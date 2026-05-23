# 用户命名空间 ID 映射机制深度分析

## 1. 概述

用户命名空间（User Namespace）是 LXC 实现非特权容器的核心机制。通过 ID 映射，容器内的 root（uid 0）可以映射到宿主机上的普通用户（如 uid 100000），从而实现安全隔离。

**核心源文件**：
- `src/lxc/idmap_utils.c` — 映射写入与辅助工具
- `src/lxc/conf.c` — userns_exec_* 系列函数
- `src/lxc/confile_utils.c` — 配置解析
- `src/lxc/start.c` — 启动时映射应用

---

## 2. 数据结构

### 2.1 id_map（单条映射条目）

```c
// src/lxc/conf.h:188-192
struct id_map {
    enum idtype idtype;       // ID_TYPE_UID 或 ID_TYPE_GID
    unsigned long hostid;     // 宿主机起始 ID
    unsigned long nsid;       // 命名空间内起始 ID
    unsigned long range;      // 映射范围（连续 ID 数量）
    struct list_head head;    // 链表节点
};
```

### 2.2 配置中的存储

```c
// src/lxc/conf.h (lxc_conf 结构体)
struct lxc_conf {
    struct list_head id_map;          // id_map 链表头
    struct id_map *root_nsuid_map;    // nsid==0 的 UID 映射（缓存）
    struct id_map *root_nsgid_map;    // nsid==0 的 GID 映射（缓存）
    // ...
};
```

### 2.3 配置格式

```
lxc.idmap = u 0    100000 65536
lxc.idmap = g 0    100000 65536
#            ^  ^     ^      ^
#          type nsid hostid range
#
# 含义: 容器 uid 0-65535 映射到宿主机 uid 100000-165535
```

---

## 3. 配置解析流程

```
lxc.idmap = u 0 100000 65536
     |
     v
parse_idmaps() [confile_utils.c:32-130]
     |
     +-> 解析 type: "u" -> ID_TYPE_UID, "g" -> ID_TYPE_GID
     +-> 解析 nsid:   lxc_safe_ulong("0")       -> 0
     +-> 解析 hostid: lxc_safe_ulong("100000")  -> 100000
     +-> 解析 range:  lxc_safe_ulong("65536")   -> 65536
     |
     v
set_config_idmaps() [confile.c:2261-2304]
     |
     +-> 分配 struct id_map
     +-> 填充字段
     +-> list_add_tail(&map->head, &conf->id_map)
     +-> 如果 nsid == 0: 缓存到 root_nsuid_map / root_nsgid_map
```

---

## 4. 映射写入内核

### 4.1 核心函数：lxc_map_ids()

```c
// src/lxc/idmap_utils.c:129-257
int lxc_map_ids(struct list_head *idmap, pid_t pid)
```

**决策逻辑**（选择写入方式）：

```
+---------------------------+
| maps_host_root ?          |
|   (映射了 hostid==0)      |
+---------------------------+
         |                |
        YES              NO
         |                |
         v                v
  Direct write     +-------------------+
  (root has        | newuidmap found?  |
   CAP_SETFCAP)    | newgidmap found?  |
                   +-------------------+
                      |            |
                    BOTH          NO
                      |            |
                      v            v
                Use shadow    Direct write
                helpers       (only own uid)
```

### 4.2 直接写入（write_id_mapping）

```c
// src/lxc/idmap_utils.c:23-65
write_id_mapping(ID_TYPE_UID, pid, "0 100000 65536\n", len) {
    // 1. 非 root 写 GID 时，先禁用 setgroups
    if (euid != 0 && idtype == GID) {
        open("/proc/<pid>/setgroups") -> write("deny\n")
    }
    // 2. 写入映射
    open("/proc/<pid>/uid_map") -> write("0 100000 65536\n")
}
```

写入格式：`nsid hostid range\n`（每行一条映射，内核限制总共 ≤ 4KB）

### 4.3 使用 newuidmap/newgidmap

当非特权用户需要多条映射时，必须使用 setuid 辅助工具：

```c
// 构造命令: "newuidmap <pid> 0 100000 65536"
sprintf(mapbuf, "new%cidmap %d", u_or_g, pid);
// 追加每条映射: " <nsid> <hostid> <range>"
for each map:
    sprintf(pos, " %lu %lu %lu", map->nsid, map->hostid, map->range);

// 执行
run_command(cmd_output, sizeof(cmd_output),
            lxc_map_ids_exec_wrapper, mapbuf);
// 内部: execl("/bin/sh", "sh", "-c", "newuidmap <pid> ...", NULL)
```

### 4.4 辅助工具检测

```c
// src/lxc/idmap_utils.c:74-118
idmaptool_on_path_and_privileged("newuidmap", CAP_SETUID) {
    // 检查:
    // 1. 二进制是否在 PATH 中
    // 2. 是否设置了 setuid bit (S_ISUID)
    // 3. 或者是否有 CAP_SETUID file capability
    // 返回: 1=可用, 0=权限不足, -ENOENT=不存在
}
```

### 4.5 优化：简单映射跳过 shadow

