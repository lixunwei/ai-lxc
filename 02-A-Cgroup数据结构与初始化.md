# LXC Cgroup 子系统深度分析（一）：数据结构与初始化

## 1. 概述

LXC 的 cgroup 子系统由 **cgfsng**（cgroup filesystem next generation）驱动实现，位于 `src/lxc/cgroups/` 目录。该驱动是 LXC 唯一的 cgroup 后端，支持三种布局：

| 布局 | 枚举值 | 说明 |
|------|--------|------|
| LEGACY | `CGROUP_LAYOUT_LEGACY` | 纯 cgroup v1 |
| HYBRID | `CGROUP_LAYOUT_HYBRID` | cgroup v1 + v2 混合 |
| UNIFIED | `CGROUP_LAYOUT_UNIFIED` | 纯 cgroup v2 |

核心文件：

| 文件 | 行数 | 职责 |
|------|------|------|
| `cgroup.h` | ~280 | 抽象层定义：结构体、操作表 |
| `cgroup.c` | ~100 | 初始化入口与资源释放 |
| `cgfsng.c` | ~3600 | 驱动核心实现 |
| `cgroup2_devices.c/h` | ~700 | cgroup v2 BPF 设备控制 |
| `cgroup_utils.c/h` | ~130 | 工具函数 |

---

## 2. 核心数据结构

### 2.1 `struct hierarchy` — 层级描述

定义于 `cgroup.h:99-169`，描述一个已挂载的 cgroup 层级：

```c
struct hierarchy {
    /* 该层级包含的控制器列表，如 ["memory", "pids"] */
    char **controllers;

    /* 挂载点的文件描述符（相对于 /sys/fs/cgroup） */
    int dfd_base;

    /* === Monitor cgroup === */
    int    dfd_mon;     // monitor cgroup 目录 fd
    char  *path_mon;    // monitor cgroup 路径

    /* === Container cgroup === */
    int    dfd_con;     // 容器 cgroup 目录 fd
    char  *path_con;    // 容器 cgroup 路径

    /* === Limit cgroup（资源限制） === */
    int    dfd_lim;     // limit cgroup 目录 fd
    char  *path_lim;    // limit cgroup 路径

    /* cgroup v2 特有 */
    __u32  at_flags;    // openat2 标志
    __u64  at_mnt;      // 挂载 ID

    /* 当前进程在该层级中的 cgroup 路径 */
    char  *at_base;
};
```

**三套 fd/path 的设计意图**：
- **monitor** (`dfd_mon/path_mon`)：LXC 监控进程的 cgroup
- **container** (`dfd_con/path_con`)：容器进程实际运行的 cgroup
- **limit** (`dfd_lim/path_lim`)：资源限制所写入的 cgroup

当配置了 `lxc.cgroup.dir.container.inner`（namespace_dir）时，limit 和 container 路径不同：limit cgroup 是外层，container cgroup 是内层子目录。这提供了"限制在外、隔离在内"的两级结构。

### 2.2 `struct cgroup_ops` — 操作表

定义于 `cgroup.h:185-276`，是 cgroup 子系统的核心接口：

```c
struct cgroup_ops {
    /* === 元信息 === */
    char *driver;               // "cgfsng"
    char *version;              // "1.0.0"
    cgroup_layout_t cgroup_layout;  // LEGACY / HYBRID / UNIFIED

    /* === 层级管理 === */
    struct hierarchy **hierarchies; // 所有已发现的层级数组
    struct hierarchy *unified;      // 指向统一层级（v2），可为 NULL
    int dfd_mnt;                    // /sys/fs/cgroup 的 fd

    /* === cgroup 路径 === */
    char *cgroup_pattern;       // 来自 lxc.cgroup.pattern
    char *monitor_cgroup;       // monitor cgroup 完整路径
    char *container_cgroup;     // container cgroup 完整路径
    char *container_limit_cgroup; // limit cgroup 完整路径

    /* === BPF 设备控制 === */
    struct bpf_program *cgroup2_devices;

    /* === 回调函数（约 25 个） === */
    /* 生命周期 */
    bool (*monitor_create)(...)；
    bool (*monitor_enter)(...)；
    bool (*monitor_delegate_controllers)(...)；
    void (*monitor_destroy)(...)；
    bool (*payload_create)(...)；
    bool (*payload_enter)(...)；
    bool (*payload_delegate_controllers)(...)；
    void (*payload_destroy)(...)；

    /* 资源控制 */
    int  (*set)(...)；          // 写入 cgroup 参数
    int  (*get)(...)；          // 读取 cgroup 参数
    bool (*setup_limits)(...)；  // 批量设置限制

    /* 冻结 */
    int  (*freeze)(...)；
    int  (*unfreeze)(...)；

    /* 设备 */
    bool (*devices_activate)(...)；

    /* 权限 */
    bool (*chown)(...)；

    /* CRIU */
    bool (*criu_escape)(...)；

    /* 信息查询 */
    int  (*num_hierarchies)(...)；
    bool (*get_hierarchies)(...)；
    const char *(*get_cgroup)(...)；
    int  (*get_limiting_cgroup)(...)；
};
```

