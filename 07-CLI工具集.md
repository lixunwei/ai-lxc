# 07 - CLI 工具集

## 1. 工具框架（tools/arguments.c/h）

### 1.1 参数解析机制

所有 CLI 工具共享统一的参数解析框架（`arguments.c:170-274`）：

```
lxc_arguments_parse()
  ├── build_shortopts()        ← 从 longopts 自动生成 shortopts
  │
  ├── getopt_long() 循环
  │   ├── 公共参数处理：
  │   │   -n (容器名)
  │   │   -o (日志文件)
  │   │   -l (日志级别)
  │   │   -q (安静模式)
  │   │   -P (lxcpath)
  │   │   --rcfile (配置文件)
  │   │   --help / --version / --usage
  │   │
  │   └── args->parser()       ← 各工具自定义选项解析回调
  │
  ├── 默认容器名处理：
  │   └── 未给 -n 时，取第一个剩余参数作为容器名
  │       （lxc-autostart / lxc-unshare 除外）
  │
  └── args->checker()          ← 解析后一致性检查回调
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
用法: lxc-start -n <name> [options]

关键选项:
  -d            后台运行（daemon 模式）
  -F            前台运行
  -f <rcfile>   指定配置文件
  -c <console>  指定控制台输出设备
  -p <pidfile>  写入 PID 文件
  -s key=val    定义配置项
  --share-*     共享 namespace

流程: 解析参数 → 加载配置 → c->start()
```

源码位于 `tools/lxc_start.c:36-260`。

### 2.2 lxc-stop — 停止容器

```
用法: lxc-stop -n <name> [options]

关键选项:
  -k / --kill     强制杀死
  --nokill        不发送 kill 信号
  --nolock        不加锁
  --nowait        不等待
  -t <timeout>    超时时间
  -r / --reboot   重启

流程: 解析参数 → c->stop() 或 c->shutdown() 或 c->reboot()
```

源码位于 `tools/lxc_stop.c:24-209`。

### 2.3 lxc-create — 创建容器

```
用法: lxc-create -n <name> -t <template> [options]

关键选项:
  -t <template>      模板名（如 download, ubuntu）
  -B <backingstore>  存储后端（dir/lvm/loop/rbd/zfs/btrfs）
  --dir <dir>        自定义 rootfs 目录
  --lvname/--vgname  LVM 相关
  --fstype/--fssize  文件系统类型和大小

流程: 参数校验 → c->create(template, bdevtype, ...)
```

源码位于 `tools/lxc_create.c:26-260`。

### 2.4 lxc-destroy — 销毁容器

```
用法: lxc-destroy -n <name> [options]

关键选项:
  -f / --force       强制（先停止再删除）
  -s / --snapshots   同时删除快照和依赖克隆

流程: 解析参数 → [force: c->stop()] → c->destroy()
```

源码位于 `tools/lxc_destroy.c:25-180`。

### 2.5 lxc-copy — 克隆/复制容器

```
用法: lxc-copy -n <name> -N <newname> [options]

关键选项:
  -e / --ephemeral   临时容器
  -s / --snapshot    快照模式
  -R / --rename      重命名
  -B <backingstore>  目标存储后端
  -m bind=<src:dst>  额外 bind mount
  -m overlay=<src:dst>  额外 overlay mount
  --tmpfs            使用 tmpfs
  --keepname/keepmac/keepdata  保留原有属性

流程: 参数解析 → c->clone() 或 c->rename()
```

源码位于 `tools/lxc_copy.c:53-233`。

---

## 3. 运行时交互工具

### 3.1 lxc-attach — 附着到容器

```
用法: lxc-attach -n <name> [-- command]

关键选项:
  -e / --elevated-privileges  提升权限
  -a / --arch                指定架构
  -s / --namespaces          选择 namespace（如 NETWORK|IPC）
  --keep-env / --clear-env   环境变量策略
  -v <var=val>               设置环境变量
  -u <uid> / -g <gid>        指定用户/组
  --context <selinux>        SELinux 上下文

流程: 解析参数 → c->attach() → 两次 fork → setns → exec
```

源码位于 `tools/lxc_attach.c:65-433`。

### 3.2 lxc-console — 连接控制台

```
用法: lxc-console -n <name> [options]

关键选项:
  -t <ttynum>   指定 tty 编号
  -e <prefix>   escape 前缀（默认 Ctrl-a）

流程: 解析参数 → c->console() → mainloop 处理 I/O
```

源码位于 `tools/lxc_console.c:30-143`。

### 3.3 lxc-execute — 执行模式启动

```
用法: lxc-execute -n <name> -- command

与 lxc-start 的区别:
  - 不启动 init 系统
  - 直接执行指定命令
  - 使用 lxc.execute.cmd 配置
  - 支持 uid/gid 指定

流程: 解析参数 → c->start() 传入 execute 标志
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
用法: lxc-ls [options]

关键选项:
  -1            每行一个
  -f / --fancy  表格格式
  -F <columns>  自定义列（NAME,STATE,PID,RAM,SWAP,AUTOSTART,GROUPS,INTERFACE,IPV4,IPV6）
  --filter <re> 正则过滤
  --active/--frozen/--running/--stopped  状态过滤
  -g <groups>   按组过滤
  --nesting     嵌套显示

流程: 枚举容器 → 查询信息 → 格式化输出
```

源码位于 `tools/lxc_ls.c:148-260+`。

### 4.3 lxc-monitor — 状态监控

```
用法: lxc-monitor -n <name> [options]

关键选项:
  -Q / --quit   通知 lxc-monitord 退出

流程: 连接 monitord → 循环接收状态变更事件
      如果 monitord 未运行则自动启动
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
用法: lxc-freeze -n <name>
      lxc-unfreeze -n <name>

流程: c->freeze() 或 c->unfreeze()
      底层操作 cgroup freezer.state 或 cgroup.freeze
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
用法: lxc-checkpoint -n <name> [options]

关键选项:
  -D <dir>                 检查点目录
  -r / --restore           恢复模式
  -s / --stop              检查点后停止
  -p / --pre-dump          增量预转储
  --predump-dir <dir>      预转储目录
  --action-script <script> 动作脚本

流程: 参数准备 → 调用 CRIU dump/restore
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
根据 argv[0] 的名称分发：
  lxc-start  → lxc_start_main()
  lxc-stop   → lxc_stop_main()
  lxc-attach → lxc_attach_main()
  ...

也支持子命令形式：
  lxc start → lxc_start_main()
  lxc stop  → lxc_stop_main()
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
liblxc_sources          ← 主静态库的全部源码
liblxc_ext_sources      ← tools/tests 用的子集源码

条件编译：
  want_apparmor  → 追加 apparmor.c
  want_seccomp   → 追加 seccomp.c
  want_selinux   → 追加 selinux.c
```

### 8.2 工具构建（tools/meson.build）

```
tools_commands_dynamic_link:
  ├── 各工具单独编译
  └── 动态链接 liblxc

lxc-monitor:
  └── 静态链接 (link_whole: [liblxc_static])

want_tools_multicall:
  ├── 所有 lxc_*.c 编成单个 "lxc" 二进制
  └── 安装 lxc-xxx → lxc 的符号链接
```

### 8.3 检查脚本

- `cmd/lxc-checkconfig.in` — 检查系统内核配置是否满足 LXC 运行要求
- `cmd/lxc-update-config.in` — 升级旧版容器配置到新格式