如果只有 2 条映射且都是映射自己的 uid/gid（identity mapping），则不需要 newuidmap：

```c
// src/lxc/idmap_utils.c:193-205
if (use_shadow && list_len == 2) {
    use_shadow = false;
    for each map:
        if (uid map: nsid==hostuid && hostid==hostuid && range==1) continue;
        if (gid map: nsid==hostgid && hostid==hostgid && range==1) continue;
        use_shadow = true;  // 非 identity mapping，仍需 shadow
}
```

---

## 5. userns_exec 系列函数

LXC 提供多个在临时用户命名空间中执行操作的辅助函数：

### 5.1 userns_exec_1()（通用）

```c
// src/lxc/conf.c:4397-4468
userns_exec_1(conf, fn, data, fn_name) {
    // 1. 计算最小映射
    get_minimal_idmap(conf, NULL, NULL, &idmap);

    // 2. 创建 pipe 用于同步
    pipe2(pipe_fds);

    // 3. clone 子进程 (CLONE_NEWUSER)
    pid = lxc_raw_clone_cb(run_userns_fn, &d, CLONE_NEWUSER);

    // 4. 父进程: 写入 ID 映射
    lxc_map_ids(&idmap, pid);

    // 5. 父进程: 通知子进程继续
    write(pipe_fds[1], "1", 1);

    // 6. 等待子进程完成
    wait_for_pid(pid);
}
```

**子进程行为**：
```c
run_userns_fn(void *data) {
    // 等待父进程写好映射
    read(pipe_fd[0], &c, 1);
    // 现在有了正确的 UID/GID，执行回调
    return fn(arg);
}
```

### 5.2 userns_exec_mapped_root()（chown 到容器 root）

```c
// src/lxc/conf.c:4763-4910+
userns_exec_mapped_root(path, path_fd, conf) {
    // 1. 获取容器 root 映射到的宿主机 ID
    container_host_uid = get_mapped_rootid(conf, ID_TYPE_UID);
    container_host_gid = get_mapped_rootid(conf, ID_TYPE_GID);

    // 2. 如果调用者是 root，直接 fchown
    if (hostuid == 0) {
        fchown(target_fd, container_host_uid, container_host_gid);
        return;
    }

    // 3. 非 root: 构建临时映射，在子 userns 中 fchown
    //    映射: u:0->container_host_uid:1, u:hostuid->hostuid:1
    //    映射: g:0->container_host_gid:1, g:hostgid->hostgid:1
    add_idmap_entry(&idmap, UID, 0, container_host_uid, 1);
    add_idmap_entry(&idmap, UID, hostuid, hostuid, 1);
    add_idmap_entry(&idmap, GID, 0, container_host_gid, 1);
    add_idmap_entry(&idmap, GID, hostgid, hostgid, 1);

    // 4. clone 子进程，传递 fd，子进程执行 fchown(fd, 0, 0)
    //    (在子 userns 中 uid 0 == 宿主机的 container_host_uid)
}
```

### 5.3 userns_exec_minimal()（精简版）

支持传入两个回调（parent_fn + child_fn），使用 socketpair 双向通信。

### 5.4 userns_exec_full()（完整映射版）

使用容器的完整 ID 映射（而非最小映射），适用于需要访问容器全部 ID 范围的操作。

---

## 6. ID 转换辅助函数

```c
// 宿主 ID -> 命名空间 ID（查找 hostid 是否在映射范围内）
int mapped_hostid(unsigned id, const struct lxc_conf *conf, enum idtype idtype);

// 获取容器 root (nsid=0) 对应的宿主 ID
uid_t get_mapped_rootid(const struct lxc_conf *conf, enum idtype idtype);
// 例: 配置 "u 0 100000 65536" -> 返回 100000

// 找到一个未被映射的命名空间 ID
unsigned int find_unmapped_nsid(const struct lxc_conf *conf, enum idtype idtype);
```

---

## 7. 容器启动时的 ID 映射时序

```
lxc-start
  |
  +-> handler = lxc_init(name, conf)
  |
  +-> lxc_spawn(handler)
        |
        +-> lxc_clone(child_fn, CLONE_NEWUSER | CLONE_NEWNS | ...)
        |     |
        |     +-> [child] 阻塞在 sync pipe，等待映射
        |
        +-> [parent] lxc_map_ids(&conf->id_map, child_pid)
        |     |
        |     +-> write "/proc/<child>/uid_map"
        |     +-> write "/proc/<child>/gid_map"
        |
        +-> [parent] 通知 child 继续
        |
        +-> [child] 现在有了正确的 uid/gid 映射
        |     +-> setgroups(0, NULL)  // 清空附加组
        |     +-> setgid(0)           // 在 userns 中变为 gid 0
        |     +-> setuid(0)           // 在 userns 中变为 uid 0
        |     +-> 继续 mount/pivot_root/exec
```

关键点：`CLONE_NEWUSER` 必须在其他命名空间之前（或同时）创建，因为 userns 赋予了在新 netns/mntns 中操作的权限。

---

## 8. 集成场景

### 8.1 rootfs 所有权修复

非特权容器需要 rootfs 文件属于容器的映射 root：

