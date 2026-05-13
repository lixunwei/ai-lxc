# LXC 命令深度分析：lxc-attach

## 1. 概述

`lxc-attach` 允许在运行中的容器命名空间内执行进程。它使用 **两次 fork + CLONE_PARENT**
设计来正确进入所有命名空间，同时支持完整的安全上下文（LSM、Capabilities、Seccomp）和终端转发。

**典型用法**：
```bash
lxc-attach -n mycontainer                          # 进入容器 shell
lxc-attach -n mycontainer -- /bin/bash             # 执行指定命令
lxc-attach -n mycontainer -e -s 'NETWORK|MOUNT'    # 只进入网络和挂载命名空间
lxc-attach -n mycontainer --clear-env -- env        # 清除环境变量
```

---

## 2. CLI 入口

**文件**：`src/lxc/tools/lxc_attach.c:320-456`

### 2.1 参数解析

```c
lxc_attach_main()                                [lxc_attach.c:320-321]
  └── lxc_arguments_parse(&my_args, argc, argv)
        └── my_parser() 处理特有选项           [lxc_attach.c:149-260]
```

| 选项 | 含义 |
|------|------|
| `-n <name>` | 容器名 |
| `-s <namespaces>` | 要进入的命名空间（`NETWORK\|MOUNT\|PID\|...`） |
| `-e` | 提升权限 |
| `-R` | 重新挂载 /sys 和 /proc |
| `--clear-env` | 清除环境变量 |
| `--keep-env` | 保持宿主机环境 |
| `--keep-var <var>` | 保留指定环境变量 |
| `--set-var <K=V>` | 设置环境变量 |
| `-u <uid>` | 指定运行 UID |
| `-g <gid>` | 指定运行 GID |
| `-c <context>` | SELinux 安全上下文 |
| `--` | 之后为要执行的命令 |

### 2.2 终端模式检测

```c
// lxc_attach.c:400-401
if (isatty(STDIN_FILENO) || isatty(STDOUT_FILENO) || isatty(STDERR_FILENO))
    // 启用终端模式，分配 PTY
```

### 2.3 命令分发

```c
// lxc_attach.c:432-444
if (有指定命令):
    c->attach_run_wait(c, &attach_options, command.program, command.argv)
else:
    c->attach(c, lxc_attach_run_shell, ...)   // 启动默认 shell
```

---

## 3. API 层

**文件**：`src/lxc/lxccontainer.c:3985-4035`

```
lxcapi_attach()                                  [lxccontainer.c:3985-4000]
  └── lxc_attach(c, exec_function, exec_payload, options, pid)

do_lxcapi_attach_run_wait()                      [lxccontainer.c:4002-4022]
  ├── lxc_attach(c, exec_function, exec_payload, options, &pid)
  └── lxc_wait_for_pid_status(pid)               // 等待附着进程退出
```

---

## 4. 核心实现：两次 Fork 设计

**文件**：`src/lxc/attach.c:1437-1824`

### 4.1 设计原理

为什么需要两次 fork？

1. **第一次 fork**：创建临时中间进程（transient process），用于进入目标命名空间
2. **第二次 fork + CLONE_PARENT**：创建实际的附着进程，使其父进程为原始调用者（而非中间进程）

```
原始进程（父）
  │
  ├── fork() → 中间进程（临时）
  │               ├── setns() 进入所有目标命名空间
  │               ├── fork(CLONE_PARENT) → 附着进程
  │               │                          ├── 安全设置
  │               │                          └── exec(目标命令)
  │               └── 发送附着进程 PID → 退出
  │
  └── waitpid(附着进程)
```

`CLONE_PARENT` 使附着进程的父进程指向原始进程而非中间进程，这样：
- 原始进程可以直接 waitpid 附着进程
- 中间进程可以安全退出
- 附着进程的 PPID 正确

### 4.2 完整流程

