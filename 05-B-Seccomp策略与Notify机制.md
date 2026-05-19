# LXC 安全子系统深度分析（二）：Seccomp 策略与 Notify 机制

## 1. 概述

LXC 的 Seccomp 支持提供系统调用过滤，阻止容器内进程调用危险的系统调用。实现支持两种策略格式（v1/v2）、多架构适配、以及先进的 **Seccomp Notify** 机制（允许 supervisor 代理处理被拦截的系统调用）。

核心文件：

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/lxc/lxcseccomp.h` | ~80 | 数据结构定义 |
| `src/lxc/seccomp.c` | ~1700 | 完整实现 |

---

## 2. 数据结构

### 2.1 `struct lxc_seccomp`

`lxcseccomp.h:67-77`：

```c
struct lxc_seccomp {
    char *seccomp;          // 策略文件路径（lxc.seccomp.profile）
    bool allow_nesting;     // 是否允许嵌套容器的 seccomp

    scmp_filter_ctx seccomp_ctx;  // libseccomp 过滤器上下文

    /* Seccomp Notify 相关 */
    struct {
        int notify_fd;      // notify 文件描述符
        char *cookie;       // 认证 cookie
        struct sockaddr_un proxy_addr;  // 代理地址
        // ... notify 缓冲区等
    } notifier;
};
```

---

## 3. 策略解析

### 3.1 v1 格式（已过时）

`seccomp.c:45-71`：

v1 格式是简单的系统调用号白名单：

```
# v1 策略文件示例
1       # 每行一个系统调用号
2       # 允许列表中的调用
3       # 其余全部拒绝
```

解析逻辑：
```c
// 逐行读取系统调用号
// 所有列出的调用被允许
// 未列出的调用返回 EPERM
```

### 3.2 v2 格式

v2 格式功能更强，支持 allowlist/denylist、架构指定、参数过滤：

#### 策略文件格式

```
2                           # 版本号
allowlist                   # 或 denylist
[all]                       # 架构标签
read allow                  # syscall action
write allow
[x86]                       # 仅 x86 架构
ioctl allow [1,0x5401,SCMP_CMP_EQ]  # 带参数过滤
[x86_64]
mount errno 1               # 返回特定错误码
```

#### Action 解析

`seccomp.c:74-156`：

| Action 名称 | 对应常量 | 说明 |
|-------------|----------|------|
| `kill` | `SCMP_ACT_KILL` | 杀死进程 |
| `errno N` | `SCMP_ACT_ERRNO(N)` | 返回错误码 N |
| `allow` | `SCMP_ACT_ALLOW` | 允许执行 |
| `trap` | `SCMP_ACT_TRAP` | 发送 SIGSYS |
| `log` | `SCMP_ACT_LOG` | 记录日志但允许 |
| `notify` | `SCMP_ACT_NOTIFY` | 转发给 supervisor |

#### 参数过滤

`seccomp.c:158-295`：

```
格式：[index, value, op, mask]
  index: 参数索引（0-5）
  value: 比较值
  op:    SCMP_CMP_EQ / NE / GT / GE / LT / LE / MASKED_EQ
  mask:  位掩码（仅 MASKED_EQ 使用）
```

### 3.3 架构处理

`seccomp.c:298-495`，LXC 自动处理多架构兼容：

#### 架构检测

```c
get_hostarch() {
    struct utsname uts;
    uname(&uts);
    // 返回: lxc_seccomp_arch_amd64 / arm64 / ppc64 / ...
}
```

#### 架构组扩展

当遇到架构标签时，LXC 自动扩展为多个兼容架构（`seccomp.c:664-780`）：

| 主机架构 | `[all]` 扩展到 |
|----------|----------------|
| x86_64 | x86_64 + x86 + x32 |
| arm64 | arm64 + arm |
| ppc64 | ppc64 + ppc |
| mips64 | mips64 + mips + mips64n32 |
| s390x | s390x + s390 |

这确保 32 位兼容程序的系统调用也被正确过滤。

#### 架构标签

`seccomp.c:806-994`：

| 标签 | 含义 |
|------|------|
| `[all]` | 所有支持的架构 |
| `[x86_64]` | 仅 64 位 x86 |
| `[x86]` | 仅 32 位 x86 |
| `[x32]` | x32 ABI |
| `[arm64]` | 仅 64 位 ARM |
| `[arm]` | 仅 32 位 ARM |

### 3.4 策略构建流程

`seccomp.c:1001-1156`：

```
parse_config(conf)
  1. Read first policy line -> determine version (1 or 2)
  2. v1: parse_config_v1()
  3. v2:
     a. Read allowlist/denylist declaration
     b. Set default action:
        - allowlist -> default SCMP_ACT_KILL (deny unlisted)
        - denylist -> default SCMP_ACT_ALLOW (allow unlisted)
     c. Create filter contexts for needed arches
     d. Read rules line by line:
        - Parse arch tag [xxx]
        - Parse "syscall_name action [args]"
        - Call seccomp_rule_add() for the matching ctx
     e. Merge all arch ctx values (seccomp_merge)
```

---

## 4. 策略加载与应用

### 4.1 加载前检测

`seccomp.c:1165-1203`：

```c
// 检查是否应该跳过 seccomp 加载
if (nesting && already_confined) {
    // 容器嵌套且已被外层 seccomp 限制
    // 不加载内层策略（避免冲突）
    return;
}

// 检查 /proc/self/status 中的 Seccomp 字段
// 值为 2 表示已被过滤
```

### 4.2 加载流程

`seccomp.c:1258-1313`：

```
lxc_seccomp_load(conf)
  1. parse_config(conf)           // 解析策略文件
  2. seccomp_load(seccomp_ctx)    // 加载到内核
  3. 可选：seccomp_export_pfc()   // 导出人类可读格式到日志
  4. if (notify action used):
       listener_fd = seccomp_notify_fd(seccomp_ctx)
       // 获取 notify fd 用于后续事件处理
