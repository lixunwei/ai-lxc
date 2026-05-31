# PID Namespace 与 Init 进程管理深度分析

## 1. 概述

PID namespace 为容器提供独立的进程 ID 空间。容器内的第一个进程成为 PID 1（init），承担信号分发和孤儿进程收割的职责。LXC 提供两种 init 模式：系统 init（lxc-start）和包装器 init（lxc-execute）。

**核心源文件**：
- `src/lxc/start.c` — handler/monitor 进程、信号处理、主循环
- `src/lxc/initutils.c` — `lxc_container_init()` init 包装器实现
- `src/lxc/execute.c` — lxc-execute 模式入口
- `src/lxc/cmd/lxc_init.c` — lxc-init CLI 工具

---

## 2. 进程树总览

### 2.1 lxc-start 模式（系统容器）

```
lxc-start (CLI)
  |
  +-- [daemonize: double-fork]
  |
  +-- handler/monitor (宿主侧常驻进程)
        |  - signalfd 监听信号
        |  - epoll mainloop
        |  - 状态管理 (STARTING->RUNNING->STOPPED)
        |
        +-- clone(CLONE_NEWPID|...) --> 容器 init (PID 1)
                                          |
                                          +-- /sbin/init
                                          +-- 系统服务进程...
```

### 2.2 lxc-execute 模式（应用容器）

```
lxc-execute (CLI)
  |
  +-- handler/monitor
        |
        +-- clone(CLONE_NEWPID|...) --> lxc-init (PID 1, 包装器)
                                          |
                                          +-- fork() --> 用户命令 (PID 2)
                                          |
                                          +-- wait loop (信号转发 + 收割)
```

---

## 3. PID Namespace 创建

### 3.1 clone 标志构建

```c
// start.c:1697-1715
handler->ns_clone_flags |= CLONE_NEWPID;  // 如果配置了 PID namespace
handler->clone_flags = handler->ns_on_clone_flags | CLONE_PIDFD;
```

### 3.2 容器进程创建

```c
// start.c:1896-1933 (首选路径: clone3)
struct clone_args clone_args = {
    .flags      = handler->clone_flags,  // 包含 CLONE_NEWPID
    .pidfd      = &handler->pidfd,
    .exit_signal = SIGCHLD,
};
handler->pid = lxc_clone3(&clone_args, ...);

// 回退路径: lxc_raw_clone_cb()
handler->pid = lxc_raw_clone_cb(do_start, handler,
                                CLONE_PIDFD | flags, ...);
```

**关键点**：`CLONE_PIDFD` 与 `CLONE_NEWPID` 一起使用，让 handler 获得容器 init 的 pidfd，用于后续的安全信号发送。

---

## 4. 两种 Init 模式

### 4.1 lxc-start（系统 init）

容器直接执行自己的 `/sbin/init`（或用户指定的 init 程序）：

```c
// lxc_start.c:161-177
// 默认 args = {"/sbin/init", NULL}
c->start(c, 0, args);  // useinit=0 表示系统 init 模式
```

容器 init 作为 PID 1 直接运行，由容器的 init 系统（systemd/openrc/sysvinit）管理所有进程。

### 4.2 lxc-execute（应用 init 包装器）

LXC 提供 `lxc-init` 作为轻量级 PID 1：

```c
// initutils.c:482-700+ lxc_container_init()
lxc_container_init(argc, argv, quiet) {
    // 1. 屏蔽所有信号（SIGILL/SIGSEGV/SIGBUS 除外）
    sigfillset(&mask);
    sigdelset(&mask, SIGILL/SIGSEGV/SIGBUS);
    pthread_sigmask(SIG_SETMASK, &mask, &omask);

    // 2. 为所有信号安装 interrupt_handler
    for (i = 1; i < NSIG; i++)
        sigaction(i, &act, NULL);

    // 3. fork 子进程执行用户命令
    pid = fork();
    if (pid == 0) {
        // 子进程: 恢复默认信号处理
        signal(i, SIG_DFL);
        pthread_sigmask(SIG_SETMASK, &omask, NULL);
        setsid();
        ioctl(STDIN_FILENO, TIOCSCTTY, 0);
        execvp(argv[0], argv);  // 用户命令变为 PID 2
    }

    // 4. PID 1: 进入信号处理 + 收割循环
    for (;;) {
        // 处理信号 + waitpid 收割所有子进程
    }
}
```

