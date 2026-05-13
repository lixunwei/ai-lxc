# LXC 深度分析：lxc_setup() 容器设置编排

## 1. 概述

`lxc_setup()` 是 LXC 容器子进程内的 **核心设置编排函数**，在 `clone()` 创建的子进程中运行，
负责将一个空白的命名空间环境转变为一个可用的容器。

**调用位置**：`src/lxc/start.c:1460-1464`，在子进程收到 `START_SYNC_POST_CONFIGURE` 同步信号后调用。

**函数位置**：`src/lxc/conf.c:3764-3895`

---

## 2. 调用时序

```
lxc_spawn()                                      [start.c]
  ├── clone() → 子进程
  │     ├── 等待 START_SYNC_POST_CONFIGURE（父进程完成 cgroup/网络等）
  │     ├── ★ lxc_setup(handler)                  [start.c:1460-1464]
  │     ├── LSM 进程标签设置
  │     ├── PR_SET_NO_NEW_PRIVS
  │     ├── seccomp 加载
  │     ├── start 钩子
  │     └── exec(init_cmd)
  │
  └── 父进程：
        ├── 配置 cgroup
        ├── 配置网络
        ├── 发送 START_SYNC_POST_CONFIGURE
        └── 等待子进程
```

---

## 3. lxc_setup() 执行步骤详解

### 步骤 1：准备子进程侧 rootfs

```c
lxc_rootfs_prepare_child(handler)                [conf.c:3766-3772]
```

准备子进程侧的 rootfs 状态。读取 `handler->conf->rootfs` 和同步状态。
失败则中止：`"Failed to prepare rootfs"`

### 步骤 2：准备根文件系统

```c
lxc_setup_rootfs_prepare_root(handler)           [conf.c:3774-3776]
  → 辅助函数在 conf.c:3366-3404
```

核心操作：
- 打开 `/` 作为 `rootfs.dfd_host`
- 标记依赖挂载
- 挂载/绑定 rootfs
- 运行 `pre-mount` 钩子
- 调用 `lxc_mount_rootfs()` 执行实际挂载

系统调用：`openat()`, `mount()`, 钩子进程 spawn

### 步骤 3：设置 UTS 主机名

```c
if (!handler->nsfd[LXC_NS_UTS])
    setup_utsname(lxc_conf->utsname)             [conf.c:3778-3782]
```

通过 `sethostname()` 在 UTS 命名空间中设置主机名。
仅在没有共享 UTS 命名空间 fd 时执行。

### 步骤 4：会话密钥环

```c
lxc_setup_keyring(lsm_se_context, ...)           [conf.c:3784-3788]
```

除非 `keyring_disable_session`，调用 `keyctl(KEYCTL_JOIN_SESSION_KEYRING)` 创建新的内核密钥环会话。
同时设置 LSM 密钥环标签。

### 步骤 5：网络配置

```c
if (CLONE_NEWNET) {
    从父进程接收 veth 名称                         [conf.c:3790-3794]
    配置容器网络命名空间                           [conf.c:3796-3798]
}
```

子进程侧的网络配置：设置 IP 地址、网关、启用接口等。
使用 netlink 和 ioctl 系统调用。

### 步骤 6：/dev 自动挂载

```c
if (autodev > 0)
    lxc_setup_autodev(handler)                    [conf.c:3800-3804]
```

在容器的 `/dev` 上挂载 tmpfs，大小由 `autodevtmpfssize` 配置。

### 步骤 7：第一阶段自动挂载（proc/sys）

```c
lxc_mount_auto_mounts(handler,
    auto_mounts & ~LXC_AUTO_CGROUP_MASK)          [conf.c:3806-3811]
```

挂载 `/proc`、`/sys` 等基本文件系统（**排除 cgroup**，留到后面）。
根据配置选择 rw/ro/mixed 模式。

### 步骤 8：fstab 挂载

```c
setup_mount_fstab(&rootfs, fstab, name, lxcpath) [conf.c:3813-3815]
```

如果配置了 `lxc.mount.fstab`，打开外部 fstab 文件，逐条执行挂载。

### 步骤 9：内联挂载条目

```c
setup_mount_entries(&rootfs, mount_entries, ...)  [conf.c:3817-3821]
```

处理所有 `lxc.mount.entry` 配置项。遍历链表，逐条解析并执行挂载。

### 步骤 10：ID 映射挂载同步

```c
lxc_sync_wake_parent(START_SYNC_IDMAPPED_MOUNTS) [conf.c:3823-3826]
lxc_idmapped_mounts_child(handler)               [conf.c:3827-3828]
```

**父子进程协作**：
1. 子进程通知父进程已准备好 idmapped 挂载
2. 父进程创建 detached 的 idmapped 挂载
3. 子进程接收并 attach 这些挂载

