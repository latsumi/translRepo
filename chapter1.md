### 接管RISC-V硬件

2019年9月27日

#### 总览

引导加载操作系统到RISC-V硬件上是很简单的一件事。这里有很多不同的方法来搞定，但是我将用我自己的方式来解决它。也就是说——从物理内存地址0x8000_0000开始。幸运的是，QEMU支持读取ELF文件，因此它知道将我们的代码放在该地址。 在整个过程中，我们将通过查看QEMU中包含的 qemu/hw/riscv/virt.c 文件来获取相关信息。首先，内存映射在开头列出：

```c
static const struct MemmapEntry {
	hwaddr base;
	hwaddr size;
} virt_memmap[] = {
	[VIRT_DEBUG] =       {        0x0,         0x100 },
	[VIRT_MROM] =        {     0x1000,       0x11000 },
	[VIRT_TEST] =        {   0x100000,        0x1000 },
	[VIRT_CLINT] =       {  0x2000000,       0x10000 },
	[VIRT_PLIC] =        {  0xc000000,     0x4000000 },
	[VIRT_UART0] =       { 0x10000000,         0x100 },
	[VIRT_VIRTIO] =      { 0x10001000,        0x1000 },
	[VIRT_DRAM] =        { 0x80000000,           0x0 },
	[VIRT_PCIE_MMIO] =   { 0x40000000,    0x40000000 },
	[VIRT_PCIE_PIO] =    { 0x03000000,    0x00010000 },
	[VIRT_PCIE_ECAM] =   { 0x30000000,    0x10000000 },
};
```

因此，我们的机器从DRAM（VIRT_DRAM）的字节0开始运行，该字节从物理地址0x8000_0000开始。进一步了解之后，我们将对CLINT（0x200_0000），PLIC（0xc00_0000），UART（0x1000_0000）和VIRTIO（0x1000_1000）进行编程。现在暂时不必担心这些意味着什么，但是你已经可以很方便地看到将要发生的事情！

完成之后，我们需要在RISC-V汇编中完成以下工作：

1. 选择一个CPU引导加载程序（通常是ID为＃0的）
2. BSS段清0
3. 回到Rust！

RISC-V汇编类似于MIPS汇编，除了我们不必在寄存器前加前缀之外。所有说明均来自RISC-V文档，您可以在以下地址获得它的拷贝：https//github.com/riscv/riscv-isa-manual。我们的目标是编写RV64GC指令集（RISC-V 64位，包括标准扩展和压缩指令扩展）。

#### 选择一个引导加载程序

在这个时间点上，我们不想担心并行性，竞争条件或者关于这个方面的任何其他问题。相反，我们让我们的单个CPU内核（RISC-V中称为HARTs[硬件线程]）完成所有工作。现在，我们将深入研究特权级机制，以弄清楚我们在这里讨论的寄存器的作用。因此，请在以下地址获取拷贝：https://github.com/riscv/riscv-isa-manual。

我们将从第3.1.5小节（硬件线程ID和寄存器mhartid）开始。该寄存器将告诉我们硬件线程编号。根据规范，我们必须始终拥有ID为#0的硬件线程。因此，让我们使用#0开始启动。

在 src/asm/ 目录中创建文件boot.S。在这里，我们将启动进入Rust。

```assembly
# boot.S
# bootloader for SoS
# Stephen Marz
# 8 February 2019
.option norvc
.section .data

.section .text.init
.global _start
_start:
	# Any hardware threads (hart) that are not bootstrapping
	# need to wait for an IPI
	csrr	t0, mhartid
	bnez	t0, 3f
	# SATP should be zero, but let's make sure
	csrw	satp, zero
.option push
.option norelax
	la		gp, _global_pointer
.option pop

3:
	wfi
	j	3b
```

在这里，csrr表示“控制状态寄存器读操作”，因此我们将hart标识符读入寄存器t0中，看它是否为零。如果不是，我们就将其寄放（忙碌等待）。之后，我们将监管者地址转换和保护（satp）寄存器设置为0。这就是我们最终将控制MMU的方式。由于我们暂时还不需要虚拟内存，因此可以通过使用csrw（控制状态寄存器写操作）向其写入零来禁用它。一些开发板的复位向量会将mhartid加载到该hart的a0寄存器中。但是，另一些开发板可能选择不这样做，因此我选择直接从mhartid寄存器里读取硬件线程ID。

