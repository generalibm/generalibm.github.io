---
title: rCore lab1 spec
subtitle: Interrupt, Exception
description: Write an OS from zero to one based on RISC-V ISA in Rust. 
tags: []
categories: Tech
date: 2021-01-14 14:26:06
toc: true
---

[rCore lab1](https://rcore-os.github.io/rCore-Tutorial-deploy/docs/lab-1/practice.html)

Exceptions

>   Also called interrupt. An unscheduled event that disrupts program execution; used to detect overflow.

Interrupts

>   An exception that comes from outside of the processor. (Some architectures use the term`interrupt` for all exceptions.)

<!--more-->

## 实验题

### 1.  原理：
在 `rust_main` 函数中，执行 `ebreak` 命令后至函数结束前，`sp` 寄存器的值是怎样变化的？

- 计算机 (qemu) 加电之后，会进行自检 （POST，Power-On self-Test），然后跳转到 bootloader (openSBI) 入口，在 bootloader 中，会进行内核环境（内存，外设等）探测。随后，bootloader 将内核 (rCore-Tutorial/os/target/riscv64imac-unknown-none-elf/debug/os) 加载到内存中 0x80200000 地址，同时从 M 态切换到 S 态，此后跳转到 os 中运行，即 `os/src/main.rs` 。 

- 在分支 lab-1 的代码中 `os/src/main.rs:31` 将 os 跳转到 `os/src/entry.asm` 中。该汇编程序完成的功能如注释所述：编译链接 os 时，将入口地址设为 `_start` ，此文件的功能即实现 `_start`。

    ```asm
        .section .text.entry
        .globl _start
        # 目前 _start 的功能：将预留的栈空间写入 $sp，然后跳转至 rust_main
    _start:
        la sp, boot_stack_top
        jal rust_main

        # 回忆：bss 段是 ELF 文件中只记录长度，而全部初始化为 0 的一段内存空间
        # 这里声明字段 .bss.stack 作为操作系统启动时的栈
        .section .bss.stack
        .global boot_stack
    boot_stack:
        # 16K 启动栈大小
        .space 4096 * 16
        .global boot_stack_top
    boot_stack_top:
        # 栈结尾
    ```
    在 `_start` 中开辟了 16K 内存作为启动栈，然后将 0x80200000 + 16K 的位置作为 `boot_stack_top` 赋值给 sp 寄存器。此后跳转到 `rust_main` 中。
    ```rust
    pub extern "C" fn rust_main() {
        interrupt::init();

        println!("Hello, rCore-Tutorial!");

        unsafe { llvm_asm!("ebreak") };

        panic!("end of rust_main");
    }
    ```
    在`rust_main`中，首先进行中断操作的初始化工作 `interrupt::init()` ，然后打印一行提示 `println!("Hello, rCore-Tutorial!")` 。此后触发一个断点异常（breakpoint exception）`unsafe { llvm_asm!("ebreak") }` 。

- 代码 `unsafe { llvm_asm!("ebreak") }` 会转入到 `os/src/interrupt/interrupt.asm` 。该文件完成的工作是 ：

    1）保存 32 个通用寄存器的值到内存中；

    2）还保存 2 个 CSR （sstatus 和 sepc）的值到内存中；

    3）保存 sp 值，scause 值，stval 值到 `a0`, `a1`, `a2` 寄存寄中作为 `handle_interrupt` 函数参数；

    4）跳转到 `os/src/interrupt/handler.rs::handle_interrupt` 进行中断处理；

    5）中断处理完成后，依次恢复 2）和 1）中保存在内存的值到相应寄存器中去。

     这样就完成了从中断现场（Context，大小为`34 * 8`）保护到中断处理，再到中断现场恢复操作。而这过程是由于 `cargo build` 过程生成了 `*.rs` 文件和 `*.asm` 文件中函数的符号表（symbol table），因此整个中断处理过程的函数调用过程可以理解为：

    ```asm
    inerrupt.asm::__interrupt -> handler.rs::handle_interrupt -> inerrupt.asm::__restore
    ```

     在上述过程中 `sp` 值的变化为： `inerrupt.asm::__interrupt` 过程 `sp` 值会 `- 34 * 8` ；`handler.rs::handle_interrupt` 过程可能也会有或多或少的函数调用参数入栈和出栈操作，因此 `sp` 值前后不变；`inerrupt.asm::__restore` 过程 `sp` 值会 `+ 34 * 8` 。

### 2.  分析：
如果去掉 `rust_main` 后的 `panic` 会发生什么，为什么？

- rCore `os/src/main.rs::rust_main` 中的 `panic` 是在 `os/src/panic.rs` 中通过声明 panic 回调函数`panic_handler`实现的。在 `os/src/panic.rs` 中引用了 `core::panic::PanicInfo` 结构，因此可以在 `panic_handler` 中获知调用 `panic` 处的信息，如函数名，行号等。在 `panic_handler` 的最后调用了 `shudown` 函数，该函数在 `os/src/sbi.rs::shutdown->sbi_call->ecall` 中实现，是调用 M 模式的 `ecall`。但是，在我的机器中，`sib_shutdown` 似乎有点[问题](https://github.com/rcore-os/rCore-Tutorial/issues/127)。

- 如果正常 `shutdown`，那么 `qemu` 会关机。

- 反过来，如果去掉 `panic` 即不能正常 `shutdown`；那么  `rust_main` 执行完后，就会回到 `entry.asm:14 `中 `jal rust_main` 后面执行，而之后的代码已经不可控。

- 试着去掉 `panic` 后之后再 `make run` 发现，`os` 会继续运行，而且打印很多不可识别字符，在经过一段时间崩溃。

- 加入实验 3.1 的捕获操作后，在次 `make run` 会发现 `os/src/interrupt/handler.rs::handle_interrupt`可以捕获到 **访问不存在的地址** 错误。

### 3.  实验

#### 1. 如果程序访问不存在的地址，会得到 `Exception::LoadFault`。模仿捕获 `ebreak` 和时钟中断的方法，捕获 `LoadFault`（之后 `panic` 即可）。

重写 `os/src/interrupt/handler::handler_interrupt` 函数如下：

```rust
pub fn handle_interrupt(context: &mut Context, scause: Scause, stval: usize) {
        // 返回的 Context 必须位于放在内核栈顶
    match scause.cause() {
            // 断点中断（ebreak）
        Trap::Exception(Exception::Breakpoint) => breakpoint(context),
            // 访问不存在地址
        Trap::Exception(Exception::LoadFault) => panic!("访问不存在地址"),
            // 时钟中断
        Trap::Interrupt(Interrupt::SupervisorTimer) => supervisor_timer(context),
            // 其他情况，终止当前线程
        _ => fault(context, scause, stval),
        };
}
```

#### 2. 在处理异常的过程中，如果程序想要非法访问的地址是 `0x0`，则打印 `SUCCESS!`。

**参考答案：**

- 如果程序因无效访问内存造成异常，这个访问的地址会被存放在 `stval` 中，而它已经被我们作为参数传入 `handle_interrupt` 了，因此直接判断即可


#### 3. 添加或修改少量代码，使得运行时触发这个异常，并且打印出 `SUCCESS!`。

*要求：不允许添加或修改任何 unsafe 代码*

**参考答案：**

- 解法 1：在 `interrupt/handler.rs` 的 `breakpoint` 函数中，将 `context.sepc += 2` 修改为 `context.sepc = 0`（则 `sret` 时程序会跳转到 `0x0`）

- 解法 2：去除 `rust_main` 中的 `panic` 语句，并在 `entry.asm` 的 `jal rust_main` 之后，添加一行读取 `0x0` 地址的指令（例如 `jr x0` 或 `ld x1, (x0)`）