```

### 4.3 启动时调用点

```
start.c:1060-1065   读取 seccomp 配置
start.c:795-799     设置 notify proxy（如果使用）
start.c:1491-1493   加载 seccomp 策略（在 exec 之前）
```

---

## 5. Seccomp Notify 机制

Seccomp Notify 是 Linux 5.0+ 引入的特性，允许 supervisor 进程拦截和处理容器内的系统调用。

### 5.1 架构

```
+-----------------+      +------------------+      +--------------+
| Container proc  |      | LXC Supervisor   |      | Notify Proxy |
|                 |      |                  |      | (optional)   |
| syscall()       |--1-->| notify_receive() |--3-->| handle req   |
|                 |      |       [2]        |<--4--| send reply   |
|                 |<--5--| notify_respond() |      |              |
+-----------------+      +------------------+      +--------------+

1. Syscall hits SCMP_ACT_NOTIFY
2. seccomp_notify_receive() gets request
3. Forward to external proxy (if configured)
4. Receive proxy reply
5. seccomp_notify_respond() returns result to container proc
```

### 5.2 初始化

`seccomp.c:1585-1602`（init）、`seccomp.c:1604-1658`（proxy setup）：

```
seccomp_notify_init()
  +-- Allocate notify request/response buffers
  `-- Init notify state

seccomp_notify_setup_proxy()
  +-- Create Unix domain socket
  +-- Connect to address from lxc.seccomp.notify.proxy
  +-- Allocate notify buffer
  `-- Register with mainloop
```

### 5.3 Notify Fd 传递

```
container side:
  seccomp.c:1660-1672  seccomp_notify_send_fd()
  // Send notify fd to supervisor over Unix socket

supervisor side:
  seccomp.c:1674-1688  seccomp_notify_recv_fd()
  // Receive the container notify fd

  seccomp.c:1690-1706  seccomp_notify_add()
  // Register the notify fd in the event loop
```

也可通过命令通道获取运行中容器的 notify fd：
```
commands.c:753-770   lxc_cmd_get_seccomp_notify_fd()
commands.c:1616-1673 回调：添加 listener fd
```

### 5.4 事件处理主循环

`seccomp.c:1400-1583`，notify handler：

```
seccomp_notify_handler(fd, mainloop)
  +-- seccomp_notify_receive(fd, &req)        // Receive syscall request
  |     // req contains: syscall number, args, pid, id
  |
  +-- If proxy is connected:
  |     +-- seccomp_notify_id_valid(fd, req.id)  // Validate request still live
  |     |     // Process may already have exited
  |     |
  |     +-- Send request to proxy                // Over Unix socket
  |     |     // Includes req + cookie + extra fd
  |     |
  |     +-- Receive proxy reply                  // resp + flags
  |     |
  |     `-- seccomp_notify_respond(fd, &resp)    // Return to kernel
  |           // resp contains: error, val, flags
  |
  `-- No proxy:
        `-- Return default reply -ENOSYS         // Deny without killing proc
```

### 5.5 默认回退

`seccomp.c:1361-1377`：

```c
// 无 proxy 或 proxy 通信失败时的默认行为
resp.error = -ENOSYS;   // 系统调用不可用
resp.val   = 0;
seccomp_notify_respond(fd, &resp);
```

---

## 6. 配置项

`confile.c:262-265`：

| 配置项 | 说明 |
|--------|------|
| `lxc.seccomp.profile` | Seccomp 策略文件路径 |
| `lxc.seccomp.allow_nesting` | 嵌套容器中是否应用 seccomp |
| `lxc.seccomp.notify.proxy` | Notify proxy 的 Unix socket 地址 |
| `lxc.seccomp.notify.cookie` | Notify 请求的认证 cookie |

配置读写（`confile.c`）：
- Setter：`confile.c:1193-1252`
- Getter：`confile.c:4353-4395`
- Clearer：`confile.c:5109-5148`

---

## 7. 策略文件示例

### 7.1 基础 denylist

```
2
denylist
[all]
kexec_load errno 1
open_by_handle_at errno 1
init_module errno 1
finit_module errno 1
delete_module errno 1
```

### 7.2 带参数过滤的 allowlist

```
2
allowlist
[x86_64]
read allow
write allow
openat allow
close allow
ioctl allow [1,0x5401,SCMP_CMP_EQ]
mmap allow
```

### 7.3 带 Notify 的策略

```
2
denylist
[all]
mount notify          # 由 supervisor 处理 mount 调用
umount2 notify        # 由 supervisor 处理 umount 调用
mknod notify          # 由 supervisor 处理设备节点创建
```

---

## 8. 设计总结

1. **双版本兼容**：v1 格式保持向后兼容（简单数字列表），v2 格式提供完整的安全策略表达能力。

2. **自动多架构**：自动检测主机架构并扩展兼容架构过滤器，无需用户手动处理 32/64 位差异。

3. **Notify 代理模型**：通过 Unix socket 将系统调用处理委托给外部 proxy，实现了灵活的策略扩展（如允许容器内有限的 mount 操作）。

4. **嵌套感知**：检测当前进程是否已被 seccomp 过滤，嵌套场景下自动跳过重复加载。

5. **Cookie 认证**：Notify 机制通过 cookie 防止未授权的 fd 注入。

6. **优雅降级**：Notify proxy 不可用时，默认返回 `-ENOSYS` 而非杀死进程，提高容错性。
