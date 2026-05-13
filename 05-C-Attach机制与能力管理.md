# LXC 安全子系统深度分析（三）：Attach 机制、能力管理与 ID 映射

## 1. 概述

本文档深度分析 LXC 的三个关键安全机制：
- **Attach**：将进程安全地注入到运行中的容器
- **Capabilities**：Linux 能力的细粒度管理
- **ID 映射**：User Namespace 的 UID/GID 映射

以及 **CRIU** 集成用于容器检查点/恢复。

---

## 2. Attach 机制

### 2.1 数据结构

**用户 API 选项**（`attach_options.h:90-158`）：

```c
struct lxc_attach_options_t {
    int attach_flags;           // 要进入的 namespace 标志集
    int namespaces;             // 具体的 namespace 掩码
    long personality;           // 进程 personality
    char *initial_cwd;          // 初始工作目录

    /* 环境变量控制 */
    lxc_attach_env_policy_t env_policy;  // LXC_ATTACH_KEEP_ENV / CLEAR_ENV
    char **extra_env_vars;      // 附加环境变量
    char **extra_keep_env;      // 清除环境时保留的变量

    /* 标准 I/O */
    int stdin_fd;
    int stdout_fd;
    int stderr_fd;

    /* 安全 */
    char *lsm_label;            // 覆盖 LSM 标签
    uid_t uid;                  // 目标 UID
    gid_t gid;                  // 目标 GID
    lxc_groups_t groups;        // 附加组策略
};

/* 默认值宏 */
#define LXC_ATTACH_OPTIONS_DEFAULT {
    .attach_flags   = LXC_ATTACH_DEFAULT,
    .namespaces     = -1,           // 所有 namespace
    .personality    = -1,           // 继承
    .env_policy     = LXC_ATTACH_KEEP_ENV,
    .stdin_fd       = 0,
    .stdout_fd      = 1,
    .stderr_fd      = 2,
    .uid            = (uid_t)-1,    // 自动
    .gid            = (gid_t)-1,    // 自动
};
```

**执行命令结构**（`attach_options.h:183-186`）：

```c
struct lxc_attach_command_t {
    char  *program;     // 要执行的程序
    char **argv;        // 参数数组
};
```

**内部上下文**（`attach.c:84-105`）：

```c
struct attach_context {
    int init_pid;           // 容器 init 进程 PID
    int init_pidfd;         // init 进程的 pidfd
    int dfd_init_pid;       // /proc/<init_pid> 的 fd
    int dfd_self_pid;       // /proc/self 的 fd

    /* 安全信息 */
    uid_t init_ns_uid;      // init 进程的 namespace UID
    gid_t init_ns_gid;      // init 进程的 namespace GID
    __u64 capability_mask;  // init 进程的 capability bounding set

    /* LSM */
    struct lsm_ops *lsm_ops;
    char *lsm_label;

    /* Namespace 相关 */
    int ns_inherited;       // 继承的 namespace 掩码
    int ns_fd[LXC_NS_MAX]; // 各 namespace 的 fd
};
```

### 2.2 两次 Fork 设计

`lxc_attach()` 使用两次 fork + `CLONE_PARENT` 的巧妙设计（`attach.c:1437-1824`）：

```
                    lxc_attach() [caller]
                         │
                    fork ①
                    ┌────┴────┐
                    │         │
               [parent]   [transient process]
                    │         │
                    │    setns() 进入容器 namespace
                    │         │
                    │    clone(CLONE_PARENT) ②
                    │    ┌────┴────┐
                    │    │         │
                    │  [transient] [attached process]
                    │    │         │
                    │    │    do_attach() → exec
                    │   exit      │
                    │             │
                    └─ waitpid ───┘
                    (attached process 的 parent 是 caller!)
```

**为什么需要 CLONE_PARENT？**

```
attach.c:1542-1548, 1612-1619

问题：
- parent 需要将 transient 进程加入容器的 cgroup
- 但 setns() 必须在 transient 进程内执行
- 如果 attached 进程是 transient 的子进程，
  它的 parent 就在容器 namespace 里

解决方案：
- transient 进程做 setns() 后，用 CLONE_PARENT 创建 attached 进程
- CLONE_PARENT 使 attached 进程的 parent 等同于 transient 的 parent
  即原始 caller 进程
- transient 进程随后退出
- caller 直接 waitpid attached 进程
```