#### 清空bss段

全局未初始化变量的值为0，因为它们是在BSS段中被分配的。但是，我们作为操作系统方，有责任确保该处内存确实为0。幸运的是，通过链接脚本，我们定义了两个字段_bss_start和_bss_end，它们分别告诉我们BSS段的开始和结束位置。因此，我们在 .option pop 标号下面的 `3:` 之前添加以下内容。

```assembly
# The BSS section is expected to be zero
	la 		a0, _bss_start
	la		a1, _bss_end
	bgeu	a0, a1, 2f
1:
	sd		zero, (a0)
	addi	a0, a0, 8
	bltu	a0, a1, 1b
2:	
```

在这里，我们使用sd指令（存储双字[64位]）将零存储到内存地址a0中，该地址循环逐渐移向_bss_end。

#### 进入Rust

由于大多数人或正常人类不喜欢在汇编语言上呆太久，所以我们会尽快进入Rust部分。虽然有些人可能会说，Rust编程是困难的部分。但我们不会与借用检查器作过多纠缠。

为了进入Rust并使CPU处于可预测的模式，我们将使用mret指令（陷入返回函数）。这使我们可以将mstatus寄存器设置为特权模式。 因此，我们将以下内容添加到boot.S中：

```assembly
# Control registers, set the stack, mstatus, mepc,
# and mtvec to return to the main function.
# li		t5, 0xffff;
# csrw	medeleg, t5
# csrw	mideleg, t5
la		sp, _stack
# We use mret here so that the mstatus register
# is properly updated.
li		t0, (0b11 << 11) | (1 << 7) | (1 << 3)
csrw	mstatus, t0
la		t1, kmain
csrw	mepc, t1
la		t2, asm_trap_vector
csrw	mtvec, t2
li		t3, (1 << 3) | (1 << 7) | (1 << 11)
csrw	mie, t3
la		ra, 4f
mret
4:
	wfi
	j	4b
```

这里有很多东西，有的被注释掉了。但是，我们正在做的是将位[12:11]设置为11，即“机器模式”。这将使我们能够访问所有指令和寄存器。当然，我们可能已经处于这种模式，但是让我们再设置一遍。然后，位[7]和位[3]将启用粗糙级别的中断。但是，我们仍然需要通过mie（机器中断使能）寄存器来启用特定的中断，这在最后进行。mepc寄存器是“机器异常程序计数器”，这是我们要返回的内存地址。在Rust中定义的符号kmain是我们从汇编中逃出的船票。mtvec（机器陷入向量）是一个内核函数，只要有陷入（例如系统调用，非法指令甚至计时器中断），就会调用该函数。

在完成Rust的main函数后，我们将ra（返回地址）设置为main的地址。然后，mret指令接收我们刚做的所有修改，并通过mepc寄存器跳转返回，这就是我们最终进入Rust的地方！

（2019年9月29日添加）我们已经引用了asm_trap_vector，但是我们没有修改它。但是，现在我们很快将在 src/asm/ 下创建一个名为trap.S的文件，并将以下内容添加到其中：

```assembly
# trap.S
# Assembly-level trap handler.
.section .text
.global asm_trap_vector
asm_trap_vector:
    # We get here when the CPU is interrupted
	# for any reason.
    mret
```

#### 裸机Rust的世界！

现在我们已经来到了Rust，需要编辑lib.rs，它是由cargo命令创建的。不要更改lib.rs的文件名，否则cargo将永远不会知道我们写了什么。接下来，lib.rs将成为我们的入口点，工具集以及我们用来导入其他Rust模块的工具。不要将kmain视为执行代码。取而代之的是，它将初始化我们需要的所有内容，然后引发“大爆炸”，也就是说，让所有内容都动起来。操作系统主要是异步的。我们将使用计时器中断来使我们的内核发挥作用，因此我们不能使用我们惯常使用的单线程编程方法。

