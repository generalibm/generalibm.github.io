---
title: rCore lab2
description: Memory Management.
keywords: [Allocator, Frame, PPN, FrameTracker, Buddy System, Slab, Heap allocator, rCore]
tags: [rCore,] 
categories: Tech
toc: true
date: 2021-01-17 10:40:09
updated: 2021-01-22
---
## Introduction

**Memory Management** is an important component in **Operating System**, and plays key role both in Process/Thread implementation and I/O subsystem. It is a classic problem for OS designing and many book s illustrate Memory Management at length. However, implementing a memory management system is not easy, especially from zero to one. But it is also perhaps the most helpful approach to understanding the key principles of memory management. [rCore lab 2](https://rcore-os.github.io/rCore-Tutorial-deploy/docs/lab-2/practice.html) is focusing on the implementation of physical memory management through some classic methods like abstracting memory with **frame**. 

<!--more-->

## 实验指导

### 动态物理内存分配

rCore 通过使用 `#[global_allocator]` 标记实现自己的全局内存分配器，从而自己管理内存分配。在 `os/src/memory/heap.rs` 中使用已经实现的伙伴系统分配器 `buddy_system_allocator::LockHeap` crate 来完成自己的堆分配。

```rust
use buddy_system_allocator::LockedHeap;

/// 进行动态内存分配所用的堆空间
///
/// 大小为 [`KERNEL_HEAP_SIZE`]，即 0x80_0000
/// 这段空间编译后会被放在操作系统执行程序的 bss 段，因为已经初始化 0
static mut HEAP_SPACE: [u8; KERNEL_HEAP_SIZE] = [0; KERNEL_HEAP_SIZE];

#[global_allocator]
static HEAP: LockedHeap = LockedHeap::empty();

/// 初始化操作系统运行时堆空间
pub fn init() {
    // 告诉分配器使用这一段预留的空间作为堆
    // 因为 Heap 是 static mut 型，所以要用 unsafe
    unsafe {
        HEAP.lock()
            .init(HEAP_SPACE.as_ptr() as usize, KERNEL_HEAP_SIZE);
    }
    
}

/// 空间分配错误的回调，直接 panic 退出
#[alloc_error_handler]
fn alloc_error_handler(_: alloc::alloc::Layout) -> ! {
    panic!("alloc error")
}
```

在 rCore 的堆分配的代码中，按照**字节**分配，总共在 .bss 上分配了 8MB 。因而可以使用 alloc care 中的 vec 来进行动态内存分配。比如：

```rust
pub extern "C" fn rust_main() {
    interrupt::init();
    memory::init();
    
    let mut vec = Vec::new();
    for i in 0..(0x80_0000/16) {
        vec.push(i);
    }
    assert_eq!(vec.len(), (0x80_0000/16));
    for (i, value) in vec.into_iter().enumerate() {
        assert_eq!(value, i);
    }
    println!("end of main");
    panic!();
}
```

如果想分配 `0x80_0000` 个 `i32` 或者 `u8`，都会[分配失败](https://github.com/rcore-os/rCore-Tutorial/issues/129)。

### 物理内存探测

物理地址不仅仅能访问 DRAM，也可以访问外设。RISCV 通过 MMIO (Memory Mapped I/O) 技术将外设映射到一段物理地址，于是外设和 DRAM 的地址访问得到了统一。

在 qemu 模拟的 RISCV Virt 计算机中物理内存（DRAM）的地址范围为 [0x80000000, 0x88000000)，即 128M ，是 qemu 的默认值。可以通过 `-m` 来指定其 DRAM 的大小。

在 rCore 中对物理地址进行了抽象封装，即 `os/src/memory/address.rs` 中的 `pub struct PhysicalAddress(pub usize)` 。实现了类似 `usize` 的诸如加、减等运算操作。为了完成与 `usize` 类型基于 layout 的转换（reinterpreting）需要加上 `[#repr(C)]` 。

```rust
#[repr(C)]
#[derive(Copy, Clone, Debug, Default, Eq, PartialEq, Ord, PartialOrd, Hash)]
pub struct PhysicalAddress(pub usize);

impl PhysicalAddress {
    /// 取得页内偏移
    pub fn page_offset(&self) -> usize {
        self.0 % PAGE_SIZE
    }
}

macro_rules! implement_address_to_page_number {
    // 这里面的类型转换实现 [`From`] trait，会自动实现相反的 [`Into`] trait
    ($address_type: ty, $page_number_type: ty) => {
        impl From<$page_number_type> for $address_type {
            /// 从页号转换为地址
            fn from(page_number: $page_number_type) -> Self {
                Self(page_number.0 * PAGE_SIZE)
            }
        }
        impl From<$address_type> for $page_number_type {
            /// 从地址转换为页号，直接进行移位操作
            ///
            /// 不允许转换没有对齐的地址，这种情况应当使用 `floor()` 和 `ceil()`
            fn from(address: $address_type) -> Self {
                assert!(address.0 % PAGE_SIZE == 0);
                Self(address.0 / PAGE_SIZE)
            }
        }
        impl $page_number_type {
            /// 将地址转换为页号，向下取整
            pub const fn floor(address: $address_type) -> Self {
                Self(address.0 / PAGE_SIZE)
            }
            /// 将地址转换为页号，向上取整
            pub const fn ceil(address: $address_type) -> Self {
                Self(address.0 / PAGE_SIZE + (address.0 % PAGE_SIZE != 0) as usize)
            }
        }
    };
}
implement_address_to_page_number! {PhysicalAddress, PhysicalPageNumber}

/// 为各种仅包含一个 usize 的类型实现运算操作
macro_rules! implement_usize_operations {
    ($type_name: ty) => {
        /// `+`
        impl core::ops::Add<usize> for $type_name {
            type Output = Self;
            fn add(self, other: usize) -> Self::Output {
                Self(self.0 + other)
            }
        }
        /// `+=`
        impl core::ops::AddAssign<usize> for $type_name {
            fn add_assign(&mut self, rhs: usize) {
                self.0 += rhs;
            }
        }
        /// `-`
        impl core::ops::Sub<usize> for $type_name {
            type Output = Self;
            fn sub(self, other: usize) -> Self::Output {
                Self(self.0 - other)
            }
        }
        /// `-`
        impl core::ops::Sub<$type_name> for $type_name {
            type Output = usize;
            fn sub(self, other: $type_name) -> Self::Output {
                self.0 - other.0
            }
        }
        /// `-=`
        impl core::ops::SubAssign<usize> for $type_name {
            fn sub_assign(&mut self, rhs: usize) {
                self.0 -= rhs;
            }
        }
        /// 和 usize 相互转换
        impl From<usize> for $type_name {
            fn from(value: usize) -> Self {
                Self(value)
            }
        }
        /// 和 usize 相互转换
        impl From<$type_name> for usize {
            fn from(value: $type_name) -> Self {
                value.0
            }
        }
        impl $type_name {
            /// 是否有效（0 为无效）
            pub fn valid(&self) -> bool {
                self.0 != 0
            }
        }
        /// {} 输出
        impl core::fmt::Display for $type_name {
            fn fmt(&self, f: &mut core::fmt::Formatter<'_>) -> core::fmt::Result {
                write!(f, "{}(0x{:x})", stringify!($type_name), self.0)
            }
        }
    };
}
implement_usize_operations! {PhysicalAddress}
implement_usize_operations! {PhysicalPageNumber}

```

对于 `PhysicalAddress` 只实现了页偏移方法 `page_offset(&self)->usize` ，其他方法由于与 `PhysicalPageNumber` 有很多重复，rCore 在这里将其通过 `macro_rules!` 将两者进行了合并，而后通过 `implement_address_to_page_number!{PhysicalAddress, PhysicalPageNumber}` 和 `implement_usize_operations! {PhysicalAddress}` 在编译期进行了宏展开，因此效果上简化了一些代码。

### 物理内存管理

通常物理内存的分配是以页帧（Frame）为单位进行分配的，rCore 以 4KB 为一页帧的大小。在代码 `os/src/memory/frame/frame_tracker.rs:22` 中定义了物理页结构 `pub struct FrameTracker(pub(super) PhysicalPageNumber)` ，即物理页号。在代码 `os/src/memory/frame/allocator.rs:20` 中将其与一个 `Allocator` 一起作为结构 `struct FrameAllocator<T: Allocator>` 的成员，从而实现了物理页分配与具体算法无关的策略。在 `os/src/memory/frame/allocator.rs:13` 中将默认 `Allocator` 设为 `AllocatorImpl = StackedAllocator`，并用spin::Mutex 将其保护起来。



### Memory Privilege

内存分配完毕后，还需要将 RISCV 中 sstatus 寄存器的 SUM (permit Supervisor User Memory access) bit 位设为 `1`，从而可以从 S 态访问 U 态的（虚拟）内存 —— 参考 risv-privileded 4.1.1.2 Memory Privilege in sstatus Register。rCore 中实现在 `os/src/memory/mod.rs`中。

```rust
/// 初始化内存相关的子模块
///
/// - [`heap::init`]
pub fn init() {
    heap::init();
    // 允许内核读写用户态内存
    unsafe { riscv::register::sstatus::set_sum() };

    println!("mod memory initialized");
}
```

## 实验指导思考题

   ### 动态分配的内存地址在哪个范围里？

>   参考答案：
>
>   在 .bss 段中，因为我们用来存放动态分配的这段是一个静态初始化为 `0` 的数组，算是内核代码的一部分。
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
>   因此动态分配内存在 `[0x80a1ffb8, 0x8800_0000) 范围。

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
>   | <----- | Low address    | ----- | ----- | high address | ----->   |
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

>   参考答案：
>
>   我们以一个朴素的分配器算法为例：将每一次内存分配记录用链表存起来。
>
>   分配器最初必须具有一个节点的静态空间。而每当它仅剩一个节点空间时，都可以用它来为自己分配一块更大的空间。如此，就实现了分配器动态分配自己。
>
>   再考虑到，每次分配 1KB 或 1MB 都需要额外保存一份元信息。如果只用静态分配，就必须按最坏情况（每次都只分配最小单元）来预先留好空间。使用动态分配就可以减少空间浪费。