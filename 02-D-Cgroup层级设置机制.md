# Cgroup 层级设置机制深度分析

## 1. 概述

LXC 的 cgroup 管理通过 `cgfsng` 驱动实现（"cgroup filesystem next generation"），支持三种布局模式。核心逻辑集中在 `src/lxc/cgroups/cgfsng.c`（约 3800 行）。

**关键设计理念**：
- Monitor（管理进程）和 Payload（容器进程）使用**独立 cgroup**
- 资源限制应用在 **limit cgroup**，容器实际运行在其下的 **inner cgroup**（叶节点）
- cgroup v2 设备控制通过 **BPF 程序**替代文件写入

---

## 2. 核心数据结构

### 2.1 hierarchy（单个层级描述）

```c
struct hierarchy {
    cgroupfs_type_magic_t fs_type;  // LEGACY_HIERARCHY | UNIFIED_HIERARCHY

    // Payload container cgroup
    int dfd_con;       // fd -> 容器进程所在的叶 cgroup
    char *path_con;    // 完整路径

    // Limit cgroup (资源限制写入位置)
    int dfd_lim;       // fd -> 限制 cgroup (可能 == dfd_con)
    char *path_lim;    // 完整路径

    // Monitor cgroup
    int dfd_mon;       // fd -> monitor 进程 cgroup

    // Mountpoint & base
    int dfd_mnt;       // /sys/fs/cgroup 或 /sys/fs/cgroup/<controller>
    char *at_mnt;
    int dfd_base;      // 当前进程/init 的 cgroup (创建的起点)
    char *at_base;

    // Unified 专属
    struct {
        char **delegate;       // 可委托的文件列表
        unsigned int utilities; // DEVICES_CONTROLLER | FREEZER_CONTROLLER
    };

    char **controllers;  // 此层级管理的控制器列表
};
```

### 2.2 cgroup_ops（操作集合）

```c
struct cgroup_ops {
    const char *driver;       // "cgfsng"
    const char *version;      // "1.0.0"
    int dfd_mnt;              // /sys/fs/cgroup 顶层 fd
    cgroup_layout_t cgroup_layout;  // LEGACY | HYBRID | UNIFIED

    char **cgroup_use;              // lxc.cgroup.use 指定的控制器
    char *cgroup_pattern;           // lxc.cgroup.pattern (e.g. "lxc/%n")
    char *container_cgroup;         // payload inner cgroup 路径
    char *container_limit_cgroup;   // payload limit cgroup 路径
    char *monitor_cgroup;           // monitor cgroup 路径

    struct hierarchy **hierarchies; // 所有层级数组 (NULL 终止)
    struct hierarchy *unified;      // 指向统一层级 (不单独释放)
    struct bpf_program *cgroup2_devices; // BPF 设备控制程序

    // 函数指针 (下文详述)
    bool (*monitor_create)(...);
    bool (*payload_create)(...);
    bool (*monitor_enter)(...);
    bool (*payload_enter)(...);
    int (*set)(...);
    int (*get)(...);
    bool (*setup_limits)(...);
    bool (*chown)(...);
    // ... 等 20+ 个操作
};
```

### 2.3 布局类型

```c
typedef enum {
    CGROUP_LAYOUT_UNKNOWN = -1,
    CGROUP_LAYOUT_LEGACY  =  0,  // 全 v1 (各控制器独立挂载)
    CGROUP_LAYOUT_HYBRID  =  1,  // v1 + v2 混合
    CGROUP_LAYOUT_UNIFIED =  2,  // 纯 v2 (单一层级)
} cgroup_layout_t;
```

---

## 3. 初始化流程

```
cgroup_init(conf)
  |
  +-> cgroup_ops_init()
        |
        +-> 分配 cgroup_ops, 设置 layout = UNKNOWN
        +-> initialize_cgroups()
        |     |
        |     +-> 打开 /sys/fs/cgroup (ops->dfd_mnt)
        |     +-> 加载 lxc.cgroup.use (控制器过滤)
        |     +-> __initialize_cgroups()
        |           |
        |           +-> unpriv_systemd_create_scope() // 非特权: 创建 systemd scope
        |           +-> 读取 /proc/1/cgroup (root) 或 /proc/self/cgroup (非root)
        |           +-> 逐行解析, 构建 hierarchy 数组
        |           +-> 确定 layout_mask -> cgroup_layout
        |
        +-> 填充函数指针 (monitor_create, payload_enter, set, get, ...)
```

