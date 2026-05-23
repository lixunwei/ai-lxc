# Mount Namespace 与 Rootfs 挂载流程深度分析

## 1. 概述

LXC 容器启动时，mount namespace 的构建是最复杂的环节之一。整个流程涉及：rootfs 挂载、mount 传播隔离、伪文件系统挂载、用户自定义挂载、pivot_root 切换。

**核心源文件**：
- `src/lxc/conf.c` — 主要挂载逻辑（`lxc_setup()` 主函数）
- `src/lxc/mount_utils.c` — 新/旧 mount API 封装
- `src/lxc/confile.c` / `confile_utils.c` — 挂载配置解析

---

## 2. 挂载操作完整时序

```
lxc_setup() [conf.c:3764-3921]
  |
  |-- [1] lxc_rootfs_prepare_child()           // 准备 rootfs fd
  |-- [2] lxc_setup_rootfs_prepare_root()      // 挂载 rootfs
  |       |-- open("/")                         // 打开宿主根
  |       |-- turn_into_dependent_mounts()      // 切断 mount 传播
  |       |-- run_hooks("pre-mount")            // pre-mount 钩子
  |       +-- lxc_mount_rootfs()                // storage backend mount
  |
  |-- [3] setup_utsname()                       // hostname
  |-- [4] lxc_setup_keyring()                   // session keyring
  |-- [5] lxc_setup_network_in_child_namespaces() // 网络配置
  |
  |-- [6] mount_autodev()                       // tmpfs /dev (新 mount API)
  |-- [7] lxc_mount_auto_mounts(~CGROUP)        // /proc, /sys 自动挂载
  |-- [8] setup_mount_fstab()                   // lxc.mount.fstab 文件
  |-- [9] setup_mount_entries()                 // lxc.mount.entry 条目
  |-- [10] lxc_idmapped_mounts_child()          // idmapped mount 应用
  |
  |-- [11] lxc_mount_auto_mounts(CGROUP)        // cgroup 挂载
  |-- [12] run_hooks("mount")                   // mount 钩子
  |-- [13] run_hooks("autodev") + lxc_fill_autodev() // 填充 /dev 设备节点
  |
  |-- [14] lxc_create_tmp_proc_mount()          // 临时 /proc (LSM 使用)
  |-- [15] lxc_setup_devpts_child()             // devpts 挂载
  |-- [16] lxc_setup_console()                  // 控制台绑定
  |-- [17] lxc_create_ttys()                    // TTY 设备
  |-- [18] lxc_setup_dev_symlinks()             // /dev 符号链接
  |
  |-- [19] lxc_setup_rootfs_switch_root()       // pivot_root 切换！
  |-- [20] make_shmount_dependent_mount()       // 共享挂载隧道隔离
  +-- [21] lxc_setup_boot_id()                  // /proc/sys/kernel/random/boot_id
```

---

## 3. Rootfs 挂载（步骤 2）

### 3.1 lxc_setup_rootfs_prepare_root()

```c
// conf.c:3366-3404
lxc_setup_rootfs_prepare_root(conf, name, lxcpath) {
    // 1. 打开宿主机根目录 fd（用于后续 pivot 回退）
    conf->rootfs.dfd_host = open("/", O_PATH|O_DIRECTORY);

    // 2. 将所有挂载点切为 dependent（阻止传播到宿主）
    turn_into_dependent_mounts(&conf->rootfs);

    // 3. 执行 pre-mount 钩子
    run_lxc_hooks(name, "pre-mount", conf, NULL);

    // 4. 调用 storage 后端的 mount 操作
    lxc_mount_rootfs(&conf->rootfs);
}
```

### 3.2 lxc_mount_rootfs()

```c
// conf.c:1157-1193
lxc_mount_rootfs(rootfs) {
    if (!rootfs->path) {
        // 无独立 rootfs，使用宿主根（共享 rootfs 模式）
        mount("", "/", NULL, MS_SLAVE|MS_REC, 0);
        rootfs->dfd_mnt = open("/");
        return;
    }

    // 通过 storage 后端挂载
    // dir -> bind mount
    // btrfs -> bind mount subvol
    // overlay -> overlayfs mount
    // loop -> loop device + mount
    rootfs->storage->ops->mount(rootfs->storage);

    // 打开挂载后的 rootfs fd
    rootfs->dfd_mnt = open(rootfs->mount);
}
```

### 3.3 Mount 传播隔离