### 2.3 详细执行流程

#### 阶段一：准备上下文

`get_attach_context()`，`attach.c:383-495`：

```
get_attach_context(container, options)
  1. 打开 /proc/self 和 /proc/<init_pid>
  2. 获取 init pidfd（通过 lxc_cmd_get_init_pidfd）
  3. 验证 pidfd 可用（发送信号 0 测试）
     如果不可用 → 回退到传统 PID 模式
  4. 自动检测容器的 namespace 和继承的 namespace
  5. 解析 init 进程的 /proc/<pid>/status
     提取：UID、GID、CapBnd（capability bounding set）
  6. 初始化 LSM 操作和标签
```

#### 阶段二：准备 Namespace

两种路径（`attach.c:546-624`）：

**pidfd 路径**（现代内核，`__prepare_namespaces_pidfd()`）：
```c
// 使用 pidfd_getfd() 获取容器进程的 namespace fd
// 更安全：避免 PID 重用导致的竞争
```

**nsfd 路径**（兼容旧内核，`__prepare_namespaces_nsfd()`）：
```c
// 打开 /proc/<pid>/ns/{user,mnt,pid,net,...}
// 传统方式，可能有 PID 重用风险
```

#### 阶段三：进入 Namespace

`attach_namespaces()`，`attach.c:680-701`：

**pidfd 路径**（`__attach_namespaces_pidfd()`）：
```c
// setns(pidfd, 0)  一次性进入所有 namespace
```

**nsfd 路径**（`__attach_namespaces_nsfd()`）：
```c
// 按顺序逐个 setns：
// user → mnt → pid → uts → ipc → net → cgroup → time
// 顺序很重要：user namespace 必须最先进入
```

#### 阶段四：安全设置

`do_attach()`，`attach.c:1134-1383`：

```
do_attach(payload)
  1. attach_namespaces()          // 进入容器 namespace
  2. 接收 LSM label fd            // 从 parent 通过 socket
  3. 设置附加组
     ├── LXC_ATTACH_SETGROUPS → setgroups(gids)
     └── 其他 → 清除或保持
  4. setgid(target_gid)           // 切换组
  5. setuid(target_uid)           // 切换用户
  6. 应用 LSM 标签
     process_label_set_at(label_fd, label)
  7. PR_SET_NO_NEW_PRIVS          // 防止权限提升
  8. drop_capabilities()          // 丢弃多余能力
  9. 设置环境变量
  10. 执行用户指定的函数或命令
```

#### 阶段五：IPC 与清理

```
IPC 机制：
  - socketpair 用于 parent 和 attached 进程通信
  - 同步协议确保安全的执行顺序
  - 安全屏障关闭敏感 fd（attach_context_security_barrier）

清理：
  - put_namespaces()       关闭 namespace fd
  - put_attach_context()   释放上下文资源
```

### 2.4 公共 API 调用

`lxccontainer.c:3985-4022`：

```c
// API 入口
lxcapi_attach(container, exec_function, exec_payload, options, pid) {
    current_config = c->lxc_conf;
    return lxc_attach(container->name, container->config_path,
                      exec_function, exec_payload, options, pid);
}

// 便捷封装：运行命令并等待
do_lxcapi_attach_run_wait(container, options, program, argv) {
    lxc_attach_command_t command = { program, argv };
    lxc_attach(c, lxc_attach_run_command, &command, options, &pid);
    return lxc_wait_for_pid_status(pid);
}
```

### 2.5 CLI 工具

`lxc_attach.c:321-457`：

```
lxc_attach_main()
  1. 构建 attach_options = LXC_ATTACH_OPTIONS_DEFAULT
  2. 设置 namespace 掩码、personality
  3. 设置 env_policy、uid/gid、LSM label
  4. if (有命令):
       c->attach_run_wait(c, options, program, argv)
  5. else:
       c->attach(c, NULL, NULL, options, &pid)  // 启动 shell
       lxc_wait_for_pid_status(pid)
```

---

## 3. 能力（Capabilities）管理

### 3.1 能力表

`conf.c:179-223`，`caps_opt[]` 静态表映射字符串到能力常量：

