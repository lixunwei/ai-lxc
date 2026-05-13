# LXC Init 与 Execute 配置深度分析

## 1. 概述

LXC 提供两种容器运行模式：
- **System Container（lxc-start）**：运行完整的 init 系统，由 `lxc.init.*` 控制
- **Application Container（lxc-execute）**：直接运行指定程序，由 `lxc.execute.cmd` 控制

两种模式的初始化命令选择、UID/GID 设置、工作目录等都通过配置项控制。

---

## 2. 配置项详解

### 2.1 lxc.init.cmd

**功能**：指定系统容器的 init 进程命令。

```
解析：set_config_init_cmd()  →  conf->init_cmd  [set_config_string_item()]
默认值："/sbin/init"（在 lxccontainer.c 中硬编码）
```

**存储**：`struct lxc_conf` 中的 `char *init_cmd`（`conf.h`）

**使用流程**：

```
lxccontainer.c: do_lxcapi_start(c, useinit, argv)
  ├── if argv == NULL:
  │     ├── useinit && conf->execute_cmd → 使用 execute_cmd
  │     ├── conf->init_cmd → 使用 init_cmd
  │     └── 回退到 "/sbin/init"
  ├── lxc_string_split_quoted(cmd) → 将命令字符串拆分为 argv[]
  └── 传递 argv 到 start 引擎
```

**示例**：
```ini
lxc.init.cmd = /bin/systemd --system
```

### 2.2 lxc.init.uid / lxc.init.gid

**功能**：指定 init 进程运行的用户 ID 和组 ID。

```
解析：set_config_init_uid() → lxc_safe_uint(value, &conf->init_uid)
解析：set_config_init_gid() → lxc_safe_uint(value, &conf->init_gid)
默认值：0（root）
```

**存储**：
- `uid_t init_uid` — init 进程的 UID
- `gid_t init_gid` — init 进程的 GID

**运行时应用**（`start.c`）：
```
容器子进程设置：
  new_uid = handler->conf->init_uid
  new_gid = handler->conf->init_gid
  → setgid(new_gid)
  → setuid(new_uid)
```

### 2.3 lxc.init.groups

**功能**：指定 init 进程的附加组列表。

```
解析：set_config_init_groups()
  → 逗号分隔解析每个 GID
  → 存入 conf->init_groups.list[] 和 conf->init_groups.size
```

**存储**：
```c
typedef struct {
    gid_t *list;
    size_t size;
} lxc_groups_t;
```

**运行时应用**（`start.c`）：
```
if init_groups.size > 0:
    lxc_setgroups(init_groups.list, init_groups.size)
else:
    lxc_drop_groups()   // 清空附加组
```

### 2.4 lxc.init.cwd

**功能**：指定 init 进程的工作目录。

```
解析：set_config_init_cwd() → set_config_path_item(&conf->init_cwd, value)
默认值：NULL（不改变工作目录）
```

**运行时应用**（`start.c`）：
```
if conf->init_cwd:
    stat(init_cwd) → 检查目录存在
    如不存在，创建目录（mkdir -p 语义）
    chdir(init_cwd)
```

### 2.5 lxc.execute.cmd

**功能**：指定 lxc-execute 模式下的默认命令。

```
解析：set_config_execute_cmd() → set_config_string_item(&conf->execute_cmd, value)
```

**存储**：`char *execute_cmd`，`bool is_execute`（`conf.h:414, 524`）

---

## 3. lxc-start 与 lxc-execute 的区别

### 3.1 lxc-start 路径（系统容器）

```
lxc-start
  └── c->start(c, 0, argv)     // useinit = 0
        └── do_lxcapi_start()
              ├── 选择 init 命令（见 2.1）
              └── __lxc_start()
                    └── lxc_spawn()
                          └── clone() → 子进程
                                ├── 设置 UID/GID/CWD/Groups
                                └── execvp(init_cmd, argv)
```

### 3.2 lxc-execute 路径（应用容器）

```
lxc-execute
  └── c->start(c, 1, argv)     // useinit = 1
        └── do_lxcapi_start()
              ├── 标记 conf->is_execute = true
              ├── 选择命令：execute_cmd 优先于 init_cmd
              └── lxc_execute()          [execute.c]
                    ├── handler->ops = execute_start_ops
                    └── __lxc_start()
                          └── 子进程执行 lxc-init 包装器
                                └── lxc-init 再 exec 目标程序
```

