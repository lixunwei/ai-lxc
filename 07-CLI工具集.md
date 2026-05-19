# 07 - CLI 工具集

## 1. 工具框架（tools/arguments.c/h）

### 1.1 参数解析机制

所有 CLI 工具共享统一的参数解析框架（`arguments.c:170-274`）：

```
lxc_arguments_parse()
  +- build_shortopts()        <- auto-build shortopts from longopts
  |
  +- getopt_long() loop
  |  +- common option handling:
  |  |  -n (container name)
  |  |  -o (log file)
  |  |  -l (log level)
  |  |  -q (quiet mode)
  |  |  -P (lxcpath)
  |  |  --rcfile (config file)
  |  |  --help / --version / --usage
  |  |
  |  `- args->parser()       <- per-tool custom option parser
  |
  +- default container name handling:
  |  `- if -n is absent, use first remaining arg as name
  |     (except lxc-autostart / lxc-unshare)
  |
  `- args->checker()          <- post-parse consistency check
```

### 1.2 统一数据结构

`struct lxc_arguments`（`arguments.h:22-135`）是一个大型结构体，包含所有工具共用和专用的字段：

| 字段组 | 说明 |
|--------|------|
| 公共 | name, lxcpath, log, quiet, rcfile |
| 启动相关 | daemonize, console, pidfile, share_ns |
| 停止相关 | timeout, kill, nokill, nolock |
| 复制相关 | newname, newpath, snapshot, keepname |
| 列表相关 | ls_fancy, ls_filter, ls_groups |
| 其他 | freeze, checkpoint, unshare 等 |

### 1.3 多路径支持

`-P/--lxcpath` 可多次出现，默认补全全局 `lxc.lxcpath`（`arguments.c:148-167, 243-248`）。

---

## 2. 容器管理工具

### 2.1 lxc-start — 启动容器

```
Usage: lxc-start -n <name> [options]

Key options:
  -d            run in background (daemon mode)
  -F            run in foreground
  -f <rcfile>   set config file
  -c <console>  set console output device
  -p <pidfile>  write PID file
  -s key=val    define config item
  --share-*     share namespace

Flow: parse args -> load config -> c->start()
```

源码位于 `tools/lxc_start.c:36-260`。

### 2.2 lxc-stop — 停止容器

```
Usage: lxc-stop -n <name> [options]

Key options:
  -k / --kill     force kill
  --nokill        skip kill signal
  --nolock        skip locking
  --nowait        do not wait
  -t <timeout>    timeout
  -r / --reboot   reboot

Flow: parse args -> c->stop() or c->shutdown() or c->reboot()
```

源码位于 `tools/lxc_stop.c:24-209`。

### 2.3 lxc-create — 创建容器

```
Usage: lxc-create -n <name> -t <template> [options]

Key options:
  -t <template>      template name (e.g. download, ubuntu)
  -B <backingstore>  storage backend (dir/lvm/loop/rbd/zfs/btrfs)
  --dir <dir>        custom rootfs dir
  --lvname/--vgname  LVM related
  --fstype/--fssize  filesystem type and size

Flow: validate args -> c->create(template, bdevtype, ...)
```

源码位于 `tools/lxc_create.c:26-260`。

### 2.4 lxc-destroy — 销毁容器

```
Usage: lxc-destroy -n <name> [options]

Key options:
  -f / --force       force (stop before delete)
  -s / --snapshots   delete snapshots and dependent clones too

Flow: parse args -> [force: c->stop()] -> c->destroy()
```

源码位于 `tools/lxc_destroy.c:25-180`。

### 2.5 lxc-copy — 克隆/复制容器

```
Usage: lxc-copy -n <name> -N <newname> [options]

Key options:
  -e / --ephemeral   ephemeral container
  -s / --snapshot    snapshot mode
  -R / --rename      rename
  -B <backingstore>  target storage backend
  -m bind=<src:dst>  extra bind mount
  -m overlay=<src:dst>  extra overlay mount
  --tmpfs            use tmpfs
  --keepname/keepmac/keepdata  keep original attrs

Flow: parse args -> c->clone() or c->rename()
```

源码位于 `tools/lxc_copy.c:53-233`。

---

## 3. 运行时交互工具

### 3.1 lxc-attach — 附着到容器

```
Usage: lxc-attach -n <name> [-- command]

