### 环境配置

2019年9月27日

Rust现在无需额外配置即可支持RISC-V！这使得我们无需工具链即可轻松构建。 但我们仍然需要地址为ch0.old.html的旧教程所提到的QEMU模拟器。

Rust将需要安装一些东西，您可以使用 `rustup` 命令添加这些。如果没有rustup，可以从 https://www.rust-lang.org/tools/install 下载。

```
rustup default nightly
rustup target add riscv64gc-unknown-none-elf
cargo install cargo-binutils
```

由于我会用到Rust的一些特性（用 `#![features]` 表示），因此即使RISC-V是稳定版本，我们也必须使用nightly测试版Rust。

#### 构建

访问位于Github的博客仓库，你将看到一个名为 .cargo/config 的文件。你可以编辑它以适配你的系统。

```rust
[build]
target = "riscv64gc-unknown-none-elf"
rustflags = ['-Clink-arg=-Tsrc/lds/virt.lds']

[target.riscv64gc-unknown-none-elf]
runner = "qemu-system-riscv64 -machine virt -cpu rv64 -smp 4 -m 128M -drive if=none,format=raw,file=hdd.dsk,id=foo -device virtio-blk-device,scsi=off,drive=foo -nographic -serial mon:stdio -bios none -device virtio-rng-device -device virtio-gpu-device -device virtio-net-device -device virtio-tablet-device -device virtio-keyboard-device -kernel "	
```

配置文件显示了我们要构建的目标，即riscv64gc。 我们还需要指定链接脚本，以便将内容放置在src/lds/virt.lds中。最后，当我们输入cargo run时，将会启动一个进程，它将运行riscv64 qemu。另外，请注意 -kernel 后有一个空格。这是因为cargo将自动指定可执行文件，该可执行文件的名称是通过Cargo.toml配置的。

#### 完成！

环境配置部分让人厌烦，没关系，以后情况会变得更好的（ArchLinux部分确实如此！），现在马上要到有趣的部分了！