### 3.1 布局检测逻辑

解析 `/proc/{1|self}/cgroup` 每一行：

```
# cgroup v2 统一层级
0::/path           --> unified, layout_mask |= UNIFIED

# cgroup v1 legacy 层级  
3:memory:/path     --> legacy, layout_mask |= LEGACY
2:cpu,cpuacct:/p   --> legacy (co-mounted controllers)
```

最终判定：
- `UNIFIED only` → `CGROUP_LAYOUT_UNIFIED`
- `LEGACY only` → `CGROUP_LAYOUT_LEGACY`  
- 两者都有 → `CGROUP_LAYOUT_HYBRID`

### 3.2 层级构建细节

**Legacy 层级**：每行创建一个 `struct hierarchy`
```
/sys/fs/cgroup/memory/     -> hierarchy[0] controllers=["memory"]
/sys/fs/cgroup/cpu,cpuacct/ -> hierarchy[1] controllers=["cpu","cpuacct"]
/sys/fs/cgroup/pids/       -> hierarchy[2] controllers=["pids"]
```

**Unified 层级**：读取 `cgroup.controllers` 获得可用控制器
```
/sys/fs/cgroup/            -> hierarchy[0] controllers=["memory","pids","cpu","io",...]
```

---

## 4. 路径构建算法

### 4.1 Monitor cgroup 路径

优先级（从高到低）：

| 优先级 | 配置源 | 生成路径 |
|--------|--------|----------|
| 1 | `lxc.cgroup.dir.monitor` | 直接使用配置值 |
| 2 | `lxc.cgroup.dir` | `<dir>/lxc.monitor.<name>-<N>` |
| 3 | `lxc.cgroup.pattern` | `<pattern(%n=name)>/monitor-<N>` |
| 4 | 默认 | `lxc.monitor.<name>-<N>` |

### 4.2 Payload cgroup 路径

| 优先级 | 配置源 | 生成路径 |
|--------|--------|----------|
| 1 | `lxc.cgroup.dir.container` + `.namespace_dir` | limit=`<container_dir>`, inner=`<limit>/<namespace_dir>` |
| 2 | `lxc.cgroup.dir` | `<dir>/lxc.payload.<name>-<N>` |
| 3 | `lxc.cgroup.pattern` | `<pattern(%n=name)>/payload-<N>` |
| 4 | 默认 | `lxc.payload.<name>-<N>` |

### 4.3 冲突解决

路径末尾预留后缀空间 `CGROUP_CREATE_RETRY`（"-NNN"）。创建失败时递增 idx（0→999），自动避免路径冲突。

### 4.4 目录创建（__cgroup_tree_create）

逐组件 `mkdirat()` 创建路径，安全检查：
- 禁止绝对路径
- 禁止 `..` 向上遍历
- 使用 `PROTECT_OPATH_DIRECTORY` + `PROTECT_LOOKUP_BENEATH` 防止符号链接攻击

---

## 5. Monitor vs Payload 分离

### 5.1 为什么要分离？

```
               /sys/fs/cgroup/.../
                       |
          +------------+------------+
          |                         |
  lxc.monitor.myct/          lxc.payload.myct/    <-- limit cgroup
  (handler parent)                  |
                              ns/  (可选)          <-- inner/leaf cgroup
                           (container init)
```

**目的**：
1. Monitor 的资源消耗不计入容器配额
2. 容器无法通过杀死 cgroup 内所有进程来影响 monitor
3. cgroup v2 "no internal processes" 规则要求：如果目录有 subtree_control，则进程必须在叶节点

### 5.2 进入流程