Key options:
  -e / --elevated-privileges  raise privileges
  -a / --arch                 set arch
  -s / --namespaces           choose namespace (e.g. NETWORK|IPC)
  --keep-env / --clear-env    env strategy
  -v <var=val>                set env var
  -u <uid> / -g <gid>         set user/group
  --context <selinux>         SELinux context

Flow: parse args -> c->attach() -> double fork -> setns -> exec
```

源码位于 `tools/lxc_attach.c:65-433`。

### 3.2 lxc-console — 连接控制台

```
Usage: lxc-console -n <name> [options]

Key options:
  -t <ttynum>   select tty number
  -e <prefix>   escape prefix (default Ctrl-a)

Flow: parse args -> c->console() -> mainloop handles I/O
```

源码位于 `tools/lxc_console.c:30-143`。

### 3.3 lxc-execute — 执行模式启动

```
Usage: lxc-execute -n <name> -- command

Diff vs lxc-start:
  - no init system
  - run target command directly
  - uses lxc.execute.cmd config
  - supports uid/gid override

Flow: parse args -> pass execute flag into c->start()
```

源码位于 `tools/lxc_execute.c:31-180`。

---

## 4. 监控与信息工具

### 4.1 lxc-info — 查询容器信息

```
用法: lxc-info -n <name> [options]

关键选项:
  -p    显示 PID
  -i    显示 IP 地址
  -s    显示状态
  -S    显示资源统计
  -c <key>  显示指定配置项

支持的信息: Name, State, PID, IPs, CPU use, BlkIO use,
            Memory use, KMem use, Link, TX/RX bytes
```

源码位于 `tools/lxc_info.c:31-394`。

### 4.2 lxc-ls — 列出容器

```
Usage: lxc-ls [options]

Key options:
  -1            one per line
  -f / --fancy  table format
  -F <columns>  custom columns (NAME,STATE,PID,RAM,SWAP,AUTOSTART,GROUPS,INTERFACE,IPV4,IPV6)
  --filter <re> regex filter
  --active/--frozen/--running/--stopped  state filter
  -g <groups>   filter by group
  --nesting     nested display

Flow: enumerate containers -> query info -> format output
```

源码位于 `tools/lxc_ls.c:148-260+`。

### 4.3 lxc-monitor — 状态监控

```
Usage: lxc-monitor -n <name> [options]

Key options:
  -Q / --quit   ask lxc-monitord to exit

Flow: connect to monitord -> receive state-change events in a loop
      auto-start if monitord is not running
```

源码位于 `tools/lxc_monitor.c:34-260`。

### 4.4 lxc-top — 实时监控

```
用法: lxc-top [options]

关键选项:
  -d <delay>    刷新间隔（秒）
  -b            批处理模式
  -s <sortby>   排序字段
  -r            反向排序

功能: 类似 top 的界面，显示各容器的 CPU/内存/IO 使用
```

源码位于 `tools/lxc_top.c:88-180+`。

---

## 5. 容器操作工具

### 5.1 lxc-freeze / lxc-unfreeze — 冻结/解冻

```
Usage: lxc-freeze -n <name>
       lxc-unfreeze -n <name>

Flow: c->freeze() or c->unfreeze()
      backend uses cgroup freezer.state or cgroup.freeze
```

### 5.2 lxc-snapshot — 快照管理

```
用法: lxc-snapshot -n <name> [options]

关键选项:
  (无选项)         创建快照
  -L / --list      列出快照
  -r <snapname>    恢复快照到新容器
  -d <snapname>    删除快照
  -c <comment>     添加注释
  -C               显示注释
```

源码位于 `tools/lxc_snapshot.c:23-220`。

### 5.3 lxc-checkpoint — 检查点/恢复

```
Usage: lxc-checkpoint -n <name> [options]

Key options:
  -D <dir>                 checkpoint dir
  -r / --restore           restore mode
  -s / --stop              stop after checkpoint
  -p / --pre-dump          incremental pre-dump
  --predump-dir <dir>      pre-dump dir
  --action-script <script> action script

