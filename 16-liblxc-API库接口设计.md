# liblxc API 库接口设计深度分析

## 1. 概述

liblxc 是 LXC 的 C 语言共享库，提供面向对象风格的容器管理 API。所有操作围绕 `struct lxc_container` 展开，通过函数指针实现方法调用，类似 C 语言的"虚函数表"模式。

**核心源文件**：
- `src/lxc/lxccontainer.h` — 公共 API 头文件（约 1170 行）
- `src/lxc/lxccontainer.c` — 实现（约 5400 行）

---

## 2. struct lxc_container 结构

### 2.1 私有字段

```c
struct lxc_container {
    /* 私有字段 */
    char *name;              // 容器名称
    char *configfile;        // 配置文件完整路径
    char *pidfile;           // PID 文件路径
    struct lxc_lock *slock;  // 信号量锁（磁盘级）
    struct lxc_lock *privlock; // 私有锁（内存级）
    int numthreads;          // 引用计数（privlock 保护）
    struct lxc_conf *lxc_conf; // 内部配置结构

    /* 公共字段 */
    char *error_string;      // 最后一次错误描述
    int error_num;           // 最后一次错误码
    bool daemonize;          // 是否后台运行
    char *config_path;       // 配置目录路径

    /* 方法（函数指针）... */
};
```

### 2.2 方法分类总览

| 类别 | 方法 | 数量 |
|------|------|------|
| 状态查询 | `is_defined`, `state`, `is_running`, `init_pid`, `init_pidfd` | 5 |
| 生命周期 | `start`, `startl`, `stop`, `destroy`, `reboot`, `reboot2`, `shutdown`, `wait` | 8 |
| 创建/克隆 | `create`, `createl`, `clone`, `rename` | 4 |
| 配置管理 | `load_config`, `save_config`, `set_config_item`, `get_config_item`, `clear_config`, `clear_config_item`, `get_keys`, `get_running_config_item`, `config_file_name`, `get_config_path`, `set_config_path` | 11 |
| 运行时控制 | `freeze`, `unfreeze`, `want_daemonize`, `want_close_all_fds`, `set_timeout`, `may_control` | 6 |
| Cgroup | `get_cgroup_item`, `set_cgroup_item` | 2 |
| 网络 | `get_interfaces`, `get_ips`, `attach_interface`, `detach_interface` | 4 |
| Attach/Console | `attach`, `attach_run_wait`, `attach_run_waitl`, `console`, `console_getfd`, `console_log`, `devpts_fd` | 7 |
| 快照 | `snapshot`, `snapshot_list`, `snapshot_restore`, `snapshot_destroy`, `snapshot_destroy_all`, `destroy_with_snapshots` | 6 |
| 设备 | `add_device_node`, `remove_device_node` | 2 |
| 挂载 | `mount`, `umount` | 2 |
| 迁移 | `checkpoint`, `restore`, `migrate` | 3 |
| 安全 | `seccomp_notify_fd`, `seccomp_notify_fd_active` | 2 |
| **总计** | | **62** |

---

## 3. 容器对象生命周期

### 3.1 创建对象：lxc_container_new()

```c
struct lxc_container *c = lxc_container_new("myct", NULL);
// NULL 表示使用默认 lxcpath (/var/lib/lxc)
```

内部流程：

```
lxc_container_new("myct", NULL)
  |
  +-> malloc(sizeof(struct lxc_container))
  +-> 设置 config_path (默认或自定义)
  +-> 复制 name
  +-> numthreads = 1
  +-> 创建 slock (磁盘锁, 基于文件)
  +-> 创建 privlock (内存锁, 进程内)
  +-> 构建 configfile 路径
  +-> 如果配置文件存在: load_config()
  +-> 检测未完成的创建 (ongoing_create)
  |     INCOMPLETE -> destroy + clear
  |     ONGOING -> 跳过
  |     FAILED -> 报错
  +-> daemonize = true
  +-> 绑定 62 个函数指针
  +-> return c
```

### 3.2 引用计数

```c
// 增加引用
int lxc_container_get(struct lxc_container *c);
// numthreads++, 返回 1 成功, 0 失败（已释放）

// 减少引用
int lxc_container_put(struct lxc_container *c);
// numthreads--, 到 0 时自动释放, 返回 1 表示已释放
```

**竞态保护**：

