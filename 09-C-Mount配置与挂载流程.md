# LXC Mount 配置与挂载流程深度分析

## 1. 概述

LXC 的挂载系统是容器文件系统隔离的核心，由三个配置项控制：

| 配置项 | 功能 | 存储位置 |
|--------|------|----------|
| `lxc.mount.entry` | 单条挂载条目（可多次指定） | `conf->mount_entries`（链表） |
| `lxc.mount.fstab` | 外部 fstab 文件路径 | `conf->fstab`（字符串） |
| `lxc.mount.auto` | 自动挂载标志（proc/sys/cgroup） | `conf->auto_mounts`（位掩码） |

挂载顺序在 `lxc_setup()` 中严格控制（`conf.c:3806-3841`）：

```
1. lxc_mount_auto_mounts(非 cgroup 部分)    — proc、sysfs
2. setup_mount_fstab()                       — 外部 fstab 文件
3. setup_mount_entries()                     — 内联挂载条目
4. lxc_mount_auto_mounts(cgroup 部分)        — cgroup 文件系统
```

---

## 2. lxc.mount.entry — 内联挂载条目

### 2.1 配置格式

```ini
lxc.mount.entry = <source> <target> <type> <options> <dump> <fsck>
```

与标准 fstab 格式相同，但增加了 LXC 特有选项。

**示例**：
```ini
# 绑定挂载宿主机目录
lxc.mount.entry = /data data none bind,create=dir 0 0

# 只读绑定挂载
lxc.mount.entry = /usr/share/doc usr/share/doc none bind,ro 0 0

# tmpfs 挂载
lxc.mount.entry = tmpfs tmp tmpfs nosuid,nodev,mode=1777 0 0

# 使用相对路径（相对于 rootfs）
lxc.mount.entry = /host/config etc/app/config none bind,relative,create=file 0 0
```

### 2.2 解析流程

```
set_config_mount()                              [confile.c:2426-2452]
  ├── 将挂载条目字符串追加到 conf->mount_entries 链表
  └── 使用 add_elem_to_mount_list() 创建 string_entry 节点
```

`mount_entries` 在 `struct lxc_conf` 中定义为 `struct list_head`（`conf.h:442-446`），
在 conf 创建时通过 `INIT_LIST_HEAD()` 初始化（`conf.c:3110-3111`）。

### 2.3 挂载选项解析

#### 标准 VFS 选项

`mount_opt[]` 数组（`conf.c:132-176`）映射选项字符串到内核 MS_* 标志：

| 选项 | 标志 | 说明 |
|------|------|------|
| `ro` | `MS_RDONLY` | 只读 |
| `rw` | ~`MS_RDONLY` | 读写（清除只读标志） |
| `nosuid` | `MS_NOSUID` | 禁止 SUID |
| `nodev` | `MS_NODEV` | 禁止设备文件 |
| `noexec` | `MS_NOEXEC` | 禁止执行 |
| `sync` | `MS_SYNCHRONOUS` | 同步写入 |
| `dirsync` | `MS_DIRSYNC` | 目录同步 |
| `remount` | `MS_REMOUNT` | 重新挂载 |
| `bind` | `MS_BIND` | 绑定挂载 |
| `rbind` | `MS_BIND \| MS_REC` | 递归绑定挂载 |
| `relatime` | `MS_RELATIME` | 相对访问时间 |
| `lazytime` | `MS_LAZYTIME` | 延迟时间更新 |
| `strictatime` | `MS_STRICTATIME` | 严格访问时间 |

传播选项：

| 选项 | 标志 |
|------|------|
| `shared` | `MS_SHARED` |
| `slave` | `MS_SLAVE` |
| `private` | `MS_PRIVATE` |
| `unbindable` | `MS_UNBINDABLE` |
| `rshared` | `MS_SHARED \| MS_REC` |
| `rslave` | `MS_SLAVE \| MS_REC` |
| `rprivate` | `MS_PRIVATE \| MS_REC` |
| `runbindable` | `MS_UNBINDABLE \| MS_REC` |

解析函数 `parse_vfs_attr()`（`conf.c:1942-2000`）逗号分隔遍历选项字符串，
匹配到的标志通过位或运算合并，未匹配的选项保留为 `data` 字符串传给内核。