打开lib.rs时，删除其中的所有内容。那里没有任何我们需要的或者可用于我们的内核的东西。相反，对于Rust，我们需要定义一些新东西。由于我们没有使用标准库（无论如何它不是为我们的内核构建的），在继续推进之前，我们必须先定义abort和panic_handler。所以，目前我们的进展还是零。

```rust
// Steve Operating System
// Stephen Marz
// 21 Sep 2019
#![no_std]
#![feature(panic_info_message,asm)]

// ///////////////////////////////////
// / RUST MACROS
// ///////////////////////////////////
#[macro_export]
macro_rules! print
{
	($($args:tt)+) => ({

	});
}
#[macro_export]
macro_rules! println
{
	() => ({
		print!("\r\n")
	});
	($fmt:expr) => ({
		print!(concat!($fmt, "\r\n"))
	});
	($fmt:expr, $($args:tt)+) => ({
		print!(concat!($fmt, "\r\n"), $($args)+)
	});
}

// ///////////////////////////////////
// / LANGUAGE STRUCTURES / FUNCTIONS
// ///////////////////////////////////
#[no_mangle]
extern "C" fn eh_personality() {}
#[panic_handler]
fn panic(info: &core::panic::PanicInfo) -> ! {
	print!("Aborting: ");
	if let Some(p) = info.location() {
		println!(
					"line {}, file {}: {}",
					p.line(),
					p.file(),
					info.message().unwrap()
		);
	}
	else {
		println!("no information available.");
	}
	abort();
}
#[no_mangle]
extern "C"
fn abort() -> ! {
	loop {
		unsafe {
            // The asm! syntax has changed in Rust.
            // For the old, you can use llvm_asm!, but the
            // new syntax kicks ass--when we actually get to use it.
			asm!("wfi");
		}
	}
}
```

我们使用#![no_std]告诉Rust我们将不会使用标准库。然后，我们告诉Rust允许我们的代码使用panic消息和内联汇编功能。我们要做的第一件事是创建一个空的eh_personality函数。#[no_mangle]关闭了Rust的名字修饰，因此函数名必然是eh_personality。然后，extern "C"告诉Rust使用C样式的ABI。

然后，#[panic_handler]告诉Rust我们定义的下一个函数将是我们的panic处理程序。Rust有一些触发panic的方式，我们将通过断言隐式调用它。我让这个函数能够打印出触发panic的源文件和行号。但是，我们还没有写print!或println!但是我们知道Rust的print和println的格式。附带说明一下，“!”这个符号表示此函数不会返回。如果Rust检测到它返回了一个值，编译器将会报出一个错误。

最后，我们编写中止函数。这一切都是通过wfi（等待中断）指令进行循环。这将关闭正在运行的硬件线程，直到发生另一个中断。

#### 我们在Rust中了！

我们现在正式处在Rust中了，因此我们需要在boot.S中编写指定入口点，即kmain。因此，将以下代码添加到我们的lib.rs文件中：

```rust
#[no_mangle]
extern "C"
fn kmain() {
	// Main should initialize all sub-systems and get
	// ready to start scheduling. The last thing this
	// should do is start the timer.
}
```

当kmain返回时，它会进入wfi循环并挂起。这就是我们所期望的，因为我们还没有真正告诉内核要做什么事情。所以现在的情况是，我们在Rust中，不幸的是，实际上直到我们实现print!函数之前，我们将不会在屏幕上看到任何输出。但是，所有的东西都应该被编译！通常，好的作家会以一些引用或结束语结尾，但是我却不是一个好的作家。

当你键入make run时，你的操作系统将尝试引导并进入Rust。但是，由于我们尚未编写与操作系统进行通信的驱动程序，将不会看到任何内容显示出来。键入CTRL-A，然后按“x”退出模拟器。另外，你也可以通过键入CTRL-A然后单击“c”来查看自己的位置。你在QEMU的控制台中。键入“info registers”以查看模拟器当前运行到操作系统的位置。

