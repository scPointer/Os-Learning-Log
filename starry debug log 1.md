# starry debug log 1

## 1. 报告问题

@Azure-stars 告诉我出现了这样一个bug。

在`axfs/src/dev.rs`中：

```rust
    pub fn read_one(&mut self, buf: &mut [u8]) -> DevResult<usize> {
        // info!("block id: {}", self.block_id);
        let read_size = if self.offset == 0 && buf.len() >= BLOCK_SIZE {
            // whole block
            self.dev
                .read_block(self.block_id, &mut buf[0..BLOCK_SIZE])?;
            self.block_id += 1;
            BLOCK_SIZE
        } else {
            // partial block
            let mut data = [0u8; BLOCK_SIZE];
            let start = self.offset;
            let count = buf.len().min(BLOCK_SIZE - self.offset);

            self.dev.read_block(self.block_id, &mut data)?;
            // info!("start: {} count: {}", start, count);
            buf[..count].copy_from_slice(&data[start..start + count]);

            self.offset += count;
            if self.offset >= BLOCK_SIZE {
                self.block_id += 1;
                self.offset -= BLOCK_SIZE;
            }
            count
        };
        Ok(read_size)
    }
```

运行到 `self.dev.read_block(self.block_id, &mut data)?:` 时报错

```
[    2.105405 0:3 axruntime::lang items:5] panicked at modules/axfs/src/dev.rs:53:47:range end index 18446744069414584324 out of range for slice of length 512
```

但是如果加上调试输出`assert!(self.offset <= BLOCK SIZE);`

报错就会变成

```
[    2.143353 0:3 axhal::arch::riscv::trap:81] S page fault from kernel, addr: 0xffffffff00080000 sepc:FFFFFFC080232F6
[    2.144294 0:3 axruntime::lang_items:5] panicked at modules/axhal/src/arch/riscv/trap.rs:86:17:
not implemented:
S page fault from kernel
```

且 assert 本身没有问题

## 2. 尝试排查

询问得知这是单核运行 `busybox echo hello `，且是 starry 当前版本做了简单修改后得到的。

发现其中出错的buffer下标为 `0xffffffff00000004`，这不是一个 block_id 或者 offset 在 usize 下被错误设置为 -1 之类的值，而是一个很奇怪的类似地址的值，但没有后续。

## 3. 寻找具体bug位置

首先在本地复现了这个bug。

然后找到调用路径为

```
axprocess/src/api.rs:97 load_app
108 -> axfs/src/api/mod.rs:40 read
41 -> axfs/src/api/file.rs:131 File::open
132 -> axfs/src/api/file.rs:71 OpenOptions::open
72 -> axfs/src/api/fops.rs:165 File::open
166 -> axfs/src/api/fops.rs:116 File::_open_at
141 -> axfs/src/api/fatfs.rs:68 FileWrapper<'static>::get_attr
70 -> rust-fatfs/src/file.rs:412 File<'_, IO, TP, OCC>::seek
```

其中倒数第二步是内核想要拿到文件的size，因此通过`seek(SeekFrom::End(0))`尝试找到文件末尾，从而获取文件大小信息。

而 fatfs 中的数据块是链表式的，因此为了找到 busybox(size=1387560B) 这个文件的末尾，需要链表跳跃 2710 次，其中每一次在 rust-fatfs 中还有若干层调用，但总会走到第一节提到的`read_one`函数中去拿数据块的元信息。

加入调试输出发现，内核总是跳到几十次或者几百次的时候就报了上面的错误，而且每次报错的次数不一样。通过修改内核栈大小，这个出错的位置也会大幅变动。

由此确定是栈或者分配内存的问题。

## 4. 确定问题

把这个信息回馈给 @Azure-stars ，他检查发现是 TrapContext 写错了，汇编里的 TrapContext 跟 Rust 代码里定义的不一样，导致内核栈被奇怪的地址覆盖了。

这确实符合报错位置比较随机的情况，也解释了为什么一个数组的下标会变成 `0xffffffff00000004` 这样一个看上去就像地址的数。

## 5. 总结

- 随机的报错位置、甚至输出与否都会改变报错情况，说明很可能是栈有问题

- 改 .cargo 里的库调试时记得先 make clean，因为rust默认.cargo的不会被更新就不会自动检查（上文没有涉及，是@Azure-stars 调试时说改 rust-fatfs 没有反应，所以我额外提一下）

- 参数尽量避免重定义。虽然实际出错的地方没有涉及到，但我也发现 arceos / starry 本身的参数比较乱，容易重定义：

比如 axprocess里的 KERNEL_STACK_SIZE 是 0x40000，arceos本身还有一个 axconfig::TASK_STACK_SIZE，后者本来应该是启动栈大小，但是它名字又叫 task。而在 axtask/src/run_queue.rs:32 又是用 TASK_STACK_SIZE 来作为应用的栈大小，这两个常量就混用了。然后 axtask/src/run_queue.rs:242 还有个局部定义的 IDLE_TASK_STACK_SIZE。



update：最后把栈大小常量统一到 axconfig 里了