### 4.3 模式对比

| 特性 | lxc-start | lxc-execute |
|------|-----------|-------------|
| PID 1 | 容器自己的 /sbin/init | lxc-init 包装器 |
| 用户命令 PID | 由 init 系统管理 | PID 2 |
| 信号转发 | init 自行处理 | lxc-init 负责转发 |
| 孤儿收割 | init 自行收割 | lxc-init wait() 循环 |
| 适用场景 | 完整系统容器 | 单应用容器 |
| 优雅关机 | init 响应 SIGPWR | lxc-init SIGTERM→SIGKILL |

---

## 5. 信号处理机制

### 5.1 Handler 进程的 signalfd

```c
// start.c:1044-1050
// handler 使用 signalfd 代替传统信号处理
sigfillset(&mask);
handler->sigfd = signalfd(-1, &mask, SFD_CLOEXEC);

// 加入 epoll mainloop
lxc_mainloop_add_handler(descr, handler->sigfd, signal_handler, handler);
```

### 5.2 信号转发规则

```c
// start.c:454-497 signal_handler()

// SIGHUP -> 转换为 SIGTERM 发送给容器 init
if (siginfo.ssi_signo == SIGHUP) {
    pidfd_send_signal(handler->pidfd, SIGTERM) 或 kill(pid, SIGTERM);
}

// 其他非 SIGCHLD 信号 -> 原样转发
if (siginfo.ssi_signo != SIGCHLD) {
    pidfd_send_signal(handler->pidfd, siginfo.ssi_signo);
}

// SIGCHLD -> 检查是否来自容器 init
if (siginfo.ssi_pid == handler->pid) {
    // 容器 init 退出 -> 关闭 mainloop
    return LXC_MAINLOOP_CLOSE;
} else {
    // 来自其他进程（如 transient），忽略
    return LXC_MAINLOOP_CONTINUE;
}
```

### 5.3 lxc-init 的信号转发

```c
// initutils.c:639-675 (lxc-init 的主循环)
switch (was_interrupted) {
    case SIGPWR:
    case SIGTERM:
        // 优雅关机: 先 SIGTERM 所有子进程
        shutdown = 1;
        prevent_forking();
        kill(-1, SIGTERM);  // 发给所有进程
        alarm(1);           // 1秒后超时
        break;

    case SIGALRM:
        // 超时: SIGKILL 所有子进程
        kill(-1, SIGKILL);
        break;

    default:
        // 其他信号: 转发给用户命令
        kill(pid, was_interrupted);
        break;
}
```

---

## 6. 容器 Init 退出检测

### 6.1 waitid + WNOWAIT

```c
// start.c:423-452
// 使用 WNOWAIT 非消费式检查 init 是否已退出
info.si_pid = 0;
waitid(P_PID, handler->pid, &info, WEXITED | WNOWAIT | WNOHANG);
if (info.si_pid == handler->pid)
    handler->init_died = true;

// 退出状态解析
switch (info.si_code) {
    case CLD_EXITED:    // 正常退出
        exit_status = info.si_status << 8;
    case CLD_KILLED:    // 被信号杀死
    case CLD_DUMPED:    // core dump
        exit_status = info.si_status << 8 | 0x7f;
}
```

**WNOWAIT 的意义**：不消费 wait 状态，允许后续再次 `waitpid()` 获取退出码。这对于在 mainloop 中多次检查状态很重要。

---

## 7. 容器停止流程

### 7.1 优雅关机（lxc-stop / shutdown）

```
lxc-stop -n myct
  |
  +-> c->shutdown(c, timeout)
        |
        +-> 发送 SIGPWR (默认 haltsignal)
        |   (systemd 收到 SIGPWR 会启动 poweroff.target)
        |
        +-> 等待容器进入 STOPPED 状态
        |   - 轮询 pidfd 或使用 state client socket
        |
        +-> 超时未停止:
              +-> c->stop(c)   // 发送 SIGKILL
```

### 7.2 强制停止

```c
// lxccontainer.c:2091-2117
lxcapi_stop(c) {
    // 通过 lxc_cmd 向 handler 发送 stop 命令
    // handler 收到后 kill(init_pid, SIGKILL)
}
```

### 7.3 关机信号配置

