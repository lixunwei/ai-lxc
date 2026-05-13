# LXC 深度分析：容器内 /proc 与 /sys 的可见性控制

## 1. 概述

LXC 通过精心设计的挂载策略控制容器内 `/proc` 和 `/sys` 的可见性。核心机制包括：

- **自动挂载表**（`default_mounts[]`）：定义不同模式下的挂载行为
- **Bind mount 遮蔽**：将敏感路径绑定挂载为只读或挂载 `/dev/null`
- **只读重挂载**：防止容器修改宿主内核参数
- **AppArmor 策略**：在 LSM 层面禁止访问危险文件

核心实现位于 `src/lxc/conf.c`，配置解析在 `src/lxc/confile.c`。

---

## 2. 配置项：`lxc.mount.auto`

### 2.1 解析

`confile.c:2317-2415`，支持以下值：

#### /proc 挂载模式

| 配置值 | 效果 |
|--------|------|
| `proc` / `proc:rw` | 以读写方式挂载 `/proc` |
| `proc:mixed` | 挂载 `/proc`，但 `/proc/sys` 只读（保留 `/proc/sys/net` 可写） |

#### /sys 挂载模式

| 配置值 | 效果 |
|--------|------|
| `sys:rw` | 以读写方式挂载 `/sys` |
| `sys:ro` | 以只读方式挂载 `/sys` |
| `sys:mixed` | `/sys` 只读，但 `/sys/devices/virtual/net` 可写 |

#### cgroup 挂载模式

| 配置值 | 效果 |
|--------|------|
| `cgroup` / `cgroup:mixed` | 自动挂载 cgroup（默认权限） |
| `cgroup:ro` | 只读挂载 cgroup |
| `cgroup:rw` | 读写挂载 cgroup |
| `cgroup:force` | 强制挂载（即使无 CAP_SYS_ADMIN） |
| `cgroup-full` | 完整 cgroup 挂载（含所有子系统） |
| `cgroup2` / `cgroup2:ro` / `cgroup2:force` | cgroup v2 挂载 |
| `shmounts:<host>:<container>` | 共享挂载辅助 |

### 2.2 标志位定义

`conf.h:264-336`，使用位掩码组合：

```c
#define LXC_AUTO_PROC_RW       0x001
#define LXC_AUTO_PROC_MIXED    0x002
#define LXC_AUTO_SYS_RW        0x004
#define LXC_AUTO_SYS_RO        0x008
#define LXC_AUTO_SYS_MIXED     0x010
#define LXC_AUTO_CGROUP_RO     0x020
#define LXC_AUTO_CGROUP_RW     0x040
#define LXC_AUTO_CGROUP_MIXED  0x080
#define LXC_AUTO_CGROUP_FULL   0x100
// ... 更多组合
```

---

## 3. 自动挂载表

### 3.1 `default_mounts[]` 数组

`conf.c:483-512`，定义了所有自动挂载条目。以下是完整的挂载表：

#### /proc 相关条目

```c
/* proc:rw — 简单挂载 */
{ LXC_AUTO_PROC_RW, "%r/proc", "proc", "proc", 0, NULL },

/* proc:mixed — 挂载后遮蔽敏感路径 */
{ LXC_AUTO_PROC_MIXED, "%r/proc", "proc", "proc", 0, NULL },
// 步骤1：移动 /proc/sys/net 到临时位置
{ LXC_AUTO_PROC_MIXED, "%r/proc/sys/net", "%r/proc/tty", NULL, MS_BIND, NULL },
// 步骤2：将 /proc/sys 绑定挂载为自身（准备重挂载）
{ LXC_AUTO_PROC_MIXED, "%r/proc/sys", "%r/proc/sys", NULL, MS_BIND, NULL },
// 步骤3：重挂载 /proc/sys 为只读
{ LXC_AUTO_PROC_MIXED, NULL, "%r/proc/sys", NULL, MS_REMOUNT|MS_BIND|MS_RDONLY, NULL },
// 步骤4：将 /proc/sys/net 移回原位（保持可写）
{ LXC_AUTO_PROC_MIXED, "%r/proc/tty", "%r/proc/sys/net", NULL, MS_BIND, NULL },
// 步骤5：/proc/sysrq-trigger 只读
{ LXC_AUTO_PROC_MIXED, "%r/proc/sysrq-trigger", "%r/proc/sysrq-trigger", NULL, MS_BIND, NULL },
{ LXC_AUTO_PROC_MIXED, NULL, "%r/proc/sysrq-trigger", NULL, MS_REMOUNT|MS_BIND|MS_RDONLY, NULL },
```