#### LXC 特有选项

`parse_lxc_mount_attrs()` / `parse_mount_attrs()`（`conf.c:2185-2243`）处理以下扩展选项：

| 选项 | 功能 |
|------|------|
| `create=dir` | 挂载前自动创建目标目录 |
| `create=file` | 挂载前自动创建目标文件 |
| `optional` | 挂载失败不报错（跳过） |
| `relative` | 源路径相对于 rootfs 解释 |
| `idmap=<path>` | 使用指定的 ID 映射进行挂载 |

### 2.4 挂载执行流程

```
setup_mount_entries()                           [conf.c:2555-2565]
  ├── 将 mount_entries 链表写入临时 memfd 文件
  ├── setmntent(memfd) 打开为 fstab 流
  └── mount_file_entries(rootfs, mntent_stream, ...)
        └── 逐条读取 getmntent_r()
              └── mount_entry(mntent, rootfs, ...)
```

`mount_entry()` 的分发逻辑（`conf.c:2293-2350`）：

```
mount_entry()
  ├── 解析 LXC 属性（create/optional/relative/idmap）
  ├── 解析 VFS 标志和传播标志
  ├── 判断挂载位置：
  │     ├── 绝对路径且以 rootfs 为前缀 → mount_entry_on_absolute_rootfs()
  │     ├── 目标是系统路径（/proc, /sys 等） → mount_entry_on_systemfs()
  │     └── 其他 → mount_entry_on_relative_rootfs()
  └── 最终都调用 mount_entry_on_generic()
```

`mount_entry_on_generic()`（`conf.c:2075-2174`）：

```
mount_entry_on_generic()
  ├── mount_entry_create_dir_file()     // 处理 create=dir/file
  ├── safe_mount(source, target, fstype, mountflags, data)
  │     └── 实际调用 mount() 系统调用
  ├── if (bind && remount_flags):
  │     └── mount(NULL, target, NULL, remountflags | MS_REMOUNT, NULL)
  │           // 绑定挂载需要第二次 mount 来应用 ro 等选项
  └── if (propagation_flags):
        └── mount(NULL, target, NULL, pflags, NULL)
              // 设置传播属性
```

**重要细节 — 绑定挂载的两步操作**：

Linux 内核的限制：`mount --bind` 时会忽略 `ro` 等标志。因此 LXC 需要：
1. 第一步：`mount(src, dst, NULL, MS_BIND, NULL)` — 创建绑定挂载
2. 第二步：`mount(NULL, dst, NULL, MS_REMOUNT|MS_BIND|MS_RDONLY, NULL)` — 重新挂载为只读

---

## 3. lxc.mount.fstab — 外部 fstab 文件

### 3.1 配置

```ini
lxc.mount.fstab = /path/to/fstab
```

### 3.2 解析

```
set_config_mount_fstab()                        [confile.c:2306-2315]
  └── set_config_string_item(&conf->fstab, value)
```

### 3.3 应用

```
setup_mount_fstab(rootfs, fstab_path, ...)      [conf.c:2462-2480]
  ├── setmntent(fstab_path, "r")               // 打开 fstab 文件
  └── mount_file_entries(rootfs, stream, ...)   // 与 mount.entry 共用挂载路径
```

fstab 文件格式与 `/etc/fstab` 完全相同，共用 `mount_file_entries()` 处理函数。

---

## 4. lxc.mount.auto — 自动挂载

### 4.1 配置语法

```ini
lxc.mount.auto = proc:mixed sys:ro cgroup:mixed:force
```

多个挂载类型用空格分隔，每个类型可带修饰符。

### 4.2 解析与位掩码

`set_config_mount_auto()`（`confile.c:2317-2423`）将每个 token 解析为位掩码：

#### proc 挂载

| 配置值 | 位掩码 | 行为 |
|--------|--------|------|
| `proc` | `LXC_AUTO_PROC_RW` | 可读写挂载 /proc |
| `proc:mixed` | `LXC_AUTO_PROC_MIXED` | /proc/sys 只读，/proc/sys/net 可写 |
| `proc:rw` | `LXC_AUTO_PROC_RW` | 完全可读写 |

#### sys 挂载

