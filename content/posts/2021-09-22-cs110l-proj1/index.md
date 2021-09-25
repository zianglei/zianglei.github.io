---
title: "CS110L Proj-1: DEET 简易调试器"
date: 2021-09-22T17:17:00+08:00
toc: true
description: gdb 的精简版
---

> 实现 DEET 调试器，一个简易版的 GDB，支持进程调试、断点和单步调试等。通过 DEET 可以练习在 Rust 中如何使用多进程，如何使用 ptrace 管理进程等

## Milestone 1：Run the inferior

第一步是要将我们要调试的进程运行起来，并通过 ptrace 控制调试进程的执行状态。

rust 提供了 [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html) 运行子进程，为了方便我们跟踪子进程的状态，我们需要使用 [nix::sys::ptrace](https://docs.rs/nix/0.22.1/nix/sys/ptrace/index.html) 提供的 traceme，在 pre_exec 阶段设置由 ptrace 跟踪子进程。

之后在 exec 执行后，子进程会进入暂停状态，我们需要通过 waitpid 检查子进程是否进入暂停状态，在暂停状态 waitpid 会返回 WaitStatus::stopped，可以通过检查返回值判断子进程是否处于 stopped 状态。

在子进程停止后，我们需要让子进程再继续运行，在 inferior 结构体加入 cont 函数，调用 ptrace 的 cont 函数，并使用 wait（也是调用的 waitpid）等待子进程执行完毕，通过返回状态判断子进程是否结束/信号原因退出/暂停状态

## Milestone 2：Stopping, resuming, and restarting the inferior

在将子进程运行起来之后，我们需要有能力控制子进程，可以停止、恢复和重启子进程（例如子进程死锁之后需要强制停止子进程）。

如果一个进程被 ptrace 追踪，那么当使用 CTRL+C（发送 SIGINT 信号）的时候，会暂停而不是停止，方便后续查看调用栈等操作。

因此在 Milestone 2，DEET 需要加入 continue 命令，负责恢复被暂停的进程。此外，需要考虑以下几种情况：

- 如果当前没有被暂停的子进程，continue 需要打印提示信息
- 如果当前有被暂停的进程，又重新执行 run 命令，那么需要将之前的子进程杀死，重新创建新的子进程
- 如果在退出 DEET 的时候，子进程还在暂停状态，我们也需要杀死子进程

判断子进程是否在运行可以使用 try_wait，根据返回的状态判断是已经退出还是正在运行。

## Milestone 3：Printing a backtrace

打印子进程的调用栈，通过 ptrace::getregs 获得子进程的寄存器组，从中得到 %rip 寄存器的值。但是为了得到有用的调试信息（例如调用函数名，行数等），我们还需要读取「调试符号」信息，这些信息包含代码中的函数名、入参名、变量名、行号等映射信息，通常以 DWARF 格式嵌入在二进制文件中（编译时需要开启调试选项）。

DWARF 格式比较复杂，在 DEET 中使用 gimli 库解析，在 gimli_wrapper.rs 提供了若干工具函数。

我们需要在 Debugger 初始化的时候，读入调试信息数据，存储在 Debugger 中。当调用 backtrace 时，通过工具函数读取调试信息（例如 get_function_from_addr, get_line_from_addr 等）

如果想要递归读取调用栈，根据 x86 栈结构 [这里]()，可以通过 %rbp 读取到调用者的 %rip 和 %rbp，一直递归到函数名为 main 即可

## Milestone 4：Print stopped location

利用 wait 返回的 Stopped status 包含的 %rip 地址，打印出来即可

## Milestone 5：Setting breakpoints

> 支持形如 break *0x123456 打断点的形式
>
> 需要在 Inferior 没有运行时也可以打断点

为了能够在 Inferior 运行之前打断点，我们需要在 Debugger 里面存储所有的断点，然后在 Inferior 运行时将断点加到 Inferior 上。

GDB 实现在某个地址打断点的方式是替换相应地址的字节为 0xcc （INT 中断指令）。由于 ptrace 只能对一个 usize 进行操作，无法直接操作一个字节，因此我们需要使用位操作对某个字节进行替换。指导网页上已经给出了相关实现：

```rust
use std::mem::size_of;

fn align_addr_to_word(addr: usize) -> usize {
    addr & (-(size_of::<usize>() as isize) as usize)
}

impl Inferior {
    fn write_byte(&mut self, addr: usize, val: u8) -> Result<u8, nix::Error> {
        let aligned_addr = align_addr_to_word(addr);
        let byte_offset = addr - aligned_addr;
        let word = ptrace::read(self.pid(), aligned_addr as ptrace::AddressType)? as u64;
        let orig_byte = (word >> 8 * byte_offset) & 0xff;
        let masked_word = word & !(0xff << 8 * byte_offset);
        let updated_word = masked_word | ((val as u64) << 8 * byte_offset);
        ptrace::write(
            self.pid(),
            aligned_addr as ptrace::AddressType,
            updated_word as *mut std::ffi::c_void,
        )?;
        Ok(orig_byte as u8)
    }
}
```

除了在 Inferior::new 函数中加上断点，我们还需要实现在 inferior 停止的时候添加断点，也即在 breakpoint 命令处判断是否存在 inferior，存在就直接更改断点地址的内存数据为 0xcc

## Milestone 6：Continuing from breakpoints

Milestone 5 在断点位置设置了 0xcc，断点位置也是下一个指令的第一个字节，x86中还存在多字节指令的情况，因此我们需要将 0xcc 替换成指令原来的值。

但是在替换之后，我们的断点实际上就不存在了，我们想要的其实是之后再执行到断点位置时仍然能暂停。因此我们需要先将断点位置替换为原来的字节，然后单步执行（调用 ptrace 的 step，如果此处程序终止，就停止后续流程），之后再把原来的断点位置换为 0xcc，最后调用 cont 继续执行，这样就恢复了断点。

上述流程意味着我们需要记录设置断点的位置原本的内存值。在 Milestone 5 我们使用 usize 表示一个断点，在 Milestone 6 我们需要使用 Hashmap 维护 struct breakpoint，在程序停止位置判断是否存在断点。

注意在保存 breakpoint 的时候，使用的 key 是 breakpoint 的地址，但是在 inferior 因为断点停止时，rip 的值是 breakpoint 的地址 + 1 （因为已经执行过 0xcc），所以需要使用 rip - 1 查找断点

## Milestone 7：Setting breakpoints on symbols

之前的断点只支持指定内存地址，在 Milestoßne 7 中需要继续支持在行号和函数名。

只需要在 Debugger 的 Breakpoint 命令里加上解析行号和函数名的分支即可，然后统一添加断点

## Extension 1：Next command

实现 next command，执行流程和 continue 命令相似，伪代码如下

```
get old line number

while true {
	ptrace::step()
	if inferior terminates, break
	else {
		if inferior stopped at a breakpoint previously {
			reset breakpoint
			clear flag
		}
		if signal is SIGTRAP {
			if there is a breakpoint, restore it, and set a flag
				break
			else
				if line number != old line number, reach the next line, break;
		} else {
			other reason, break
		}
	}
}
```

根据这个逻辑实现的类似于 gdb 中的 step 功能，因为在函数调用发生后无法准确通过行号判断是否函数调用执行完毕。而且由于 DWARF_wrapper 仅支持在当前可执行文件找到源代码，因此只能获取当前 target 的源代码。

如果想要实现类似 gdb 的 next 功能，一个可能的想法是记录函数的调用栈，当函数调用完成，调用栈为空的时候，表明执行到了当前范围的下一条指令。

## Extension 2：Print the code

第二个扩展是在 Stopped 的时候打印当前行的代码，由于 DWRAF 返回的 Line 结构体包含 file 和 line_number，因此直接读取源文件并打印对应行的字符串即可。



