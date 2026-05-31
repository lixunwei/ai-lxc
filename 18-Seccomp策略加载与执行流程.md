# Seccomp 策略加载与执行流程深度分析

## 1. 概述

LXC 通过 libseccomp 库为容器提供系统调用过滤。支持两种策略格式（v1/v2）、多架构过滤、以及 `SECCOMP_RET_USER_NOTIF` 机制（允许 monitor 代理处理被拦截的系统调用）。

**核心源文件**：
- `src/lxc/seccomp.c` — 策略解析、过滤器生成、加载、notify 代理
- `src/lxc/lxcseccomp.h` — 数据结构定义
- `src/lxc/start.c` — 生命周期集成点
- `src/lxc/conf.c` — 父子进程间 notify fd 传递

---

## 2. 数据结构

### 2.1 seccomp 配置结构

```c
// lxcseccomp.h:54-63
struct seccomp_notify {
    bool wants_supervision;   // 是否有 notify 规则
    int notify_fd;            // 内核 listener fd (SECCOMP_RET_USER_NOTIF)
    int proxy_fd;             // 连接到外部代理的 socket
    struct sockaddr_un proxy_addr;
    // request/response 缓冲区和 cookie
};

// lxcseccomp.h:67-76
struct lxc_seccomp {
    char *seccomp;            // 策略文件路径
    scmp_filter_ctx seccomp_ctx;  // libseccomp 主上下文
    unsigned int allow_nesting;    // 嵌套容器支持
    struct seccomp_notify notifier;
};
```

### 2.2 在 lxc_conf 中的位置

```c
struct lxc_conf {
    ...
    struct lxc_seccomp seccomp;  // seccomp 策略配置
    ...
};
```

---

## 3. 策略文件格式

### 3.1 v1 格式（简单白名单）

```
1       # version
open    # 允许的系统调用名或编号
read
write
close
...
```

每行一个系统调用，默认动作为 `SCMP_ACT_KILL`（未列出的调用将终止进程）。

### 3.2 v2 格式（支持架构、动作、参数条件）

```
2                          # version
denylist                   # 模式: allowlist 或 denylist
[x86_64]                   # 架构段
kexec_load errno 1         # syscall + action
open_by_handle_at errno 1
[x86]                      # 32位兼容段
kexec_load errno 1
[all]                      # 所有架构通用
mount notify               # 使用 notify 代理处理
keyctl kill                # 直接杀死进程
```

#### 支持的动作

| 动作 | 含义 |
|------|------|
| `kill` | SCMP_ACT_KILL，终止进程 |
| `errno N` | SCMP_ACT_ERRNO(N)，返回错误码 |
| `allow` | SCMP_ACT_ALLOW，允许调用 |
| `trap` | SCMP_ACT_TRAP，发送 SIGSYS |
| `notify` | SCMP_ACT_NOTIFY，转发到 monitor 代理 |
| `log` | SCMP_ACT_LOG，记录但允许 |
| `trace` | SCMP_ACT_TRACE，ptrace 通知 |

#### 参数条件

```
# 格式: syscall action [index,value,op] 或 [index,value,op,mask]
# op: NE, LT, LE, EQ, GE, GT, MASKED_EQ
clone errno 0 [0,0x7e020000,MASKED_EQ,0x7e020000]
```

---

## 4. 完整执行流程

