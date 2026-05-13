# LXC 核心生命周期深度分析：容器 API 对象

## 1. 概述

`struct lxc_container` 是 LXC 对外暴露的核心 API 对象，定义了容器的全部操作接口。
它使用 **函数指针表** 模式，类似于面向对象语言中的虚方法表。

**定义文件**：`src/lxc/lxccontainer.h:50-760`
**实现文件**：`src/lxc/lxccontainer.c`（约 5400 行）

---

## 2. struct lxc_container 数据成员

**位置**：`lxccontainer.h:50-96`

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `char *` | 容器名 |
| `configfile` | `char *` | 配置文件路径 |
| `pidfile` | `char *` | PID 文件路径 |
| `slock` | `struct lxc_lock *` | 磁盘锁（保护文件操作） |
| `privlock` | `struct lxc_lock *` | 内存锁（保护内存数据） |
| `numthreads` | `int` | 引用计数（线程安全） |
| `lxc_conf` | `struct lxc_conf *` | 容器配置结构体 |
| `error_string` | `char *` | 最近的错误信息 |
| `error_num` | `int` | 最近的错误码 |
| `daemonize` | `bool` | 是否守护进程化 |
| `config_path` | `char *` | 配置文件搜索路径 |

---

## 3. 函数指针 API 表

**位置**：`lxccontainer.h:117-760`

### 3.1 生命周期管理

| 方法 | 签名 | 说明 |
|------|------|------|
| `is_defined` | `bool (*)(c)` | 容器是否已定义 |
| `start` | `bool (*)(c, useinit, argv)` | 启动容器 |
| `stop` | `bool (*)(c)` | 强制停止 |
| `shutdown` | `bool (*)(c, timeout)` | 优雅关闭 |
| `reboot` | `bool (*)(c)` | 重启 |
| `reboot2` | `bool (*)(c, timeout)` | 带超时重启 |
| `wait` | `bool (*)(c, state, timeout)` | 等待指定状态 |
| `freeze` | `bool (*)(c)` | 冻结 |
| `unfreeze` | `bool (*)(c)` | 解冻 |
| `state` | `const char *(*)(c)` | 获取当前状态 |
| `is_running` | `bool (*)(c)` | 是否运行中 |
| `init_pid` | `pid_t (*)(c)` | 获取 init PID |
| `init_pidfd` | `int (*)(c)` | 获取 init pidfd |

### 3.2 容器创建与销毁

| 方法 | 签名 | 说明 |
|------|------|------|
| `create` | `bool (*)(c, t, bdevtype, specs, flags, argv)` | 创建容器 |
| `createl` | `bool (*)(c, t, bdevtype, specs, flags, ...)` | 创建（可变参数） |
| `destroy` | `bool (*)(c)` | 销毁容器 |
| `destroy_with_snapshots` | `bool (*)(c)` | 销毁容器及快照 |
| `rename` | `bool (*)(c, newname)` | 重命名 |

### 3.3 克隆与快照

| 方法 | 签名 | 说明 |
|------|------|------|
| `clone` | `struct lxc_container *(*)(c, newname, ...)` | 克隆容器 |
| `snapshot` | `int (*)(c, commentfile)` | 创建快照 |
| `snapshot_list` | `int (*)(c, &snapshots)` | 列出快照 |
| `snapshot_restore` | `bool (*)(c, snapname, newname)` | 恢复快照 |
| `snapshot_destroy` | `bool (*)(c, snapname)` | 销毁快照 |
| `snapshot_destroy_all` | `bool (*)(c)` | 销毁所有快照 |

### 3.4 配置管理

| 方法 | 签名 | 说明 |
|------|------|------|
| `load_config` | `bool (*)(c, alt_file)` | 加载配置 |
| `save_config` | `bool (*)(c, alt_file)` | 保存配置 |
| `set_config_item` | `bool (*)(c, key, value)` | 设置配置项 |
| `get_config_item` | `int (*)(c, key, retv, inlen)` | 获取配置项 |
| `get_running_config_item` | `char *(*)(c, key)` | 获取运行时配置 |
| `get_keys` | `int (*)(c, key, retv, inlen)` | 列出配置键 |
| `clear_config` | `void (*)(c)` | 清除全部配置 |
| `clear_config_item` | `bool (*)(c, key)` | 清除指定配置 |
| `set_config_path` | `bool (*)(c, path)` | 设置配置搜索路径 |
| `get_config_path` | `const char *(*)(c)` | 获取配置搜索路径 |

### 3.5 运行时交互