#### /sys 相关条目

```c
/* sys:rw — 读写挂载 */
{ LXC_AUTO_SYS_RW, "%r/sys", "sysfs", "sysfs", 0, NULL },

/* sys:ro — 只读挂载 */
{ LXC_AUTO_SYS_RO, "%r/sys", "sysfs", "sysfs", MS_RDONLY, NULL },

/* sys:mixed — 只读但保留网络设备可写 */
// 步骤1：在 /proc/sys 下临时挂载一个读写 sysfs
{ LXC_AUTO_SYS_MIXED, "%r/proc/sys", "sysfs", "sysfs", 0, NULL },
// 步骤2：挂载只读 sysfs 到 /sys
{ LXC_AUTO_SYS_MIXED, "%r/sys", "sysfs", "sysfs", MS_RDONLY, NULL },
// 步骤3：从临时 sysfs 绑定 devices/virtual/net 到 /sys（可写）
{ LXC_AUTO_SYS_MIXED, "%r/proc/sys/devices/virtual/net", "%r/sys/devices/virtual/net",
  NULL, MS_BIND, NULL },
// 步骤4：卸载临时 sysfs
{ LXC_AUTO_SYS_MIXED, NULL, "%r/proc/sys", NULL, /* umount */, NULL },
```

### 3.2 `%r` 占位符

`%r` 表示容器的 rootfs 路径，在实际挂载时被替换为 `conf->rootfs.path` + `conf->rootfs.mount`。

---

## 4. 挂载执行引擎

### 4.1 `lxc_mount_auto_mounts()`

`conf.c:471-657`，自动挂载的核心执行函数：

```
lxc_mount_auto_mounts(handler, flags)
  1. 准备 /proc 挂载点（518-539）
     ├── 检查 /proc 是否已存在
     ├── 必要时创建目录
     └── 确保挂载点可用

  2. 准备 /sys 挂载点（541-562）
     └── 同上

  3. 遍历 default_mounts[] 表（565-616）
     for entry in default_mounts[]:
         if (entry.flags & requested_flags):
             替换 %r 占位符
             执行挂载操作

  4. 挂载 cgroup（618-648）
     ├── 根据 CAP_SYS_ADMIN 决定默认权限
     └── 调用 handler->cgroup_ops->mount()

  5. 共享挂载辅助（650-653）
```

### 4.2 调用时机

自动挂载在 `lxc_setup()` 中被调用两次（`conf.c:3764-3921`）：

```
lxc_setup() 流程:

  1. lxc_rootfs_prepare_child()           // 准备 rootfs
  2. lxc_setup_rootfs_prepare_root()      // 设置根文件系统（3774-3776）
  3. mount_autodev()                      // 挂载 /dev tmpfs（3800-3804）

  4. ★ 第一次 automount：proc + sys      // （3806-3811）
     lxc_mount_auto_mounts(flags & ~CGROUP)
     // 只挂载 proc 和 sys，不挂载 cgroup
     // 因为 cgroup 可能依赖 /sys 存在

  5. setup_mount_fstab()                  // 处理 lxc.mount.fstab（3813-3815）
  6. setup_mount_entries()                // 处理 lxc.mount.entry（3817-3820）
  7. 处理 idmapped mounts（3823-3828）

  8. ★ 第二次 automount：cgroup only     // （3835-3841）
     lxc_mount_auto_mounts(flags & CGROUP)
     // 此时 /sys 已存在，可以安全挂载 /sys/fs/cgroup

  9. 运行 mount hooks（3843-3848）
  10. lxc_setup_boot_id()                 // 替换 boot_id（3850-3852）
  11. lxc_setup_devpts()                  // 设置 devpts
  12. lxc_setup_rootfs_switch_root()      // pivot_root（3887-3891）
  13. setup_sysctl_parameters()           // 写入 sysctl（3906-3912）
```

---

## 5. `/proc` 可见性控制详解

### 5.1 `proc:rw` 模式

```
/proc                       ← proc 文件系统，完全读写
├── sys/                    ← 可写（可修改内核参数！）
│   ├── net/               ← 可写
│   └── kernel/            ← 可写
├── sysrq-trigger           ← 可写（危险！）
├── kcore                   ← 可读（但受 AppArmor 限制）
└── ...
```