### 2.3 `struct lxc_cgroup` — 配置存储

定义于 `conf.h:61-90`，存储用户配置的 cgroup 元信息：

```c
struct lxc_cgroup {
    union {
        /* 控制器条目（lxc.cgroup.xxx = yyy） */
        struct {
            int version;        // 1 或 2
            char *subsystem;    // 如 "memory.max"
            char *value;        // 如 "1G"
        };

        /* 元配置 */
        struct {
            char *controllers;          // 可用控制器列表
            char *dir;                  // lxc.cgroup.dir
            char *monitor_dir;          // lxc.cgroup.dir.monitor
            char *monitor_pivot_dir;    // lxc.cgroup.dir.monitor.pivot
            char *container_dir;        // lxc.cgroup.dir.container
            char *namespace_dir;        // lxc.cgroup.dir.container.inner
            bool relative;              // lxc.cgroup.relative
            char *systemd_scope;        // systemd scope 名称
        };
    };
};
```

### 2.4 辅助结构与枚举

```c
/* cgroup 文件系统类型识别 */
enum cgroupfs_type_magic_t {
    CGROUP_SUPER_MAGIC   = 0x27e0eb,   // cgroup v1
    CGROUP2_SUPER_MAGIC  = 0x63677270, // cgroup v2
    TMPFS_MAGIC          = 0x01021994,
};

/* 内部控制器标志（非 Linux 内核标准） */
#define DEVICES_CONTROLLER  (1U << 0)  // 设备控制器
#define FREEZER_CONTROLLER  (1U << 1)  // 冻结控制器

/* 外部传递 cgroup 信息的上下文 */
struct cgroup_fd {
    int fd;
    int layout;
};

struct cgroup_ctx {
    int layout;
    struct cgroup_fd *fds;   // 传递给外部的 fd 数组
};
```

---

## 3. 初始化流程

### 3.1 入口：`cgroup_init()`

`cgroup.c:25-50`：

```
cgroup_init(conf)
  +-- arg check
  +-- cgroup_ops_init(conf)        // detect cgroup env
  |     +-- alloc cgroup_ops
  |     +-- initialize_cgroups()   // core probe
  |     +-- install callbacks
  +-- ops->data_init(ops)          // read lxc.cgroup.pattern
  +-- print driver / layout info
```

### 3.2 环境探测：`initialize_cgroups()`

`cgfsng.c:3255-3479`，核心初始化逻辑：

#### 步骤一：确定 PID 来源

```c
// 特权模式且非 relative：读 /proc/1/cgroup（host 视角）
// 否则：读 /proc/self/cgroup（当前进程视角）
if (conf->cgroup_meta.relative || geteuid())
    pid = lxc_raw_getpid();  // self
else
    pid = 1;                  // init
```

#### 步骤二：逐行解析 `/proc/<pid>/cgroup`

对 `/proc/<pid>/cgroup` 文件的每一行：

```
格式：hierarchy-ID:controllers:path
示例：0::/user.slice/user-1000.slice        # unified (v2)
示例：7:memory:/system.slice/lxc.service    # legacy (v1)
```