```c
// 1. Monitor 进入（handler parent process）
cgfsng_monitor_enter(ops, handler)
    -> lxc_writeat(h->dfd_mon, "cgroup.procs", monitor_pid)

// 2. Payload 进入（容器 init 进程）
cgfsng_payload_enter(ops, handler)
    -> lxc_writeat(h->dfd_con, "cgroup.procs", container_pid)
```

### 5.3 Limit cgroup vs Inner cgroup

```
lxc.payload.myct/           <-- LIMIT cgroup (资源限制写在这里)
    |-- memory.max = 512M
    |-- cpu.max = 100000 100000
    |
    +-- ns/                 <-- INNER/LEAF cgroup (进程实际运行)
         |-- cgroup.procs = <container_pid>
```

- 资源限制（`memory.max`, `cpu.max`）写入 `dfd_lim`
- 进程写入 `dfd_con`（叶节点）
- 若无 `namespace_dir` 配置，`dfd_lim == dfd_con`（合并为一个 cgroup）

---

## 6. 控制器委托（Delegation）

### 6.1 Unified (v2) 控制器启用

在 cgroup v2 中，子 cgroup 的控制器不会自动启用。LXC 必须逐层写入 `cgroup.subtree_control`：

```c
__cgfsng_delegate_controllers(ops, cgroup_path) {
    // 构建 "+memory +pids +cpu +io" 字符串
    for each controller:
        add_controllers += "+<ctrl> "

    // 从 base 开始逐层写入
    for each component in path:
        lxc_writeat(dfd_cur, "cgroup.subtree_control", add_controllers);
        dfd_cur = open(next_component);
}
```

**示例**：路径 `lxc.payload.myct/ns`
```
/sys/fs/cgroup/<base>/cgroup.subtree_control  <- write "+memory +pids +cpu +io"
/sys/fs/cgroup/<base>/lxc.payload.myct/cgroup.subtree_control <- write "+memory +pids +cpu +io"
# ns/ 是叶节点，不需要写入 subtree_control
```

### 6.2 非特权容器的 chown

非特权容器需要 chown cgroup 目录给容器的用户命名空间 root：

**Unified (v2)**：
```c
// chown 以下文件给 container uid 0 映射的 host uid
chown: "."
chown: "cgroup.procs"
chown: "cgroup.threads"
chown: "cgroup.subtree_control"
chown: "memory.oom.group"
// 加上 /sys/kernel/cgroup/delegate 列出的文件
```

**Legacy (v1)**：
```c
chown: "."        // 目录本身
chown: "tasks"
chown: "cgroup.procs"
```

---

## 7. 配置到 cgroup 文件的映射

### 7.1 写入路径

```
lxc.cgroup.memory.limit_in_bytes = 512M     (v1)
lxc.cgroup2.memory.max = 536870912          (v2)
```

处理流程：

```c
cgfsng_set(ops, key="memory.limit_in_bytes", value="512M", ...) {
    // 1. 提取控制器名: "memory"
    controller = "memory"

    // 2. 特殊处理: v2 devices -> BPF
    if (unified && controller == "devices") {
        device_cgroup_rule_parse(&device, key, value);
        lxc_cmd_add_bpf_device_cgroup(name, lxcpath, &device);
        return;
    }

    // 3. 获取 limit cgroup 路径
    path = lxc_cmd_get_limit_cgroup_path(name, lxcpath, controller);

    // 4. 找到对应 hierarchy
    h = get_hierarchy(ops, "memory");

    // 5. 写入文件
    fullpath = h->path_lim + "/" + key;  // e.g. .../memory.limit_in_bytes
    lxc_write_to_file(fullpath, "512M", ...);
}
```

### 7.2 setup_limits（启动时批量应用）

容器启动时，`cgfsng_setup_limits()` 遍历配置中所有 `lxc.cgroup.*` 条目：

```c
cgfsng_setup_limits(ops, handler) {
    for each cgroup_item in conf->cgroup_list:
        // 解析 controller 和 filename
        h = get_hierarchy(ops, controller);
        // 写入 h->dfd_lim/<filename>
        lxc_writeat(h->dfd_lim, filename, value, strlen(value));
}
```

### 7.3 v2 设备控制（BPF）

cgroup v2 移除了 `devices.*` 文件，改用 BPF：

