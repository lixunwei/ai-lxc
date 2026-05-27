# LXC Attach 实现原理深度分析

## 1. 概述

`lxc-attach` 允许在已运行的容器中执行命令或启动交互式 shell。其核心挑战是：将一个新进程"注入"到容器的所有命名空间中，同时正确处理安全上下文（LSM、capabilities、uid/gid）。

**核心源文件**：
- `src/lxc/tools/lxc_attach.c` — CLI 入口
- `src/lxc/attach.c` — 核心实现（约 1800 行）
- `src/lxc/attach_options.h` — 选项结构定义

---

## 2. 两次 Fork 设计

这是 attach 最关键的架构决策：

```
lxc-attach (原始进程, "grandparent")
    |
    +-- fork() --> 中间进程 ("transient")
                      |
                      |-- setns() 进入容器命名空间
                      |-- clone(CLONE_PARENT) --> 最终进程 ("attached")
                      |                             |
                      |                             +-- do_attach()
                      |                             +-- exec(command)
                      +-- 退出 (孤儿收割由容器 init 处理)
```

**为什么需要两次 fork？**

1. **PID namespace 语义**：`setns(CLONE_NEWPID)` 只影响**子进程**的 PID namespace，而非当前进程。因此必须在 setns 之后再 clone 一次。

2. **CLONE_PARENT**：让最终进程的父进程是 grandparent（而非 transient），这样 grandparent 可以直接 `waitpid()` 获取退出状态。

3. **中间进程快速退出**：transient 在 clone 后立即退出，其创建的子进程被容器 init (PID 1) 接管为孤儿。

---

## 3. 完整流程时序图

```
+----------------+     +----------------+     +-----------------+     +---------+
| lxc-attach     |     | Transient      |     | Attached proc   |     | Monitor |
| (grandparent)  |     | (intermediate) |     | (final child)   |     | daemon  |
+----------------+     +----------------+     +-----------------+     +---------+
       |                       |                       |                    |
       |-- lxc_cmd_get_init_pid/pidfd --------------->|                    |
       |<-- pid/pidfd --------------------------------|                    |
       |                       |                       |                    |
       |-- get_attach_context()--->                    |                    |
       |   (ns fds, caps, LSM, personality)           |                    |
       |                       |                       |                    |
       |-- socketpair(IPC) --->|                       |                    |
       |                       |                       |                    |
       |== fork() ============>|                       |                    |
       |                       |                       |                    |
       |-- cgroup_attach(pid)->|                       |                    |
       |-- sync(CGROUP) ------>|                       |                    |
       |                       |                       |                    |
       |                       |-- attach_namespaces() |                    |
       |                       |   setns(USER)         |                    |
       |                       |   setns(MNT)          |                    |
       |                       |   setns(PID)          |                    |
       |                       |   setns(UTS)          |                    |
       |                       |   setns(IPC)          |                    |
       |                       |   setns(NET)          |                    |
       |                       |   setns(CGROUP)       |                    |
       |                       |   setns(TIME)         |                    |
       |                       |                       |                    |
       |                       |== clone(CLONE_PARENT)=>|                   |
       |                       |                       |                    |
       |<-- sync(attached_pid)-|                       |                    |
       |                       +-- exit()              |                    |
       |                                               |                    |
       |-- send LSM fd --------------------------->|   |                    |
       |                                               |                    |
       |                                               |-- recv LSM fd      |
       |                                               |-- lxc_switch_uid_gid()
       |                                               |-- set LSM label    |
       |                                               |-- drop_capabilities()
       |                                               |-- set environment  |
       |                                               |-- exec(command)    |
       |                                               |                    |
       |<-- waitpid(attached_pid) --------------------|                    |
       +                                               +                    +
```

---

## 4. 命名空间进入

### 4.1 进入顺序

定义在 `src/lxc/namespace.c:38-47`：

```
USER -> MNT -> PID -> UTS -> IPC -> NET -> CGROUP -> TIME
```

**USER 必须第一个**：进入 user namespace 后才有权限进入其他 namespace。

### 4.2 两种进入方式