```c
static struct caps_opt caps_opt[] = {
    { "chown",              CAP_CHOWN },
    { "dac_override",       CAP_DAC_OVERRIDE },
    { "dac_read_search",    CAP_DAC_READ_SEARCH },
    { "fowner",             CAP_FOWNER },
    { "fsetid",             CAP_FSETID },
    { "kill",               CAP_KILL },
    { "setgid",             CAP_SETGID },
    { "setuid",             CAP_SETUID },
    { "setpcap",            CAP_SETPCAP },
    { "linux_immutable",    CAP_LINUX_IMMUTABLE },
    { "net_bind_service",   CAP_NET_BIND_SERVICE },
    { "net_broadcast",      CAP_NET_BROADCAST },
    { "net_admin",          CAP_NET_ADMIN },
    { "net_raw",            CAP_NET_RAW },
    { "ipc_lock",           CAP_IPC_LOCK },
    { "ipc_owner",          CAP_IPC_OWNER },
    { "sys_module",         CAP_SYS_MODULE },
    { "sys_rawio",          CAP_SYS_RAWIO },
    { "sys_chroot",         CAP_SYS_CHROOT },
    { "sys_ptrace",         CAP_SYS_PTRACE },
    { "sys_pacct",          CAP_SYS_PACCT },
    { "sys_admin",          CAP_SYS_ADMIN },
    { "sys_boot",           CAP_SYS_BOOT },
    { "sys_nice",           CAP_SYS_NICE },
    { "sys_resource",       CAP_SYS_RESOURCE },
    { "sys_time",           CAP_SYS_TIME },
    { "sys_tty_config",     CAP_SYS_TTY_CONFIG },
    { "mknod",              CAP_MKNOD },
    { "lease",              CAP_LEASE },
    { "audit_write",        CAP_AUDIT_WRITE },
    { "audit_control",      CAP_AUDIT_CONTROL },
    { "setfcap",            CAP_SETFCAP },
    // ... 更多
};
```

### 3.2 配置与解析

`confile.c:199-200`：

```
lxc.cap.drop = sys_admin net_admin   # 丢弃指定能力
lxc.cap.keep = net_bind_service      # 仅保留指定能力（其余丢弃）
```

**互斥关系**：`cap.drop` 和 `cap.keep` 不能同时使用。

解析器（`confile.c:2454-2545`）：
```c
add_cap_entry(conf, value, is_keep) {
    // 解析空格分隔的能力名列表
    // 通过 caps_opt[] 表将名称转为 CAP_* 常量
    // 添加到 conf->caps（drop）或 conf->keepcaps（keep）链表
}
```

### 3.3 能力执行

`conf.c:2827-2927`：

#### 判断逻辑

```c
has_cap(cap, conf) {
    if (conf->keepcaps 非空):
        // keep 模式：只有在 keepcaps 列表中的才保留
        return cap 在 keepcaps 列表中
    else:
        // drop 模式：不在 caps 列表中的都保留
        return cap 不在 caps 列表中
}
```

#### 丢弃能力

```c
capabilities_deny(conf) {
    // drop 模式：遍历 conf->caps 列表
    for (cap in conf->caps):
        prctl(PR_CAPBSET_DROP, cap)
}
```

#### 保留能力

```c
capabilities_allow(conf) {
    // keep 模式：丢弃不在 keepcaps 列表中的所有能力
    for (cap = 0; cap < CAP_LAST_CAP; cap++):
        if (cap 不在 conf->keepcaps 中):
            prctl(PR_CAPBSET_DROP, cap)
}
```

### 3.4 Ambient Capabilities

`caps.c:88-170`，用于非特权容器：

```c
lxc_ambient_caps_up() {
    // 提升 ambient capabilities
    // 在容器启动时，让非 root 进程也能获得必要能力
    prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_RAISE, cap, 0, 0);
}

lxc_ambient_caps_down() {
    // 清除所有 ambient capabilities
    prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_CLEAR_ALL, 0, 0, 0);
}
```

调用时机：
- `start.c:1273`：启动前提升 ambient caps
- `start.c:1627`：执行后清除 ambient caps

### 3.5 低级辅助函数

`caps.c:25-209`：