```c
cgfsng_devices_activate(ops, handler) {
    // 1. 解析所有 lxc.cgroup2.devices.* 配置
    // 2. 编译为 BPF_PROG_TYPE_CGROUP_DEVICE 程序
    // 3. BPF_PROG_ATTACH 到容器的 cgroup fd
    bpf_program_cgroup_attach(ops->cgroup2_devices, BPF_CGROUP_DEVICE,
                              h->dfd_lim, BPF_F_ALLOW_MULTI);
}
```

---

## 8. 完整时序图

容器启动时 cgroup 相关操作时序：

```
+------------------+     +-------------------+     +------------------+
| lxc_handler      |     | cgroup_ops        |     | Kernel cgroup    |
+------------------+     +-------------------+     +------------------+
        |                         |                         |
        |-- cgroup_init() ------->|                         |
        |                         |-- open /sys/fs/cgroup ->|
        |                         |-- parse /proc/*/cgroup->|
        |                         |-- build hierarchies[] ->|
        |                         |                         |
        |-- monitor_create() ---->|                         |
        |                         |-- mkdirat(monitor_cg) ->|
        |                         |                         |
        |-- monitor_enter() ----->|                         |
        |                         |-- write cgroup.procs -->|
        |                         |                         |
        |-- payload_create() ---->|                         |
        |                         |-- mkdirat(limit_cg) --->|
        |                         |-- mkdirat(inner_cg) --->|
        |                         |                         |
        |-- delegate_controllers->|                         |
        |                         |-- write subtree_ctrl -->|
        |                         |   (each path level)     |
        |                         |                         |
        |-- setup_limits() ------>|                         |
        |                         |-- write memory.max ---->|
        |                         |-- write cpu.max ------->|
        |                         |-- write pids.max ------>|
        |                         |                         |
        |-- devices_activate() -->|                         |
        |                         |-- BPF_PROG_ATTACH ----->|
        |                         |                         |
        |-- chown() (unpriv) ---->|                         |
        |                         |-- chown cgroup files -->|
        |                         |                         |
        |-- payload_enter() ----->|                         |
        |                         |-- write cgroup.procs -->|
        |                         |   (container pid)       |
        +                         +                         +
```

---

## 9. 销毁流程

```c
// Monitor 销毁
cgfsng_monitor_destroy(ops, handler) {
    // 杀死 monitor cgroup 中的所有进程
    // 递归删除 monitor cgroup 目录
    cgroup_tree_prune(h->dfd_base, ops->monitor_cgroup);
}

// Payload 销毁
cgfsng_payload_destroy(ops, handler) {
    // 先清理 inner cgroup (ns/)
    // 再清理 limit cgroup
    cgroup_tree_prune(h->dfd_base, ops->container_limit_cgroup);
}
```

清除使用 `cgroup_kill_tree()` 先杀进程，再 `unlinkat(dfd, dir, AT_REMOVEDIR)` 删除空 cgroup。

---

## 10. 特殊场景

### 10.1 Systemd 集成

非特权用户启动容器时，先通过 DBus 请求 systemd 创建 scope：
```c
unpriv_systemd_create_scope(ops, conf)
    // 创建 transient scope unit
    // 容器在 scope 内获得 cgroup 委托权限
```

### 10.2 Hybrid 布局处理

Hybrid 模式下，LXC 同时维护 legacy 和 unified hierarchies：
- Legacy 控制器通过对应 `/sys/fs/cgroup/<ctrl>/` 目录
- Unified 控制器通过 `/sys/fs/cgroup/unified/`
- 设备和 freezer 在 unified 中作为"utility controllers"特殊处理

### 10.3 cgroup.use 过滤

配置 `lxc.cgroup.use = memory;pids` 时，仅初始化指定控制器对应的 hierarchy，忽略其余。

### 10.4 相对路径模式

`lxc.cgroup.relative = 1` 时：
- 使用 `/proc/self/cgroup`（而非 `/proc/1/cgroup`）作为 base
- 容器 cgroup 创建在当前进程所在 cgroup 之下
- 主要用于嵌套容器场景