```
get()er:                        put()er:
  if numthreads < 1 -> bail      lock(privlock)
  lock(privlock)                 numthreads--
  if numthreads < 1 -> bail      if numthreads < 1:
  numthreads++                     unlock()
  unlock()                         free(container)
```

双重检查 `numthreads < 1`（锁前+锁后）避免 use-after-free。

### 3.3 释放

```c
lxc_container_free(c) {
    free(configfile)
    free(error_string)
    lxc_putlock(slock)
    lxc_putlock(privlock)
    free(name)
    lxc_conf_free(lxc_conf)
    free(config_path)
    free(c)
}
```

---

## 4. 锁模型

```
+-----------------------------------+
| container_mem_lock(c)             |
| - 保护 struct lxc_container 内存  |
| - 用于 numthreads, 配置读写等     |
+-----------------------------------+
         |
         v
+-----------------------------------+
| container_disk_lock(c)            |
| - 保护磁盘上的配置文件            |
| - 内部先获取 mem_lock 再获取      |
|   文件锁 (slock)                  |
+-----------------------------------+
         |
         v
+-----------------------------------+
| thread_mutex (全局)               |
| - 保护进程级数据 (如 fd 表)       |
+-----------------------------------+
```

**设计原则**：锁只在 API 函数内部持有，不跨函数。这避免了调用者导致的死锁，但意味着两个进程操作同一容器时需要调用者自行协调。

---

## 5. API 模式与约定

### 5.1 do_lxcapi / lxcapi 二层模式

```c
// 内部实现（不加锁）
static bool do_lxcapi_is_defined(struct lxc_container *c) {
    // 实际逻辑
    return file_exists(c->configfile);
}

// 公开包装（可能加锁）
static bool lxcapi_is_defined(struct lxc_container *c) {
    // 有些添加锁，有些直接调用 do_ 版本
    return do_lxcapi_is_defined(c);
}
```

### 5.2 返回值约定

| 返回类型 | 成功 | 失败 |
|----------|------|------|
| `bool` | `true` | `false` |
| `int` | `0` 或正数 | `-1` 或负 errno |
| `const char *` | 有效指针（静态，不可 free） | `NULL` |
| `char *` | 有效指针（**调用者负责 free**） | `NULL` |
| `char **` | 数组（**调用者负责 free**） | `NULL` |
| `pid_t` | 有效 PID | `-1` |

### 5.3 "查询长度再分配"模式

```c
// get_config_item 支持两次调用模式:
// 第一次: 传 NULL 获取所需长度
int len = c->get_config_item(c, "lxc.net.0.type", NULL, 0);
char *buf = malloc(len + 1);
// 第二次: 传缓冲区获取值
c->get_config_item(c, "lxc.net.0.type", buf, len + 1);
```

### 5.4 Varargs "l" 变体

```c
// 数组版本
char *argv[] = {"ls", "-la", NULL};
c->start(c, 0, argv);

// Varargs 版本（对绑定语言更友好）
c->startl(c, 0, "ls", "-la", NULL);
```

---

## 6. 关键 API 详解

### 6.1 创建容器

```c
// 使用模板创建
c->create(c, "download", NULL, NULL, 0, args);
// template = "download"
// 调用 /usr/share/lxc/templates/lxc-download 脚本

// 或使用 OCI 模板
c->create(c, "oci", NULL, NULL, 0, oci_args);
```

### 6.2 启动/停止

```c
c->start(c, useinit, argv);
// useinit=0: 使用 lxc-start 启动
// useinit=1: 使用 lxc-execute (运行指定程序而非 /sbin/init)

c->stop(c);
// 发送 SIGKILL 给容器 init

c->shutdown(c, timeout);
// 发送 SIGPWR，等待 timeout 秒优雅关闭
```

### 6.3 Attach

```c
// 异步 attach（返回 PID）
pid_t pid;
c->attach(c, exec_function, exec_payload, options, &pid);
waitpid(pid, &status, 0);

// 同步 attach（等待完成）
int ret = c->attach_run_wait(c, options, "ls", "-la", NULL);
```

### 6.4 快照

```c
// 创建快照
int idx = c->snapshot(c, NULL);  // 返回快照索引

// 列出快照
struct lxc_snapshot *snaps;
int n = c->snapshot_list(c, &snaps);

// 恢复快照
c->snapshot_restore(c, "snap0", "restored_ct");
```

### 6.5 迁移 (CRIU)