辅助函数 `lxc_idmapped_mounts_child()`（`conf.c:2776-2824`）处理实际的挂载附加。

### 步骤 11：打开 /dev 目录 fd

```c
openat(rootfs.dfd_mnt, "dev", ...)
  → rootfs.dfd_dev                               [conf.c:3830-3833]
```

为后续 devpts 和设备节点操作准备目录 fd。`ENOENT` 错误被容忍。

### 步骤 12：第二阶段自动挂载（cgroup）

```c
lxc_mount_auto_mounts(handler,
    auto_mounts & LXC_AUTO_CGROUP_MASK)           [conf.c:3835-3841]
```

现在 `/sys` 已经可用，完成 cgroup 文件系统的挂载。
调用 `cgroup_ops->mount()` 委托给 cgroup 驱动。

### 步骤 13：mount 钩子

```c
run_hooks(hooks[LXCHOOK_MOUNT])                   [conf.c:3843-3845]
```

执行 `lxc.hook.mount` 配置的脚本。此时 rootfs 和所有挂载已完成，但尚未 pivot_root。

### 步骤 14：rootfs 覆盖检查

```c
lxc_rootfs_overmounted(rootfs)                    [conf.c:3847-3848]
```

验证 rootfs 没有被意外覆盖挂载。

### 步骤 15：/dev 设备填充

```c
run_hooks(hooks[LXCHOOK_AUTODEV])                 [conf.c:3850-3852]
lxc_setup_autodev(handler)                        [conf.c:3854-3858]
```

1. 运行 `lxc.hook.autodev` 钩子
2. 在 `/dev` 下创建基本设备节点：
   - `mknod()` 创建 null, zero, full, random, urandom, tty 等
   - 绑定挂载 /dev/console
   - 创建设备符号链接

### 步骤 16：验证 start 钩子

```c
verify_start_hooks(hooks[LXCHOOK_START], rootfs)  [conf.c:3860-3862]
```

检查 `lxc.hook.start` 指定的脚本在容器 rootfs 内是否存在（`access()` 调用）。

### 步骤 17：临时 procfs（LSM 用）

```c
lxc_setup_proc_filesystem(transient_procfs_mnt)   [conf.c:3864-3866]
```

挂载临时 procfs，供 LSM 操作使用。

### 步骤 18：devpts 设置

```c
lxc_setup_devpts_child(handler)                   [conf.c:3868-3871]
lxc_finish_devpts_child(handler)                  [conf.c:3872-3874]
```

创建新的 devpts 实例并挂载到 `/dev/pts`：

```
lxc_setup_devpts_child()                          [conf.c:1432-1594]
  ├── umount2("/dev/pts", MNT_DETACH)            // 卸载旧 devpts
  ├── mkdirat(dfd_dev, "pts", 0755)              // 创建 pts 目录
  ├── fsopen("devpts") → fsconfig() → fsmount()  // 新式挂载 API
  │     options: gid=5, newinstance, ptmxmode=0666,
  │              mode=0620, max=<pty_max>
  ├── move_mount() → /dev/pts                    // 移动挂载到位
  ├── unlinkat(dfd_dev, "ptmx")                  // 删除旧 ptmx
  ├── mknodat(dfd_dev, "ptmx", ...)              // 创建新 ptmx 设备
  │   或 symlinkat("pts/ptmx", dfd_dev, "ptmx")  // 或创建符号链接
  └── 设置正确权限
```

### 步骤 19：控制台设置

```c
lxc_setup_console(handler, rootfs, console, ...)  [conf.c:3876-3879]
```

将 PTY 从端绑定挂载到 `/dev/console`，设置 stdin/stdout/stderr。

### 步骤 20：TTY 设备创建

```c
lxc_setup_ttys(handler)                           [conf.c:3881-3883]
```

根据 `lxc.tty.max` 创建额外的 TTY 设备（`/dev/tty1`, `/dev/tty2`, ...），
通过 PTY 分配 + 绑定挂载实现。

### 步骤 21：/dev 符号链接

```c
lxc_setup_dev_symlinks(&rootfs)                   [conf.c:3885-3887]
  → 辅助函数在 conf.c:688-717
```

确保以下标准符号链接存在：

| 链接 | 目标 |
|------|------|
| `/dev/fd` | `/proc/self/fd` |
| `/dev/stdin` | `/proc/self/fd/0` |
| `/dev/stdout` | `/proc/self/fd/1` |
| `/dev/stderr` | `/proc/self/fd/2` |

使用 `fstatat()` + `symlinkat()`。`EEXIST` 和 `EROFS` 被容忍。

### 步骤 22：切换根文件系统

