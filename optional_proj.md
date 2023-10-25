# 

## 选题：在 `arch64` 和 `x86_64` 下支持基于换栈的 `axbacktrace`

目前 `axbacktrace` 模块（ https://github.com/Azure-stars/Starry/tree/feat/module/modules/axbacktrace ）仅支持 `riscv64`的实现，你需要修改这个模块以及 `alter_trap` 模块（ https://github.com/Azure-stars/Starry/tree/feat/module/crates/alter_trap ），使得它们能在 `arch64`和`x86_64`架构下运行。

### 说明和要求

- `axbacktrace`涉及栈帧的内容，`alter_trap`涉及中断异常处理及特权寄存器（CSR）的内容，你可能需要复习这些知识，以及查询它们在三种架构下的实现。

- `alter_trap`依赖于 Arceos 的另一个模块 `kernel_guard`，你可能需要事先了解它的功能以及实现原理

- 请确保你的实现满足以下需求
  
  - `backtrace`不能依赖其他模块，不能由异常中断导致死循环。
    
    因为通常只有在 `panic`发生时我们才会调用 `backtrace`，因此，在这个模块中你不能信任内存管理、trap 等等模块，它们都可能已经出现问题，而 `backtrace`自己不能再重复触发问题。
    
    事实上，`alter_trap`模块正是为了上面这个目的而实现的。它将暂时接管异常中断处理，以保证 `backtrace` 模块运行过程中不会跑到默认的中断处理函数导致触发 `panic`。
  
  - `backtrace`查询、回退栈帧位置正确。
    
     `backtrace`模块目前给出的是16进制地址输出。它有输出不代表输出的结果是正确的。你可以在内核代码中插入一个 `panic`，故意触发 `backtrace`，然后通过反汇编或者 `gdb`检查`backtrace` 触发时输出的地址是否是栈帧的地址。写一个文档记录你的工作。
  
  - `alter_trap`不会被中断打断、不会被抢占打断。
    
    详见 `alter_trap`代码中对 `kernel_guard`模块的使用
  
  - 参照目前这两个模块的文档，给出完善的文档说明。

### 一些进阶内容

如果你仍有余力，可以尝试以下的一部分扩展内容

- 使 `backtrace` 输出具体的函数名和位置。类似用户态 `rust` 程序报错后的 `backtrace`

- 尝试让 Arceos 的宏内核支持 `arch64` 和 `x86_64`。 Arceos Unikernel 已经支持了这两种架构。而从 Arceos Unikernel 到 Arceos 宏内核，添加的 **架构相关** 的代码不多，`backtrace`所涉及的异常中断处理就已经涉及了其中的大部分。只需要在其他架构相关的地方设置切换，就可以让 Arceos 宏内核支持这两个架构。

## 选题：针对比赛测例，设计一个类似 `strace` 的模块以提供更详细的 syscall 说明

在 rCore-Tutorial 教程和 OS 比赛初赛时，我们通常会对 OS 需要支持的 syscall 以及它们的详细参数给出定义文档，要求同学们写的内核按照文档里的语义实现。如比赛初赛的 syscall 文档在（ https://github.com/oscomp/testsuits-for-oskernel/blob/master/oscomp_syscalls.md ）。但是比赛决赛第一阶段/第二阶段不会给出 syscall 文档，需要大家根据实际测例去分析需要的 syscall 和参数。

`strace` 是 `Linux` 下的一个工具，可以记录程序运行时的所有 syscall 调用。目前 `Starry` （ https://github.com/Azure-stars/Starry ）已支持 2023年比赛决赛要求的所有测例，你可以先尝试用它运行所有决赛测例，给出每个 syscall 的运行次数，以此熟悉环境。（一些 syscall 会被反复调用，因此你不能立即输出所有 syscall 的调用信息）

之后，你需要设计一个或多个模块，使得在 Arceos 中加入这个模块后，可以运行一组测例并输出使用的 syscall 的统计信息，精确到参数。

例如运行的测例多次调用了 `sys_mmap`，那么模块可能分析出以下信息（你也可以用另外的格式）