```
+--------------------------------------------------+
|  1. Config Parse                                 |
|  lxc.seccomp.profile = /path/to/policy           |
+--------------------------------------------------+
          |
          v
+--------------------------------------------------+
|  2. lxc_read_seccomp_config(conf)                |
|  - 初始化 libseccomp 上下文 (SCMP_ACT_KILL)      |
|  - 禁用 NNP (SCMP_FLTATR_CTL_NNP = 0)          |
|  - 启用 SCMP_FLTATR_ATL_TSKIP (如可用)          |
|  - 打开策略文件                                   |
|  - 调用 parse_config() -> v1/v2 解析器           |
+--------------------------------------------------+
          |
          v
+--------------------------------------------------+
|  3. parse_config_v2()                            |
|  - 解析模式 (allowlist/denylist)                  |
|  - 按架构段创建子上下文 (get_new_ctx)             |
|  - 解析每条规则 -> do_resolve_add_rule()          |
|  - seccomp_merge() 合并多架构上下文              |
|  - 标记 wants_supervision (如有 notify 规则)      |
+--------------------------------------------------+
          |
          v
+--------------------------------------------------+
|  4. lxc_seccomp_load(conf) [容器子进程中]         |
|  - seccomp_load() 安装 BPF 过滤器到内核          |
|  - 如有 notify: seccomp_notify_fd() 获取监听 fd  |
|  - 设为 blocking, 存入 notifier.notify_fd        |
+--------------------------------------------------+
          |
          v
+--------------------------------------------------+
|  5. fd 传递 (子进程 -> 父进程 handler)            |
|  - lxc_seccomp_send_notifier_fd() [子进程]       |
|  - lxc_seccomp_recv_notifier_fd() [handler]      |
|  - 通过 sync socket 传递                         |
+--------------------------------------------------+
          |
          v
+--------------------------------------------------+
|  6. lxc_seccomp_setup_proxy() [handler 侧]       |
|  - 连接到 proxy_addr Unix socket                 |
|  - 查询 SECCOMP_GET_NOTIF_SIZES                  |
|  - 分配 req/resp 缓冲区                          |
|  - 注册 seccomp_notify_handler 到 mainloop       |
+--------------------------------------------------+
          |
          v
+--------------------------------------------------+
|  7. seccomp_notify_handler() [运行时]             |
|  - seccomp_notify_receive() 接收内核通知         |
|  - seccomp_notify_id_valid() 验证                |
|  - 打开 /proc/<pid> 和 /proc/<pid>/mem           |
|  - 通过 seqpacket socket 转发给外部代理          |
|  - 接收代理回复                                   |
|  - seccomp_notify_respond() 回复内核             |
+--------------------------------------------------+
```

---

## 5. 策略解析详解

### 5.1 入口函数

```c
// seccomp.c:1205-1256
int lxc_read_seccomp_config(struct lxc_conf *conf) {
    // 1. 初始化默认动作为 KILL
    conf->seccomp.seccomp_ctx = seccomp_init(SCMP_ACT_KILL);

    // 2. 关键属性设置
    seccomp_attr_set(ctx, SCMP_FLTATR_CTL_NNP, 0);      // 不由 seccomp 设置 NNP
    seccomp_attr_set(ctx, SCMP_FLTATR_ATL_TSKIP, 1);    // 跳过不匹配的架构

    // 3. 打开策略文件并解析
    FILE *f = fopen(conf->seccomp.seccomp, "r");
    parse_config(f, conf);
}
```

### 5.2 版本分发

```c
// seccomp.c:1121-1156
static int parse_config(FILE *f, struct lxc_conf *conf) {
    // 读取第一行获取版本号
    int version = atoi(line);
    switch (version) {
        case 1: return parse_config_v1(f, ...);
        case 2: return parse_config_v2(f, ...);
    }
}
```

### 5.3 多架构上下文管理

```c
// seccomp.c:358-495  get_new_ctx()
static scmp_filter_ctx get_new_ctx(enum lxc_hostarch_t n_arch,
                                   uint32_t action, ...) {
    // 根据目标架构创建新的 libseccomp 上下文
    scmp_filter_ctx ctx = seccomp_init(action);

    // 添加目标架构标记
    seccomp_arch_add(ctx, SCMP_ARCH_X86);  // 或 ARM, AARCH64 等

    // 移除本机架构（如果不匹配）
    seccomp_arch_remove(ctx, seccomp_arch_native());

    return ctx;
}
```

**多架构合并**：在 x86_64 宿主上，可能创建 x86、x32、x86_64 三个独立上下文，最后通过 `seccomp_merge()` 合并为单一 BPF 程序：

```c
// seccomp.c:1036-1083
seccomp_merge(conf->seccomp.seccomp_ctx, compat_ctx[0]);  // 合并 x86
seccomp_merge(conf->seccomp.seccomp_ctx, compat_ctx[1]);  // 合并 x32
```

### 5.4 规则添加

