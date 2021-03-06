# 操作系统探秘：用Rust语言编写操作系统
## 原文：The Adventures of OS: Making a RISC-V Operating System using Rust

2019年9月26日

### 目标

RISC-V（“risc five”）和Rust编程语言都以R开头，因此自然而然地可以将它们组合在一起。在此博客中，我们将用Rust编写一个针对RISC-V架构的操作系统（主要目标）。如果你已经配置了完整的RISC-V开发环境，则可以跳过下面的设置部分直接看启动部分。如果没有配置环境，将会很难上手。
本教程将从头开始逐步构建一个操作系统，你可以向你的朋友或父母展示最终的成果——如果他们还很年轻的话。由于我对此还不是很熟悉，所以决定将“每篇发布的博文都会随着时间的流逝而完善”作为本教程的“特性”。我将添加和明晰更多的细节。期待着您的反馈！

### 内容前瞻

第0章：环境配置

第1章：接管RISC-V硬件

第2章：通信

第3.1章：基于页面的内存分配

第3.2章：内存管理单元

第4章：处理中断和陷入

第5章：外部中断

第6章：进程存储空间

第7章：系统调用

第8章：启动一个进程

第9章：块设备

第10章：文件系统

第11章：用户态进程

未来关于这个操作系统的一些更新将会发布在 https://blog.stephenmarz.com/category/os/

我将github的仓库更改为使用tag来标记每一章。不幸的是，我很晚才这么做。第1章到第9章位于名为“chapters”的目录中。第10章和第11章由github仓库中的tag来标记。也许有一天，我将会把所有章节都打上tag的。

完成后，你将拥有一个使用RISC-V架构的操作系统，该操作系统可以基于文件系统来调度用户程序，包括shell、ls（列出文件）、cd（更改目录）等等。

这并非旨在完成一个万能的顶尖操作系统，而是要演示Rust编写的操作系统的效率。