```c
// conf.c:3275-3360
turn_into_dependent_mounts(rootfs) {
    // 递归将宿主挂载树变为 slave
    // 确保容器内的 mount/umount 不会传播到宿主
    mount("", "/", NULL, MS_SLAVE|MS_REC, NULL);
}
```

---

## 4. /dev 自动挂载（步骤 6）

使用**新 mount API**（`fsopen`/`fsconfig`/`fsmount`/`move_mount`）：

```c
// conf.c:990-1048
mount_autodev(name, rootfs, autodevtmpfssize, lxcpath) {
    // 1. 创建 /dev 目录
    mkdirat(rootfs->dfd_mnt, "dev", 0755);

    // 2. 使用新 mount API 挂载 tmpfs
    fd_fs = fs_prepare("tmpfs", -EBADF, "", 0, 0);  // fsopen("tmpfs")
    fs_set_property(fd_fs, "mode", "0755");          // fsconfig(SET_STRING)
    fs_set_property(fd_fs, "size", "500000");        // fsconfig(SET_STRING)
    fs_attach(fd_fs, rootfs->dfd_mnt, "dev", ...);   // fsmount + move_mount

    // 3. 创建 /dev/pts 目录
    mkdirat(rootfs->dfd_mnt, "dev/pts", 0755);
}
```

### 4.1 /dev 设备节点填充

```c
// lxc_fill_autodev() 创建:
// /dev/null, /dev/zero, /dev/full, /dev/urandom, /dev/random
// /dev/tty, /dev/console
// 通过 mknod 或 bind mount 从宿主 /dev 绑定
```

### 4.2 /dev 符号链接

```c
// lxc_setup_dev_symlinks() 创建:
// /dev/fd -> /proc/self/fd
// /dev/stdin -> /proc/self/fd/0
// /dev/stdout -> /proc/self/fd/1
// /dev/stderr -> /proc/self/fd/2
```

---

## 5. 自动挂载（/proc, /sys）

### 5.1 第一批自动挂载（步骤 7）

`lxc_mount_auto_mounts(handler, auto_mounts & ~CGROUP_MASK)` 挂载：

| 目标 | 类型 | 条件 |
|------|------|------|
| /proc | proc | `LXC_AUTO_PROC_MIXED` 或 `LXC_AUTO_PROC_RW` |
| /sys | sysfs | `LXC_AUTO_SYS_MIXED` 或 `LXC_AUTO_SYS_RW` |
| /sys/firmware/efi/efivars | efivarfs | 若存在 |

对于 `MIXED` 模式，某些路径以只读方式 bind mount 覆盖：
```
/proc/sys      -> bind mount (ro)
/proc/sysrq-trigger -> bind mount (ro)
/sys           -> bind mount (部分只读)
```

### 5.2 第二批自动挂载 — Cgroup（步骤 11）

`lxc_mount_auto_mounts(handler, auto_mounts & CGROUP_MASK)` 挂载：

```
# cgroup v1
/sys/fs/cgroup/memory/     -> bind mount 容器的 cgroup 路径
/sys/fs/cgroup/cpu/        -> bind mount 容器的 cgroup 路径
...

# cgroup v2
/sys/fs/cgroup/            -> bind mount 容器的 cgroup 路径
```

必须在 /sys 挂载之后才能执行，故分两批。

---

## 6. 用户自定义挂载

### 6.1 lxc.mount.fstab（步骤 8）

```
# /etc/lxc/myct.fstab (标准 fstab 格式)
/data  /var/lib/lxc/myct/rootfs/data  none  bind  0 0
tmpfs  /var/lib/lxc/myct/rootfs/tmp   tmpfs defaults 0 0
```

处理流程：
```c
setup_mount_fstab(rootfs, fstab_path, name, lxcpath) {
    f = fopen(fstab_path);
    mount_file_entries(rootfs, f, lxcpath);
    // 逐行解析 + mount_entry_on_generic()
}
```

### 6.2 lxc.mount.entry（步骤 9）

```
lxc.mount.entry = /data data none bind 0 0
lxc.mount.entry = tmpfs tmp tmpfs defaults 0 0
```

支持的 LXC 扩展挂载选项（`parse_lxc_mount_attrs()`）：

| 选项 | 含义 |
|------|------|
| `create=dir` | 自动创建目录挂载点 |
| `create=file` | 自动创建文件挂载点 |
| `optional` | 挂载失败不报错 |
| `relative` | 路径相对于 rootfs |
| `idmap=container` | 使用容器的 ID 映射 |
| `idmap=path` | 使用指定 userns 的映射 |

### 6.3 lxc.mount.auto（配置解析）

