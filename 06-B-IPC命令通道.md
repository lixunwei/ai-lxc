# LXC 基础设施深度分析（二）：IPC 命令通道

## 1. 概述

LXC 使用 **Unix 域抽象套接字** 实现容器监控进程（monitor）与客户端之间的 IPC 通信。客户端（如 `lxc-info`、`lxc-stop`、`lxccontainer.c` API）通过命令通道查询容器状态、获取 cgroup 信息、控制容器行为。

核心文件：
- `src/lxc/commands.h`（~180 行）— 协议定义
- `src/lxc/commands.c`（~2200 行）— 服务端/客户端实现
- `src/lxc/commands_utils.c/h` — 工具函数

---

## 2. 协议设计

### 2.1 命令类型枚举

`commands.h:28-57`，`lxc_cmd_t`：

```c
enum lxc_cmd_t {
    LXC_CMD_GET_TTY_FD,                 // 获取终端 fd
    LXC_CMD_GET_INIT_PID,               // 获取 init PID
    LXC_CMD_GET_INIT_PIDFD,             // 获取 init pidfd
    LXC_CMD_GET_DEVPTS_FD,              // 获取 devpts fd
    LXC_CMD_GET_SECCOMP_NOTIFY_FD,      // 获取 seccomp notify fd
    LXC_CMD_GET_STATE,                  // 查询容器状态
    LXC_CMD_STOP,                       // 停止容器
    LXC_CMD_GET_CLONE_FLAGS,            // 获取 clone 标志
    LXC_CMD_GET_CGROUP,                 // 获取 cgroup 路径
    LXC_CMD_GET_LIMIT_CGROUP,           // 获取 limit cgroup 路径
    LXC_CMD_GET_CONFIG_ITEM,            // 获取配置项
    LXC_CMD_GET_NAME,                   // 获取容器名称
    LXC_CMD_GET_LXCPATH,               // 获取 LXC 路径
    LXC_CMD_ADD_STATE_CLIENT,           // 注册状态监听
    LXC_CMD_CONSOLE_LOG,               // 获取控制台日志
    LXC_CMD_SERVE_STATE_CLIENTS,        // 通知状态监听者
    LXC_CMD_SECCOMP_NOTIFY_ADD_LISTENER,// 添加 seccomp 监听器
    LXC_CMD_ADD_BPF_DEVICE_CGROUP,      // 添加 BPF 设备规则
    LXC_CMD_FREEZE,                     // 冻结容器
    LXC_CMD_UNFREEZE,                   // 解冻容器
    LXC_CMD_GET_CGROUP2_FD,             // 获取 cgroup2 fd
    LXC_CMD_GET_LIMIT_CGROUP2_FD,       // 获取 limit cgroup2 fd
    LXC_CMD_GET_CGROUP_FD,              // 获取 cgroup fd
    LXC_CMD_GET_LIMIT_CGROUP_FD,        // 获取 limit cgroup fd
    LXC_CMD_GET_CGROUP_CTX,             // 获取 cgroup 上下文
    LXC_CMD_GET_SYSTEMD_SCOPE,          // 获取 systemd scope
    LXC_CMD_MAX,
};
```

### 2.2 消息结构

`commands.h:59-76`：

```c
/* 请求消息 */
struct lxc_cmd_req {
    lxc_cmd_t cmd;          // 命令类型
    size_t datalen;         // 数据长度
    const void *data;       // 附加数据（可选）
};

/* 响应消息 */
struct lxc_cmd_rsp {
    int ret;                // 返回值
    size_t datalen;         // 数据长度
    void *data;             // 响应数据（可选）
};
```

### 2.3 特殊响应码

`commands.h:20-26`：

```c
#define LXC_CMD_REAP_CLIENT_FD   1   // 处理完毕，关闭客户端 fd
#define LXC_CMD_KEEP_CLIENT_FD   2   // 保持客户端连接（用于状态监听）
```

---

## 3. 通信架构