**方式 1：逐个 setns（传统）**
```c
// attach.c:653-678 __attach_namespaces_nsfd()
for each ns in ns_info[] order:
    if (ns_flag & requested_namespaces):
        setns(ctx->ns_fd[i], ns_flag);
```

**方式 2：pidfd 单次 setns（内核 5.3+）**
```c
// attach.c:634-651
setns(ctx->init_pidfd, all_ns_flags);  // 一次调用进入所有命名空间
```

pidfd 方式更原子化，避免了逐个进入时的中间状态。

### 4.3 命名空间 fd 获取

```c
// get_attach_context() -> prepare_namespaces()
// 通过 /proc/<init_pid>/ns/<ns_name> 打开各命名空间 fd
// 或通过 pidfd_open(init_pid) 获取 pidfd
```

---

## 5. 与 Monitor 的通信

attach 需要从运行中的容器 monitor 进程获取信息：

```c
// 通过 Unix socket 与 monitor 通信 (src/lxc/commands.c)
lxc_cmd_get_init_pid(name, lxcpath);      // 获取容器 init PID
lxc_cmd_get_init_pidfd(name, lxcpath);    // 获取 init 的 pidfd
lxc_cmd_get_clone_flags(name, lxcpath);   // 获取容器使用的命名空间标志
lxc_cmd_get_config_item(name, "lxc.arch", ...); // 获取配置项
```

通信协议：
- 连接 `$rundir/lxc/$lxcpath/$name/command` Unix socket
- 发送带 credential (SCM_CREDENTIALS) 的请求
- 响应可携带 fd (SCM_RIGHTS)

---

## 6. Cgroup 附加

在 transient 进程进入命名空间**之前**，grandparent 将其移入容器 cgroup：

```c
// attach.c:1674-1697
cgroup_attach(conf, name, lxcpath, transient_pid) {
    // 获取容器的 cgroup 路径
    // 写入 transient_pid 到 cgroup.procs
}
// 之后通知 transient 继续
sync_wake(ipc_socket, ATTACH_SYNC_CGROUP);
```

顺序很重要：先入 cgroup，再入 namespace，确保 attached 进程受容器资源限制。

---

## 7. 安全转换

### 7.1 Capabilities 降权

```c
// attach.c:774-795 drop_capabilities()
// 读取容器 init 的 bounding set: /proc/<pid>/status -> CapBnd
for cap = 0; cap <= CAP_LAST_CAP; cap++:
    if cap NOT in container_bounding_set:
        prctl(PR_CAPBSET_DROP, cap);
```

### 7.2 LSM 标签设置

```c
// 1. Grandparent 打开 LSM proc fd（在 setuid 之前）
//    原因: setuid 后进程变为 undumpable，无法再读 /proc
fd_lsm = open("/proc/<pid>/attr/current");  // 或 exec

// 2. 通过 IPC socket 发送 fd 给 attached 进程
send_fd(ipc_socket, fd_lsm);

// 3. Attached 进程接收并设置标签
process_label_set_at(lsm_ops, fd_lsm, label, on_exec);
// AppArmor: 写 "changeprofile <label>" 到 /proc/self/attr/current
// SELinux: 写 label 到 /proc/self/attr/exec (on_exec=true)
```

### 7.3 UID/GID 切换

```c
// 对于 user namespace 容器:
// userns_setup_ids() 确定应使用的 uid/gid
// lxc_switch_uid_gid(setup_ns_uid, setup_ns_gid)
setgid(gid);
setuid(uid);
```

### 7.4 no_new_privs

```c
if (conf->no_new_privs || options->attach_flags & LXC_ATTACH_NO_NEW_PRIVS)
    prctl(PR_SET_NO_NEW_PRIVS, 1);
```

---

## 8. PTY 终端处理

### 8.1 终端检测

```c
// CLI: lxc_attach.c:400-401
if (isatty(STDIN_FILENO))
    options.attach_flags |= LXC_ATTACH_TERMINAL;
```

### 8.2 终端创建

```c
// attach.c:1491-1501
if (LXC_ATTACH_TERMINAL) {
    lxc_attach_terminal(&terminal, ...);
    // 分配 PTY pair (ptx/pts)
    // ptx 留在 grandparent 用于 I/O 转发
    // pts fd 传给 attached 进程
}
```