```
lxc_attach(c, exec_fn, exec_payload, options, pid)
  │                                              [attach.c:1437-1824]
  │
  ├─1─ 准备上下文
  │      get_attach_context(options)              [attach.c:470-494]
  │      → 获取容器 init PID、命名空间 fd、LSM 标签
  │
  ├─2─ 准备 seccomp
  │      fetch_seccomp(ctx)                       [attach.c:1059-1095]
  │      → 从运行中的容器获取 seccomp 配置
  │
  ├─3─ 创建 IPC socketpair
  │      socketpair(AF_UNIX, SOCK_DGRAM, 0, ipc_sockets)
  │                                              [attach.c:1503-1540]
  │
  ├─4─ 分配终端（如果需要）
  │      lxc_terminal_create(terminal)
  │                                              [attach.c:1491-1501]
  │
  ├─5─ 第一次 fork → 中间进程
  │      pid = fork()                             [attach.c:1542]
  │      │
  │      └── 中间进程：
  │            ├── 等待父进程的 cgroup 同步信号
  │            ├── 关闭敏感 fd
  │            ├── 进入命名空间（见 4.3）
  │            │
  │            ├── 第二次 fork + CLONE_PARENT     [attach.c:1567-1620]
  │            │     └── 附着进程（grandchild）：
  │            │           └── do_attach()         [attach.c:1621-1641]
  │            │
  │            ├── 通过 IPC 发送附着进程 PID 给父进程
  │            └── exit()                          [attach.c:1647-1663]
  │
  ├─6─ 父进程后续处理                              [attach.c:1674-1803]
  │      ├── 接收附着进程 PID
  │      ├── 将附着进程加入 cgroup
  │      ├── 处理 seccomp notify fd 交换
  │      ├── LSM 握手
  │      └── 启动终端主循环（如果有终端）
  │            或直接 waitpid
  │
  └─7─ 返回附着进程 PID
```

### 4.3 命名空间进入

**文件**：`src/lxc/attach.c:546-700`

两种实现路径：

#### pidfd 路径（现代内核）

```c
__prepare_namespaces_pidfd()                     [attach.c:546-571]
  └── setns(ctx->init_pidfd, ns_flags)
        // 一次性进入所有命名空间
```

#### nsfd 路径（传统方式）

```c
__prepare_namespaces_nsfd()                      [attach.c:573-615]
  └── for each namespace in ns_info[]:
        setns(ctx->ns_fd[i], ns_info[i].clone_flag)
        // 逐个进入各命名空间
```

命名空间类型（`ns_info[]` 数组）：

| 命名空间 | 标志 | 说明 |
|----------|------|------|
| user | CLONE_NEWUSER | 用户命名空间 |
| mount | CLONE_NEWNS | 挂载命名空间 |
| pid | CLONE_NEWPID | PID 命名空间 |
| uts | CLONE_NEWUTS | UTS 命名空间 |
| ipc | CLONE_NEWIPC | IPC 命名空间 |
| net | CLONE_NEWNET | 网络命名空间 |
| cgroup | CLONE_NEWCGROUP | Cgroup 命名空间 |
| time | CLONE_NEWTIME | 时间命名空间 |

分发入口 `attach_namespaces()`（`attach.c:680-700`）根据内核能力选择路径。

---

## 5. 附着进程设置：do_attach()

**文件**：`src/lxc/attach.c:1190-1420`

附着进程（grandchild）在新命名空间中执行以下设置：

```
do_attach(ap)                                    [attach.c:1190-1420]
  │
  ├─1─ 关闭不需要的 fd
  │
  ├─2─ 设置环境变量
  │      lxc_attach_set_environment()             [attach.c:797-909]
  │      ├── CLEAR_ENV 模式：清除所有 → 保留指定变量
  │      ├── 始终设置 container=lxc
  │      ├── 加载容器 environment / environment_runtime
  │      └── 应用 --set-var 的额外变量
  │
  ├─3─ 丢弃 Capabilities
  │      drop_capabilities()                      [attach.c:774-795]
  │      → prctl(PR_CAPBSET_DROP, cap) 逐个丢弃
  │                                              [attach.c:1208-1214]
  │
  ├─4─ 设置 LSM 安全标签
  │      ctx->lsm_ops->process_label_set(...)     [attach.c:1268-1279]
  │      → AppArmor: aa_change_profile()
  │      → SELinux: setexeccon()
  │
  ├─5─ 设置 no_new_privs
  │      prctl(PR_SET_NO_NEW_PRIVS, 1)            [attach.c:1281-1288]
  │
  ├─6─ 准备终端
  │      lxc_terminal_prepare_login(terminal_pts_fd)
  │                                              [attach.c:1333-1341]
  │      ├── lxc_make_controlling_terminal()      → ioctl(TIOCSCTTY)
  │      └── lxc_terminal_set_stdfds()            → dup2(fd, 0/1/2)
  │
  ├─7─ 设置 UID/GID
  │      setuid(uid) / setgid(gid)
  │
  ├─8─ 加载 Seccomp
  │      seccomp_load / notify fd 交换            [attach.c:1358-1368]
  │
  └─9─ 执行目标命令
         exec_function(exec_payload)
         → lxc_attach_run_command()              [attach.c:1827-1846]
           └── execvp(command->program, command->argv)
```

