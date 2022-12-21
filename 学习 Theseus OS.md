# 学习 Theseus OS

[TOC]

## 动态加载内核模块

首先需要设置 `THESEUS_CONFIG="loadable"`，或者 `make loadable` 开启。启动时，含 `#![cfg(loadable)]` 的模块会被动态加载，具体来说：

- 所有这些动态模块会被 build 到单独的对象文件中，但不会链接

- bootloader 会把这些对象文件加载到内存中，这样 theseus 内核(`nano_core`)启动时就可以加载它们，这里不需要 fs 支持

- 在可加载代码块中，调用者在运行时动态查找给定一个函数的符号，并动态调用，而不是通过常规函数调用直接调用它。这在源代码中产生了一个“软依赖(soft dependency)”，而不是一个需要静态链接的"硬依赖"。例：

```rust
type MyPrintFuncSignature = fn(&str) -> Result<(), &'static str>;
let section = mod_mgmt::get_symbol_starting_with("my_crate::print::").upgrade().unwrap();
let print_func: &MyPrintFuncSignature = unsafe { section.as_func() }.unwrap();
print_func("hello there");
```

## cargo 的支持问题

rust 缺少稳定的 abi，因此难以稳定、安全地集成(integrate)多个rust二进制文件。“集成”指使一个二进制文件调用另一个预先构建(prebuild)的二进制文件，如静态链接的库、动态链接的对象

cargo 不支持一个 crate 依赖于一个 prebuild 的 crate，如 a 依赖 b，不能先生成 b.o，再让 a 依赖 b.o 做编译。只能写进 dependencies 里然后 a,b 一起编译

或者有个不太优雅的变通方法，交给 rustc 用 C 的方式来做（参考 https://github.com/rust-lang/cargo/issues/1139 ）：

```rust
use std::io::fs;

fn main() {
    let from = Path::new("/tmp/rust-crates/a/target/liba-b2092cdbfc1953bd.rlib");
    let to = Path::new("/tmp/rust-crates/b/blah/liba-b2092cdbfc1953bd.rlib");
    fs::copy(&from, &to).unwrap();
    println!("cargo:rustc-flags=-L /tmp/rust-crates/b/blah");
}
```

这样指定库文件是可以的（事实上 theseus 的解决方案是对上面这个方法的包装）。

**基于以上的原因， theseus 想要替换一个 crate 的时候，没法简单地重新编译这个模块本身。** 如模块 page_allocator 包含接口  page_allocator::allocate_pages_at()，在初次编译时它可能对应符号 `_ZN14page_allocator17allocate_pages_at17heb9fd5c4948b3ccfE`，但如果试图用同样的接口、同样的依赖编译一个新模块，那么 cargo 会从代码层重新编译它的所有依赖，使得所有接口对应的符号中的 hash 不同，不能用进已启动(已经有了一个实现，需要替换掉)的系统内。否则会导致两个相同模块的实现同时存在。

事实上， thesues 在"两个相同模块的实现同时存在"时不会报错，而是会单纯认为这个模块被卸载了。见 `/kernel/src/mod_mgmt/src/lib.rs` 对trie的处理提到：

```rust
    /// Returns a weak reference to the `LoadedSection` whose name beings with the given `symbol_prefix`,
    /// *if and only if* the symbol map only contains a single possible matching symbol.
    /// This will also search the recursive namespace's symbol map. 
    /// 
    /// # Important Usage Note
    /// To avoid greedily matching more symbols than expected, you may wish to end the `symbol_prefix` with "`::`".
    /// This may provide results more in line with the caller's expectations; see the last example below about a trailing "`::`". 
    /// This works because the delimiter between a symbol and its trailing hash value is "`::`".
    /// 
    /// # Example
    /// * The symbol map contains `my_crate::foo::h843a613894da0c24` 
    ///   and no other symbols that start with `my_crate::foo`. 
    ///   Calling `get_symbol_starting_with("my_crate::foo")` will return 
    ///   a weak reference to the section `my_crate::foo::h843a613894da0c24`.
    /// * The symbol map contains `my_crate::foo::h843a613894da0c24` and 
    ///   `my_crate::foo::h933a635894ce0f12`. 
    ///   Calling `get_symbol_starting_with("my_crate::foo")` will return 
    ///   an empty (default) weak reference, which is the same as returing None.
    /// * (Important) The symbol map contains `my_crate::foo::h843a613894da0c24` and 
    ///   `my_crate::foo_new::h933a635894ce0f12`. 
    ///   Calling `get_symbol_starting_with("my_crate::foo")` will return 
    ///   an empty (default) weak reference, which is the same as returing None,
    ///   because it will match both `foo` and `foo_new`. 
    ///   To match only `foo`, call this function as `get_symbol_starting_with("my_crate::foo::")`
    ///   (note the trailing "`::`").
```