### 8.3 I/O 转发

```c
// Grandparent 运行 mainloop:
// stdin  -> ptx (发送到容器)
// ptx    -> stdout (从容器接收)
// 处理窗口大小变化 (SIGWINCH -> ioctl TIOCSWINSZ)
```

### 8.4 Attached 进程设置

```c
// do_attach() 中:
lxc_terminal_prepare_login(pts_fd);
// setsid() - 新会话
// ioctl(TIOCSCTTY) - 设为控制终端
// dup2(pts_fd, 0/1/2) - 重定向标准流
```

---

## 9. 环境变量处理

```c
// attach.c:797-909 lxc_attach_set_environment()
if (LXC_ATTACH_CLEAR_ENV) {
    clearenv();
    // 保留 keep_env 列表中的变量
    setenv("PATH", "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin");
}

// 始终设置
setenv("container", "lxc");

// 注入容器配置中的环境变量 (lxc.environment)
for each env in conf->environment:
    putenv(env);

// 追加用户指定的额外变量
for each env in options->extra_env_vars:
    putenv(env);
```

---

## 10. 选项结构

```c
// attach_options.h:90-178
typedef struct lxc_attach_options_t {
    lxc_attach_env_policy_t env_policy;  // LXC_ATTACH_KEEP_ENV | CLEAR_ENV
    int namespaces;           // 要进入的命名空间标志
    long personality;         // 进程 personality (32/64bit)
    char *initial_cwd;        // 进入后的工作目录
    uid_t uid, gid;           // 目标 uid/gid (-1 = 不切换)
    int attach_flags;         // 行为标志组合
    int stdin_fd, stdout_fd, stderr_fd;  // 自定义 stdio fd
    char *lsm_label;          // 自定义 LSM 标签
    lxc_groups_t groups;      // 补充组列表

    // attach_flags 常用标志:
    // LXC_ATTACH_MOVE_TO_CGROUP    - 移入容器 cgroup
    // LXC_ATTACH_DROP_CAPABILITIES - 降低 capabilities
    // LXC_ATTACH_SET_PERSONALITY   - 设置 personality
    // LXC_ATTACH_LSM_EXEC          - LSM label on exec
    // LXC_ATTACH_LSM_LABEL         - 使用自定义 LSM label
    // LXC_ATTACH_NO_NEW_PRIVS      - 设置 no_new_privs
    // LXC_ATTACH_TERMINAL          - 分配 PTY
    // LXC_ATTACH_SETGROUPS         - 设置补充组
} lxc_attach_options_t;
```

---

## 11. 安全屏障

在进入容器上下文前，关闭所有敏感 fd：

```c
// attach.c:724-737 attach_context_security_barrier()
// 关闭所有打开的 namespace fds
// 关闭 pidfd
// 关闭其他可能泄露宿主信息的 fd
// 确保进入容器时不携带宿主侧的文件描述符
```

---

## 12. 错误处理与同步

父子进程通过 socketpair 进行多阶段同步：

| 同步点 | 方向 | 含义 |
|--------|------|------|
| `ATTACH_SYNC_CGROUP` | parent→transient | cgroup 附加完成，可以继续 |
| `sync_wake_pid` | transient→parent | 通告 attached 进程的 PID |
| `sync_wait_fd` | parent→attached | 传递 LSM fd |

任何阶段失败都通过 shutdown(socket) 通知对方退出。

---

## 13. 使用示例

```bash
# 在容器中执行命令
lxc-attach -n mycontainer -- ls -la /

# 交互式 shell
lxc-attach -n mycontainer

# 指定命名空间（只进入网络和挂载）
lxc-attach -n mycontainer -s 'NETWORK|MOUNT' -- ip addr

# 提升权限（不降 capabilities）
lxc-attach -n mycontainer -e

# 自定义 LSM 标签
lxc-attach -n mycontainer -c unconfined -- /bin/bash

# 指定 uid/gid
lxc-attach -n mycontainer -u 1000 -g 1000 -- id
```
