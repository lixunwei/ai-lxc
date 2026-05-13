# LXC 命令深度分析：其他 CLI 命令

## 1. lxc-start

**文件**：`src/lxc/tools/lxc_start.c:36-260`

### 1.1 参数

| 选项 | 含义 |
|------|------|
| `-d` / `--daemon` | 守护进程模式（默认） |
| `-F` / `--foreground` | 前台运行 |
| `-f` / `--rcfile` | 指定配置文件 |
| `-s` / `--define` | 定义配置项（`key=value`） |
| `-p` / `--pidfile` | PID 文件路径 |
| `--share-net/ipc/uts` | 共享命名空间 |

### 1.2 执行流程

```
lxc_start_main()                                 [lxc_start.c:153-260]
  ├── 解析参数
  ├── 加载配置文件（rcfile 或默认 <lxcpath>/<name>/config）
  ├── 应用 -s 定义的配置覆盖
  ├── 设置 c->daemonize
  └── c->start(c, 0, args)                       [lxc_start.c:256-260]
```

### 1.3 API 路径

```
do_lxcapi_start(c, useinit=0, argv)              [lxccontainer.c:846-1112]
  ├── 检查是否有进行中的创建操作
  ├── 初始化 handler
  ├── 解析 init 命令
  │     ├── argv 优先
  │     ├── conf->init_cmd 次之
  │     └── 回退 "/sbin/init"
  ├── 守护进程模式：双 fork 解除终端关联
  │     └── 写入 pidfile
  └── lxc_start(c->name, argv, handler, ...)      [lxccontainer.c:1082-1087]
        └── → __lxc_start() → lxc_spawn() → clone() → ...
```

### 1.4 守护进程化

双 fork 模式（`-d`）：
```
父进程 → fork → 中间进程 → fork → 守护进程
  退出         setsid()            ├── 关闭 stdin/stdout/stderr
                退出                ├── 写 pidfile
                                   └── 运行容器
```

---

## 2. lxc-stop

**文件**：`src/lxc/tools/lxc_stop.c:24-210`

### 2.1 停止模式

| 模式 | 选项 | 行为 |
|------|------|------|
| shutdown | 默认 | 发送 halt 信号，等待优雅关闭 |
| kill | `--kill` | 发送 SIGKILL 强制终止 |
| reboot | `--reboot` | 发送 reboot 信号 |

### 2.2 执行流程

```
lxc_stop_main()                                  [lxc_stop.c:140-210]
  │
  ├── kill 模式：
  │     c->stop(c)                                [lxc_stop.c:178-182]
  │     → lxc_cmd_stop(name)                      // 通过 IPC 命令通道
  │
  ├── reboot 模式：
  │     c->reboot2(c, timeout)                    [lxc_stop.c:184-192]
  │
  └── shutdown 模式：
        c->shutdown(c, timeout)                   [lxc_stop.c:194-200]
        → 发送 haltsignal（默认 SIGPWR）
        → 等待容器进入 STOPPED 状态
        if 超时 && !nokill:
            c->stop(c)                            // 回退到强制停止
```

### 2.3 shutdown API

```
do_lxcapi_shutdown(c, timeout)                    [lxccontainer.c:2038-2134]
  ├── 选择信号：conf->haltsignal ?: SIGPWR
  ├── 创建状态监听客户端（等待 STOPPED）
  ├── 使用 pidfd 或 kill() 发送信号
  └── 等待状态变化（带超时）
```

---

## 3. lxc-destroy

**文件**：`src/lxc/tools/lxc_destroy.c:25-259`

### 3.1 执行流程

```
do_destroy(c)                                    [lxc_destroy.c:71-107]
  ├── 检查 clone/snapshot 依赖标记
  ├── if 容器运行中 && --force:
  │     c->stop(c)                                [lxc_destroy.c:89-97]
  ├── if lxc.ephemeral == 0:
  │     c->destroy(c)                             [lxc_destroy.c:99-107]
  └── --snapshots: 遍历并销毁依赖的快照           [lxc_destroy.c:117-188]
```

### 3.2 destroy API

```
do_lxcapi_destroy(c)                              [lxccontainer.c:3055-3075]
  └── container_destroy(c)
        ├── 检查是否有关联快照（有则拒绝）
        ├── 删除存储后端数据
        │     bdev->ops->destroy(bdev)
        ├── 删除配置文件
        ├── 删除容器目录
        └── 运行 destroy 钩子
```

---

## 4. lxc-info

**文件**：`src/lxc/tools/lxc_info.c:31-419`

### 4.1 可查询信息

| 信息类型 | 获取方式 |
|----------|----------|
| 状态 | `c->state(c)` → 直接查询 |
| PID | `c->init_pid(c)` → IPC 命令 |
| IP 地址 | `c->get_ips()` → fork + 进入 netns 枚举 |
| CPU/内存统计 | cgroup 读取 |
| 配置项 | `c->get_running_config_item()` → IPC 命令 |

### 4.2 IP 获取机制

```
do_lxcapi_get_ips(c, interface, family, scope)    [lxccontainer.c:2314-2528]
  ├── fork()
  │     └── 子进程：
  │           ├── setns(容器 netns)
  │           ├── 枚举所有网络接口
  │           ├── 获取 IP 地址（getifaddrs 或 netlink）
  │           └── 通过管道发送结果
  └── 父进程：读取管道，收集 IP 列表
```

### 4.3 运行时配置查询

```
c->get_running_config_item(key)
  → lxc_cmd_get_config_item(name, key, lxcpath)
     → 通过 IPC Unix 域套接字发送 GET_CONFIG_ITEM 命令
     → 运行中的容器 monitor 返回配置值
```

