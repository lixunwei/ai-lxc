# LXC Monitor 监控机制

## 1. 概述

LXC monitor 子系统提供了一套**发布-订阅 (pub/sub)** 机制，允许外部工具实时监听容器的状态变化和退出码。整体架构由三个组件构成：

```
┌─────────────┐     FIFO       ┌──────────────┐   Abstract Unix Socket   ┌─────────────┐
│  容器进程    │ ──────────────→ │ lxc-monitord │ ─────────────────────────→│ lxc-monitor │
│ (生产者)    │  写入 lxc_msg  │  (守护进程)   │    广播到所有订阅者      │  (消费者)   │
└─────────────┘                └──────────────┘                           └─────────────┘
```

| 组件 | 角色 | 源文件 |
|------|------|--------|
| 容器进程 | 状态变化时写入 FIFO | `src/lxc/monitor.c` |
| lxc-monitord | 转发守护进程（epoll 事件循环） | `src/lxc/cmd/lxc_monitord.c` |
| lxc-monitor | 用户空间命令行订阅工具 | `src/lxc/tools/lxc_monitor.c` |

---

## 2. 消息协议

### 2.1 消息结构体

```c
// src/lxc/monitor.h
typedef enum {
    lxc_msg_state,      // 状态变化通知
    lxc_msg_priority,   // 优先级（保留，未使用）
    lxc_msg_exit_code,  // 容器退出码
} lxc_msg_type_t;

struct lxc_msg {
    lxc_msg_type_t type;
    char name[NAME_MAX + 1];  // 容器名称
    int value;                // 状态枚举值 或 exit status
};
```

### 2.2 消息类型

| 类型 | 触发场景 | value 含义 |
|------|----------|-----------|
| `lxc_msg_state` | 容器进入新状态 | `lxc_state_t` 枚举 (STOPPED/STARTING/RUNNING/...) |
| `lxc_msg_exit_code` | 容器 init 进程退出 | `waitpid()` 返回的 status（需 `WEXITSTATUS()` 解析）|
| `lxc_msg_priority` | 保留 | 未使用 |

### 2.3 消息发送 API

```c
// 容器状态变化时调用（在 lxc_set_state() 中触发）
void lxc_monitor_send_state(const char *name, lxc_state_t state, const char *lxcpath);

// 容器退出时调用
void lxc_monitor_send_exit_code(const char *name, int exit_code, const char *lxcpath);
```

这两个函数内部调用 `lxc_monitor_fifo_send()`，以非阻塞方式写入 FIFO：
- 使用 `O_WRONLY | O_NONBLOCK` 打开 FIFO
- 如果没有 monitord 在运行（ENXIO/ENOENT），静默忽略
- 消息大小满足 `sizeof(lxc_msg) <= PIPE_BUF`，保证原子写入

---

## 3. lxc-monitord 守护进程

### 3.1 职责

lxc-monitord 是一个**轻量级事件转发守护进程**：
1. 从 FIFO 读取容器发来的 `lxc_msg`
2. 通过 abstract unix socket 广播给所有已连接的 lxc-monitor 客户端
3. 管理客户端连接的建立和断开
4. 无客户端时自动超时退出（30 秒）

### 3.2 数据结构

```c
// src/lxc/cmd/lxc_monitord.c
struct lxc_monitor {
    const char *lxcpath;        // 监控的 lxcpath
    int fifofd;                 // FIFO 文件描述符（接收容器消息）
    int listenfd;               // 监听 socket（接受客户端连接）
    int *clientfds;             // 已连接客户端 fd 数组
    int clientfds_size;         // 数组容量（按 CLIENTFDS_CHUNK=64 增长）
    int clientfds_cnt;          // 当前连接数
    struct lxc_async_descr descr; // epoll 事件循环
};
```

### 3.3 通信通道

| 通道 | 路径 | 用途 |
|------|------|------|
| FIFO | `$rundir/lxc/$lxcpath/monitor-fifo` | 容器 → monitord |
| Abstract Unix Socket | `@lxc/{hash}/{lxcpath}/monitor-sock` | monitord → 客户端 |

**FIFO 互斥**：使用 `fcntl(F_SETLK)` 文件锁确保同一 lxcpath 只有一个 monitord 实例运行。