```c
lxc_setup_rootfs_switch_root(&rootfs)             [conf.c:3889-3891]
  → 辅助函数在 conf.c:1403-1411
```

执行最关键的根切换：

```c
if (无 rootfs):     返回成功（不切换）
if (ramfs rootfs):  lxc_chroot()
else:               lxc_pivot_root()
```

`pivot_root()` 将旧根移到临时位置，新 rootfs 成为 `/`，然后卸载旧根。

### 步骤 23：共享挂载传播

```c
lxc_setup_shmount(auto_mounts, shmount)           [conf.c:3893-3895]
```

如果配置了共享挂载隧道，设置 `MS_REC|MS_SLAVE` 传播属性。

### 步骤 24：Boot ID 隔离

```c
if (autodev > 0)
    lxc_setup_boot_id()                           [best effort]
```

为容器生成独立的 `/proc/sys/kernel/random/boot_id`。通过绑定挂载 + 只读重挂载实现。
失败被忽略（best effort）。

### 步骤 25：CPU 架构（personality）

```c
setup_personality(lxc_conf->personality)           [conf.c:1597-1610]
```

如果配置了 `lxc.arch`，调用 `personality()` 系统调用设置进程执行域。
例如在 x86_64 上运行 32 位容器。

### 步骤 26：内核参数与资源限制

```c
setup_sysctl_parameters(lxc_conf)                 // 写 /proc/sys/*
setup_proc_filesystem(lxc_conf)                   // 写 /proc/self/*
setup_resource_limits(lxc_conf)                    // setrlimit()
setup_capabilities(lxc_conf)                       // capability 设置
```

---

## 4. 执行步骤全景图

```
lxc_setup(handler)
  │
  │ ──── 文件系统准备 ────
  ├─ 1. rootfs 子进程准备
  ├─ 2. rootfs 挂载（+ pre-mount 钩子）
  │
  │ ──── 基本身份 ────
  ├─ 3. UTS 主机名
  ├─ 4. 会话密钥环
  │
  │ ──── 网络 ────
  ├─ 5. 接收 veth + 配置网络
  │
  │ ──── 设备层 ────
  ├─ 6. /dev tmpfs 挂载
  │
  │ ──── 挂载层（两阶段） ────
  ├─ 7. 自动挂载 proc/sys
  ├─ 8. fstab 挂载
  ├─ 9. 内联挂载条目
  ├─10. idmapped 挂载同步
  ├─11. 打开 /dev fd
  ├─12. 自动挂载 cgroup
  │
  │ ──── 钩子与验证 ────
  ├─13. mount 钩子
  ├─14. rootfs 覆盖检查
  ├─15. autodev 钩子 + 设备填充
  ├─16. 验证 start 钩子存在
  │
  │ ──── 终端层 ────
  ├─17. 临时 procfs
  ├─18. devpts 设置
  ├─19. 控制台设置
  ├─20. TTY 设备创建
  ├─21. /dev 符号链接
  │
  │ ──── 根切换 ────
  ├─22. pivot_root / chroot
  ├─23. 共享挂载传播
  ├─24. boot_id 隔离
  │
  │ ──── 进程属性 ────
  ├─25. CPU 架构
  └─26. sysctl + proc + rlimit + capabilities
```

---

## 5. 父子进程同步点

`lxc_setup()` 内部包含与父进程的关键同步：

| 同步信号 | 方向 | 时机 | 用途 |
|----------|------|------|------|
| `START_SYNC_POST_CONFIGURE` | 父→子 | setup 开始前 | 父进程完成 cgroup/网络配置 |
| `START_SYNC_IDMAPPED_MOUNTS` | 子→父 | 步骤 10 | 请求父进程创建 idmapped 挂载 |

### 5.1 idmapped 挂载协作

```
子进程                              父进程
  │                                   │
  ├── 挂载 fstab + entries            │
  ├── 发送 IDMAPPED_MOUNTS ─────────→│
  │                                   ├── 创建 detached idmapped 挂载
  │                                   ├── 通过 SCM_RIGHTS 传递 fd
  │   ←────────────────────────────── │
  ├── 接收 fd                         │
  └── move_mount() 附加到容器         │
```

---

## 6. 错误处理策略

- **大多数步骤**：失败直接返回错误，中止容器启动
- **少数 best-effort 步骤**：
  - `/dev` 目录 fd 打开（`ENOENT` 容忍）
  - `/dev` 符号链接创建（`EEXIST`、`EROFS` 容忍）
  - boot_id 设置（完全忽略失败）
- **钩子失败**：中止启动（钩子脚本返回非零退出码）
- **每个步骤**都有对应的 `log_error()` 或 `syserror()` 日志输出
