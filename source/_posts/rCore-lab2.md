---
title: rCore lab2
description: Memory management.
keyword: [Allocator, Frame, PPN, FrameTracker, Buddy System, Slab, Heap allocator, Segement tree, rCore]
tags: [rCore,] 
categories: Tech
toc: true
date: 2021-01-17 10:40:09
updated: 2021-01-21
---

## Indroduction

**Memory Management** is an important component in **Operating System**, and plays key role both in Process/Thread implementation and I/O subsystem. It is a classic problem for OS designing and many book s introducing Memory Management at length. However, implementing a memory management system are not easy, especially from zero to one. But it is aslo perhaps the most helpful approach to unstanding the key principles of memory management. [rCore lab 2](https://rcore-os.github.io/rCore-Tutorial-deploy/docs/lab-2/practice.html) is focusing on the implementation of physical memory management through some classic methods like abstracting memory with **frame**.

<!--more-->

## 实验指导思考题

   ### 动态分配的内存地址在哪个范围里？

>   参考答案：
>
>   在 .bss 段中，因为我们用来存放动态分配的这段是一个静态没有初始化的数组，算是内核代码的一部分。
>
>   对于一般程序而言，其内存布局是通用的。动态分配内存操作系统会将其动态申请的内存放在程序内存布局中的 heap 中，但对于这里的 rCore 而言，其内存布局由链接文件 `os/src/linker.ld` 指定
>
>   ```bash
>   /* 目标架构 */
>   OUTPUT_ARCH(riscv)
>   
>   /* 执行入口 */
>   ENTRY(_start)
>   
>   /* 数据存放起始地址 */
>   BASE_ADDRESS = 0x80200000;
>   
>   SECTIONS
>   {   
>      /* . 表示当前地址（location counter） */
>      . = BASE_ADDRESS;
>   
>      /* start 符号表示全部的开始位置 */
>      kernel_start = .;
>   
>      text_start = .;
>   
>      /* .text 字段 */
>      .text : {
>          /* 把 entry 函数放在最前面 */
>          *(.text.entry)
>          /* 要链接的文件的 .text 字段集中放在这里 */
>          *(.text .text.*)
>      }
>   
>      rodata_start = .;
>   
>      /* .rodata 字段 */
>      .rodata : {
>          /* 要链接的文件的 .rodata 字段集中放在这里 */
>          *(.rodata .rodata.*)
>      }
>   
>      data_start = .;
>   
>      /* .data 字段 */
>      .data : {
>          /* 要链接的文件的 .data 字段集中放在这里 */
>          *(.data .data.*)
>      }
>   
>      bss_start = .;
>   
>      /* .bss 字段 */
>      .bss : {
>          /* 要链接的文件的 .bss 字段集中放在这里 */
>          *(.sbss .bss .bss.*)
>      }
>   
>      /* 结束地址 */
>      kernel_end = .;
>   }
>   ```
>
>   在该文件中，并没有对 Stack 和 Heap 的说明。因此是 rCore 的动态内存地址范围是在 .bbs 之后。具体说来，在 rCore Tutorial 中，通过加上 `#![feature(global_asm)]` 之后可以通过代码 
>
>   ```rust
>   pub fn rust_main() -> ! {
>      extern "C" {
>          fn text_start();
>          fn rodata_start();
>          fn data_start();
>          fn bss_start();
>          fn kernel_end();
>      };
>   
>      println!(".text_start {:#x}", text_start as usize);
>      println!(".rodata_start {:#x}", rodata_start as usize);
>      println!(".data_start {:#x}", data_start as usize);
>      println!(".bss_start {:#x}", bss_start as usize);
>      println!(".kernel_end {:#x}", kernel_end as usize);
>       
>      panic!()
>   }
>   ```
>
>   打印出 rCore 内存布局如下：
>
>   ```bash
>   .text_start 0x80200000
>   .rodata_start 0x8020c3b8
>   .data_start 0x8020f6d5
>   .bss_start 0x8020f6d5
>   .kernel_end 0x80a1ffb8
>   ```
>
>   因此动态分配内存在 `[0x80a1ffb8, 0x8800_0000]` 范围。

###   物理内存分配中，下面代码有什么问题？

```rust
/// Rust 的入口函数
///
/// 在 `_start` 为我们进行了一系列准备之后，这是第一个被调用的 Rust 函数
#[no_mangle]
pub extern "C" fn rust_main() -> ! {
    // 初始化各种模块
    interrupt::init();
    memory::init();

    // 物理页分配
    match memory::frame::FRAME_ALLOCATOR.lock().alloc() {
    Result::Ok(frame_tracker) => frame_tracker,
    Result::Err(err) => panic!("{}", err)
    };

    panic!()
}
```

>   参考答案：
>
>   这里的 `frame_tracker` 变量会在 `match` 语法里面析构。但是析构的时候，外层的 `lock()` 函数还没有释放锁，这样写会导致死锁。

## 实验

### 实验之前

-   阅读实验指导二
-   checkout 到仓库 `lab-2` 分支

### 实验题

#### 1.  原理

.bss 字段是什么含义？为什么我们需要将动态分配的内存（堆）空间放在 .bss 字段？
>   参考答案：
>
>   内存布局
>
>   | .text    | .rodata        | .data                                  | .bss                      | Stack        | Heap               |
>   | -------- | -------------- | -------------------------------------- | ------------------------- | ------------ | ------------------ |
>   | asm code | read only data | initialized data, usaually global data | initialized data with `0` | local data   | dynamic allocating |
>   | \|<----- | Low address    |                                        |                           | high address | -------->\|        |
>
>   对于一个 ELF 程序文件而言，.bss 字段一般包含全局变量的名称和长度，在执行时有操作系统分配空间并初始化为零。
>   不过，在我们执行 rust-objcopy 时，不同的字段会相应地被处理而形成一段连续的二进制数据，这段二进制数据会直接写入到 QEMU 所模拟的机器 `0x8020_0000` 位置。这是因为我们写的操作系统是直接运行在机器上的，而不是一个被操作系统加载的程序。
>
>   我们一般遇到应用程序的动态内存分配（堆）是由操作系统提供的。例如在 C 语言中的 `malloc()`，glibc 运行库会维护一个堆空间，而这个空间是通过 `brk()` 等系统调用向内核索要的。由于我们编写操作系统，自然就无法像这样获取空间。但是此时我们具有随意使用内存空间的权力，因为我们可以在内存中随意划一段空间，然后用相应的算法来实现一个堆。
>
>   至于为何堆在 .bss 字段，实际上这也不是必须的——我们完全可以随意指定一段可以访问的内存空间。不过，在代码中用全局来表示堆并将其放在 .bss 字段，是一个很简单的实现：这样堆空间就包含在内核的二进制数据之中了，而自 `KERNEL_END_ADDRESS` 以后的空间就可以给进程使用。

#### 2.  分析

我们在动态内存分配中实现了一个堆，它允许我们在内核代码中使用动态分配的内存，例如 `Vec` 和  `Box` 等。那么，如果我们在实现这个堆的过程中使用中使用 `Vec` 而不是 `[u8]` ，会出现什么结果

-   无法编译？
-   运行时错误？
-   正常运行？

>   参考答案：
>
>   **都不会**！程序会陷入一个循环：它需要在堆上分配空间，但是分配器又需要在堆上分配空间......

#### 3.  实验

1.  回答: `algorithm/src/allocator` 下有一个 `Allocator` trait，我们之前用它实现了物理页面分配。这个算法的时间和空间复杂度是多少？

> 参考答案： O(1) 和 O(N)。
>
> rCore 在代码 `src/algorithm/src/allocator/mod.rs` 中定义了 `Allocator` trait，之后在相同 mod 下的 `stacked_allocator.rs` 和 `bitmap_vector_allocator.rs` 分别对其进行了实现。其中 `stacked_allocator.rs` 中的 `StackAllocator` 作为默认分配器在 `os/src/memory/frame/allocator.rs` 中使用。
>
> 在 `stacked_allocator.rs` 中 `struct StackAllocator` 定义了 `list: Vec<(usize, usize)>` 结构，通过 `pop` 和 `push` 操作分别进行 `alloc` 和 `dealloc` ，因此其时空复杂度为 O(1) 和 O(N)。

2.  二选一：实现基于线段树的物理页面分配算法（不需要考虑合并分配）；或尝试修改 `FrameAllocator`，令其使用未被分配的页面空间（而不是全局变量）来存放页面使用状态。

#### 4.  挑战实验（选做）

1.  在 `memory/heap2.rs` 中，提供了一个手动实现堆的方法。它使用 `algorithm::VectorAllocator` 作为其根本分配算法，而我们目前提供了一个非常简单的 bitmap 算法（而且只开了很小的空间）。请在 `algorithm` crate 中利用伙伴算法实现 `VectorAllocator` trait。
2.  前面说到，堆的实现本身不能完全使用动态内存分配。但有没有可能让堆能利用动态分配的空间，这样做会带来什么好处？