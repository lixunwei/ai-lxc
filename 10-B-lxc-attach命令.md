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
  `-- lxc_arguments_parse(&my_args, argc, argv)
      `-- my_parser() handles attach-specific options [lxc_attach.c:149-260]
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
  `-- lxc_attach(c, exec_function, exec_payload, options, pid)

do_lxcapi_attach_run_wait()                      [lxccontainer.c:4002-4022]
  +-- lxc_attach(c, exec_function, exec_payload, options, &pid)
  `-- lxc_wait_for_pid_status(pid)               // wait for attached process exit
```

---

## 4. 核心实现：两次 Fork 设计

**文件**：`src/lxc/attach.c:1437-1824`

### 4.1 设计原理

为什么需要两次 fork？

1. **第一次 fork**：创建临时中间进程（transient process），用于进入目标命名空间
2. **第二次 fork + CLONE_PARENT**：创建实际的附着进程，使其父进程为原始调用者（而非中间进程）

```
original process (parent)
  |
  +-- fork() -> intermediate process (transient)
  |              +-- setns() into all target namespaces
  |              +-- fork(CLONE_PARENT) -> attached process
  |              |                        +-- security setup
  |              |                        `-- exec(target command)
  |              `-- send attached PID -> exit
  |
  `-- waitpid(attached process)
```

`CLONE_PARENT` 使附着进程的父进程指向原始进程而非中间进程，这样：
- 原始进程可以直接 waitpid 附着进程
- 中间进程可以安全退出
- 附着进程的 PPID 正确

### 4.2 完整流程

```
lxc_attach(c, exec_fn, exec_payload, options, pid)
  |                                              [attach.c:1437-1824]
  |
  +--1-- prepare context
  |      get_attach_context(options)              [attach.c:470-494]
  |      -> get container init PID, namespace fds, LSM label
  |
  +--2-- prepare seccomp
  |      fetch_seccomp(ctx)                       [attach.c:1059-1095]
  |      -> get seccomp config from the running container
  |
  +--3-- create IPC socketpair
  |      socketpair(AF_UNIX, SOCK_DGRAM, 0, ipc_sockets)
  |                                              [attach.c:1503-1540]
  |
  +--4-- allocate terminal (if needed)
  |      lxc_terminal_create(terminal)
  |                                              [attach.c:1491-1501]
  |
  +--5-- first fork -> intermediate process
  |      pid = fork()                             [attach.c:1542]
  |      |
  |      `-- intermediate process:
  |            +-- wait for parent's cgroup sync signal
  |            +-- close sensitive fds
  |            +-- enter namespaces (see 4.3)
  |            |
  |            +-- second fork + CLONE_PARENT    [attach.c:1567-1620]
  |            |   `-- attached process (grandchild):
  |            |       `-- do_attach()            [attach.c:1621-1641]
  |            |
  |            +-- send attached PID to parent over IPC
  |            `-- exit()                         [attach.c:1647-1663]
  |
  +--6-- parent follow-up                         [attach.c:1674-1803]
  |      +-- receive attached PID
  |      +-- add attached process to cgroup
  |      +-- handle seccomp notify fd exchange
  |      +-- LSM handshake
  |      `-- start terminal main loop (if terminal exists)
  |          or waitpid directly
  |
  `--7-- return attached PID
```

### 4.3 命名空间进入

**文件**：`src/lxc/attach.c:546-700`

两种实现路径：

#### pidfd 路径（现代内核）

```c
__prepare_namespaces_pidfd()                     [attach.c:546-571]
  `-- setns(ctx->init_pidfd, ns_flags)
        // Enter all namespaces at once
```

#### nsfd 路径（传统方式）

```c
__prepare_namespaces_nsfd()                      [attach.c:573-615]
  `-- for each namespace in ns_info[]:
        setns(ctx->ns_fd[i], ns_info[i].clone_flag)
        // Enter namespaces one by one
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
  |
  +--1-- close unneeded fds
  |
  +--2-- set environment
  |      lxc_attach_set_environment()             [attach.c:797-909]
  |      +-- CLEAR_ENV mode: clear all -> keep selected vars
  |      +-- always set container=lxc
  |      +-- load container environment / environment_runtime
  |      `-- apply extra vars from --set-var
  |
  +--3-- drop capabilities
  |      drop_capabilities()                      [attach.c:774-795]
  |      -> prctl(PR_CAPBSET_DROP, cap) one by one
  |                                              [attach.c:1208-1214]
  |
  +--4-- set LSM security label
  |      ctx->lsm_ops->process_label_set(...)     [attach.c:1268-1279]
  |      -> AppArmor: aa_change_profile()
  |      -> SELinux: setexeccon()
  |
  +--5-- set no_new_privs
  |      prctl(PR_SET_NO_NEW_PRIVS, 1)            [attach.c:1281-1288]
  |
  +--6-- prepare terminal
  |      lxc_terminal_prepare_login(terminal_pts_fd)
  |                                              [attach.c:1333-1341]
  |      +-- lxc_make_controlling_terminal()      -> ioctl(TIOCSCTTY)
  |      `-- lxc_terminal_set_stdfds()            -> dup2(fd, 0/1/2)
  |
  +--7-- set UID/GID
  |      setuid(uid) / setgid(gid)
  |
  +--8-- load seccomp
  |      seccomp_load / notify fd exchange        [attach.c:1358-1368]
  |
  `--9-- exec target command
         exec_function(exec_payload)
         -> lxc_attach_run_command()              [attach.c:1827-1846]
            `-- execvp(command->program, command->argv)
```