| 方法 | 签名 | 说明 |
|------|------|------|
| `attach` | `int (*)(c, exec_fn, payload, options, pid)` | 附着执行 |
| `attach_run_wait` | `int (*)(c, options, program, argv)` | 附着运行等待 |
| `console` | `int (*)(c, ttynum, stdinfd, stdoutfd, ...)` | 连接控制台 |
| `console_getfd` | `int (*)(c, ttynum, ptxfd)` | 获取控制台 fd |
| `get_ips` | `char **(*)(c, iface, family, scope)` | 获取 IP 地址 |
| `get_interfaces` | `char **(*)(c)` | 获取网络接口 |
| `get_cgroup_item` | `int (*)(c, subsys, retv, inlen)` | 读取 cgroup 值 |
| `set_cgroup_item` | `bool (*)(c, subsys, value)` | 写入 cgroup 值 |

### 3.6 辅助方法

| 方法 | 签名 | 说明 |
|------|------|------|
| `want_daemonize` | `bool (*)(c, state)` | 设置守护进程模式 |
| `want_close_all_fds` | `bool (*)(c, state)` | 是否关闭所有 fd |
| `may_control` | `bool (*)(c)` | 检查控制权限 |
| `set_timeout` | `int (*)(c, timeout)` | 设置 IPC 超时 |

---

## 4. 容器对象创建：lxc_container_new()

**位置**：`lxccontainer.c:5241-5385`

```
lxc_container_new(name, configpath)
  │
  ├─1─ 分配并初始化结构体                         [5247-5270]
  │      calloc(1, sizeof(struct lxc_container))
  │      设置 name, config_path
  │      numthreads = 1
  │
  ├─2─ 初始化锁                                   [5272-5280]
  │      slock = lxc_newlock(lxcpath, name)        // 磁盘锁
  │      privlock = lxc_newlock(NULL, NULL)         // 内存锁
  │
  ├─3─ 计算配置文件路径                            [5282-5290]
  │      configfile = <configpath>/<name>/config
  │
  ├─4─ 检查进行中的创建                           [5299-5317]
  │      ongoing_create(c)
  │      → 检查 partial 标记文件
  │
  ├─5─ 自动加载配置                               [5292-5297]
  │      if (configfile 存在):
  │          lxc_config_read(configfile, c->lxc_conf)
  │
  ├─6─ 设置默认值                                 [5319-5320]
  │      daemonize = true
  │      pidfile = NULL
  │
  └─7─ 初始化函数指针                             [5322-5385]
         c->is_defined = lxcapi_is_defined
         c->start = lxcapi_start
         c->stop = lxcapi_stop
         c->create = lxcapi_create
         c->attach = lxcapi_attach
         ... (所有 API 方法)
```

---

## 5. 线程安全与锁机制

**位置**：`lxccontainer.c:218-238`

### 5.1 锁层级

```
层级 1: thread_mutex          进程级互斥锁
          → 保护进程内全局数据

层级 2: container_mem_lock    内存锁 (privlock)
          → 保护 struct lxc_container 内存数据

层级 3: container_disk_lock   磁盘锁 (slock + privlock)
          → 保护容器磁盘数据（配置文件等）
          → 同时获取 privlock（内存锁）
```

### 5.2 使用规则

```c
// 只需要读/写内存数据
container_mem_lock(c);
// ... 操作 c->lxc_conf 等
container_mem_unlock(c);

// 需要读/写磁盘文件（自动也获取内存锁）
container_disk_lock(c);
// ... 读写配置文件
container_disk_unlock(c);
```

### 5.3 引用计数

```
lxc_container_get(c)                              [lxccontainer.c:299-310]
  ├── privlock 加锁
  ├── numthreads++
  └── privlock 解锁

lxc_container_put(c)                              [lxccontainer.c:312-343]
  ├── privlock 加锁
  ├── numthreads--
  ├── if numthreads < 1:
  │     释放所有资源
  │     free(c)
  └── privlock 解锁
```

---

## 6. 关键 API 实现

### 6.1 do_lxcapi_start()

**位置**：`lxccontainer.c:846-1112`

```
do_lxcapi_start(c, useinit, argv)
  │
  ├── 检查配置完整性                              [857-874]
  │     验证无进行中的创建
  │
  ├── 初始化 handler                              [883-893]
  │     lxc_init_handler(c->name, c->lxc_conf, lxcpath)
  │
  ├── 解析 init 命令                              [895-912]
  │     argv → conf->init_cmd → conf->execute_cmd → "/sbin/init"
  │
  ├── 守护进程化（双 fork）                       [918-1006]
  │     fork → 中间进程 → fork → 守护进程
  │     ├── 第一个 fork：父进程等待启动确认
  │     ├── 第二个 fork：中间进程退出
  │     ├── 守护进程：setsid(), chdir("/")
  │     └── 重定向 stdio 到 /dev/null
  │
  ├── 写入 pidfile                                [1010-1042]
  │     （守护进程化之后，PID 已是最终值）
  │
  ├── 选择启动模式                                [1082-1087]
  │     if useinit:
  │         lxc_execute(name, argv, handler)       // lxc-execute 模式
  │     else:
  │         lxc_start(name, argv, handler)         // lxc-start 模式
  │
  └── 清理                                        [1095-1111]
        删除 pidfile
        lxc_container_put(c)
```