**风险**：容器可以修改宿主内核参数（如 `vm.swappiness`）。

### 5.2 `proc:mixed` 模式（推荐）

```
/proc                       ← proc 文件系统
├── sys/                    ← 只读（bind + remount ro）
│   └── net/               ← 可写（移回可写挂载）
├── sysrq-trigger           ← 只读（bind + remount ro）
└── 其余                    ← 正常读写
```

**实现技巧**：`/proc/sys/net` 的保护

LXC 使用 `/proc/tty` 作为临时中转点，通过以下步骤保留 `/proc/sys/net` 的可写性：

```
原始状态：/proc/sys/net (rw)

步骤1: bind mount /proc/sys/net → /proc/tty      （保存 net）
步骤2: bind mount /proc/sys → /proc/sys           （准备 remount）
步骤3: remount /proc/sys → readonly               （只读化整个 sys）
步骤4: bind mount /proc/tty → /proc/sys/net       （恢复 net 可写）

最终：/proc/sys (ro), /proc/sys/net (rw)
```

**为什么保留 `/proc/sys/net` 可写？**
容器有自己的网络命名空间，`/proc/sys/net` 下的参数是 per-namespace 的，修改只影响容器自己，不会影响宿主。

### 5.3 boot_id 替换

`conf.c:3432-3477`，`lxc_setup_boot_id()`：

```c
// 每个容器获得唯一的 boot_id
// 防止容器读取宿主的真实 boot_id

1. 生成随机 UUID → 写入 /dev/.lxc-boot-id
2. bind mount /dev/.lxc-boot-id → /proc/sys/kernel/random/boot_id
3. remount readonly
```

### 5.4 AppArmor 层面的额外保护

`lsm/apparmor.c:90-111`，即使 proc 以读写模式挂载，AppArmor 也会阻止：

```
deny @{PROC}/kcore           rwklx,   # 内核内存映像
deny @{PROC}/sysrq-trigger   rwklx,   # SysRq 触发器
deny @{PROC}/sys/fs/**        wklx,   # 文件系统参数写入
# 但允许 /proc/sys/fs/binfmt_misc（注册二进制格式）
```

---

## 6. `/sys` 可见性控制详解

### 6.1 `sys:rw` 模式

```
/sys                         ← sysfs，完全读写
├── devices/                 ← 可写
├── class/                   ← 可写
├── fs/cgroup/              ← 可写
└── ...
```

### 6.2 `sys:ro` 模式

```
/sys                         ← sysfs，完全只读
└── 所有内容                  ← 只读
```

### 6.3 `sys:mixed` 模式（推荐）

```
/sys                                ← sysfs，只读
├── devices/virtual/net/           ← 可写（bind mount 自临时 sysfs）
│   ├── eth0/                      ← 可写
│   └── lo/                        ← 可写
└── 其余                            ← 只读
```

**实现技巧**：

```
步骤1: mount sysfs (rw) → /proc/sys           （临时挂载点）
步骤2: mount sysfs (ro) → /sys                （只读目标）
步骤3: bind /proc/sys/devices/virtual/net
         → /sys/devices/virtual/net            （可写网络设备）
步骤4: umount /proc/sys                        （清理临时挂载）
```

**为什么保留 `/sys/devices/virtual/net` 可写？**
同 `/proc/sys/net` 的理由——网络设备信息是 per-namespace 的，容器需要管理自己的网络设备参数。

---

## 7. 各模式可见性对比

### 7.1 /proc 对比

| 路径 | `proc:rw` | `proc:mixed` |
|------|-----------|--------------|
| `/proc` | rw | rw |
| `/proc/sys` | rw | **ro** |
| `/proc/sys/net` | rw | rw |
| `/proc/sysrq-trigger` | rw | **ro** |
| `/proc/kcore` | rw* | rw* |
| `/proc/sys/kernel/random/boot_id` | 宿主值 | **容器专属值(ro)** |

\* 受 AppArmor 策略限制

### 7.2 /sys 对比

| 路径 | `sys:rw` | `sys:ro` | `sys:mixed` |
|------|----------|----------|-------------|
| `/sys` | rw | ro | ro |
| `/sys/devices/virtual/net` | rw | ro | **rw** |
| `/sys/fs/cgroup` | rw | ro | ro |
| `/sys/class` | rw | ro | ro |