```
lxc.mount.auto = proc:mixed sys:mixed cgroup:mixed
```

解析为标志位组合（`LXC_AUTO_PROC_MIXED | LXC_AUTO_SYS_MIXED | LXC_AUTO_CGROUP_MIXED`）。

---

## 7. 挂载条目执行

### 7.1 mount_entry_on_generic()

```c
// conf.c:2293-2350
mount_entry_on_generic(rootfs, mntent, path, ...) {
    // 1. 处理 create=dir/file（创建挂载点）
    // 2. 解析 mount flags (MS_BIND, MS_RDONLY, MS_NOSUID, ...)
    // 3. 执行 mount()
    mount(source, target, fstype, flags, data);
    // 4. 如果需要 readonly，追加 remount
    if (MS_RDONLY && MS_BIND)
        mount(NULL, target, NULL, MS_REMOUNT|MS_BIND|MS_RDONLY, NULL);
}
```

### 7.2 新 mount API 的 bind mount

```c
// mount_utils.c:332-387
__fd_bind_mount(dfd_from, source, dfd_to, target, ...) {
    // 使用 open_tree() + OPEN_TREE_CLONE 克隆挂载
    fd_tree = open_tree(dfd_from, source,
                        OPEN_TREE_CLONE | AT_RECURSIVE);
    // move_mount 到目标位置
    move_mount(fd_tree, "", dfd_to, target, MOVE_MOUNT_F_EMPTY_PATH);
}
```

---

## 8. Idmapped Mounts

### 8.1 工作原理

内核 5.12+ 支持在 mount 时附加 ID 映射，使文件系统的 UID/GID 在访问时自动转换，无需修改磁盘数据。

### 8.2 LXC 实现

**子进程端**（`__lxc_idmapped_mounts_child`）：
```c
// conf.c:2568-2774
// 1. 创建 detached bind mount
fd_tree = open_tree(source, OPEN_TREE_CLONE);
// 2. 通过 socket 将 fd 和 userns_fd 传给父进程
send_fd(sock, fd_tree);
send_fd(sock, userns_fd);
```

**父进程端**（`lxc_idmapped_mounts_parent`）：
```c
// conf.c:3543-3584
// 1. 接收 fd
fd_tree = recv_fd(sock);
userns_fd = recv_fd(sock);
// 2. 设置 ID 映射属性
struct mount_attr attr = {
    .attr_set = MOUNT_ATTR_IDMAP,
    .userns_fd = userns_fd,
};
mount_setattr(fd_tree, "", AT_EMPTY_PATH|AT_RECURSIVE, &attr, sizeof(attr));
// 3. 移动到最终位置
move_mount(fd_tree, "", target_dfd, target_path, MOVE_MOUNT_F_EMPTY_PATH);
```

**配置方式**：
```
lxc.mount.entry = /shared shared none bind,idmap=container 0 0
```

---

## 9. pivot_root 切换（步骤 19）

### 9.1 决策逻辑

```c
// conf.c:1403-1412
lxc_setup_rootfs_switch_root(rootfs) {
    if (!rootfs->path)
        return;  // 无独立 rootfs，不切换

    if (detect_ramfs_rootfs())
        return lxc_chroot(rootfs);   // ramfs 不支持 pivot_root

    return lxc_pivot_root(rootfs);   // 正常路径
}
```

### 9.2 lxc_pivot_root() 详解

```c
// conf.c:1328-1401
lxc_pivot_root(rootfs) {
    // 1. 打开旧根 fd（用于后续操作）
    fd_oldroot = open("/", O_PATH);

    // 2. 切换工作目录到新 rootfs
    fchdir(rootfs->dfd_mnt);

    // 3. pivot_root(".", ".")
    //    新根 = "."（当前目录 = rootfs mount point）
    //    旧根挂载到新根的 "."（自身之上）
    pivot_root(".", ".");

    // 4. 回到旧根（现在在新根之上）
    fchdir(fd_oldroot);

    // 5. 将旧根变为 slave（阻止 umount 传播到宿主）
    mount("", ".", "", MS_SLAVE|MS_REC, NULL);

    // 6. 卸载旧根
    umount2(".", MNT_DETACH);

    // 7. 回到新根
    fchdir(rootfs->dfd_mnt);

    // 8. 将新根设为 shared（容器内服务需要传播）
    mount("", ".", "", MS_SHARED|MS_REC, NULL);
}
```

### 9.3 pivot_root(".","." ) 技巧