| 配置值 | 位掩码 | 行为 |
|--------|--------|------|
| `sys:ro` | `LXC_AUTO_SYS_RO` | 只读挂载 /sys |
| `sys:mixed` | `LXC_AUTO_SYS_MIXED` | /sys 只读，/sys/devices/virtual/net 可写 |
| `sys:rw` | `LXC_AUTO_SYS_RW` | 完全可读写 |

#### cgroup 挂载

| 配置值 | 位掩码 | 行为 |
|--------|--------|------|
| `cgroup` | `LXC_AUTO_CGROUP_RO` | 只读挂载 cgroup |
| `cgroup:mixed` | `LXC_AUTO_CGROUP_MIXED` | cgroup 根只读，容器自身目录可写 |
| `cgroup:ro` | `LXC_AUTO_CGROUP_RO` | 只读 |
| `cgroup:rw` | `LXC_AUTO_CGROUP_RW` | 完全可读写 |
| `cgroup:force` | 追加 `FORCE` 标志 | 强制挂载（即使 cgroup 命名空间可用） |
| `cgroup-full:*` | `FULL` 系列标志 | 挂载完整 cgroup 层级 |
| `cgroup2:*` | `CGROUP2` 系列标志 | cgroup v2 专用 |

#### shmounts

| 配置值 | 说明 |
|--------|------|
| `shmounts:<host_path>[:<cont_path>]` | 共享挂载点 |

位定义在 `conf.h:264-336`。

### 4.3 应用流程

`lxc_mount_auto_mounts()`（`conf.c:471-656`）使用 `default_mounts[]` 表：

```c
static struct mount_opt default_mounts[] = {
    { "proc",                  "/proc",               "proc",   MS_NODEV|MS_NOEXEC|MS_NOSUID },
    { "sysfs",                 "/sys",                "sysfs",  MS_RDONLY },
    { "/proc/sys",             "/proc/sys",           NULL,     MS_BIND },
    { "/proc/sys",             "/proc/sys",           NULL,     MS_REMOUNT|MS_RDONLY|MS_BIND },
    { "/proc/sysrq-trigger",   "/proc/sysrq-trigger", NULL,     MS_BIND },
    { "/proc/sysrq-trigger",   "/proc/sysrq-trigger", NULL,     MS_REMOUNT|MS_RDONLY|MS_BIND },
    ...
};
```

#### proc:mixed 的实现细节

```
1. mount("proc", "/proc", "proc", MS_NODEV|MS_NOEXEC|MS_NOSUID, NULL)
2. mount("/proc/sys", "/proc/sys", NULL, MS_BIND, NULL)           // 绑定挂载
3. mount("/proc/sys", "/proc/sys", NULL, MS_REMOUNT|MS_RDONLY|MS_BIND, NULL)  // 改为只读
4. mount("/proc/sys/net", "/proc/tty", "none", MS_BIND, NULL)     // 暂存 net 到临时点
5. umount("/proc/sys")                                              // 卸载只读 /proc/sys
6. mount("/proc/tty", "/proc/sys/net", "none", MS_MOVE, NULL)      // 移回 net（保持可写）
```

这个巧妙的 "暂存-卸载-移回" 序列使得 `/proc/sys` 整体只读，但 `/proc/sys/net` 保持可写。

#### sys:mixed 的实现细节

```
1. mount("sysfs", "/sys", "sysfs", MS_RDONLY, NULL)
2. 创建临时 sysfs 挂载
3. 绑定挂载 /sys/devices/virtual/net（可写）
4. 移除临时挂载
```

### 4.4 两阶段挂载

`lxc_setup()` 将自动挂载分为两阶段：

```c
// 阶段一：proc 和 sysfs（conf.c:3806-3810）
lxc_mount_auto_mounts(handler, auto_mounts & ~LXC_AUTO_CGROUP_MASK);

// 用户自定义挂载（可能依赖 proc/sys 已经就位）
setup_mount_fstab(...);
setup_mount_entries(...);

// 阶段二：cgroup（conf.c:3835-3841）
lxc_mount_auto_mounts(handler, auto_mounts & LXC_AUTO_CGROUP_MASK);
```

原因：cgroup 挂载可能依赖用户自定义挂载创建的目录结构。