### 3.3 关键差异

| 特性 | lxc-start | lxc-execute |
|------|-----------|-------------|
| 命令来源 | `lxc.init.cmd` | `lxc.execute.cmd` 或 CLI 参数 |
| 默认命令 | `/sbin/init` | 必须指定 |
| init 包装 | 直接 exec | 通过 `lxc-init` 包装 |
| `is_execute` | false | true |
| TTY 分配 | 完整 TTY 设置 | 简化的 TTY（`conf.c:962-968`）|
| 用途 | 完整 OS 容器 | 单进程应用容器 |

---

## 4. lxc-init 包装器

`src/lxc/cmd/lxc_init.c` 是一个轻量级的 init 进程替代品，用于 lxc-execute 模式。

### 4.1 工作流程

```
lxc_init main()
  ├── 解析参数（--name, --logpriority, -- command...）
  └── lxc_container_init()              [initutils.c]
        ├── fork()
        │     └── 子进程：
        │           ├── setsid()           // 创建新会话
        │           ├── ioctl(TIOCSCTTY)   // 获取控制终端
        │           └── execvp(argv[0], argv)
        └── 父进程（作为 PID 1）：
              ├── 安装信号处理器
              ├── 转发信号给子进程
              └── waitpid() 等待子进程退出
```

### 4.2 lxc-init 作为 PID 1 的职责

在容器的 PID 命名空间中，lxc-init 充当 PID 1 的角色：
1. **信号转发**：将收到的信号转发给实际的应用进程
2. **僵尸进程收割**：作为 init 进程负责 wait 所有孤儿进程
3. **退出码传递**：将应用进程的退出码传递给容器外部

---

## 5. CLI 参数与配置交互

### 5.1 lxc-execute 的 CLI 覆盖

```c
// src/lxc/tools/lxc_execute.c
if (my_args.uid)
    c->set_config_item(c, "lxc.init.uid", buf);  // CLI --uid 覆盖配置
if (my_args.gid)
    c->set_config_item(c, "lxc.init.gid", buf);  // CLI --gid 覆盖配置
```

### 5.2 lxc-start 的配置文件路径

lxc-start 按以下顺序加载配置：
1. 默认系统配置：`/etc/lxc/default.conf`
2. 容器特定配置：`<lxcpath>/<name>/config`
3. CLI 指定的额外配置：`-f <file>` 和 `-s key=value`

---

## 6. 配置到执行的完整数据流

```
配置文件                     struct lxc_conf              运行时
────────                     ───────────────              ──────
lxc.init.cmd = /bin/bash  →  conf->init_cmd             → execvp("/bin/bash", ...)
lxc.init.uid = 1000       →  conf->init_uid             → setuid(1000)
lxc.init.gid = 1000       →  conf->init_gid             → setgid(1000)
lxc.init.groups = 10,27   →  conf->init_groups          → setgroups([10, 27])
lxc.init.cwd = /home/user →  conf->init_cwd             → chdir("/home/user")
lxc.execute.cmd = /app    →  conf->execute_cmd          → lxc-init → execvp("/app")
```

---

## 7. 相关信号配置

Init 进程的生命周期管理还涉及以下信号配置：

| 配置项 | 默认值 | 用途 |
|--------|--------|------|
| `lxc.signal.halt` | SIGPWR | 发送给 init 以请求关机 |
| `lxc.signal.reboot` | SIGINT | 发送给 init 以请求重启 |
| `lxc.signal.stop` | SIGKILL | 强制停止容器时使用 |

**解析**（`confile.c:1865-1915`）：支持信号名（如 `SIGTERM`）和数字（如 `15`）两种格式。

**使用**（`lxccontainer.c`）：
```c
// 优雅关机
kill(handler->pid, conf->haltsignal ?: SIGPWR);

// 重启
kill(handler->pid, conf->rebootsignal ?: SIGINT);

// 强制停止
kill(handler->pid, conf->stopsignal ?: SIGKILL);
```
