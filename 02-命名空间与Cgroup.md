# 02 - 命名空间与 Cgroup

## 1. Namespace 处理（namespace.c/h）

### 1.1 支持的命名空间

LXC 支持 8 种 Linux 命名空间（`namespace.h:14-24`）：

| 索引 | 名称 | Clone Flag | proc 路径 | 用途 |
|------|------|------------|-----------|------|
| 0 | user | `CLONE_NEWUSER` | `/proc/<pid>/ns/user` | 用户/权限隔离 |
| 1 | mnt | `CLONE_NEWNS` | `/proc/<pid>/ns/mnt` | 挂载点隔离 |
| 2 | pid | `CLONE_NEWPID` | `/proc/<pid>/ns/pid` | 进程 ID 隔离 |
| 3 | uts | `CLONE_NEWUTS` | `/proc/<pid>/ns/uts` | 主机名隔离 |
| 4 | ipc | `CLONE_NEWIPC` | `/proc/<pid>/ns/ipc` | IPC 隔离 |
| 5 | net | `CLONE_NEWNET` | `/proc/<pid>/ns/net` | 网络隔离 |
| 6 | cgroup | `CLONE_NEWCGROUP` | `/proc/<pid>/ns/cgroup` | Cgroup 视图隔离 |
| 7 | time | `CLONE_NEWTIME` | `/proc/<pid>/ns/time` | 时钟偏移隔离 |

### 1.2 关键设计：User Namespace 优先

`ns_info[]` 数组中 **user namespace 被放在首位**（`namespace.c:22-37`），这是一个关键设计决策：

> 在执行 `setns()` 进入容器 namespace 时，必须先进入 user namespace，
> 否则在非特权场景下，后续进入 mount namespace 等操作会被内核权限检查拒绝。

### 1.3 主要管理函数

| 函数 | 功能 | 位置 |
|------|------|------|
| `lxc_namespace_2_cloneflag()` | 字符串 → `CLONE_NEW*` flag | `namespace.c:49-59` |
| `lxc_namespace_2_ns_idx()` | 字符串 → 枚举索引 | `namespace.c:61-70` |
| `lxc_fill_namespace_flags()` | 解析 `IPC\|MOUNT\|PID` 列表并累积 flags | `namespace.c:106-125` |

---

## 2. Cgroup 子系统

### 2.1 架构概览

```
cgroup.h          ← 抽象层定义
cgroup.c          ← 初始化入口
cgfsng.c          ← 主驱动实现（cgfsng = cgroup filesystem next generation）
cgroup2_devices.c ← BPF 设备控制（cgroup v2 专用）
cgroup_utils.c    ← 辅助函数
```

### 2.2 布局类型

LXC 支持三种 cgroup 布局（`cgroup.h:32-53`）：

| 布局 | 说明 |
|------|------|
| `CGROUP_LAYOUT_LEGACY` | 纯 cgroup v1 |
| `CGROUP_LAYOUT_HYBRID` | v1 + v2 混合（v2 仅 unified hierarchy） |
| `CGROUP_LAYOUT_UNIFIED` | 纯 cgroup v2 |

布局检测通过扫描挂载点和控制器兼容性来判定（`cgfsng.c:3270-3482`）。

### 2.3 核心数据结构

#### `struct hierarchy`（`cgroup.h:99-169`）

描述每个 cgroup 挂载层级：

```c
struct hierarchy {
    int fd;              // 挂载点 fd
    char *controllers;   // 控制器列表（如 "cpu,memory"）
    char *__mountpoint;  // 挂载路径
    char *container_base_path;  // 容器 cgroup 路径
    char *monitor_full_path;    // 监视进程 cgroup 路径
    // ...
};
```

#### `struct cgroup_ops`（`cgroup.h:185-276`）

统一的操作接口：

| 操作 | 说明 |
|------|------|
| `create` | 创建 cgroup 层级 |
| `enter` | 将进程移入 cgroup |
| `set` / `get` | 读写 cgroup 参数 |
| `freeze` / `unfreeze` | 冻结/解冻 |
| `setup_limits` | 设置资源限制 |
| `devices_activate` | 激活 BPF 设备控制 |
| `monitor_delegate_controllers` | 委派控制器 |
| ... | 其他 |

### 2.4 cgfsng 驱动

cgfsng 是 LXC 的主 cgroup 驱动，全部实现在 `cgfsng.c`（约 3600 行）。

#### 初始化流程

```
cgroup_init()                     ← cgroup.c:25-49
  └── cgroup_ops_init()           ← cgfsng.c:3538-3595
        ├── 设置默认布局
        ├── 初始化所有函数指针
        └── cgfsng_data_init()    ← 读取 lxc.cgroup.pattern
```

#### Cgroup 组织方式

```
ops->hierarchies[]     ← 所有 v1 hierarchy（每个控制器或 co-mounted 集合）
ops->unified           ← v2 unified hierarchy
ops->monitor_cgroup    ← 监视进程所在 cgroup
ops->container_cgroup  ← payload 容器的 inner cgroup
ops->container_limit_cgroup ← payload 容器的 limit cgroup
```

#### 容器 Cgroup 创建

**Monitor cgroup**（`cgfsng.c:1363-1442`）：
```
cgfsng_monitor_create()
  └── 路径: lxc.monitor.<name>
      失败时带 -1..-999 后缀重试
```

