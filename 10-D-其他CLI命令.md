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
  +-- parse args
  +-- load config file (rcfile or default <lxcpath>/<name>/config)
  +-- apply config overrides from -s
  +-- set c->daemonize
  `-- c->start(c, 0, args)                       [lxc_start.c:256-260]
```

### 1.3 API 路径

```
do_lxcapi_start(c, useinit=0, argv)              [lxccontainer.c:846-1112]
  +-- check whether a create operation is still in progress
  +-- init handler
  +-- parse init command
  |   +-- argv first
  |   +-- conf->init_cmd next
  |   `-- fallback to "/sbin/init"
  +-- daemon mode: double fork to detach from terminal
  |   `-- write pidfile
  `-- lxc_start(c->name, argv, handler, ...)      [lxccontainer.c:1082-1087]
      `-- -> __lxc_start() -> lxc_spawn() -> clone() -> ...
```

### 1.4 守护进程化

双 fork 模式（`-d`）：
```
parent process -> fork -> intermediate process -> fork -> daemon process
   exit           setsid()                         +-- close stdin/stdout/stderr
                  exit                             +-- write pidfile
                                                   `-- run container
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
  |
  +-- force-stop mode:
  |   c->stop(c)                                  [lxc_stop.c:178-182]
  |   -> lxc_cmd_stop(name)                       // via IPC command channel
  |
  +-- reboot mode:
  |   c->reboot2(c, timeout)                      [lxc_stop.c:184-192]
  |
  `-- shutdown mode:
      c->shutdown(c, timeout)                     [lxc_stop.c:194-200]
      -> send haltsignal (default SIGPWR)
      -> wait for container to enter STOPPED state
      if timeout && !nokill:
          c->stop(c)                              // fallback to forced stop
```

### 2.3 shutdown API

```
do_lxcapi_shutdown(c, timeout)                    [lxccontainer.c:2038-2134]
  +-- choose signal: conf->haltsignal ?: SIGPWR
  +-- create state listener client (wait for STOPPED)
  +-- use pidfd or signal syscall to send the signal
  `-- wait for state change (with timeout)
```

---

## 3. lxc-destroy

**文件**：`src/lxc/tools/lxc_destroy.c:25-259`

### 3.1 执行流程

```
do_destroy(c)                                    [lxc_destroy.c:71-107]
  +-- check clone/snapshot dependency markers
  +-- if container is running && --force:
  |   c->stop(c)                                  [lxc_destroy.c:89-97]
  +-- if lxc.ephemeral == 0:
  |   c->destroy(c)                               [lxc_destroy.c:99-107]
  `-- --snapshots: walk and destroy dependent snapshots [lxc_destroy.c:117-188]
```

### 3.2 destroy API

```
do_lxcapi_destroy(c)                             [lxccontainer.c:3055-3075]
  `-- container_destroy(c)
      +-- check for linked snapshots (reject if present)
      +-- delete storage backend data
      |   bdev->ops->destroy(bdev)
      +-- delete config file
      +-- delete container directory
      `-- run destroy hook
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
  +-- fork()
  |   `-- child:
  |       +-- setns(container netns)
  |       +-- enumerate all network interfaces
  |       +-- get IP addresses (getifaddrs or netlink)
  |       `-- send results through pipe
  `-- parent: read pipe and collect IP list
```

### 4.3 运行时配置查询

```
c->get_running_config_item(key)
  -> lxc_cmd_get_config_item(name, key, lxcpath)
     -> send GET_CONFIG_ITEM command over the IPC Unix domain socket
     -> running container monitor returns the config value
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
  +-- rename: c->rename(c, newname)
  `-- clone:
      c->clone(c, newname, lxcpath, flags,
               bdevtype, bdevdata, newsize, hookargs)
                                                 [lxc_copy.c:385-401]
      -> do_lxcapi_clone()                       [lxccontainer.c]
```

### 5.3 临时容器

```
ephemeral mode (-e):                             [lxc_copy.c:403-513]
  +-- create snapshot clone
  +-- set lxc.ephemeral = 1
  +-- configure extra mounts (tmpfs, etc.)
  `-- clone->start(clone, ...)                   // start the ephemeral container
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
  +-- check state is not FROZEN
  `-- cgroup_freeze(cgroup_ops)
      -> write "1" to cgroup.freeze (v2)
      -> or write "FROZEN" to freezer.state (v1)

c->unfreeze(c)                                   [lxccontainer.c:542-559]
  +-- check state is FROZEN or FREEZING
  `-- cgroup_unfreeze(cgroup_ops)
      -> write "0" to cgroup.freeze (v2)
      -> or write "THAWED" to freezer.state (v1)
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
lxc_execute_main()                               [lxc_execute.c:150-242]
  +-- parse command and options
  +-- load lxc.execute.cmd (if no CLI command)    [lxc_execute.c:108-129]
  +-- apply --uid/--gid to lxc.init.uid/gid       [lxc_execute.c:194-215]
  `-- c->start(c, 1, my_args.argv)                [lxc_execute.c:218-227]
      useinit = 1 -> take the lxc_execute() path
```

---

## 9. lxc-autostart

**文件**：`src/lxc/tools/lxc_autostart.c:23-515`

### 9.1 自动启动逻辑

```
lxc_autostart_main()                             [lxc_autostart.c:273-515]
  +-- scan all containers
  +-- filter: lxc.start.auto == 1                 [lxc_autostart.c:384-408]
  +-- filter: lxc.group matches (-g option)
  +-- sort: by lxc.start.order descending         [lxc_autostart.c:273-290]
  |
  `-- for each container, perform an action:
      +-- start: c->start(c, 0, NULL)             [lxc_autostart.c:469-475]
      |          sleep(start.delay)               // start interval
      +-- shutdown: c->shutdown(c, timeout)       [lxc_autostart.c:421-427]
      +-- force-stop: c->stop(c)                  [lxc_autostart.c:437-440]
      +-- reboot: c->reboot2(c, timeout)          [lxc_autostart.c:452-457]
      `-- list: print container name and groups   [lxc_autostart.c:415-449]
```

### 9.2 排序规则

- `lxc.start.order` 数值越大越先启动
- 相同 order 的容器顺序不确定
- 每个容器启动后等待 `lxc.start.delay` 秒

---

## 10. 命令总览

```
container lifecycle:
  lxc-create  ->  lxc-start  ->  lxc-stop  ->  lxc-destroy
                      |
                      +-- lxc-freeze / lxc-unfreeze (pause/resume)
                      +-- lxc-attach (enter container)
                      +-- lxc-info (query status)
                      +-- lxc-copy (clone/snapshot)
                      `-- lxc-snapshot (manage snapshots)

special modes:
  lxc-execute     app container mode
  lxc-autostart   batch auto-start

API call chain:
  CLI tools -> lxccontainer.c API -> core implementation (start.c/conf.c/attach.c/...)
                                   <-> IPC
                              commands.c command channel
```
