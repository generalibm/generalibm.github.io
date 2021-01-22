---
title: An openSBI shutdown bug in rCore lab
description: openSBI v0.7 has not implemented RISC-V SBI v0.2 yet?
keyword: [openSBI, RISCV, rCore]
tags: [rCore]
categories: Tech
toc: true
date: 2021-01-15 23:40:32
---

### Introduction

Over days debug I have not figured out why qemu in my machine always says something deffrent from the [project manul](https://rcore-os.github.io/rCore-Tutorial-deploy/docs/pre-lab/env.html). But now I can even send an issue to rCore group with my pull request, even though the key of the bug does lie in qemu or openSBI rather than rCore. 

<!--more-->


### 问题描述
下载最[新代码](https://github.com/rcore-os/rCore-Tutorial)后，切换到 lab-1 分支，`make run` 出现如下错误信息：
```bash
...
src/main.rs:44: 'end of rust_main'
sbi_trap_error: hart0: trap handler failed (error -2)
sbi_trap_error: hart0: mcause=0x0000000000000007 mtval=0x0000000000100000
sbi_trap_error: hart0: mepc=0x0000000080003d3e mstatus=0x8000000000007800
sbi_trap_error: hart0: ra=0x0000000080008228 sp=0x000000008001ec98
sbi_trap_error: hart0: gp=0x0000000000000000 tp=0x000000008001f000
sbi_trap_error: hart0: s0=0x000000008001eca8 s1=0x0000000000000040
sbi_trap_error: hart0: a0=0x0000000000000000 a1=0x0000000080003d2a
sbi_trap_error: hart0: a2=0x0000000080003d2a a3=0x0000000080003d2a
sbi_trap_error: hart0: a4=0x0000000000100000 a5=0x0000000000005555
sbi_trap_error: hart0: a6=0x0000000000003d2a a7=0x000000008000e0e0
sbi_trap_error: hart0: s2=0x0000000000000000 s3=0x000000008001f000
sbi_trap_error: hart0: s4=0x0000000000000000 s5=0x0000000000000000
sbi_trap_error: hart0: s6=0x0000000000000001 s7=0x0000000000000000
sbi_trap_error: hart0: s8=0x0000000000000000 s9=0x0000000000000000
sbi_trap_error: hart0: s10=0x0000000000000000 s11=0x0000000000000008
sbi_trap_error: hart0: t0=0x0000000000000000 t1=0x0000000000000000
sbi_trap_error: hart0: t2=0x000000008021622c t3=0x0000000000000000
sbi_trap_error: hart0: t4=0x0000000000000000 t5=0x0000000000000000
sbi_trap_error: hart0: t6=0x0000000000000000
```
之后，gbd 和 qemu 陷入卡死状态，目前我的处理方式是 `kill -9`。

### 我的环境
| Tool                | Version        |
| ------------------- | -------------- |
| mac OS              | 11.1           |
| rustc               | 1.46.0-nightly |
| qemu-system-riscv64 | 5.1.0          |
| openSBI             | 0.7            |
| SBI                 | 0.2            |

### 我的追踪过程
1. 根据 `os/src/main.rs:44: 'end of rust_main'` 提示，追踪到 `os/src/panic.rs::panic_handler:26` 中 `shutdown` ；
---

2. 在 `os/src/sbi.rs:44` 中 `shutdown` 调用 `sbi_call(8,0,0,0)`。根据 `os/src/sbi.rs:7` 的定义，发现是使用了 `ecall` 请求 M 态（即SBI）服务，在这里是 qemu 下的 openSBI v0.7，该版本实现了 RISCV SBI 的 v0.2 版本。
---

3. 自然而言会想到查看 SBI 手册，在[这里](https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.adoc#extension-system-shutdown-eid-0x08)找到了关于 `shutdown` 的说明。起初，看到 Replacement EID 这一列时，认为可能是我的机器环境比实验手册中环境要新一些，故阅读完[该文档](https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.adoc#system-reset-extension-eid-0x53525354-srst)之后，实现了一个新版本的 `sbi_shutdown` 取代此前`os/src/panic.rs::panic_handler:26`处的 `shutdown`，代码如下：
```rust
#[inline(always)]
fn sbi_call(eid: usize,fid: usize,  arg0: usize, arg1: usize, arg2: usize) -> Sbiret {
    let mut error;
    let mut value;
    unsafe {
        llvm_asm!("ecall"
            : "={x10}" (error), "={x11}" (value)
            : "{x10}" (arg0), "{x11}" (arg1), "{x12}" (arg2), "{x17}" (eid), "{x16}" (fid)
            : "memory"      // 如果汇编可能改变内存，则需要加入 memory 选项
            : "volatile");     // 防止编译器做激进的优化（如调换指令顺序等破坏 SBI 调用行为的优化）
    }
    Sbiret{error, value}
}

pub struct Sbiret {
    pub error: i64,
    pub value: i64,
}

const SBI_SYSTEM_RESET: usize = 0x53525354;

pub fn sbi_shutdown() -> Sbiret {
    sbi_call(SBI_SYSTEM_RESET, 0, 0, 0,0)
}
```

  而后，在 `os/src/panic.rs::panic_handler:27` 处打印 `struct Sbiret` 的值，结果为 `sbi_ret.error = -2` 和 ` sbi_ret.value = 0`；根据[文档](https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.adoc#binary-encoding)，查到 `-2` 代表`SBI_ERR_NOT_SUPPORTED`，即 SBI v0.2 不支持这个用法。于是放弃了继续改写的打算，转而查找 `shutdown` 后 `sbi_trap_error: hart0: trap handler failed (error -2)` 原因。

---

4. 经过各种 搜索引擎 的对关键字 `sbi_trap_error` `trap handler failed` `error` 检索之后，发现相关内容数量少且关系不大。只好回到 rCore 本身。
---

5. 在 rCore 的 lab-1 代码中，对 `shutdown` 和 `sbi_call` 进行了断点调试，主要涉及 `os/src/panic.rs` 和 `os/src/sbi.rs` 两个文件，其中将 `os/src/interrupt/mod.rs:15` 即 `timer::init()` 注释掉了，因为这个时钟也是通过 `ecall` 来实现的，因此会多次影响调试时的程序运行栈。结果证明是个 `loop`，只好另谋方案。
---

6. 虽然在 rCore 代码上经过了一番努力和尝试之后还是无果，但是可以得出结论问题不在 rCore ，而是在 SBI ，于是，我把目光转向到 openSBI 的[实现]()上，希望能够找到一些思路。
---

7. 在 `opensbi/lib/sbi/sbi_trap.c::216` 中，终于看到了熟悉的 `trap handler failed` 的身影，根据函数名 `sbi_trap_handler` 以及该函数注释，不难发现该函数和熟悉的rCore上的 `sbi_call` 完美契合。在 `opensbi/lib/sbi/sbi_trap.c::267` 中调用了在同一个文件中的函数 `sbi_trap_redirect` ，此后将函数的返回值作为判断条件是否进行 `trap_error` 处理；在 `opensbi/lib/sbi/sbi_trap.c::sbi_trap_error` 中可以发现，这里的处理即是打印各种信息，而这和 rCore `make run` 之后的错误信息完全一致；此后进入 `sbi_hart_hang`， 该函数内部是一个 `while (1)` 循环，这也就解释了 gdb 和 qemu 卡死的原因。因此，知道 `sbi_trap_redirect` 的返回值代表的含义就能发现问题的原因了。
---

8. 在 `sbi_trap_redirect` 中，返回值只有两个 `0` (`opensbi/lib/sbi/sbi_trap.c::194`) 和 `SBI_ENOTSUPP` (`opensbi/lib/sbi/sbi_trap.c::98`)。而根据 `make run` 的错误信息，以及 `opensbi/lib/sbi/sbi_trap.c:29` 不难知道这个不是 `0` 而是 `-2`，因此可以得出假设 `SBI_ENOTSUPP = 2`。
---

9. 果然在 `opensbi/include/sbi/sbi.error.h:19` 发现了 `#define SBI_ENOTSUPP	 SBI_ERR_NOT_SUPPORTED`，又在 `opensbi/include/sbi/sbi_ecall/interface.h:88` 中发现 `#define SBI_ERR_NOT_SUPPORTED -2`。于是假设 `SBI_ENOTSUPP = 2` 得到了证实。原本找到错误的原因应该举杯欢庆，但是对 SBI_ERR_NOT_SUPPORTED 的字面意思又难以高兴起来，其意为 rCore 中的 `os/src/sbi.rs::shutdown` 即 `sbi_call(8,0,0,0)` 也是不支持。和最初想要自己动手写一个 `sbi_shutdown` 一样，openSBI v0.7 目前都不支持。被这个问题困扰了一段时间之后，接下来的思路就是再搭建一个环境。
---

10. 在搭建 virtual box 和 ubuntu18.04 的等待过程中，突然有了个疑问：之前看到的 SBI 文档是 v0.2 吗？应该是 master 分支上吧。于是，转向查找 [SBI v0.2 文档](https://github.com/riscv/riscv-sbi-doc/blob/v0.2.0/riscv-sbi.adoc)。
---

11. 在 SBI v0.2 文档中 [`sbi_shutdown`](https://github.com/riscv/riscv-sbi-doc/blob/v0.2.0/riscv-sbi.adoc#function-listing-1) 中，对于 `Replacement Extension` 并没有指定，因此最初重写 `sbi_shutdown` 后，返回值表明没有实现这个是合理的。但是，在 `Extension ID` 列中，关于 `sbi_shutdown` 指定的值确是 `0x08`，而经过上面的探究已经 openSBI 也给出了不支持的回答。至此，问题似乎明晰又不明晰。在看完 SBI v0.2 文档之后，发现 [HSM](https://github.com/riscv/riscv-sbi-doc/blob/v0.2.0/riscv-sbi.adoc#hart-state-management-extension-extension-id-0x48534d-hsm)实现 `sbi_hart_stop`，根据其解释 

> Returns ownership of the calling hart back to the SBI implementation

结合 `sbi_shutdown` 的解释

> Puts all the harts to shut down state from supervisor point of view.

似乎可以通过 `sbi_hart_stop` 实现 `sbi_shutdown`。

---
12. 于是，再次重写 `sbi_shutdown` 如下
```rust
const HSMMID:usize = 0x48534D;
const HSM_STOP_FID:usize = 1;

pub fn sbi_hart_stop() -> Sbiret {
    sbi_call(HSMMID, HSM_STOP_FID, 0, 0, 0)
}
```
然后，用 `sbi_hart_stop` 替代 `os/src/panic.rs:26` 处的 `shutdown`。

---
13. `make run` 之后光标停在了 `src/main.rs:44: 'end of rust_main'` 之后。至此，`shutdown` 问题得到了解决，但是不确定的是这个方案优劣几何。
---
14. 既然，`sbi_shutdown` 中 `ecall` 对于 `EID = 8` 不支持，那么 `EID = 0` 的时钟是否也不支持呢？在将 `os/src/mod.rs:15` 的 `timer::init();` 解除注释后，再把 `os/src/main.rs:44`中的 `panic("end of rust_main")` 改为 `loop {} `后，`make run` 发现打印出各种 ticks ，说明 open SBI v0.7 对于 `EID = 0` 情况的 `ecall` 是支持的。

### 涉及文件
rCore-Tutorial 的 lab-1 下
- os/src/sbi.rs
- os/src/panic.rs
- os/src/main.rs

### 相关段落
- 修改后的文件 `os/src/sbi.rs`
```rust
//! 调用 Machine 层的操作
// 目前还不会用到全部的 SBI 调用，暂时允许未使用的变量或函数
#![allow(unused)]

/// SBI 调用
#[inline(always)]
fn sbi_call(eid: i32,fid: i32,  arg0: usize, arg1: usize, arg2: usize) -> Sbiret {
    let mut error;
    let mut value;
    unsafe {
        llvm_asm!("ecall"
            : "={x10}" (error), "={x11}" (value)
            : "{x10}" (arg0), "{x11}" (arg1), "{x12}" (arg2), "{x17}" (eid), "{x16}" (fid)
            : "memory"      // 如果汇编可能改变内存，则需要加入 memory 选项
            : "volatile"); // 防止编译器做激进的优化（如调换指令顺序等破坏 SBI 调用行为的优化）
    }
    Sbiret{error, value}
}

pub struct Sbiret {
    pub error: i64,
    pub value: i64,
}

const SBI_HSM_STOP_EID:i32 = 0x48534D;
const SBI_HSM_STOP_FID:i32 = 1;

const SBI_SET_TIMER: i32 = 0;
const SBI_CONSOLE_PUTCHAR: i32 = 1;
const SBI_CONSOLE_GETCHAR: i32 = 2;
const SBI_CLEAR_IPI: i32 = 3;
const SBI_SEND_IPI: i32 = 4;
const SBI_REMOTE_FENCE_I: i32 = 5;
const SBI_REMOTE_SFENCE_VMA: i32 = 6;
const SBI_REMOTE_SFENCE_VMA_ASID: i32 = 7;
const SBI_SHUTDOWN: i32 = 8;

/// 向控制台输出一个字符
///
/// 需要注意我们不能直接使用 Rust 中的 char 类型
pub fn console_putchar(c: usize) {
    sbi_call(SBI_CONSOLE_PUTCHAR, 0, c, 0, 0);
}

/// 从控制台中读取一个字符
///
/// 没有读取到字符则返回 -1
pub fn console_getchar() -> Sbiret {
    sbi_call(SBI_CONSOLE_GETCHAR, 0, 0, 0, 0)
}

/// 调用 SBI_SHUTDOWN 来关闭操作系统（直接退出 QEMU）
pub fn shutdown() -> ! {
    sbi_call(SBI_SHUTDOWN, 0, 0, 0,0);
    unreachable!()
}

/// 设置下一次时钟中断的时间
pub fn set_timer(time: usize) {
    sbi_call(SBI_SET_TIMER, 0, time, 0,0);
}

/// 关闭 hart ，等价于 SBI_SHUTDOWN ？
/// TODO: need to verify
pub fn sbi_hart_stop() -> Sbiret {
    sbi_call(SBI_HSM_STOP_EID, SBI_HSM_STOP_FID, 0, 0, 0)
}
```

- 修改后的函数 `os/src/sbi.rs::panic_handler`
```rust
#[panic_handler]
fn panic_handler(info: &PanicInfo) -> ! {
    // `\x1b[??m` 是控制终端字符输出格式的指令，在支持的平台上可以改变文字颜色等等，这里使用红色
    // 参考：https://misc.flogisoft.com/bash/tip_colors_and_formatting
    //
    // 需要全局开启 feature(panic_info_message) 才可以调用 .message() 函数
    if let Some(location) = info.location() {
        println!(
            "\x1b[1;31m{}:{}: '{}'\x1b[0m",
            location.file(),
            location.line(),
            info.message().unwrap()
        );
    } else {
        println!("\x1b[1;31mpanic: '{}'\x1b[0m", info.message().unwrap());
    }

    // This call is not expected to return under normal conditions.
    // Returns SBI_ERR_FAILED through sbiret.error only if it fails, 
    // where SBI_ERR_FAILED = -1.
    let sbiret = sbi_hart_stop();
    println!("sbiret.error = {}, sbiret.value = {}", sbiret.error, sbiret.value);
    
    unreachable!()
}
```
### 遇到问题
对于本问题的分析如上。但是对于目前给出 `shutdown` 的方案仍然感到疑惑。