**Socket 命名**：使用 FNV-1a 64-bit 哈希处理长路径名，确保不超过 `sun_path` 的 105 字节限制。

### 3.4 事件循环

```
┌─────────────────────────────────────────────┐
│               epoll 主循环                   │
│                                             │
│  fifofd  → lxc_monitord_fifo_handler()      │
│           读取 lxc_msg，广播给所有 clientfds │
│                                             │
│  listenfd → lxc_monitord_sock_accept()      │
│           accept4() 新连接，验证 uid 权限   │
│                                             │
│  clientfd → lxc_monitord_sock_handler()     │
│           处理 "quit" 命令 或 EPOLLHUP 断连 │
└─────────────────────────────────────────────┘
```

### 3.5 消息转发逻辑

```c
static int lxc_monitord_fifo_handler(int fd, uint32_t events, void *data,
                                     struct lxc_async_descr *descr)
{
    struct lxc_msg msglxc;
    ret = lxc_read_nointr(fd, &msglxc, sizeof(msglxc));
    // 广播给所有已连接客户端
    for (i = 0; i < mon->clientfds_cnt; i++)
        lxc_write_nointr(mon->clientfds[i], &msglxc, sizeof(msglxc));
    return LXC_MAINLOOP_CONTINUE;
}
```

### 3.6 权限验证

```c
// 只允许 root 或同 euid 的用户连接
if (cred.uid && cred.uid != geteuid()) {
    WARN("Monitor denied for uid %d");
    goto err1;
}
```

### 3.7 生命周期管理

- **启动**：通过双 fork + setsid() 脱离终端，避免僵尸进程
- **同步**：通过 pipe 向父进程确认 socket 已就绪
- **超时退出**：非持久模式下 mainloop 超时 30 秒，无客户端则退出
- **持久模式**：`--daemon` 参数，mainloop 永不超时（systemd 服务模式）
- **信号处理**：SIGTERM/SIGILL/SIGSEGV/SIGBUS → siglongjmp 到清理路径
- **退出命令**：客户端发送 "quit" 字符串触发退出

```c
// 退出判断（主循环）
for (;;) {
    ret = lxc_mainloop(&monitor.descr, persistent ? -1 : 1000 * 30);
    if (monitor.clientfds_cnt <= 0) break;  // 无客户端
    if (quit == LXC_MAINLOOP_CLOSE) break;  // 收到 quit
}
```

---

## 4. lxc-monitor 客户端工具

### 4.1 功能

`lxc-monitor` 是用户空间的订阅工具，连接 monitord 的 unix socket 并实时打印容器事件。

### 4.2 命令行参数

```
lxc-monitor [--name=NAME] [-Q|--quit]

  -n, --name=NAME   容器名称（支持正则表达式，默认 ".*" 匹配所有）
  -Q, --quit        通知 lxc-monitord 退出
```

### 4.3 工作流程

```
1. 解析参数
2. 编译 name 为正则表达式 (regcomp)
3. 对每个 lxcpath:
   a. 启动/确认 lxc-monitord 运行
   b. lxc_monitor_open() 连接 abstract unix socket
4. 进入主循环:
   a. poll() 等待消息
   b. 读取 struct lxc_msg
   c. regexec() 过滤容器名
   d. 打印状态变化或退出码
```

### 4.4 输出格式

```
'mycontainer' changed state to [RUNNING]
'mycontainer' changed state to [STOPPING]
'mycontainer' changed state to [STOPPED]
'mycontainer' exited with status [0]
```

### 4.5 多 lxcpath 支持

可以同时监控多个 lxcpath（通过 `-P` 多次指定），为每个 path 创建独立的 pollfd：

```c
nfds = my_args.lxcpath_cnt;
for (i = 0; i < nfds; i++) {
    lxc_tool_monitord_spawn(my_args.lxcpath[i]);
    fds[i].fd = lxc_monitor_open(my_args.lxcpath[i]);
}
```

---

## 5. 启动时序图