```c
// 启动时调用
userns_exec_mapped_root(rootfs_path, -1, conf);
// 效果: chown rootfs -> 宿主机 uid 100000 (=容器 uid 0)
```

### 8.2 Cgroup 委托

容器需要写入自己的 cgroup 文件：

```c
// src/lxc/cgroups/cgfsng.c:1690-1743
chown_cgroup_wrapper(void *data) {
    // 在 userns 中以"root"身份 chown cgroup 文件
    fchownat(fd, "cgroup.procs", 0, 0, 0);  // 容器 root
    fchownat(fd, "cgroup.threads", 0, 0, 0);
    // ...
}

// 通过 userns_exec_1 执行
userns_exec_1(conf, chown_cgroup_wrapper, &wrapper_args, "chown_cgroup");
```

### 8.3 设备节点创建

设备节点需要正确的 owner：

```c
// 在子 userns 中 mknod + chown
// uid/gid 转换通过映射自动生效
mknod("/dev/null", S_IFCHR | 0666, makedev(1, 3));
chown("/dev/null", 0, 0);  // userns 中的 0 -> 宿主机 100000
```

### 8.4 Idmapped Mounts（内核 5.12+）

新内核支持 mount 时指定 ID 映射，避免文件级 chown：

```c
// src/lxc/conf.c:3543-3584
// 使用 mount_setattr() 设置 MOUNT_ATTR_IDMAP
struct mount_attr attr = {
    .attr_set  = MOUNT_ATTR_IDMAP,
    .userns_fd = userns_fd,  // 包含映射信息的 userns fd
};
mount_setattr(tree_fd, "", AT_EMPTY_PATH | AT_RECURSIVE, &attr, sizeof(attr));
```

优势：
- 无需修改磁盘上的文件所有权
- 多个容器可以共享同一个 rootfs（不同映射）
- 性能更好（避免全量 chown）

---

## 9. 安全考量

### 9.1 setgroups deny

非 root 用户写 GID 映射前必须先 deny setgroups：
```c
write("/proc/<pid>/setgroups", "deny\n");
// 防止容器内通过 setgroups() 获取额外权限
```

### 9.2 CAP_SETFCAP 限制

较新内核要求：映射 host root (uid 0) 到子 userns 时，写入者需要 `CAP_SETFCAP`，否则容器可通过设置 file capabilities 影响祖先命名空间。

### 9.3 /etc/subuid 和 /etc/subgid

newuidmap/newgidmap 验证映射是否在 `/etc/subuid` 和 `/etc/subgid` 授权范围内：
```
# /etc/subuid
lxcuser:100000:65536

# 含义: 用户 lxcuser 被授权使用 uid 100000-165535
```

### 9.4 最小映射原则

`get_minimal_idmap()` 为临时操作计算最小映射：
- 只映射当前 uid/gid（用于操作权限）
- 映射容器 root（用于 chown 目标）
- 不暴露完整的 65536 ID 范围

---

## 10. 完整流程图

```
+-----------+     +-------------+     +----------------+     +--------+
| Config    |     | Parent Proc |     | Child (userns) |     | Kernel |
+-----------+     +-------------+     +----------------+     +--------+
     |                   |                    |                    |
     |-- parse_idmaps -->|                    |                    |
     |   u 0 100000 65536|                    |                    |
     |                   |                    |                    |
     |                   |-- clone(NEWUSER) ->|                    |
     |                   |                    |-- block on pipe -->|
     |                   |                    |                    |
     |                   |-- lxc_map_ids() ---|                    |
     |                   |   newuidmap <pid> 0 100000 65536       |
     |                   |                    |    uid_map written |
     |                   |   newgidmap <pid> 0 100000 65536       |
     |                   |                    |    gid_map written |
     |                   |                    |                    |
     |                   |-- notify(pipe) --->|                    |
     |                   |                    |-- setuid(0) ------>|
     |                   |                    |   (= host 100000) |
     |                   |                    |                    |
     |                   |                    |-- mount/exec ----->|
     |                   |                    |                    |
     +                   +                    +                    +
```

---

## 11. lxc-usernsexec 工具

`src/lxc/cmd/lxc_usernsexec.c` 提供独立的用户命名空间执行工具：

```bash
# 在指定映射的用户命名空间中执行命令
lxc-usernsexec -m u:0:100000:65536 -m g:0:100000:65536 -- /bin/bash
```

实现流程与 `userns_exec_1()` 相同：clone → 写映射 → 通知 → exec。

---

## 12. 多映射段示例

```
# 容器 uid 0 -> 宿主 100000 (root)
lxc.idmap = u 0 100000 1

# 容器 uid 1-999 -> 宿主 100001-100999 (系统用户)
lxc.idmap = u 1 100001 999

# 容器 uid 1000 -> 宿主 1000 (穿透映射，共享文件)
lxc.idmap = u 1000 1000 1

# 容器 uid 1001-65535 -> 宿主 101001-165535
lxc.idmap = u 1001 101001 64535
```

写入 `/proc/<pid>/uid_map` 的内容：
```
0 100000 1
1 100001 999
1000 1000 1
1001 101001 64535
```