```
+----------------+                    +--------------------+
| Client process |                    | Container monitor  |
|                |                    | process            |
| lxc-info       |   Unix socket      | Event loop         |
| lxc-stop       |<------------------>|   |                |
| lxccontainer   |   (abstract)       | lxc_cmd_accept()  |
|                |                    |   |                |
| lxc_cmd() -----+---- send req ----->| lxc_cmd_process() |
|                |                    |   |                |
| <--------------+--- return rsp -----| handler()         |
+----------------+                    +--------------------+
```

### 套接字路径

`commands_utils.c:75-144`：

```c
lxc_make_abstract_socket_name(name, lxcpath, suffix) {
    // 生成抽象套接字名：@lxcpath/name/suffix
    // 例如：@/var/lib/lxc/mycontainer/command
    // 使用抽象命名空间，不在文件系统中创建文件
}
```

---

## 4. 服务端实现

### 4.1 套接字创建

`commands.c:2120-2139`，`lxc_server_init()`：

```c
lxc_server_init(name, lxcpath, suffix) {
    // 生成抽象套接字名
    path = lxc_make_abstract_socket_name(name, lxcpath, "command");

    // 创建 SOCK_STREAM 套接字
    fd = lxc_abstract_unix_open(path, SOCK_STREAM, 0);

    return fd;
}
```

在 `start.c:939` 中调用：
```c
handler->conf->maincmd_fd = lxc_server_init(name, lxcpath, "command");
```

### 4.2 事件循环注册

`commands.c:2141-2154`，`lxc_cmd_mainloop_add()`：

```c
lxc_cmd_mainloop_add(name, descr, handler) {
    // 将命令套接字注册到事件循环
    // 当有新连接到来时触发 lxc_cmd_accept()
    lxc_mainloop_add_handler(descr, handler->conf->maincmd_fd,
                             lxc_cmd_accept, handler);
}
```

### 4.3 连接接受

`commands.c:2094-2118`，`lxc_cmd_accept()`：

```c
lxc_cmd_accept(fd, handler, descr, events) {
    // 接受新连接
    client_fd = accept4(fd, NULL, NULL, SOCK_CLOEXEC);

    // 为客户端连接注册独立的 handler
    // 当客户端发送命令时触发 lxc_cmd_handler()
    lxc_mainloop_add_handler(descr, client_fd,
                             lxc_cmd_handler, handler);

    return LXC_MAINLOOP_CONTINUE;
}
```

### 4.4 命令分发

`commands.c:1936-1977`，`lxc_cmd_process()`：

```c
// 命令分发表
static int (*const cmd_handlers[])(struct lxc_cmd_req *, struct lxc_handler *,
                                   struct lxc_async_descr *, int client_fd) = {
    [LXC_CMD_GET_TTY_FD]                = lxc_cmd_get_tty_fd_callback,
    [LXC_CMD_GET_INIT_PID]              = lxc_cmd_get_init_pid_callback,
    [LXC_CMD_GET_INIT_PIDFD]            = lxc_cmd_get_init_pidfd_callback,
    [LXC_CMD_GET_DEVPTS_FD]             = lxc_cmd_get_devpts_fd_callback,
    [LXC_CMD_GET_SECCOMP_NOTIFY_FD]     = lxc_cmd_get_seccomp_notify_fd_callback,
    [LXC_CMD_GET_STATE]                 = lxc_cmd_get_state_callback,
    [LXC_CMD_STOP]                      = lxc_cmd_stop_callback,
    [LXC_CMD_GET_CLONE_FLAGS]           = lxc_cmd_get_clone_flags_callback,
    [LXC_CMD_GET_CGROUP]                = lxc_cmd_get_cgroup_callback,
    [LXC_CMD_GET_LIMIT_CGROUP]          = lxc_cmd_get_limit_cgroup_callback,
    [LXC_CMD_GET_CONFIG_ITEM]           = lxc_cmd_get_config_item_callback,
    [LXC_CMD_GET_NAME]                  = lxc_cmd_get_name_callback,
    [LXC_CMD_GET_LXCPATH]               = lxc_cmd_get_lxcpath_callback,
    [LXC_CMD_ADD_STATE_CLIENT]          = lxc_cmd_add_state_client_callback,
    [LXC_CMD_CONSOLE_LOG]               = lxc_cmd_console_log_callback,
    [LXC_CMD_SERVE_STATE_CLIENTS]       = lxc_cmd_serve_state_clients_callback,
    [LXC_CMD_SECCOMP_NOTIFY_ADD_LISTENER] = lxc_cmd_seccomp_notify_add_listener_callback,
    [LXC_CMD_ADD_BPF_DEVICE_CGROUP]     = lxc_cmd_add_bpf_device_cgroup_callback,
    [LXC_CMD_FREEZE]                    = lxc_cmd_freeze_callback,
    [LXC_CMD_UNFREEZE]                  = lxc_cmd_unfreeze_callback,
    [LXC_CMD_GET_CGROUP2_FD]            = lxc_cmd_get_cgroup2_fd_callback,
    [LXC_CMD_GET_LIMIT_CGROUP2_FD]      = lxc_cmd_get_limit_cgroup2_fd_callback,
    [LXC_CMD_GET_CGROUP_FD]             = lxc_cmd_get_cgroup_fd_callback,
    [LXC_CMD_GET_LIMIT_CGROUP_FD]       = lxc_cmd_get_limit_cgroup_fd_callback,
    [LXC_CMD_GET_CGROUP_CTX]            = lxc_cmd_get_cgroup_ctx_callback,
    [LXC_CMD_GET_SYSTEMD_SCOPE]         = lxc_cmd_get_systemd_scope_callback,
};
```