---

## 5. lxc-copy

**文件**：`src/lxc/tools/lxc_copy.c:79-546`

### 5.1 操作模式

| 模式 | 选项 | 行为 |
|------|------|------|
| 克隆 | 默认 | 完整复制容器 |
| 快照 | `-s` | 创建 COW 快照（btrfs/zfs/overlay） |
| 重命名 | `-R` | 重命名容器 |
| 临时 | `-e` | 创建临时克隆，停止后自动销毁 |

### 5.2 克隆流程

```
do_clone_task(c, args)                            [lxc_copy.c:526-546]
  ├── 重命名：c->rename(c, newname)
  └── 克隆：
        c->clone(c, newname, lxcpath, flags,
                 bdevtype, bdevdata, newsize, hookargs)
                                                  [lxc_copy.c:385-401]
        → do_lxcapi_clone()                       [lxccontainer.c]
```

### 5.3 临时容器

```
临时模式 (-e):                                    [lxc_copy.c:403-513]
  ├── 创建快照克隆
  ├── 设置 lxc.ephemeral = 1
  ├── 配置额外挂载（tmpfs 等）
  └── clone->start(clone, ...)                    // 启动临时容器
```

---

## 6. lxc-snapshot

**文件**：`src/lxc/tools/lxc_snapshot.c:33-300`

| 操作 | API 调用 |
|------|----------|
| 创建快照 | `c->snapshot(c, commentfile)` |
| 列出快照 | `c->snapshot_list(c, &snapshots)` |
| 恢复快照 | `c->snapshot_restore(c, snapname, newname)` |
| 销毁快照 | `c->snapshot_destroy(c, snapname)` |
| 销毁全部 | `c->snapshot_destroy_all(c)` |

---

## 7. lxc-freeze / lxc-unfreeze

**文件**：`src/lxc/tools/lxc_freeze.c:39-96`、`src/lxc/tools/lxc_unfreeze.c:39-96`

### 7.1 冻结机制

```
c->freeze(c)                                     [lxccontainer.c:514-540]
  ├── 检查状态不是 FROZEN
  └── cgroup_freeze(cgroup_ops)
        → 写 "1" 到 cgroup.freeze（v2）
        → 或写 "FROZEN" 到 freezer.state（v1）

c->unfreeze(c)                                   [lxccontainer.c:542-559]
  ├── 检查状态是 FROZEN 或 FREEZING
  └── cgroup_unfreeze(cgroup_ops)
        → 写 "0" 到 cgroup.freeze（v2）
        → 或写 "THAWED" 到 freezer.state（v1）
```

冻结会暂停容器内所有进程的执行，但不释放其内存或资源。

---

## 8. lxc-execute

**文件**：`src/lxc/tools/lxc_execute.c:31-242`

### 8.1 与 lxc-start 的区别

| 特性 | lxc-start | lxc-execute |
|------|-----------|-------------|
| 用途 | 系统容器 | 应用容器 |
| Init | 运行 /sbin/init | 运行 lxc-init 包装器 |
| useinit | 0 | 1 |
| 命令来源 | lxc.init.cmd | CLI 参数或 lxc.execute.cmd |
| TTY | 完整设置 | 简化 |

### 8.2 执行流程

```
lxc_execute_main()                                [lxc_execute.c:150-242]
  ├── 解析命令和选项
  ├── 加载 lxc.execute.cmd（如果无 CLI 命令）     [lxc_execute.c:108-129]
  ├── 应用 --uid/--gid 到 lxc.init.uid/gid       [lxc_execute.c:194-215]
  └── c->start(c, 1, my_args.argv)                [lxc_execute.c:218-227]
        useinit = 1 → 走 lxc_execute() 路径
```

---

## 9. lxc-autostart

**文件**：`src/lxc/tools/lxc_autostart.c:23-515`

### 9.1 自动启动逻辑

```
lxc_autostart_main()                              [lxc_autostart.c:273-515]
  ├── 扫描所有容器
  ├── 过滤：lxc.start.auto == 1                   [lxc_autostart.c:384-408]
  ├── 过滤：lxc.group 匹配（-g 选项）
  ├── 排序：按 lxc.start.order 降序               [lxc_autostart.c:273-290]
  │
  └── 对每个容器执行动作：
        ├── start: c->start(c, 0, NULL)           [lxc_autostart.c:469-475]
        │          sleep(start.delay)              // 启动间隔
        ├── shutdown: c->shutdown(c, timeout)      [lxc_autostart.c:421-427]
        ├── kill: c->stop(c)                       [lxc_autostart.c:437-440]
        ├── reboot: c->reboot2(c, timeout)         [lxc_autostart.c:452-457]
        └── list: 打印容器名和组                   [lxc_autostart.c:415-449]
```

### 9.2 排序规则

- `lxc.start.order` 数值越大越先启动
- 相同 order 的容器顺序不确定
- 每个容器启动后等待 `lxc.start.delay` 秒

---

## 10. 命令总览

```
容器生命周期：
  lxc-create  ─→  lxc-start  ─→  lxc-stop  ─→  lxc-destroy
                      │
                      ├── lxc-freeze / lxc-unfreeze（暂停/恢复）
                      ├── lxc-attach（进入容器）
                      ├── lxc-info（查询状态）
                      ├── lxc-copy（克隆/快照）
                      └── lxc-snapshot（管理快照）

特殊模式：
  lxc-execute     应用容器模式
  lxc-autostart   批量自动启动

API 调用关系：
  CLI 工具 → lxccontainer.c API → 核心实现（start.c/conf.c/attach.c/...）
                                    ↕ IPC
                              commands.c 命令通道
```