```c
// seccomp.c:504-586
static int do_resolve_add_rule(uint32_t arch, char *line,
                               scmp_filter_ctx ctx, struct seccomp_v2_rule *rule) {
    // 1. 解析系统调用名
    int nr = seccomp_syscall_resolve_name(syscall_name);

    // 2. 特殊处理: reject_force_umount
    if (strcmp(line, "reject_force_umount") == 0) {
        // 拒绝带 MNT_FORCE/MNT_DETACH 的 umount2
        rule->args_num = 1;
        rule->args_value[0] = {.index=0, .value=MNT_FORCE|MNT_DETACH, .op=SCMP_CMP_MASKED_EQ};
    }

    // 3. 构建参数比较数组
    struct scmp_arg_cmp arg_cmp[6];
    for (i = 0; i < rule->args_num; i++)
        arg_cmp[i] = SCMP_CMP(index, op, value, mask);

    // 4. 添加精确规则
    seccomp_rule_add_exact_array(ctx, rule->action, nr, rule->args_num, arg_cmp);
}
```

---

## 6. 过滤器加载

### 6.1 加载时机

在容器子进程的 `do_start()` 路径中，rootfs 设置完成后、start hooks 之前：

```c
// start.c:1488-1493
ret = lxc_seccomp_load(handler->conf);
if (ret < 0)
    goto out_warn_father;
```

### 6.2 加载实现

```c
// seccomp.c:1258-1313
int lxc_seccomp_load(struct lxc_conf *conf) {
    // 1. 将 BPF 过滤器安装到内核
    ret = seccomp_load(conf->seccomp.seccomp_ctx);

    // 2. 调试: 导出 BPF 到日志
    if (lxc_log_trace())
        seccomp_export_bpf(ctx, fd);

    // 3. 如果有 notify 规则, 获取 listener fd
    if (conf->seccomp.notifier.wants_supervision) {
        conf->seccomp.notifier.notify_fd = seccomp_notify_fd(ctx);
        // 设为 blocking 模式 (用于 handler 侧 epoll)
        fcntl(notify_fd, F_SETFL, flags & ~O_NONBLOCK);
    }

    return 0;
}
```

---

## 7. Notify 代理机制

### 7.1 架构

```
+------------------+     +-----------------+     +------------------+
|  Container       |     |  Handler/       |     |  External Proxy  |
|  (filtered proc) |     |  Monitor        |     |  (e.g. lxc-agent)|
+------------------+     +-----------------+     +------------------+
        |                        |                        |
        | syscall (notify)       |                        |
        +----------------------->|                        |
        |                        | forward req + fds      |
        |                        +----------------------->|
        |                        |                        |
        |                        |      decision (allow/  |
        |                        |<-----------------------+
        |                        |      deny/modify)      |
        |  kernel responds       |                        |
        |<-----------------------+                        |
        |                        |                        |
```

### 7.2 代理设置

```c
// seccomp.c:1604-1658
int lxc_seccomp_setup_proxy(struct lxc_seccomp *seccomp,
                            struct lxc_async_descr *descr,
                            struct lxc_handler *handler) {
    // 1. 连接到配置的代理 socket
    seccomp->notifier.proxy_fd = socket(AF_UNIX, SOCK_SEQPACKET, 0);
    connect(proxy_fd, &seccomp->notifier.proxy_addr, ...);

    // 2. 查询通知缓冲区大小
    ioctl(notify_fd, SECCOMP_IOCTL_NOTIF_GET_SIZES, &sizes);

    // 3. 分配请求/响应缓冲区
    seccomp_notify_alloc(&req, &resp);

    // 4. 注册到 handler mainloop
    lxc_mainloop_add_handler(descr, notify_fd,
                             seccomp_notify_handler, handler);
}
```

### 7.3 运行时处理