---

## 8. 挂载处理函数

### 8.1 `mount_entry()`

`conf.c:2075-2174`，通用挂载入口：

```c
mount_entry(source, target, fstype, flags, opts, rootfs, ...) {
    // 1. 如果是 bind mount + rdonly，先 bind 再 remount
    // 2. 调用 mount() 系统调用
    // 3. 处理 MS_REMOUNT|MS_BIND 的组合
    //    必须保留原有的 nosuid/nodev/noexec 标志
    //    通过 statvfs() 读取当前标志后合并
}
```

### 8.2 挂载标志保留

`conf.c:2105-2154`，关键安全逻辑：

```c
// 执行 bind remount 时，必须保留底层文件系统的安全标志
struct statvfs sb;
statvfs(target, &sb);

// 合并原有标志
if (sb.f_flag & ST_NOSUID) flags |= MS_NOSUID;
if (sb.f_flag & ST_NODEV)  flags |= MS_NODEV;
if (sb.f_flag & ST_RDONLY) flags |= MS_RDONLY;
if (sb.f_flag & ST_NOEXEC) flags |= MS_NOEXEC;
```

这防止了通过 remount 去除安全限制的攻击。

### 8.3 `mount_entry_on_systemfs()`

`conf.c:2353-2369`，处理系统文件系统（proc/sys）上的自定义挂载。

### 8.4 LXC 特有挂载选项

`conf.c:2185-2244`，`parse_lxc_mount_attrs()`：

| 选项 | 说明 |
|------|------|
| `create=dir` | 如果挂载点不存在，自动创建目录 |
| `create=file` | 如果挂载点不存在，自动创建文件 |
| `optional` | 挂载失败不报错 |
| `relative` | 使用相对于 rootfs 的路径 |
| `idmap=<path>` | 使用指定的 ID 映射进行挂载 |

### 8.5 标准挂载标志

`conf.c:132-164`，`mount_opt[]` 表：

```c
static struct mount_opt mount_opt[] = {
    { "atime",       MS_NOATIME,     true  },  // 取反：去掉 NOATIME
    { "noatime",     MS_NOATIME,     false },
    { "dev",         MS_NODEV,       true  },
    { "nodev",       MS_NODEV,       false },
    { "exec",        MS_NOEXEC,      true  },
    { "noexec",      MS_NOEXEC,      false },
    { "suid",        MS_NOSUID,      true  },
    { "nosuid",      MS_NOSUID,      false },
    { "ro",          MS_RDONLY,      false },
    { "rw",          MS_RDONLY,      true  },
    { "bind",        MS_BIND,       false },
    { "rbind",       MS_BIND|MS_REC, false },
    { "remount",     MS_REMOUNT,    false },
    // ...
};
```

---

## 9. Rootfs 切换

### 9.1 pivot_root

`conf.c:1328-1400`，`lxc_pivot_root()`：

```
lxc_pivot_root(rootfs)
  1. mount("", "/", NULL, MS_REC|MS_PRIVATE, NULL)
     // 确保挂载传播为 private，防止泄漏
  2. mount(rootfs, rootfs, NULL, MS_BIND|MS_REC, NULL)
     // 绑定挂载 rootfs 到自身
  3. chdir(rootfs)
  4. pivot_root(".", ".")
     // 新 root = rootfs, 旧 root = rootfs (然后 umount)
  5. umount2(".", MNT_DETACH)
     // 卸载旧的根文件系统
  6. chdir("/")
```

### 9.2 chroot 回退

`conf.c:1212-1300`，`lxc_chroot()`：

当 rootfs 是 ramfs 时（`pivot_root` 不支持 ramfs），使用 chroot：

```
lxc_chroot(rootfs)
  1. mount(rootfs, rootfs, NULL, MS_BIND|MS_REC, NULL)
  2. chroot(rootfs)
  3. chdir("/")
  4. 逐个卸载旧挂载点
```

### 9.3 选择逻辑

`conf.c:1403-1412`：

```c
lxc_setup_rootfs_switch_root() {
    if (rootfs_is_ramfs())
        return lxc_chroot(rootfs);
    else
        return lxc_pivot_root(rootfs);
}
```

---

## 10. 设备节点创建

### 10.1 /dev tmpfs

