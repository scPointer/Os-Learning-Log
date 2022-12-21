# rust 语法补充学习

这里记录了一些在看其他 OS 是遇到的新 rust 语法。不排除这些语法只是我没见过的可能性。

### 0x00 Arc::new_cyclic

> from thesues

这个方法的原型是

```rust
impl<T> Arc<T> {
    pub fn new_cyclic(data_fn: impl FnOnce(&Weak<T>) -> T) -> Arc<T> {
```

允许在创建 Arc 之前预先拿到 Weak 指针，用于构造循环引用。例：

构造一个 TCB，其中一些东西需要用到对TCB本身的指针。

如 vm 要用 `Arc<TCB>` 初始化，按照原来的方法就需要先用一个空的 vm 构造 TCB，再用构造好的 TCB 的指针去构造其中的 vm。有了 new_cyclic 以后就可以假设已有 Weak 指针去构造完整的 TCB 了

### 0x01 makefile 联动 cfg 宏

> from thesues

环境变量 `RUSTFLAGS` 可以被传递给 rustc

1. 可以在 `.cargo/config.toml` 里写 `rustflags = "..."`
2. 可以在运行时用 `RUSTFLAGS="..." cargo run`

在 RUSTFLAGS 中添加 cfg 宏的格式是 `--cfg XYZ`

thesues 的思路是，在 makefile 中加入自定义变量 `THESEUS_CONFIG`，如

`THESEUS_CONFIG="cfg_option_1 cfg_option_2"`

然后再把这个变量按格式塞入 RUSTFLAGS 中：

```bash
export override RUSTFLAGS += $(patsubst%,--cfg %, $(THESEUS_CONFIG))
```

最后再按上面描述，在cargo run时加入环境变量即可。

> 这个方法比较像 C，可以和 rust feature 相区别，也就不必走 feature 依赖 feature 那一套方法

### 0x02 函数指针类型

> from aero

小写的 fn() 可用来表示函数指针类型，如：

```rust
pub enum SignalHandler {
    Ignore,
    Default,
    Handle(fn(usize)),
}
```

之前都是在参数里使用类似 `f: impl Fn(...) -> (...)` 或者 `where F: Fn(...) -> (...)` 的语法。

### 0x03  doctest 属性叠加

> from aero

如在注释里嵌入rust代码，但不运行运行，可以在 ````` 中使用如下属性，然后用逗号分隔：

```rust
//! ## Example
//!
//! ```rust,no_run
//! fn hello_init() {}
//! fn hello_exit() {}
//! ```
```

还有 `should_panic`  `compile_fail` `edition2018` `edition2021` 这些属性常用，但标准库中似乎更喜欢把理应 panic 或者编译失败的代码注释掉并加上标注。