```c
// seccomp.c:1400-1583
int seccomp_notify_handler(int fd, uint32_t events,
                           void *data, struct lxc_async_descr *descr) {
    // 1. 接收内核通知
    seccomp_notify_receive(notify_fd, req);

    // 2. 验证请求仍然有效（进程未退出）
    seccomp_notify_id_valid(notify_fd, req->id);

    // 3. 打开目标进程的 /proc 路径（供代理读取内存）
    int proc_fd = open("/proc/<pid>", O_DIRECTORY);
    int mem_fd  = open("/proc/<pid>/mem", O_RDWR);

    // 4. 构造转发消息
    struct seccomp_notify_proxy_msg msg = {
        .monitor_pid = handler->monitor_pid,
        .init_pid    = handler->pid,
        .req         = *req,
        .cookie      = cookie,
    };

    // 5. 通过 Unix socket + SCM_RIGHTS 发送 fds
    sendmsg(proxy_fd, &msghdr, 0);  // 包含 proc_fd, mem_fd

    // 6. 接收代理决策
    recvmsg(proxy_fd, &msghdr, 0);

    // 7. 回复内核
    seccomp_notify_respond(notify_fd, resp);

    return LXC_MAINLOOP_CONTINUE;
}
```

### 7.4 默认回退

如果代理不可用或出错：

```c
// seccomp.c:1361-1377
static void seccomp_notify_default_answer(int fd, struct seccomp_notif *req,
                                          struct seccomp_notif_resp *resp) {
    resp->id    = req->id;
    resp->error = -ENOSYS;  // 返回"功能未实现"
    resp->val   = 0;
    seccomp_notify_respond(fd, resp);
}
```

---

## 8. Fd 传递机制

seccomp notify fd 在容器子进程中创建（因为 `seccomp_load()` 在子进程执行），但需要在 handler 进程中使用：

```c
// 子进程侧 (do_start):
lxc_seccomp_load(conf);  // notify_fd 产生
lxc_seccomp_send_notifier_fd(&conf->seccomp, sync_fd);
// 通过 SCM_RIGHTS 发送 notify_fd

// 父进程侧 (handler):
lxc_seccomp_recv_notifier_fd(&conf->seccomp, sync_fd);
// 接收后存入 conf->seccomp.notifier.notify_fd
lxc_seccomp_setup_proxy(...);  // 注册到 mainloop
```

---

## 9. 特殊处理

### 9.1 reject_force_umount

v2 策略的内置宏规则：

```
reject_force_umount  # 拒绝 umount2(MNT_FORCE|MNT_DETACH)
```

自动展开为：`umount2 errno EACCES [0, MNT_FORCE|MNT_DETACH, MASKED_EQ]`

### 9.2 嵌套容器支持

```
lxc.seccomp.allow_nesting = 1
```

启用时，不为 `seccomp()` 系统调用本身添加限制，允许嵌套容器设置自己的 seccomp 过滤器。

### 9.3 NO_NEW_PRIVS 处理

LXC 通过 `SCMP_FLTATR_CTL_NNP = 0` 告诉 libseccomp 不要自动设置 `PR_SET_NO_NEW_PRIVS`，因为 LXC 在其他地方单独处理该标志。

---

## 10. 配置项

| 配置 | 说明 |
|------|------|
| `lxc.seccomp.profile` | 策略文件路径 |
| `lxc.seccomp.allow_nesting` | 允许嵌套容器设置 seccomp |
| `lxc.seccomp.notify.proxy` | notify 代理 Unix socket 地址 |
| `lxc.seccomp.notify.cookie` | 传递给代理的自定义 cookie 字符串 |

---

## 11. 与容器生命周期的时序关系

```
container_start
  |
  +-- lxc_read_seccomp_config()    [配置阶段: 解析策略文件]
  |
  +-- clone(CLONE_NEWPID|...)      [创建容器进程]
  |
  +-- do_start() [子进程]:
  |     |
  |     +-- lxc_setup()            [rootfs/mount/设备]
  |     +-- setup_capabilities()
  |     +-- lxc_seccomp_load()     [安装 BPF 过滤器]
  |     +-- send_notifier_fd()     [传递 fd 给 handler]
  |     +-- execvp()               [启动 init]
  |
  +-- handler [父进程]:
        |
        +-- recv_notifier_fd()
        +-- lxc_seccomp_setup_proxy()
        +-- lxc_poll()             [mainloop: 含 notify handler]
```

**关键顺序**：seccomp 在 capabilities 降权之后加载，确保加载过程有足够权限；加载后所有新执行的代码（包括容器 init）都受 BPF 过滤器约束。