```
mmap {
    start: usize[0x1000, 0x213200],
    len: usize[0x0, 0x20000],
    prot: u32[PROT_READ=0x1 | PROT_WRITE=0x2 | PROT_EXEC=0x4],
    flags: u32[WNOHANG=0x1],
    fd: i32{-1, [4, 6]},
    offset: usize[0x0, 0x42],
}
```

- 参数 `start` `len` 是用户给的 `usize` 参数，在每次调用 `sys_mmap` 时，`start`这个参数最小给出过 `0x1000` ，最大给出过 `0x213200` 。（每次 mmap 会要求一个地址，所以统计测例具体要哪个地址没有太大意义，统计参数取值的区间比较有意义）

- `prot` 是一个标志位，在这个测例运行时，`PROT_READ` `PROT_WRITE` `PROT_EXEC` 3个标志位都在至少一次 `sys_mmap` 调用中被用到。（注意需要标出每个标志位实际对应 u32/usize 的哪一位）

- `flags`是一个标志位，在测例运行时只用到了 `WNOHANG`（注意，syscall定义中 Flags 不止 `WNOHANG` 一种！模块需要判断哪些 `flags` 实际被测例用到）

- `fd`是文件标识符，测例给出过 `-1` （代表不映射到实际文件），也给出过真正的文件标识符值，用到的文件中，标识符最小为 `4`最大为 `6` 。（每个参数用 16 进制还是 10 进制输出？`-1` 或者 `0` 是不是个表达不同含义特殊值？需要详细规定）

- `offset` 类似 `start` 和 `len`

你可能已经注意到了，要做到上面的输出，除了模块之外，光靠模块自动分析是不够的，还需要事先做一些手动的分析。具体来说

- 对于每个 syscall，每个参数分别是什么类型？用 Rust 定义它们。
  
  - 如上面的 `sya_mmap`，它的 `start/len/offset` 是一个数值；`prot/flags` 是一个标志位，每一个 bit 代表不同的涵义；`fd`虽然也是一个数值，但它等于 `-1` 时有特殊含义；另外， `sys_open` 的参数 `path` 是一个地址；以及 `sys_sigaction`的 `action` 参数也是一个地址，但它可以为 `0`，表示另一种特殊含义。

- 你可以参考这个表格（ [RISC-V Linux syscall table | Juraj&#39;s Blog](https://jborza.com/post/2021-05-11-riscv-linux-syscalls/) ）

- 不需要统计所有 syscall，只需要考虑比赛决赛需要的 syscall

然后你的模块就可以对每次 syscall 调用，根据参数的不同类型去统计，产生类似上面举例的输出。

### 一些进阶内容

如果你仍有余力，可以尝试以下的一部分扩展内容

- 让你的输出形成可读的中文或英文文档，类似上面解释输出那样。
  
  - 这一步不一定要在内核里实现，你可以用另外的程序输入上面产生的结果，然后输出一个文档。也不要求用 Rust。

- 支持测例间 syscall 统计的“减法”。
  
  - 假设某同学已经支持了 a、b 测例集，还想支持 c 应用。TA可以用这个模块求出 a+b 需要的所有 syscall 及参数、a+b+c 需要的所有 syscall 及参数，然后“减”出**想支持c应用还需要新加的 syscall 和参数**

- 支持“合法性”检测
  
  - 假设某同学已经支持了 a、b 测例集，还想支持 c 应用。我

## 选题：支持新应用

Arceos 的宏内核可以支持原生 Linux 应用，但功能不全。你可以选择一些 Arceos 仍不支持的应用，然后：

1. 尝试在 Arceos 中运行这些应用直到出错。注意检查你的出错信息，确保它是由于 Arceos 功能缺失而报错，而不是因为你的配置错误

2. 分析错误以及缺失的功能、模块、syscall

3. 完成这些新的功能，并提供测例和说明文档，说明如何运行这个应用。如有可能，请尽可能少修改 Arceos 原本模块的代码，用新模块实现新功能，并保证 Arceos 运行其他应用时的逻辑不变。

4. 将你的工作提交到 Arceos