---

## 5. 客户端实现

### 5.1 发送命令

`commands.c:443-545`：

```c
lxc_cmd_send(name, req, rsp, lxcpath) {
    // 1. 连接到容器的命令套接字
    fd = lxc_cmd_connect(name, lxcpath, "command");

    // 2. 发送请求
    lxc_cmd_send(fd, req);

    // 3. 接收响应
    lxc_cmd_rsp_recv(fd, rsp);

    close(fd);
}

lxc_cmd_timeout(name, req, rsp, lxcpath, timeout) {
    // 带超时的命令发送（使用 poll 等待）
}

lxc_cmd(name, req, rsp, lxcpath) {
    // 无超时的命令发送
    return lxc_cmd_timeout(name, req, rsp, lxcpath, -1);
}
```

### 5.2 响应接收

`commands.c:168-337`，`lxc_cmd_rsp_recv()`：

```c
lxc_cmd_rsp_recv(fd, rsp) {
    // 1. 读取固定大小的 lxc_cmd_rsp 头
    recv(fd, rsp, sizeof(*rsp));

    // 2. 如果有附加数据，继续读取
    if (rsp->datalen > 0) {
        rsp->data = malloc(rsp->datalen);
        recv(fd, rsp->data, rsp->datalen);
    }

    // 3. 处理 fd 传递（通过 SCM_RIGHTS）
    // 某些命令会通过辅助消息传递文件描述符
}
```

---

## 6. 重要命令详解

### 6.1 获取 init PID

`commands.c:608-640`：

```
Client: lxc_cmd_get_init_pid(name, lxcpath)
Server: lxc_cmd_get_init_pid_callback(req, handler)
  -> return handler->pid (container init PID)
```

### 6.2 获取 init pidfd

`commands.c:643-682`：

```
Client: lxc_cmd_get_init_pidfd(name, lxcpath)
Server: lxc_cmd_get_init_pidfd_callback(req, handler)
  -> pass handler->pidfd via SCM_RIGHTS
```

### 6.3 停止容器

`commands.c:1105-1166`：

```
Client: lxc_cmd_stop(name, lxcpath)
Server: lxc_cmd_stop_callback(req, handler)
  -> kill(handler->pid, SIGKILL)
  -> handler->state = STOPPING
```

### 6.4 状态监听

`commands.c:1386-1453`：

```
Client: lxc_cmd_add_state_client(name, lxcpath, states_mask)
Server: lxc_cmd_add_state_client_callback(req, handler)
  -> add client fd to state watcher list
  -> return LXC_CMD_KEEP_CLIENT_FD (keep connection)
  -> notify all watchers on state change
```

### 6.5 BPF 设备规则添加

`commands.c:1455-1506`：