### 一个不好但能用的办法

通过 C API 的调用，把 rust结构按照 C 的 `memory layout` 和调用约定处理：

```rust
// in `page_allocator`
#![allow(unused)]
fn main() {

#[no_mangle]
pub extern "C" fn allocate_pages_at(num_pages: usize, ...) -> ... {
    page_allocator::allocate_pages_at(num_pages, ...)
    ...
}
}

// in `my_crate` 
extern "C" {
    fn allocate_pages_at(num_pages: usize, ...);
}
fn main() {
    unsafe {
        allocate_pages_at(15, ...);
        ...
    }
}
```

缺点是所有用到的 rust 结构都要符合 C 的规范，而且上面的这些包装写起来比较麻烦，但其实是可以自动生成的

### Theseus 的办法：写一个 theseus_cargo

> theseus_cargo 是什么？
> 
> 本身是一个 rust 写的程序，在 `tools/theseus_cargo`。它重新包装了 cargo ，它会去运行 cargo，截获输出，修改后再扔给 rustc 运行。它自己可在 theseus 内运行

如现在(运行时)要编译一个库，那么它会做以下事情：

- 调用 `tools/copy_latest_crate_objects`（也是一个 rust 写的程序），它会把新库的所有依赖复制到一个新的 `/build/deps` 文件夹里(第三方库取所有版本)，并把 core alloc 等标准库的子库也放到里面的 `/sysroot/`

- 调用 cargo 编译这个模块，并把预编译的库以 .rmeta .rlib 的形式传递给 rustc，主要利用以下参数指定
  
  - `-L dependency=<dir>`
  - `--extern <crate_name>=<crate_file>.rmeta`

这个方式其实还是不理想，但目前没有更有效的方式截获 cargo 输出并修改 rustc 命令

## 动态加载 C 程序

theseus 是 unikernel，它除了要把用户程序加载到内核态来运行，还要负责处理上面的动态加载问题

目前大多数标准库和libc实现都构建为完全链接的静态库/动态库，但在 theseus 中不适用：

- 通常程序的依赖链在 syscall 结束，因此从链接库的角度看，不需要包含内核信息。但 theseus 的用户程序需要链接到内核，而内核里的模块又是可替换的，所以只能是**运行时动态链接**。

- 所以在 theseus 中，独立的 C library 没法按完全静态链接的二进制文件形式直接使用

对于这个问题， theseus 在 build 用户程序时做完整的静态链接，但保留了完整的重定向信息，然后在加载 elf 时重写了这些信息

### build 时静态链接，tlibc

首先，为了把 syscall 改成函数调用，它需要自己写一套 libc，在这里是叫 `tlibc`，在根目录下 `/tlibc`

对 C 程序时的链接过程(在 theseus 内)分三步

- 编译链接 tlibc 相关的东西，生成 `tlibc.o`

- 编译 C 程序

- 结果链接到 tlibc（在 C 上没有上面 rust 的链接 prebuild 的库的问题）

具体指令为

```bash
x86_64-elf-gcc                             \
    -mno-red-zone -nostdlib -nostartfiles  \
    -ffunction-sections -fdata-sections    \
    -mcmodel=large                         \
    -Wl,-gc-sections                       \
    -Wl,--emit-relocs                      \
    -o dummy_works                         \
    path/to/crtbegin.o                     \
    dummy.c                                \
    path/to/tlibc.o                        \
    path/to/crtend.o
```

其中：

- `-mno-red-zone, -mcmodel=large`: 是为了匹配 theseus 在 rust 这边的 configuration。
  
  - `-mno-red-zone` 本身也是为了把程序放在内核态(否则中断会破坏 red zone)

- `-Wl,--emit-relocs`: 为了包含重定向信息(.rela
   段)

### 加载时修改重定向信息

因为上面生成的 elf 是完全静态链接的，所以可能包含一些重复的实例(theseus cells)，或者是已经在运行的单例，例：物理页的帧分配器。

为了解决这个问题，theseus 重写了上面这个静态链接的elf文件的重定位部分（用 `applications/loadc/`，也是一个 rust 程序），对于已加载到 theseus 的部分，修改其引用；对于未初始化/未使用的部分，保持不变。

loadc 还负责加载 elf 的段等等