```
lxc-monitor                    lxc-monitord                     容器进程
    │                               │                              │
    │── fork()/fork() ──────────────│                              │
    │                        setsid()                              │
    │                        创建 FIFO                             │
    │                        创建 socket                           │
    │◄── pipe 同步 ─────────────────│                              │
    │                               │                              │
    │── connect(socket) ───────────→│                              │
    │                        accept4() + 权限检查                  │
    │                               │                              │
    │                               │◄── FIFO write ───────────── │ lxc_monitor_send_state()
    │                               │    (lxc_msg: state=RUNNING)  │
    │                               │                              │
    │◄── socket write ──────────────│                              │
    │   (广播 lxc_msg)              │                              │
    │                               │                              │
    │ printf("changed state")       │                              │
    │                               │                              │
```

---

## 6. 与容器启动引擎的集成

monitor 消息在 `lxc_set_state()` 中自动触发：

```c
// src/lxc/state.c (简化)
int lxc_set_state(const char *name, struct lxc_handler *handler, lxc_state_t state)
{
    handler->state = state;
    lxc_monitor_send_state(name, state, handler->lxcpath);
    // 同时通知 state_socket_pair 等待者
    lxc_cmd_serve_state_clients(name, handler);
    return 0;
}
```

退出码在 `lxc_poll()` 中容器退出时发送：

```c
lxc_monitor_send_exit_code(name, handler->exit_status, handler->lxcpath);
```

---

## 7. Systemd 集成

LXC 提供 systemd 服务文件 `lxc-monitord.service`，以 `--daemon` 模式运行 monitord：

```ini
# config/init/systemd/lxc-monitord.service.in
[Service]
ExecStart=@LIBEXECDIR@/lxc/lxc-monitord --daemon @LXCPATH@
```

持久模式下 mainloop 超时设为 -1（永不超时），即使无客户端也不退出。

---

## 8. 与 lxc_cmd 通道的区别

LXC 有两套 IPC 机制，用途不同：

| 特性 | Monitor (FIFO + Socket) | Command (Unix Socket) |
|------|-------------------------|----------------------|
| 方向 | 容器 → 外部观察者 | 外部 → 容器 handler |
| 协议 | 简单广播 struct lxc_msg | 请求-响应 lxc_cmd_req/rsp |
| 守护进程 | lxc-monitord 中转 | 容器 handler 直接服务 |
| 用途 | 状态监听、退出码 | 获取状态/PID/cgroup/配置 |
| 连接方式 | 多对多 (pub/sub) | 一对一 (client/server) |
| 订阅者 | lxc-monitor 工具 | lxc-info/lxc-stop/API |

---

## 9. 使用场景

1. **容器编排**：自动化脚本监听容器状态，触发后续操作
2. **运维监控**：实时感知容器崩溃、重启事件
3. **调试**：观察容器生命周期各阶段的状态转换
4. **集成测试**：等待容器进入特定状态后执行断言

### 典型用法

```bash
# 监控所有容器
lxc-monitor -n ".*"

# 监控特定前缀的容器
lxc-monitor -n "^web-.*$"

# 监控多个 lxcpath
lxc-monitor -n ".*" -P /var/lib/lxc -P /home/user/.local/share/lxc

# 让 monitord 退出
lxc-monitor -Q
```

### 编程接口

```c
#include <lxc/lxccontainer.h>

// 打开监控连接
int fd = lxc_monitor_open("/var/lib/lxc");

// 阻塞读取下一个事件（带超时）
struct lxc_msg msg;
int ret = lxc_monitor_read_timeout(fd, &msg, 30);

if (ret > 0) {
    printf("Container %s: state=%s\n", msg.name, lxc_state2str(msg.value));
}

close(fd);
```

---

## 10. 关键设计决策

| 决策 | 原因 |
|------|------|
| FIFO 而非 socket 做生产者通道 | 容器侧无需等待消费者就绪，O_NONBLOCK 即可 |
| 双 fork 启动 monitord | 避免僵尸进程，完全脱离控制终端 |
| Abstract unix socket | 无需文件系统路径管理，自动清理 |
| FNV-1a 哈希路径 | 解决 sun_path 105 字节长度限制 |
| PIPE_BUF 原子写 | BUILD_BUG_ON 保证 sizeof(lxc_msg) <= PIPE_BUF |
| 30 秒超时退出 | 非持久模式下避免无用守护进程残留 |
| SO_PEERCRED 权限检查 | 只允许同 uid 或 root 连接 |