### 6.2 do_lxcapi_shutdown()

**位置**：`lxccontainer.c:2038-2134`

```
do_lxcapi_shutdown(c, timeout)
  ├── 选择信号：conf->haltsignal ?: SIGPWR
  ├── 添加状态监听客户端（等待 STOPPED）
  ├── 通过 pidfd 或 kill() 发送信号
  └── 等待状态变化（带超时）
        timeout == 0: 非阻塞
        timeout == -1: 无限等待
        timeout > 0: 等待 N 秒
```

### 6.3 do_lxcapi_clone()

**位置**：`lxccontainer.c:3728-3929`

```
do_lxcapi_clone(c, newname, lxcpath, flags, bdevtype, bdevdata, newsize, hookargs)
  ├── 锁定源容器
  ├── 创建新容器对象
  ├── 复制配置
  │     调整主机名（MAC 地址随机化）
  │     调整钩子路径
  │     调整 fstab 路径
  ├── fork 子进程复制 rootfs
  │     bdev->ops->clone(src_bdev, dst_bdev)
  ├── 保存新配置
  └── 运行 clone 钩子
```

### 6.4 配置操作

```
do_lxcapi_set_config_item(c, key, value)          [lxccontainer.c:3133-3149]
  ├── container_mem_lock(c)
  ├── lxc_set_config_item_locked(conf, key, value)
  │     → lxc_get_config(key)->set(key, value, conf)
  └── container_mem_unlock(c)

do_lxcapi_get_config_item(c, key, retv, inlen)    [lxccontainer.c:2533-2552]
  ├── container_mem_lock(c)
  ├── lxc_get_config(key)->get(key, retv, inlen, conf)
  └── container_mem_unlock(c)

do_lxcapi_load_config(c, alt_file)                [lxccontainer.c:659-700]
  ├── 主配置文件：container_disk_lock()            // 需要磁盘锁
  ├── 备选文件：container_mem_lock()               // 只需内存锁
  └── load_config_locked(c, fname)
        → lxc_config_read(fname, c->lxc_conf)

do_lxcapi_save_config(c, alt_file)                [lxccontainer.c:2600-2663]
  ├── 加载默认配置（如果需要）
  ├── container_lock()
  ├── 打开目标文件
  └── write_config(fd, c->lxc_conf)
```

---

## 7. API 调用模式

所有 API 实现遵循统一模式：

```c
// 内部实现函数
static bool do_lxcapi_xxx(struct lxc_container *c, ...)
{
    // 参数验证
    // 加锁
    // 实际逻辑
    // 解锁
    // 返回结果
}

// 公开包装函数（处理错误、锁定、容器名绑定）
WRAP_API(bool, lxcapi_xxx)
// 或
WRAP_API_1(bool, lxcapi_xxx, type1)
// 或
WRAP_API_2(bool, lxcapi_xxx, type1, type2)
```

`WRAP_API*` 宏自动生成包装函数，提供：
- 参数类型检查
- 容器对象有效性验证
- 统一错误处理

---

## 8. IPC 与运行时查询

运行中容器的信息通过 IPC 命令通道获取：

```
c->get_running_config_item(key)
  → lxc_cmd_get_config_item(name, key, lxcpath)
      → Unix 域套接字 → 发送 GET_CONFIG_ITEM 命令
      → monitor 进程处理 → 返回配置值

c->init_pid(c)
  → lxc_cmd_get_init_pid(name, lxcpath)
      → Unix 域套接字 → 发送 GET_INIT_PID 命令

c->get_ips(iface, family, scope)
  → fork() + setns(netns) + 枚举接口 + 管道传回
```

---

## 9. 容器对象生命周期

```
创建：lxc_container_new("mycontainer", NULL)
  │
  ├── 引用：lxc_container_get(c)   // numthreads++
  │
  ├── 使用：c->start() / c->stop() / c->attach() / ...
  │
  ├── 释放：lxc_container_put(c)   // numthreads--
  │
  └── 销毁：numthreads < 1 时自动 free
```