**统一层级（v2）处理**：
1. 行以 `0::` 开头 → 标记 `layout_mask |= UNIFIED`
2. 在 `ops->dfd_mnt` 下打开 `"unified"` 子目录
3. `fhas_fs_type()` 验证是 `CGROUP2_SUPER_MAGIC`
4. 读取当前 cgroup 路径（优先 systemd_scope，否则从文件解析）
5. `unified_hierarchy_delegated()` 检查是否有写权限
6. 读取 `cgroup.controllers` 获取可用控制器列表

**传统层级（v1）处理**：
1. 解析 controllers 字段（如 `memory`、`cpu,cpuacct`）
2. `stable_order()` 排序控制器名称（确保一致性）
3. 在 `ops->dfd_mnt` 下打开对应挂载点
4. `fhas_fs_type()` 验证是 `CGROUP_SUPER_MAGIC`
5. `prune_init_scope()` 清理路径末尾的 `/init.scope`
6. `legacy_hierarchy_delegated()` 检查写权限
7. `skip_hierarchy()` 过滤未配置使用的控制器

#### 步骤三：确定最终布局

```c
if (ops->unified) {
    if (layout_mask & CGFSNG_LAYOUT_LEGACY)
        layout = CGROUP_LAYOUT_HYBRID;      // v1+v2 共存
    else
        layout = CGROUP_LAYOUT_UNIFIED;     // 纯 v2
} else {
    layout = CGROUP_LAYOUT_LEGACY;          // 纯 v1
}
```

### 3.3 Delegation 检测

#### 统一层级 delegation

`cgfsng.c:3153-3170`：

```c
unified_hierarchy_delegated() {
    // 读取 /sys/kernel/cgroup/delegate 获取可委派文件列表
    delegatable_files = __list_cgroup_delegate();

    // 检查这些文件在当前 cgroup 目录下是否可写
    for (file in delegatable_files) {
        if (!faccessat(dfd, file, W_OK, 0))
            continue;  // 可写，检查下一个
        return false;   // 不可写，该层级不可用
    }
    return true;
}
```

#### Delegatable 文件列表

`cgfsng.c:3110-3150`：

```c
__list_cgroup_delegate() {
    // 尝试读取 /sys/kernel/cgroup/delegate
    if (读取失败) {
        // 使用默认列表
        return ["cgroup.procs", "cgroup.threads",
                "cgroup.subtree_control", "memory.oom.group"];
    }
    // 解析文件内容，跳过 cgroup.procs（总是需要 chown）
    return parsed_list;
}
```

### 3.4 回调函数注册

`cgfsng.c:3556-3598`，`cgroup_ops_init()` 成功后注册所有操作：

```c
ops->monitor_create               = cgfsng_monitor_create;
ops->monitor_enter                = cgfsng_monitor_enter;
ops->monitor_destroy              = cgfsng_monitor_destroy;
ops->monitor_delegate_controllers = cgfsng_monitor_delegate_controllers;
ops->payload_create               = cgfsng_payload_create;
ops->payload_enter                = cgfsng_payload_enter;
ops->payload_destroy              = cgfsng_payload_destroy;
ops->payload_delegate_controllers = cgfsng_payload_delegate_controllers;
ops->setup_limits                 = cgfsng_setup_limits;
ops->devices_activate             = cgfsng_devices_activate;
ops->freeze                       = cgroup_freeze;
ops->unfreeze                     = cgroup_unfreeze;
ops->chown                        = cgfsng_chown;
ops->set                          = cgfsng_set;
ops->get                          = cgfsng_get;
ops->criu_escape                  = cgfsng_criu_escape;
// ... 等
```

---

## 4. Cgroup 创建

### 4.1 Monitor Cgroup 创建

`cgfsng_monitor_create()`，`cgfsng.c:1363-1442`：

#### 路径确定优先级

```
1. conf->cgroup_meta.monitor_dir        (user-specified)
2. conf->cgroup_meta.dir + "/lxc.monitor." + name + RETRY_SUFFIX
3. cgroup_pattern replace %n + "/lxc.monitor" + RETRY_SUFFIX
4. "lxc.monitor." + name + RETRY_SUFFIX  (default)
```

#### 创建与重试