---

## 6. 环境变量处理

**文件**：`src/lxc/attach.c:797-909`

```
lxc_attach_set_environment(options)
  │
  ├── CLEAR_ENV 模式：
  │     ├── clearenv()                            // 清除所有环境变量
  │     ├── 保留 --keep-var 指定的变量
  │     ├── 设置 TERM, HOME, USER（从容器 /etc/passwd）
  │     └── 设置 PATH 默认值
  │
  ├── 始终设置：
  │     container=lxc                             // 标识在容器内
  │
  ├── 加载容器配置中的环境变量：
  │     lxc.environment = KEY=VALUE
  │     lxc.environment.runtime = KEY=VALUE
  │
  └── 应用用户指定的额外变量：
        --set-var KEY=VALUE
```

---

## 7. 终端转发

### 7.1 PTY 分配

```
lxc_terminal_create(terminal)                    [terminal.c:1103-1110]
  ├── openpty(&ptx, &pty, ...)
  └── 返回 ptx/pty fd 对
```

### 7.2 I/O 转发

父进程启动主循环进行双向 I/O 转发：

```
宿主机终端 (stdin/stdout)
       ↕ 读写
PTY 主端 (ptx fd)  ←→  主循环  ←→  PTY 从端 (pty fd)
                                          ↕
                                   附着进程 (stdin/stdout)
```

主循环使用 epoll/io_uring 监听两端，实现字符级实时转发。
SIGWINCH 信号用于同步终端窗口大小变化。

---

## 8. 安全上下文处理

### 8.1 LSM 标签

```
get_attach_context()                             [attach.c:470-494]
  └── 读取容器 init 进程的 LSM 标签
        /proc/<pid>/attr/current (AppArmor)
        /proc/<pid>/attr/current (SELinux)

do_attach()
  └── lsm_ops->process_label_set(label)
        AppArmor: aa_change_profile(label)
        SELinux:  setexeccon(label)
```

### 8.2 Capabilities

```
drop_capabilities(ctx)                           [attach.c:774-795]
  └── for cap in 0..CAP_LAST_CAP:
        if cap 不在保留列表:
            prctl(PR_CAPBSET_DROP, cap)
```

### 8.3 Seccomp

```
fetch_seccomp(ctx)                               [attach.c:1059-1095]
  └── 从容器的 seccomp notify fd 获取策略

do_attach()
  └── seccomp_load(filter)
      + 通过 IPC 交换 notify fd（如果使用 seccomp notify）
                                                 [attach.c:1358-1368, 1773-1780]
```

---

## 9. 完整生命周期

```
lxc-attach -n mycontainer -- /bin/bash

1. CLI 解析 → name=mycontainer, command=/bin/bash, 检测终端模式
                                                 [lxc_attach.c:321-456]

2. c->attach_run_wait(c, options, "/bin/bash", argv)
                                                 [lxccontainer.c:4002-4022]

3. lxc_attach() → 获取容器 PID/命名空间/LSM/seccomp
                                                 [attach.c:1471-1540]

4. fork() → 中间进程
     ├── setns() 进入所有命名空间                  [attach.c:680-700]
     └── fork(CLONE_PARENT) → 附着进程
                                                 [attach.c:1567-1620]

5. 附着进程 do_attach()：
     ├── 设置环境变量                              [attach.c:797-909]
     ├── 丢弃 capabilities                        [attach.c:774-795]
     ├── 设置 LSM 标签                             [attach.c:1268-1279]
     ├── 设置 no_new_privs                         [attach.c:1281-1288]
     ├── 准备终端（PTY → stdin/stdout/stderr）      [attach.c:1333-1341]
     └── execvp("/bin/bash", argv)                [attach.c:1827-1846]

6. 父进程：运行终端主循环，转发 I/O
                                                 [attach.c:1797-1803]

7. 用户看到 bash 提示符，在容器命名空间中操作
```
