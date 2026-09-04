# GuestOS

GuestOS 是一个基于 [rCore-Tutorial-v3](https://github.com/rcore-os/rCore-Tutorial-v3) 完成的 RISC-V 教学操作系统。项目从 `no_std` Rust 内核出发，在 rCore 框架上逐步实现了日志、特权级切换、进程管理、虚拟内存、文件系统和进程间通信等功能。

GuestOS 不是一个通用操作系统发行版，而是一份面向操作系统原理学习的实验代码与报告集。代码主要运行在 QEMU `virt` 平台，同时保留 K210 的构建支持。

## 实现概览

| 模块 | 实现内容 | 主要位置 |
| --- | --- | --- |
| 内核日志 | `error!` / `warn!` / `info!` / `debug!` / `trace!` 分级彩色日志，通过 `LOG` 构建变量选择级别 | `os/src/console.rs` |
| Trap 与系统调用 | U/S 态切换、寄存器上下文保存与恢复、时钟中断及系统调用分发 | `os/src/trap/`, `os/src/syscall/` |
| 进程管理 | PCB、PID 分配、内核栈、上下文切换、`fork` / `exec` / `spawn` / `waitpid` / `exit` | `os/src/task/`, `os/src/syscall/process.rs` |
| 内存管理 | 物理页帧分配、SV39 三级页表、内核/用户地址空间、`mmap` / `munmap` 参数检查与映射 | `os/src/mm/`, `os/src/syscall/memory.rs` |
| 文件系统 | easy-fs 块缓存、inode/VFS、文件描述符、管道，以及 `open` / `read` / `write` / `dup` / `fstat` / `linkat` / `unlinkat` | `easy-fs/`, `os/src/fs/`, `os/src/syscall/flinker.rs` |
| 进程间通信 | 基于每进程邮箱和管道的 `mail_read` / `mail_write` 接口 | `os/src/fs/mail.rs`, `os/src/syscall/fs.rs` |
| 用户态支持 | 用户库、堆分配、Shell 和文件/进程/管道测试程序 | `user/` |

## 相对 rCore 教学框架的修改范围

本项目保留了 rCore-Tutorial-v3 的整体分层，主要工作集中在下列实验扩展：

1. **Lab 1 — 日志与启动流程**
   实现五级日志宏，跟踪 QEMU 上电、RustSBI 初始化以及跳转到 `0x80200000` 的内核启动过程。
2. **Lab 2 — Trap 与访存检查**
   完成 U 态/S 态之间的 trap 上下文切换，并对 `sys_write` 传入的用户地址进行边界检查。
3. **Lab 3 — 时间与优先级**
   增加 `get_time` 和 `set_priority` 系统调用，并完成过 stride 调度实验与公平性分析。需要注意：当前 `main` 分支使用 FIFO 就绪队列，`stride.rs` 中的实验实现已注释，因此 stride 属于报告中记录的阶段性成果，不是当前默认调度策略。
4. **Lab 4 — 虚拟内存**
   实现页帧分配、SV39 页表和进程地址空间，添加 `mmap` / `munmap`，并修正内核态访问用户态 `TimeVal` 时的地址转换问题。
5. **Lab 5 — 进程创建**
   在 `fork + exec` 之外实现 `spawn`，直接从文件加载新程序并建立父子关系，减少无效的地址空间复制。
6. **后续扩展 — 邮箱 IPC 与文件链接**
   代码中进一步加入每进程邮箱、`mail_read` / `mail_write`，以及 `linkat` / `unlinkat` / `fstat` 等接口。其中链接映射主要用于教学验证，不具备生产文件系统的完整持久化和安全语义。

## 实验成果

- 内核可在 QEMU 中通过 RustSBI 启动，完成内存、trap、时钟中断、文件系统和 `initproc` 初始化。
- 用户程序可通过系统调用完成进程创建、等待与退出、动态内存映射、文件 I/O 和管道通信。
- `reports/` 保留了 Lab 1–5 的实验过程、关键问题分析、运行截图和参考资料。
- 仓库中保留了 `hello_world`、Shell、管道、文件和进程相关的用户态程序，方便在 QEMU 中继续实验。

部分实验截图：

| 分级日志 | 内存管理测试 | `spawn` 测试 |
| --- | --- | --- |
| ![Lab 1 日志输出](reports/lab1/info.png) | ![Lab 4 内存管理测试](reports/lab4/result.png) | ![Lab 5 spawn 测试](reports/lab5/result1.png) |

## 仓库结构

```text
GuestOS/
├── os/             # Rust 内核：trap、内存、调度、系统调用、文件系统
├── user/           # 用户库与用户态程序
├── easy-fs/        # no_std 简易文件系统
├── easy-fs-fuse/   # 宿主机端文件系统镜像打包工具
├── bootloader/      # QEMU / K210 的 RustSBI 二进制
└── reports/         # Lab 1–5 实验报告与截图
```

## 运行方式

### 方式一：使用现成 Docker 环境

仓库顶层 Makefile 默认使用 `dinghao188/rcore-tutorial` 镜像：

```bash
make docker
```

进入容器后，仓库挂载在 `/mnt`：

```bash
cd /mnt/os
make run BOARD=qemu LOG=info
```

### 方式二：本机构建

本项目固定使用 `nightly-2021-01-30`，并依赖带 RISC-V 支持的 QEMU 和 Cargo binutils。

```bash
rustup toolchain install nightly-2021-01-30
rustup target add riscv64gc-unknown-none-elf --toolchain nightly-2021-01-30
rustup component add rust-src llvm-tools-preview --toolchain nightly-2021-01-30
cargo +nightly-2021-01-30 install cargo-binutils

cd os
make run BOARD=qemu LOG=info
```

`LOG` 支持 `error`、`warn`、`info`、`debug` 和 `trace`。它通过 Rust `option_env!` 在编译时写入内核，更换级别后需要重新构建。

## 实验报告

- [Lab 1：日志与 RustSBI 启动流程](reports/lab1.md)
- [Lab 2：Trap 与用户内存访问](reports/lab2.md)
- [Lab 3：进程调度与优先级](reports/lab3.md)
- [Lab 4：SV39 页表与内存映射](reports/lab4.md)
- [Lab 5：进程创建与 `spawn`](reports/lab5.md)

## 已知限制

- 项目依赖 2021 年的 Rust nightly 和旧版 QEMU/Rust 生态，在最新工具链上可能需要适配。
- 当前默认调度器为 FIFO，优先级接口不会改变就绪队列的取出顺序。
- `spawn` 的内核路径已实现，但当前用户库封装未向它传递参数列表。
- 邮箱 IPC 和文件链接是教学性实现，未覆盖生产内核所需的完整并发、持久化与错误恢复语义。

## 致谢与许可证

项目基于 rCore-Tutorial-v3 教学框架开发，感谢 rCore 社区及课程维护者。仓库按 [GNU General Public License v3.0](LICENSE) 开源。