Flow: prepare args -> call CRIU dump/restore
```

源码位于 `tools/lxc_checkpoint.c:37-220`。

### 5.4 lxc-cgroup — Cgroup 操作

```
用法: lxc-cgroup -n <name> <state-object> [value]

无 value: 读取 cgroup 参数
有 value: 写入 cgroup 参数

示例: lxc-cgroup -n myct memory.limit_in_bytes 256M
```

源码位于 `tools/lxc_cgroup.c:21-135`。

### 5.5 lxc-device — 设备管理

```
用法: lxc-device -n <name> add|del <device> [name]

功能: 向运行中容器添加/删除设备节点或网卡
      自动判断目标是网络接口还是普通设备节点
```

源码位于 `tools/lxc_device.c:23-174`。

### 5.6 lxc-autostart — 批量自动启动

```
用法: lxc-autostart [options]

关键选项:
  -k / --kill        杀死
  -s / --shutdown     优雅关闭
  -r / --reboot      重启
  -g <groups>         按组选择
  -t <timeout>        超时

功能: 批量对标记了 autostart 的容器执行操作
```

源码位于 `tools/lxc_autostart.c:55-220`。

---

## 6. 其他工具

### 6.1 lxc-unshare — 命名空间隔离执行

```
用法: lxc-unshare -s <namespaces> -- command

关键选项:
  -s <ns>       要 unshare 的 namespace
  -u <user>     目标用户名
  -H <hostname> 主机名
  -i <ifname>   迁移的网卡
  -d            后台运行
  -M            重挂载默认文件系统

功能: 在新 namespace 中运行命令（无需完整容器）
```

源码位于 `tools/lxc_unshare.c:54-200`。

### 6.2 lxc-config — 查询全局配置

```
用法: lxc-config -l
      lxc-config <key>

功能: 列出或查询 LXC 全局配置项
```

源码位于 `tools/lxc_config.c:10-70`。

### 6.3 lxc（multicall 二进制）

`lxc_multicall.c` 实现了 BusyBox 风格的多命令合体二进制（`tools/lxc_multicall.c:31-108`）：

```
Dispatch by argv[0] name:
  lxc-start  -> lxc_start_main()
  lxc-stop   -> lxc_stop_main()
  lxc-attach -> lxc_attach_main()
  ...

Also supports subcommand form:
  lxc start -> lxc_start_main()
  lxc stop  -> lxc_stop_main()
```

---

## 7. 系统辅助程序（cmd/）

### 7.1 lxc-init（cmd/lxc_init.c）

容器内的 init 进程。在容器启动时作为 PID 1 运行，负责：
- 信号转发
- 僵尸进程回收
- 关机处理

### 7.2 lxc-monitord（cmd/lxc_monitord.c）

后台监控守护进程：
- 接收容器状态变更事件
- 向订阅客户端（如 `lxc-monitor`、daemon 容器）分发通知
- 支持 Unix socket 通信

### 7.3 lxc-user-nic（cmd/lxc_user_nic.c）

**setuid 辅助程序**，为非特权容器准备网络：
- 创建 veth pair
- 配置桥接
- 管理网络配额（`/etc/lxc/lxc-usernet`）

### 7.4 lxc-usernsexec（cmd/lxc_usernsexec.c）

**setuid 辅助程序**，在用户 namespace 中执行命令。

---

## 8. 构建系统

### 8.1 主库构建（src/lxc/meson.build）

```
liblxc_sources          <- all source files of main static library
liblxc_ext_sources      <- subset used by tools/tests

Conditional build:
  want_apparmor  -> add apparmor.c
  want_seccomp   -> add seccomp.c
  want_selinux   -> add selinux.c
```

### 8.2 工具构建（tools/meson.build）

```
tools_commands_dynamic_link:
  +- build each tool separately
  `- dynamically link liblxc

lxc-monitor:
  `- statically link (link_whole: [liblxc_static])

want_tools_multicall:
  +- build all lxc_*.c into one "lxc" binary
  `- install symlink lxc-xxx -> lxc
```

### 8.3 检查脚本

- `cmd/lxc-checkconfig.in` — 检查系统内核配置是否满足 LXC 运行要求
- `cmd/lxc-update-config.in` — 升级旧版容器配置到新格式