```
# 默认: SIGPWR
lxc.signal.halt = SIGPWR

# 可自定义为 SIGRTMIN+3 等
lxc.signal.halt = SIGRTMIN+3

# reboot 信号
lxc.signal.reboot = SIGINT

# stop 信号 (强制)
lxc.signal.stop = SIGKILL
```

---

## 8. 守护化（Daemonize）

### 8.1 双 fork 模式

```c
// lxccontainer.c:918-1000
if (c->daemonize) {
    // 第一次 fork
    pid_t pid1 = fork();
    if (pid1 > 0) {
        // 原始进程: 等待启动确认
        wait_on_state_socket(state_fd);
        exit(0);
    }

    // 第二次 fork
    pid_t pid2 = fork();
    if (pid2 > 0)
        _exit(0);  // 第一个子进程退出

    // 第二个子进程: 成为守护进程
    setsid();
    chdir("/");
    // 关闭/重定向 stdio
    close(0); close(1); close(2);
    open("/dev/null", O_RDWR);
    dup2(0, 1); dup2(0, 2);
}
```

### 8.2 启动同步

```c
// start.c:913-936
// handler 创建 state_socket_pair
// 当容器进入 RUNNING 状态时，通知等待的父进程
lxc_serve_state_clients(name, handler, RUNNING);
```

---

## 9. 状态机

```
                  +----------+
                  | STOPPED  |<---------+
                  +----------+          |
                       |                |
                  lxc-start             |
                       |                |
                       v                |
                  +----------+          |
                  | STARTING |          |
                  +----------+          |
                       |                |
              setup complete            |
                       |                |
                       v                |
                  +----------+     +----------+
                  | RUNNING  |---->| STOPPING |
                  +----------+     +----------+
                       |                |
                  error/kill            |
                       |                |
                       v                |
                  +----------+          |
                  | ABORTING |---------+
                  +----------+

                  +----------+     +----------+
                  | FREEZING |---->|  FROZEN  |
                  +----------+     +----------+
                       ^                |
                       |           unfreeze
                       |                |
                       +---  THAWED <---+
```

状态转换通过 `lxc_set_state()` 触发，同时通过 FIFO 通知 lxc-monitord。

---

## 10. Handler 进程 Mainloop

```c
// start.c:763-821 lxc_poll()
lxc_mainloop_open(&descr);

// 事件源 1: 信号 fd
lxc_mainloop_add_handler(&descr, handler->sigfd, signal_handler);

// 事件源 2: 控制台 I/O
if (console)
    lxc_terminal_mainloop_add(&descr, &handler->conf->console);

// 事件源 3: 命令 socket (lxc-info, lxc-stop 等通信)
lxc_cmd_mainloop_add(name, &descr, handler);

// 事件源 4: seccomp 代理 (如果启用)
if (seccomp_notify_fd)
    lxc_mainloop_add_handler(&descr, seccomp_fd, ...);

// 运行 mainloop 直到 init 退出
lxc_mainloop(&descr, -1);

// init 退出后清理
lxc_mainloop_close(&descr);
```

---

## 11. 孤儿收割

### 11.1 系统 init 模式

容器内的 `/sbin/init`（如 systemd）作为 PID 1 自动收割所有孤儿进程。这是 Linux 内核的 PID namespace 语义：当父进程退出时，孤儿进程被 reparent 到 namespace 的 init。

### 11.2 lxc-init 包装器模式

```c
// initutils.c:635-700
for (;;) {
    waited_pid = wait(&status);  // 收割任何子进程

    if (waited_pid < 0) {
        if (errno == ECHILD) {
            // 没有更多子进程
            if (shutdown)
                exit(exit_with);  // 关机完成
            // 否则继续等待（可能有新的孤儿）
        }
        continue;
    }

    if (waited_pid == pid) {
        // 用户命令退出
        have_status = 1;
        exit_with = WEXITSTATUS(status);
        if (!shutdown)
            shutdown = 1;
            kill(-1, SIGTERM);  // 通知所有子进程
            alarm(1);
    }
}
```

---

## 12. PID Namespace 嵌套

LXC 支持嵌套容器（container-in-container）。嵌套时：

- 外层容器的 init 是内层容器 handler 的 PID namespace 的 parent
- 内层 `CLONE_NEWPID` 创建新的 PID namespace 层
- 内层 PID 1 独立于外层
- 信号只能从父 PID namespace 向子 namespace 发送，反向不可