```c
lxc_caps_down() {
    // 临时降低 effective capabilities
    // 用于需要暂时放弃权限的操作
}

lxc_caps_up() {
    // 恢复 effective capabilities
}

lxc_caps_init() {
    // 初始化能力状态
    // 如果以 setuid root 运行，降低到实际用户权限
}
```

---

## 4. ID 映射

### 4.1 数据结构

`conf.h:178-192`：

```c
struct id_map {
    enum idtype {
        ID_TYPE_UID,
        ID_TYPE_GID,
    } idtype;

    unsigned long hostid;    // 主机端起始 ID
    unsigned long nsid;      // 容器端起始 ID
    unsigned long range;     // 映射范围大小
};
```

### 4.2 配置

`confile.c:232`：

```
# 格式：u/g nsid hostid range
lxc.idmap = u 0 100000 65536    # UID: 容器 0-65535 → 主机 100000-165535
lxc.idmap = g 0 100000 65536    # GID: 同上
```

解析器（`confile.c:2261-2303`）：
```c
set_config_idmaps(key, value, conf) {
    // 解析 "u/g nsid hostid range" 格式
    // 创建 id_map 结构体
    // 添加到 conf->id_map 链表
}
```

### 4.3 映射写入

`idmap_utils.c:23-65`，`write_id_mapping()`：

```c
write_id_mapping(type, pid, idmap_list) {
    // 构造映射文件路径
    path = "/proc/<pid>/uid_map" 或 "/proc/<pid>/gid_map"

    // 对于 GID 映射，需要先禁用 setgroups
    if (type == GID) {
        // 写入 /proc/<pid>/setgroups → "deny"
        // 这是安全要求：非特权用户必须先禁用 setgroups
    }

    // 写入映射："nsid hostid range\n"
    for (map in idmap_list):
        if (map->idtype == type):
            buf += sprintf("%lu %lu %lu\n", map->nsid, map->hostid, map->range)

    write(path, buf)
}
```

### 4.4 newuidmap/newgidmap 辅助工具

`idmap_utils.c:74-258`：

#### 权限检测

```c
idmaptool_on_path_and_privileged(tool_name) {
    // 检查 newuidmap/newgidmap 是否：
    // 1. 存在于 PATH 中
    // 2. 具有 setuid bit 或 CAP_SETUID/CAP_SETGID
    // 返回是否可用
}
```

#### 映射执行

```c
lxc_map_ids(conf, pid) {
    // 决策：直接写还是用辅助工具
    if (非特权 && newuidmap 可用):
        // 使用 newuidmap/newgidmap
        // 这些 setuid 工具可以设置超出用户权限范围的映射
        // 它们会验证 /etc/subuid 和 /etc/subgid
        run_command("newuidmap", pid, nsid1, hostid1, range1, ...)
        run_command("newgidmap", pid, nsid1, hostid1, range1, ...)
    else:
        // 直接写入 /proc/<pid>/uid_map
        write_id_mapping(UID, pid, conf->id_map)
        write_id_mapping(GID, pid, conf->id_map)
}
```

### 4.5 辅助函数

```c
// 查找主机 ID 对应的容器 ID
mapped_hostid(hostid, idmap_list)  // idmap_utils.c:260-273

// 查找未被映射的容器 ID
find_unmapped_nsid(conf, type)     // idmap_utils.c:275-292

// 添加映射条目
mapped_nsid_add(conf, nsid, type)  // idmap_utils.c:323-339
mapped_hostid_add(conf, hostid, type)  // idmap_utils.c:363-391
```

---

## 5. CRIU 集成

### 5.1 架构

LXC 不自行实现检查点/恢复，而是作为 CRIU 的"参数生成器"：

```
LXC                              CRIU
 │                                 │
 │  构建 criu 命令行参数            │
 │  设置容器状态                    │
 │  execv("criu", args)  ────────→ │
 │                                 │  执行 checkpoint/restore
 │  ←───────────────────────────── │
 │  检查退出状态                    │
```

### 5.2 核心实现

`criu.c`：

#### 参数生成

`exec_criu()`，`criu.c:159-611`：