这是一个巧妙的用法：
- `new_root` = "." = 当前目录 = 容器 rootfs
- `put_old` = "." = 相同目录
- 效果：旧根被挂载到新根之上，然后通过 `umount2(MNT_DETACH)` 移除
- 优势：不需要额外的挂载点来放置旧根

### 9.4 chroot 回退

当根文件系统是 ramfs 时（不支持 pivot_root），使用 chroot：

```c
// conf.c:1212-1300
lxc_chroot(rootfs) {
    // 遍历 /proc/self/mountinfo 卸载所有非 rootfs 的挂载
    // chroot(rootfs->mount)
    // chdir("/")
}
```

---

## 10. Mount 传播模型

```
+--------------------------------------------------+
|  Host mount namespace (shared)                   |
|                                                  |
|  /sys/fs/cgroup/ (shared)                        |
|  /var/lib/lxc/myct/rootfs/ (shared)              |
+--------------------------------------------------+
         |
         | CLONE_NEWNS
         v
+--------------------------------------------------+
|  Container mount namespace                       |
|                                                  |
|  turn_into_dependent_mounts():                   |
|    "/" -> MS_SLAVE|MS_REC  (切断向上传播)         |
|                                                  |
|  mount rootfs, /dev, /proc, /sys, etc.           |
|  (这些挂载对宿主不可见)                           |
|                                                  |
|  pivot_root 之后:                                 |
|    new "/" -> MS_SHARED|MS_REC  (容器内部共享)    |
|    (独立 peer group，不与宿主共享)                |
+--------------------------------------------------+
```

**为什么新根要设为 shared？**

容器内 systemd 等服务会创建子 mount namespace（如 systemd-udevd），子 namespace 默认继承为 slave。如果容器根是 slave 而非 shared，子 namespace 内的挂载传播语义会错乱。

---

## 11. 新 Mount API 使用总结

LXC 封装了新 mount API（`mount_utils.c`）：

| 函数 | 系统调用 | 用途 |
|------|----------|------|
| `fs_prepare()` | `fsopen()` + `fsconfig(CMD_CREATE)` | 打开文件系统上下文 |
| `fs_set_property()` | `fsconfig(SET_STRING)` | 设置挂载属性 |
| `fs_set_flag()` | `fsconfig(SET_FLAG)` | 设置布尔标志 |
| `fs_attach()` | `fsmount()` + `move_mount()` | 创建并挂载 |
| `__fd_bind_mount()` | `open_tree(CLONE)` + `move_mount()` | fd-based bind mount |
| `create_detached_idmapped_mount()` | `open_tree(CLONE)` + `mount_setattr(IDMAP)` | idmapped mount |

运行时检测能力：
```c
// mount_utils.c:538-611
can_use_mount_api()  // 检测 fsopen/fsmount/move_mount 是否可用
can_use_bind_mounts() // 检测 open_tree + move_mount 是否可用
```

不可用时回退到传统 `mount()` 系统调用。

---

## 12. 关键数据结构

```c
// conf.h
struct lxc_rootfs {
    char *path;           // 配置中的 rootfs 路径 (e.g. "/var/lib/lxc/ct/rootfs")
    char *mount;          // 挂载点路径
    char *options;        // 挂载选项
    int dfd_mnt;          // 挂载后的目录 fd
    int dfd_dev;          // /dev 目录 fd
    int dfd_host;         // 宿主根目录 fd
    struct lxc_storage *storage;   // storage 后端实例
    struct lxc_mount_options mnt_opts;  // 解析后的挂载选项
};

// mount entry (from lxc.mount.entry / fstab)
struct lxc_mount_entry {
    char *mnt_fsname;     // 源
    char *mnt_dir;        // 目标
    char *mnt_type;       // 文件系统类型
    char *mnt_opts;       // 选项字符串
    unsigned long flags;  // MS_* 标志
    // LXC 扩展属性
    bool optional;
    bool relative;
    int create_type;      // MNT_CREATE_DIR / MNT_CREATE_FILE
    char *idmap_path;     // idmap userns 路径
};
```

---

## 13. 安全保护

1. **路径验证**：所有挂载目标使用 `PROTECT_LOOKUP_BENEATH_XDEV`，防止符号链接跨越文件系统边界
2. **Overmount 检测**：`lxc_rootfs_overmounted()` 在所有挂载完成后验证 rootfs 未被恶意覆盖
3. **传播隔离**：`MS_SLAVE|MS_REC` 确保容器操作不影响宿主
4. **fd-based 操作**：尽量使用 `dfd_mnt` + `openat/mkdirat` 而非路径字符串，防止 TOCTOU 攻击