---

## 6. 环境变量处理

**文件**：`src/lxc/attach.c:797-909`

```
lxc_attach_set_environment(options)
  |
  +-- CLEAR_ENV mode:
  |   +-- clearenv()                              // clear all env vars
  |   +-- keep vars requested by --keep-var
  |   +-- set TERM, HOME, USER (from container /etc/passwd)
  |   `-- set default PATH
  |
  +-- always set:
  |   container=lxc                               // mark execution as inside container
  |
  +-- load env vars from container config:
  |   lxc.environment = KEY=VALUE
  |   lxc.environment.runtime = KEY=VALUE
  |
  `-- apply extra vars from the user:
      --set-var KEY=VALUE
```

---

## 7. 终端转发

### 7.1 PTY 分配

```
lxc_terminal_create(terminal)                    [terminal.c:1103-1110]
  +-- openpty(&ptx, &pty, ...)
  `-- return ptx/pty fd pair
```

### 7.2 I/O 转发

父进程启动主循环进行双向 I/O 转发：

```
host terminal (stdin/stdout)
       ^  I/O  v
PTY master (ptx fd)  <->  main loop  <->  PTY slave (pty fd)
                                               ^
                                               |
                                               v
                                        attached process (stdin/stdout)
```

主循环使用 epoll/io_uring 监听两端，实现字符级实时转发。
SIGWINCH 信号用于同步终端窗口大小变化。

---

## 8. 安全上下文处理

### 8.1 LSM 标签

```
get_attach_context()                             [attach.c:470-494]
  `-- read the LSM label of the container init process
      /proc/<pid>/attr/current (AppArmor)
      /proc/<pid>/attr/current (SELinux)

do_attach()
  `-- lsm_ops->process_label_set(label)
      AppArmor: aa_change_profile(label)
      SELinux:  setexeccon(label)
```

### 8.2 Capabilities

```
drop_capabilities(ctx)                           [attach.c:774-795]
  `-- for cap in 0..CAP_LAST_CAP:
      if cap is not in the keep list:
          prctl(PR_CAPBSET_DROP, cap)
```

### 8.3 Seccomp

```
fetch_seccomp(ctx)                               [attach.c:1059-1095]
  `-- get policy from the container seccomp notify fd

do_attach()
  `-- seccomp_load(filter)
      + exchange notify fd over IPC (if seccomp notify is in use)
                                                [attach.c:1358-1368, 1773-1780]
```

---

## 9. 完整生命周期

```
lxc-attach -n mycontainer -- /bin/bash

1. CLI parse -> name=mycontainer, command=/bin/bash, detect terminal mode
                                                 [lxc_attach.c:321-456]

2. c->attach_run_wait(c, options, "/bin/bash", argv)
                                                 [lxccontainer.c:4002-4022]

3. lxc_attach() -> get container PID / namespaces / LSM / seccomp
                                                 [attach.c:1471-1540]

4. fork() -> intermediate process
     +-- setns() into all namespaces             [attach.c:680-700]
     `-- fork(CLONE_PARENT) -> attached process
                                                 [attach.c:1567-1620]

5. Attached process do_attach():
     +-- set environment                         [attach.c:797-909]
     +-- drop capabilities                       [attach.c:774-795]
     +-- set LSM label                           [attach.c:1268-1279]
     +-- set no_new_privs                        [attach.c:1281-1288]
     +-- prepare terminal (PTY -> stdin/stdout/stderr)
                                                [attach.c:1333-1341]
     `-- execvp("/bin/bash", argv)              [attach.c:1827-1846]

6. Parent process: run terminal main loop and forward I/O
                                                 [attach.c:1797-1803]

7. User sees the bash prompt and works inside container namespaces
```