```c
// Checkpoint（冻结 + dump 状态）
struct migrate_opts opts = {
    .directory = "/tmp/checkpoint",
    .stop = true,
};
c->migrate(c, MIGRATE_DUMP, &opts, sizeof(opts));

// Restore（恢复）
c->migrate(c, MIGRATE_RESTORE, &opts, sizeof(opts));
```

---

## 7. 独立函数

不依赖容器对象的全局函数：

```c
// 获取库版本
const char *lxc_get_version(void);

// 获取全局配置
const char *lxc_get_global_config_item("lxc.lxcpath");
// 返回: "/var/lib/lxc"

// 列出已定义的容器（有配置文件）
int n = list_defined_containers("/var/lib/lxc", &names, &containers);

// 列出活跃容器（正在运行，通过扫描 /proc/net/unix 中的 monitor socket）
int n = list_active_containers("/var/lib/lxc", &names, &containers);

// 列出所有容器（定义 + 活跃的并集）
int n = list_all_containers("/var/lib/lxc", &names, &containers);

// 检查配置键是否受支持
bool lxc_config_item_is_supported("lxc.net.0.type");

// 检查 API 扩展是否存在
bool lxc_has_api_extension("mount_injection");

// 获取容器状态名称列表
const char **lxc_get_wait_states(&states);
// STOPPED, STARTING, RUNNING, STOPPING, ABORTING, FREEZING, FROZEN, THAWED
```

---

## 8. 语言绑定友好性

liblxc 的设计对语言绑定非常友好：

| 特性 | 对绑定的好处 |
|------|-------------|
| 纯 C ABI（无 C++） | Go/Python/Ruby cgo/ctypes 直接调用 |
| 单一句柄 (`lxc_container *`) | 一个不透明指针 = 一个对象 |
| 函数指针 = 方法 | 自然映射到面向对象接口 |
| 简单返回类型 (bool/int/char*) | 无复杂结构需要跨语言序列化 |
| 引用计数 (get/put) | 与 GC 语言集成简单 |
| "查询长度"模式 | 避免缓冲区溢出 |
| Varargs `l` 变体 | 某些语言更容易调用 |

**已知绑定**：
- **Python**: `python3-lxc` (官方维护)
- **Go**: `go-lxc` (官方维护)
- **Ruby**: `ruby-lxc`
- **Lua**: `lua-lxc`

---

## 9. 典型使用示例

```c
#include <lxc/lxccontainer.h>

int main() {
    struct lxc_container *c;

    // 1. 创建容器对象
    c = lxc_container_new("test1", NULL);
    if (!c) return 1;

    // 2. 配置
    c->set_config_item(c, "lxc.net.0.type", "veth");
    c->set_config_item(c, "lxc.net.0.link", "lxcbr0");
    c->set_config_item(c, "lxc.idmap", "u 0 100000 65536");
    c->set_config_item(c, "lxc.idmap", "g 0 100000 65536");

    // 3. 创建 rootfs
    if (!c->create(c, "download", NULL, NULL, 0,
                   "-d", "ubuntu", "-r", "jammy", "-a", "amd64", NULL)) {
        fprintf(stderr, "Create failed\n");
        goto out;
    }

    // 4. 启动
    if (!c->start(c, 0, NULL)) {
        fprintf(stderr, "Start failed\n");
        goto out;
    }

    // 5. 等待运行
    c->wait(c, "RUNNING", -1);
    printf("Container PID: %d\n", c->init_pid(c));

    // 6. 在容器中执行命令
    c->attach_run_wait(c, NULL, "/bin/ls", "-la", "/", NULL);

    // 7. 停止
    c->shutdown(c, 30);

out:
    lxc_container_put(c);
    return 0;
}
```

编译：
```bash
gcc -o test test.c -llxc
```

---

## 10. 错误处理模式

```c
// API 函数内部模式
static bool do_lxcapi_start(struct lxc_container *c, ...) {
    if (!c)                          // 1. 参数校验
        return false;

    if (!c->is_defined(c))          // 2. 前置条件检查
        return false;

    if (container_mem_lock(c))      // 3. 加锁（如需）
        return false;

    // ... 实际操作 ...

    if (ret < 0) {                  // 4. 错误记录
        ERROR("Failed to start %s", c->name);
        container_mem_unlock(c);
        return false;
    }

    container_mem_unlock(c);        // 5. 解锁
    return true;
}
```