```
exec_criu(type, opts)
  1. 基础参数（criu.c:183-289）
     criu dump/restore
       --images-dir <dir>
       --work-dir <dir>
       --shell-job          （如果是 shell）
       --manage-cgroups
       --file-locks
       --link-remap

  2. cgroup 路径映射（criu.c:290-375）
     --cgroup-root <controller>:<path>
     // 为每个 cgroup hierarchy 生成映射

  3. dump/pre-dump 特有参数（criu.c:377-451）
     --leave-running        （pre-dump 不停止容器）
     --tcp-established      （保存 TCP 连接）
     --ext-unix-sk          （外部 Unix socket）
     --ghost-limit          （Ghost 文件大小限制）

  4. restore 特有参数（criu.c:452-580）
     --restore-detached
     --pidfile <file>
     --root <rootfs>
     // 网络设备恢复
     --veth-pair <host>=<container>
     --ext-mount-map <src>=<dst>

  5. execv("criu", argv)
```

#### 公共入口

```c
__criu_pre_dump(handler)    // criu.c:1319
    // 增量 checkpoint（不停止容器）

__criu_dump(handler)        // criu.c:1324
    // 完整 checkpoint（停止容器）

__criu_restore(handler)     // criu.c:1341
    // 从 checkpoint 恢复容器
```

#### 特性检测

`__criu_check_feature()`，`criu.c:628`：

```c
// 检查 CRIU 版本和支持的特性
// 决定可以使用哪些参数
```

---

## 6. 安全机制协作图

```
容器启动安全流程:

  ┌─────────────────────────────────────────────────┐
  │                    启动阶段                       │
  │                                                  │
  │  1. LSM prepare → 生成/加载安全策略               │
  │  2. Capabilities init → 设置 ambient caps        │
  │  3. ID mapping → 写入 uid_map/gid_map            │
  │  4. User namespace → CLONE_NEWUSER               │
  │                                                  │
  ├─────────────────────────────────────────────────┤
  │                  容器进程内部                      │
  │                                                  │
  │  5. Capabilities enforce → drop/keep              │
  │  6. Seccomp load → 加载系统调用过滤器              │
  │  7. LSM label → 设置进程安全标签                   │
  │  8. PR_SET_NO_NEW_PRIVS → 防止权限提升            │
  │                                                  │
  ├─────────────────────────────────────────────────┤
  │                    运行阶段                       │
  │                                                  │
  │  9. Seccomp notify → 拦截敏感系统调用             │
  │ 10. LSM → 强制访问控制                            │
  │ 11. Capabilities → 权限边界                       │
  │                                                  │
  └─────────────────────────────────────────────────┘

Attach 安全流程:

  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │  1. 获取目标   │    │  2. 进入      │    │  3. 安全设置  │
  │     namespace │───→│     namespace │───→│              │
  │  - pidfd      │    │  - setns()   │    │  - setuid    │
  │  - ns fds     │    │  - 顺序进入   │    │  - setgid    │
  │  - caps 检查  │    │              │    │  - LSM label │
  │              │    │              │    │  - drop caps │
  │              │    │              │    │  - NO_NEW_PRIVS│
  └──────────────┘    └──────────────┘    └──────────────┘
```

---

## 7. 设计总结

1. **两次 Fork + CLONE_PARENT**：Attach 机制通过精巧的进程创建模式解决了"需要 parent 设置 cgroup，但 child 需要在容器 namespace 中"的矛盾。

2. **pidfd 优先**：现代内核下使用 pidfd 避免 PID 重用竞争，旧内核自动回退到 nsfd。

3. **能力互斥设计**：`cap.drop`（黑名单）和 `cap.keep`（白名单）互斥使用，避免配置冲突。

4. **Ambient Capabilities**：支持非 root 容器进程获得必要能力，是非特权容器的关键特性。

5. **ID 映射双路径**：特权模式直接写 `/proc/<pid>/uid_map`，非特权模式委托给 setuid 辅助工具（newuidmap/newgidmap）。

6. **CRIU 解耦**：LXC 不嵌入 CRIU 逻辑，而是作为参数生成器，降低耦合度并支持 CRIU 独立升级。

7. **纵深防御**：多层安全机制协作——Namespace 隔离 + Capabilities 限制 + Seccomp 过滤 + LSM 强制访问 + NO_NEW_PRIVS 锁定，任一层被绕过仍有其他层保护。