```
Client: lxc_cmd_add_bpf_device_cgroup(name, lxcpath, rule)
Server: lxc_cmd_add_bpf_device_cgroup_callback(req, handler)
  -> bpf_cgroup_devices_update(handler->cgroup_ops, rule)
  -> atomically replace BPF program
```

### 6.6 冻结/解冻

`commands.c:1682-1743`：

```
Client: lxc_cmd_freeze(name, lxcpath, timeout)
Server: lxc_cmd_freeze_callback(req, handler)
  -> handler->cgroup_ops->freeze(timeout)
```

### 6.7 获取 cgroup 路径

`commands.c:940-987`：

```
Client: lxc_cmd_get_cgroup(name, controller, lxcpath)
Server: lxc_cmd_get_cgroup_callback(req, handler)
  -> find hierarchy for controller
  -> return hierarchy->path_con (container cgroup path)
```

---

## 7. fd 传递机制

部分命令需要传递文件描述符（如 pidfd、cgroup fd、devpts fd）。LXC 使用 Unix 域套接字的 **SCM_RIGHTS** 辅助消息：

```c
// 发送 fd（服务端）
struct cmsghdr cmsg = {
    .cmsg_level = SOL_SOCKET,
    .cmsg_type = SCM_RIGHTS,
    .cmsg_len = CMSG_LEN(sizeof(int)),
};
*(int *)CMSG_DATA(&cmsg) = target_fd;
sendmsg(client_fd, &msg, 0);

// 接收 fd（客户端）
recvmsg(fd, &msg, 0);
received_fd = *(int *)CMSG_DATA(CMSG_FIRSTHDR(&msg));
```

支持 fd 传递的命令：
- `GET_INIT_PIDFD` → init 进程的 pidfd
- `GET_DEVPTS_FD` → devpts 文件系统 fd
- `GET_SECCOMP_NOTIFY_FD` → seccomp notify fd
- `GET_TTY_FD` → 终端 fd
- `GET_CGROUP_FD` / `GET_CGROUP2_FD` → cgroup 目录 fd

---

## 8. 客户端 API 使用

`lxccontainer.c` 中的 API 通过命令通道与运行中的容器通信：

```c
// 获取 init PID
do_lxcapi_init_pid(c) {
    return lxc_cmd_get_init_pid(c->name, c->config_path);
}

// 获取 init pidfd
do_lxcapi_init_pidfd(c) {
    return lxc_cmd_get_init_pidfd(c->name, c->config_path);
}

// 获取 devpts fd
do_lxcapi_devpts_fd(c) {
    return lxc_cmd_get_devpts_fd(c->name, c->config_path);
}

// 控制台日志
do_lxcapi_console_log(c, log) {
    return lxc_cmd_console_log(c->name, log, c->config_path);
}

// 状态查询
lxc_getstate(name, lxcpath) {
    return lxc_cmd_get_state(name, lxcpath);
}
```

---

## 9. 工具函数

`commands_utils.c:30-201`：

```c
// 连接到容器命令套接字
lxc_cmd_connect(name, lxcpath, suffix)

// 通过命令套接字获取容器状态
lxc_cmd_sock_get_state(name, lxcpath)
lxc_cmd_sock_rcv_state(fd, timeout)

// 注册状态监听
lxc_add_state_client(fd, handler, states_mask)

// 通知所有状态监听者
lxc_cmd_notify_state_listeners(name, handler, state)
```

---

## 10. 设计总结

1. **抽象套接字**：使用 Linux 抽象命名空间避免文件系统竞争和清理问题，套接字随进程退出自动消失。

2. **数组分发表**：命令类型直接作为数组索引，O(1) 分发，无需查找或 switch/case。

3. **fd 传递**：通过 SCM_RIGHTS 安全传递文件描述符，避免路径解析和权限问题。

4. **状态监听模型**：`ADD_STATE_CLIENT` 命令保持连接（`KEEP_CLIENT_FD`），实现推送式状态通知，无需轮询。

5. **事件循环集成**：命令套接字注册到主事件循环，与信号、终端 I/O 等统一处理，避免额外线程。

6. **运行时控制**：通过命令通道可实现冻结/解冻、BPF 规则更新等运行时操作，无需重启容器。