```
for idx = 0 to 999:
    for each hierarchy:
        cgroup_tree_create(hierarchy, monitor_cgroup, NULL, false)
    if all succeed:
        break
    else:
        rollback created ones(cgroup_tree_prune)
        change suffix to "-1", "-2", ...
        retry
```

最多重试 999 次（`idx == 1000` 时报 `ERANGE`），防止名称冲突。

### 4.2 Payload Cgroup 创建

`cgfsng_payload_create()`，`cgfsng.c:1449-1552`：

#### 路径确定优先级

```
1. conf->cgroup_meta.container_dir      (user-specified)
   +-- with namespace_dir: limit=container_dir, container=container_dir/namespace_dir
   +-- without namespace_dir: limit == container(no isolation)
2. conf->cgroup_meta.dir + "/lxc.payload." + name + RETRY_SUFFIX
3. cgroup_pattern replace %n + "/lxc.payload" + RETRY_SUFFIX
4. "lxc.payload." + name + RETRY_SUFFIX  (default)
```

#### Limit vs Container 的两级结构

```
/sys/fs/cgroup/.../<limit_cgroup>/          <- resource limits here
                    +-- <namespace_dir>/    <- container procs run here
```

这种设计允许：
- 在 limit cgroup 层设置 `memory.max`、`cpu.max` 等限制
- 容器进程运行在内层子目录，拥有独立的 cgroup namespace 视图

### 4.3 Controller Delegation（cgroup v2）

`cgfsng.c:2989-3073`，仅在 **纯 unified** 布局下生效：

```c
// 构造要启用的控制器字符串
// 例如："+memory +pids +cpu +io"
for (controller in unified->controllers)
    buf += "+" + controller + " ";

// 沿路径逐级写入 cgroup.subtree_control
// 必须从祖先到子目录逐级启用
for (level = root; level < target; level++) {
    write(level + "/cgroup.subtree_control", buf);
    open(level + "/" + next_component);
}
```

这是 cgroup v2 的 delegation 模型要求：父目录必须先在 `cgroup.subtree_control` 中启用控制器，子目录才能使用。

---

## 5. 配置项一览

以下配置项影响 cgroup 行为（`confile.c:191-207`）：

| 配置项 | 说明 |
|--------|------|
| `lxc.cgroup.<controller>.<key>` | cgroup v1 控制器参数 |
| `lxc.cgroup2.<controller>.<key>` | cgroup v2 控制器参数 |
| `lxc.cgroup.dir` | cgroup 目录基础路径 |
| `lxc.cgroup.dir.monitor` | monitor cgroup 路径 |
| `lxc.cgroup.dir.monitor.pivot` | monitor pivot cgroup 路径 |
| `lxc.cgroup.dir.container` | 容器 cgroup 路径 |
| `lxc.cgroup.dir.container.inner` | 容器内层 namespace 目录 |
| `lxc.cgroup.relative` | 是否使用相对 cgroup 路径 |
| `lxc.cgroup.pattern` | cgroup 命名模板（`%n` = 容器名） |

---

## 6. 资源释放

`cgroup.c:52-99`，`cgroup_exit()` 负责完整清理：

```
cgroup_exit(ops)
  +-- free all hierarchies
  |     +-- close dfd_base / dfd_mon / dfd_con / dfd_lim
  |     +-- free path_mon / path_con / path_lim / at_base
  |     +-- free controllers array
  +-- close dfd_mnt
  +-- free cgroup_pattern / monitor_cgroup / container_cgroup
  +-- free BPF device program(bpf_program)
  +-- free ops itself
```

---

## 7. 关键设计总结

1. **fd-based 操作**：cgfsng 全面使用文件描述符（`openat/fstatat/mkdirat`）而非路径字符串操作 cgroup 文件系统，避免 TOCTOU 竞争和路径拼接错误。

2. **三套 cgroup 路径**：monitor / container / limit 的分离提供了灵活的权限和资源控制模型。

3. **自动布局检测**：通过解析 `/proc/<pid>/cgroup` 自动适配三种 cgroup 布局，对用户透明。

4. **重试机制**：cgroup 创建支持最多 999 次后缀递增重试，处理并发容器启动时的名称冲突。

5. **delegation 感知**：主动检测 cgroup 文件的写权限，确保在非特权场景下也能正确工作。