**Payload cgroup**（`cgfsng.c:1449-1552`）：
```
cgfsng_payload_create()
  └── 路径: lxc.payload.<name>
      同样支持重试
```

#### v1 / v2 处理差异

| 特性 | cgroup v1 | cgroup v2 |
|------|-----------|-----------|
| 层级 | 多层级，每个控制器独立挂载 | 单一 unified 层级 |
| 冻结 | 写 `freezer.state` | 写 `cgroup.freeze` |
| 设备控制 | 写 `devices.allow/deny` | BPF 程序 |
| 控制器委派 | 直接可用 | 写 `cgroup.subtree_control` |
| 可委派文件 | 无此概念 | 读 `/sys/kernel/cgroup/delegate` |

### 2.5 资源限制设置

```
cgfsng_setup_limits()              ← cgfsng.c:2905-2951
  ├── 仅对 unified(v2) 的 conf->cgroup2 生效
  ├── devices 条目 → 走 BPF devices 流程
  └── 其他控制器 → 直接写 cgroup.* 文件
```

---

## 3. BPF 设备控制（cgroup2_devices.c/h）

### 3.1 设计目标

在 cgroup v2 中，设备访问控制不再通过文件（`devices.allow/deny`），而是使用 BPF 程序类型 `BPF_PROG_TYPE_CGROUP_DEVICE`。

### 3.2 核心数据结构

`struct bpf_program`（`cgroup2_devices.h:39-71`）：封装了 BPF 程序的指令缓冲区和元数据。

### 3.3 BPF 程序构建流程

```
bpf_program_init()                 ← 初始化，从 bpf_cgroup_dev_ctx 读取：
  │                                   access_type, major, minor
  │                                   cgroup2_devices.c:178-200
  │
  ├── bpf_program_append_device()  ← 为每条设备规则生成条件跳转指令
  │                                   cgroup2_devices.c:202-281
  │                                   支持 allowlist/denylist 策略
  │
  └── bpf_program_finalize()       ← 生成默认返回值
                                      cgroup2_devices.c:284-299
```

### 3.4 BPF 程序加载与更新

| 操作 | 说明 | 位置 |
|------|------|------|
| `bpf_cgroup_devices_attach()` | 将 BPF 程序 attach 到 cgroup | `cgroup2_devices.c:595-614` |
| `bpf_cgroup_devices_update()` | 动态更新；优先 `BPF_F_REPLACE`，不支持则 append | `cgroup2_devices.c:616-706` |
| `bpf_devices_cgroup_supported()` | 检测内核是否支持 BPF 设备 cgroup | `cgroup2_devices.c:520-554` |

### 3.5 启用条件

在 cgfsng 中，BPF 设备控制仅在以下条件满足时启用（`cgfsng.c:2968-2976`）：

1. 纯 unified（cgroup v2）布局
2. unified hierarchy 有 `devices` utility controller

---

## 4. 冻结/解冻（freezer.c）

### 4.1 Legacy 方式（cgroup v1）

通过 `freezer.state` 文件驱动（`freezer.c:28-86`）：

```
do_freeze_thaw()
  ├── 初始化 cgroup ops
  ├── 写 freezer.state = "FROZEN" / "THAWED"
  └── 循环读取 freezer.state 直到状态匹配
```

调用入口：
- `lxc_freeze()` — 先通知 `FREEZING` 状态，完成后通知 `FROZEN/RUNNING`
- `lxc_unfreeze()` — 类似

### 4.2 Unified 方式（cgroup v2）

仅在纯 unified 布局下支持（`cgfsng.c:2192-2218`）：

```
写 cgroup.freeze = 1    ← 冻结
写 cgroup.freeze = 0    ← 解冻
用 epoll 等待状态变化
```

---

## 5. 能力管理（caps.c/h）

### 5.1 接口设计

当系统有 libcap 时提供完整功能，否则退化为空操作（`caps.h:12-67`）。

### 5.2 核心函数

| 函数 | 功能 | 位置 |
|------|------|------|
| `lxc_caps_down()` | 非 root 时清除 effective capabilities | `caps.c:25-47` |
| `lxc_caps_up()` | 将 permitted 集合复制到 effective | `caps.c:49-86` |
| `lxc_ambient_caps_up()` | permitted → inheritable → `PR_CAP_AMBIENT_RAISE` | `caps.c:88-139` |
| `lxc_ambient_caps_down()` | 清空 ambient 和 inheritable | `caps.c:141-170` |
| `lxc_caps_init()` | setuid-root 场景：`PR_SET_KEEPCAPS` → 降权 → 恢复 | `caps.c:172-209` |
| `lxc_caps_last_cap()` | 读 `/proc/sys/kernel/cap_last_cap` 或探测 | `caps.c:211-276` |
| `lxc_proc_cap_is_set()` | 查询进程 capability | `caps.c:278-302` |
| `lxc_file_cap_is_set()` | 查询文件 capability | `caps.c:304-324` |

### 5.3 Capability 在容器中的应用

在容器配置中，通过以下配置项控制：

- `lxc.cap.drop` — 从容器中移除的能力
- `lxc.cap.keep` — 容器中保留的能力（白名单模式）

启动时，LXC 会根据配置调整容器 init 进程的 capability 集合，实现最小权限原则。