---

## 5. 挂载辅助工具

### 5.1 safe_mount()

`mount_utils.c` 中的安全挂载封装：

```c
int safe_mount(const char *src, const char *dest, const char *fstype,
               unsigned long mountflags, const void *data,
               const char *rootfs)
```

功能：
- 防止符号链接攻击（验证目标路径不逃逸 rootfs）
- 支持 `AT_EMPTY_PATH` 进行 fd-based mount
- 使用 `open_tree()` + `move_mount()` 作为新式挂载 API

### 5.2 mount_entry_create_dir_file()

处理 `create=dir` 和 `create=file` 选项（`conf.c:2246-2289`）：

```
mount_entry_create_dir_file(mntent)
  ├── create=dir:
  │     └── mkdir_p(target, 0755)
  └── create=file:
        ├── mkdir_p(dirname(target), 0755)
        └── open(target, O_CREAT, 0644)
```

### 5.3 remount 安全标志保留

绑定重新挂载时保留原始安全标志（`conf.c`）：

```c
statvfs(target, &sb);
// 保留原有的 nosuid, nodev, noexec 等标志
required_flags = sb.f_flag & (ST_NOSUID | ST_NODEV | ST_NOEXEC | ST_RDONLY);
mount(NULL, target, NULL, MS_REMOUNT | MS_BIND | required_flags | extra_flags, NULL);
```

---

## 6. 配置示例

### 6.1 典型系统容器

```ini
# 自动挂载
lxc.mount.auto = proc:mixed sys:mixed cgroup:mixed

# 外部 fstab
lxc.mount.fstab = /var/lib/lxc/mycontainer/fstab

# 内联挂载
lxc.mount.entry = /data srv/data none bind,create=dir 0 0
lxc.mount.entry = tmpfs run tmpfs nosuid,nodev,mode=0755 0 0
lxc.mount.entry = tmpfs tmp tmpfs nosuid,nodev,mode=1777,size=512m 0 0
```

### 6.2 只读根文件系统 + 可写层

```ini
lxc.mount.auto = proc:mixed sys:ro
lxc.rootfs.options = ro

# 可写区域
lxc.mount.entry = tmpfs var/run tmpfs nosuid,nodev 0 0
lxc.mount.entry = tmpfs var/tmp tmpfs nosuid,nodev,mode=1777 0 0
lxc.mount.entry = /persistent/log var/log none bind,create=dir 0 0
```

### 6.3 共享宿主机资源

```ini
# 共享 GPU 设备
lxc.mount.entry = /dev/dri dev/dri none bind,optional,create=dir 0 0

# 共享 X11 socket
lxc.mount.entry = /tmp/.X11-unix tmp/.X11-unix none bind,create=dir 0 0

# 共享 Pulseaudio
lxc.mount.entry = /run/user/1000/pulse run/user/1000/pulse none bind,create=dir 0 0
```

---

## 7. 挂载流程全景图

```
lxc_setup()
  │
  ├─1─ lxc_mount_auto_mounts(非 cgroup)
  │      ├── proc → /proc（按 auto_mounts 标志选择模式）
  │      ├── sysfs → /sys
  │      ├── /proc/sys 只读绑定（mixed 模式）
  │      ├── /proc/sysrq-trigger 只读绑定
  │      └── shmounts 设置
  │
  ├─2─ setup_mount_fstab(conf->fstab)
  │      └── 读取外部 fstab，逐条调用 mount_entry()
  │
  ├─3─ setup_mount_entries(conf->mount_entries)
  │      └── 遍历链表，逐条调用 mount_entry()
  │            ├── parse_lxc_mount_attrs()    // 解析 LXC 扩展选项
  │            ├── parse_vfs_attr()           // 解析 VFS 标志
  │            ├── mount_entry_create_dir_file()  // 创建挂载点
  │            ├── safe_mount()               // 执行挂载
  │            ├── remount（绑定挂载 + ro）    // 两步操作
  │            └── 设置传播属性
  │
  ├─4─ lxc_mount_auto_mounts(cgroup)
  │      └── cgroup_ops->mount(cgroup_ops, handler)
  │
  └─5─ hook.mount 钩子（如果配置）
```