`conf.c:990-1048`，`mount_autodev()`：

```c
// 在容器 rootfs 的 /dev 上挂载 tmpfs
mount("none", dev_path, "tmpfs", MS_NOSUID, "size=500000,mode=755");
```

### 10.2 标准设备节点

`conf.c:1058-1065`，`lxc_fill_autodev()` 创建的设备：

```c
static struct lxc_device_node lxc_devices[] = {
    { "/dev/full",    S_IFCHR, 1, 7  },
    { "/dev/null",    S_IFCHR, 1, 3  },
    { "/dev/random",  S_IFCHR, 1, 8  },
    { "/dev/tty",     S_IFCHR, 5, 0  },
    { "/dev/urandom", S_IFCHR, 1, 9  },
    { "/dev/zero",    S_IFCHR, 1, 5  },
};
```

### 10.3 符号链接

`conf.c:681-686`，`dev_symlinks[]`：

```c
{ "/proc/self/fd",   "/dev/fd"     },
{ "/proc/self/fd/0", "/dev/stdin"  },
{ "/proc/self/fd/1", "/dev/stdout" },
{ "/proc/self/fd/2", "/dev/stderr" },
```

---

## 11. 嵌套容器支持

### 11.1 嵌套辅助挂载

`conf.c:2492-2494`，`nesting_helpers`：

```c
// 为嵌套容器提供额外的 proc/sys 挂载点
{ "dev/.lxc/proc", "proc", "proc", ... },
{ "dev/.lxc/sys",  "sys",  "sysfs", ... },
```

当 `lxc.mount.auto` 配置中包含嵌套支持时，这些辅助挂载会被添加到 `/dev/.lxc/` 下。内层容器可以从这些位置获取"干净的"proc 和 sysfs 实例。

### 11.2 通过 `make_anonymous_mount_file()`

`conf.c:2482-2553`：

```c
// 如果检测到需要嵌套支持
// 在生成的 fstab 中追加 nesting_helpers 条目
// 这些条目会在 setup_mount_fstab() 中被处理
```

---

## 12. 完整挂载流程图

```
容器启动
  │
  ├── lxc_setup_rootfs_prepare_root()
  │     └── 设置 rootfs 挂载（bind/loop/...）
  │
  ├── mount_autodev()
  │     └── /dev ← tmpfs + 创建设备节点 + 符号链接
  │
  ├── ★ lxc_mount_auto_mounts() 第一次
  │     ├── /proc ← proc 文件系统
  │     │     └── mixed: /proc/sys → ro, /proc/sys/net → rw
  │     └── /sys ← sysfs 文件系统
  │           └── mixed: /sys → ro, /sys/devices/virtual/net → rw
  │
  ├── setup_mount_fstab() + setup_mount_entries()
  │     └── 用户自定义挂载（lxc.mount.fstab / lxc.mount.entry）
  │
  ├── ★ lxc_mount_auto_mounts() 第二次
  │     └── /sys/fs/cgroup ← cgroup 文件系统
  │
  ├── lxc_setup_boot_id()
  │     └── /proc/sys/kernel/random/boot_id ← 容器专属 UUID (ro)
  │
  ├── lxc_setup_rootfs_switch_root()
  │     └── pivot_root / chroot
  │
  └── setup_sysctl_parameters()
        └── 写入容器 sysctl 参数（此时已在新 rootfs 内）
```

---

## 13. 安全设计总结

1. **分层防护**：挂载标志（ro/nosuid/nodev）+ AppArmor 策略 + Namespace 隔离，多层保护防止容器影响宿主。

2. **最小可写原则**：`mixed` 模式仅开放 per-namespace 的路径（`/proc/sys/net`、`/sys/devices/virtual/net`），其余全部只读。

3. **标志保留机制**：bind remount 时通过 `statvfs()` 读取并保留原有安全标志（nosuid/nodev/noexec），防止通过 remount 降级安全性。

4. **boot_id 隔离**：每个容器获得唯一的 boot_id，防止容器间和容器-宿主间的身份关联。

5. **两次挂载策略**：proc/sys 先挂载，cgroup 后挂载，解决了 cgroup 依赖 `/sys` 存在的顺序问题。

6. **嵌套容器感知**：通过 `/dev/.lxc/proc` 和 `/dev/.lxc/sys` 为内层容器提供干净的文件系统实例。
