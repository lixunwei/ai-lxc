# Hook 钩子机制详解

## 1. 概述

LXC 提供 10 种钩子（Hook）类型，允许用户在容器生命周期的关键节点执行自定义脚本。钩子以 fork+exec 方式运行，接收标准化的环境变量和参数，脚本返回非零值将中止当前操作。

**核心源文件**：
- `src/lxc/conf.h` — `enum lxchooks` 枚举定义
- `src/lxc/conf.c` — `run_lxc_hooks()` 执行引擎
- `src/lxc/start.c` — 生命周期中的钩子调用点、环境变量设置
- `src/lxc/lxccontainer.c` — clone/destroy 钩子
- `src/lxc/confile.c` — 配置解析

---

## 2. 钩子类型枚举

```c
// conf.h:338-351
enum lxchooks {
    LXCHOOK_PRESTART,     // "pre-start"
    LXCHOOK_PREMOUNT,     // "pre-mount"
    LXCHOOK_MOUNT,        // "mount"
    LXCHOOK_AUTODEV,      // "autodev"
    LXCHOOK_START,        // "start"
    LXCHOOK_STOP,         // "stop"
    LXCHOOK_POSTSTOP,     // "post-stop"
    LXCHOOK_CLONE,        // "clone"
    LXCHOOK_DESTROY,      // "destroy"
    LXCHOOK_START_HOST,   // "start-host"
    NUM_LXC_HOOKS,        // = 10
};

// conf.c:101-112
char *lxchook_names[NUM_LXC_HOOKS] = {
    "pre-start", "pre-mount", "mount", "autodev", "start",
    "stop", "post-stop", "clone", "destroy", "start-host"
};
```

存储方式：

```c
// conf.h
struct lxc_conf {
    ...
    struct list_head hooks[NUM_LXC_HOOKS];  // 每种类型一个链表
    unsigned int hooks_version;              // 0=legacy, 1=new
    ...
};
```

---

## 3. 钩子生命周期时序

```
lxc-start
  |
  +-- [1] "pre-start"        (start.c:1035, 宿主侧, clone 之前)
  |
  +-- clone(CLONE_NEWPID|...)
  |
  +-- [parent] 保存 namespace fd -> hook_argv[]
  |     |
  |     +-- [2] "start-host"  (start.c:2089, 宿主侧, 子进程已创建)
  |
  +-- [child] do_start():
        |
        +-- lxc_setup():
        |     |
        |     +-- [3] "pre-mount"  (conf.c:3395, 容器侧, mount 之前)
        |     |
        |     +-- mount rootfs, 执行所有挂载
        |     |
        |     +-- [4] "mount"      (conf.c:3843, 容器侧, mount 完成后)
        |     |
        |     +-- [5] "autodev"    (conf.c:3851, 容器侧, /dev 填充后)
        |     |
        |     +-- pivot_root, 设置设备/网络/capabilities...
        |
        +-- [6] "start"           (start.c:1495, 容器侧, exec 之前)
        |
        +-- execvp(init)

容器停止:
  |
  +-- [7] "stop"              (start.c:1133, 宿主侧, 容器退出后)
  |                            (带 LXC_TARGET=stop|reboot)
  |
  +-- [8] "post-stop"         (start.c:1176, 宿主侧, 清理阶段)

容器克隆:
  +-- [9] "clone"             (lxccontainer.c:3659, 宿主侧)

容器销毁:
  +-- [10] "destroy"          (lxccontainer.c:2953, 宿主侧)
```

---

## 4. 钩子详细说明

| 钩子 | 执行上下文 | 时机 | 用途 |
|------|-----------|------|------|
| `pre-start` | 宿主 | clone 之前 | 准备外部资源（网络、存储）|
| `pre-mount` | 容器 | rootfs mount 之前 | 修改挂载环境 |
| `mount` | 容器 | 所有挂载完成后 | 额外挂载或修改文件 |
| `autodev` | 容器 | /dev 自动填充后 | 创建额外设备节点 |
| `start` | 容器 | exec init 之前 | 最后的容器内定制 |
| `start-host` | 宿主 | 子进程创建后 | 宿主侧的后期配置 |
| `stop` | 宿主 | 容器退出后 | 清理外部资源 |
| `post-stop` | 宿主 | 完全停止后 | 最终清理 |
| `clone` | 宿主 | 克隆完成后 | 修改克隆体配置 |
| `destroy` | 宿主 | 销毁之前 | 清理关联资源 |

---

## 5. 执行引擎

### 5.1 run_lxc_hooks()

```c
// conf.c:3924-3948
int run_lxc_hooks(const char *name, char *hookname,
                  struct lxc_conf *conf, char *argv[]) {
    int which;

    // 查找钩子类型索引
    for (which = 0; which < NUM_LXC_HOOKS; which++) {
        if (strequal(hookname, lxchook_names[which]))
            break;
    }
    if (which >= NUM_LXC_HOOKS)
        return -1;

    // 遍历该类型的所有钩子脚本
    list_for_each_entry(entry, &conf->hooks[which], head) {
        char *hook = entry->val;
        ret = run_script_argv(name, conf->hooks_version,
                              "lxc", hook, hookname, argv);
        if (ret < 0)
            return -1;  // 任何一个失败则中止
    }
    return 0;
}
```

### 5.2 run_script_argv()

```c
// utils.c:576+
int run_script_argv(const char *name, unsigned int hook_version,
                    const char *section, const char *script,
                    const char *hookname, char *argv[]) {
    pid_t pid = fork();
    if (pid == 0) {
        // 子进程: 执行钩子脚本
        // hook_version 0: execl(script, script, name, section, hookname, ...)
        // hook_version 1: execvp(script, argv_with_ns_fds)
        execvp(script, ...);
        _exit(EXIT_FAILURE);
    }
    // 父进程: 等待脚本完成
    waitpid(pid, &status, 0);
    return WEXITSTATUS(status) == 0 ? 0 : -1;
}
```

---

## 6. 环境变量

### 6.1 通用环境变量

在 `start.c:980-1030` 中设置，所有钩子可用：

| 变量 | 来源 | 说明 |
|------|------|------|
| `LXC_NAME` | 容器名 | 始终设置 |
| `LXC_CONFIG_FILE` | `conf->rcfile` | 配置文件路径 |
| `LXC_ROOTFS_MOUNT` | `conf->rootfs.mount` | rootfs 挂载点 |
| `LXC_ROOTFS_PATH` | `conf->rootfs.path` | rootfs 源路径 |
| `LXC_CONSOLE` | `conf->console.path` | 控制台设备路径 |
| `LXC_CONSOLE_LOGPATH` | `conf->console.log_path` | 控制台日志路径 |
| `LXC_CGNS_AWARE` | `"1"` | 支持 cgroup namespace |
| `LXC_LOG_LEVEL` | 日志级别 | 当前日志级别 |
| `LXC_HOOK_VERSION` | `"0"` 或 `"1"` | 钩子协议版本 |

### 6.2 启动阶段特有

| 变量 | 时机 | 说明 |
|------|------|------|
| `LXC_PID` | start-host 之前 | 容器 init 的 PID（宿主视角）|

### 6.3 停止阶段特有

| 变量 | 时机 | 说明 |
|------|------|------|
| `LXC_TARGET` | stop/post-stop | `"stop"` 或 `"reboot"` |

### 6.4 克隆阶段特有

| 变量 | 时机 | 说明 |
|------|------|------|
| `LXC_SRC_NAME` | clone | 源容器名称 |

### 6.5 网络相关

| 变量 | 说明 |
|------|------|
| `LXC_NET_TYPE` | 网络类型 (veth/macvlan/...) |
| `LXC_NET_PARENT` | 宿主侧接口名 |
| `LXC_NET_PEER` | 对端接口名 |

---

## 7. 钩子参数 (argv)

### 7.1 Hook Version 0 (传统模式)

```
argv[0] = 脚本路径
argv[1] = 容器名
argv[2] = "lxc" (section)
argv[3] = 钩子名 ("pre-start", "mount", ...)
argv[4...] = namespace fd 路径 (仅 stop 钩子)
```

### 7.2 Hook Version 1 (新模式)

```
argv[0] = 脚本路径
argv[1...N] = namespace fd 路径
              格式: "<ns>:/proc/<monitor_pid>/fd/<fd_num>"
              例: "mnt:/proc/1234/fd/7"
                  "net:/proc/1234/fd/8"
                  "pid:/proc/1234/fd/9"
```

**Namespace fd 传递机制**：

```c
// start.c:149-176
static int lxc_try_preserve_namespace(...) {
    // 打开 /proc/<init_pid>/ns/<ns> 获取 fd
    fd = lxc_preserve_ns(handler->pid, ns);

    // 构建路径字符串: "ns:/proc/<monitor_pid>/fd/<fd>"
    strnprintf(handler->nsfd_paths[idx],
               "%s:/proc/%d/fd/%d",
               ns_info[idx].proc_name, handler->monitor_pid, fd);

    // 添加到 hook argv
    handler->hook_argv[handler->hook_argc] = handler->nsfd_paths[idx];
    handler->hook_argc++;
}
```

这允许钩子脚本通过 `nsenter --net=/proc/1234/fd/8` 等方式进入容器的特定命名空间。

---

## 8. 配置语法

### 8.1 基本配置

```
# 单个钩子
lxc.hook.pre-start = /usr/share/lxc/hooks/setup-network.sh
lxc.hook.mount = /usr/share/lxc/hooks/extra-mounts.sh
lxc.hook.stop = /usr/share/lxc/hooks/cleanup.sh

# 同一类型多个钩子（按配置顺序执行）
lxc.hook.start = /path/to/hook1.sh
lxc.hook.start = /path/to/hook2.sh
```

### 8.2 钩子版本

```
# 传统模式 (默认): 信息通过 argv 传递
lxc.hook.version = 0

# 新模式: namespace fd 作为 argv 传递
lxc.hook.version = 1
```

### 8.3 配置解析

```c
// confile.c:1366+  set_config_hooks()
// 解析 "lxc.hook.<type>" → 添加到 conf->hooks[type] 链表
static int set_config_hooks(const char *key, const char *value,
                            struct lxc_conf *lxc_conf, void *data) {
    // 从 key 提取后缀 (pre-start, mount, ...)
    // 匹配 lxchook_names[] 找到索引
    // add_hook(lxc_conf, which, value)
}
```

---

## 9. 典型使用场景

### 9.1 自动网络配置

```bash
#!/bin/bash
# lxc.hook.start-host = /etc/lxc/hooks/setup-nat.sh
# 在容器启动后配置 NAT 规则
iptables -t nat -A POSTROUTING -s 10.0.3.0/24 -o eth0 -j MASQUERADE
```

### 9.2 挂载额外文件系统

```bash
#!/bin/bash
# lxc.hook.mount = /etc/lxc/hooks/mount-shared.sh
# 在容器挂载完成后添加共享目录
mount --bind /data/shared $LXC_ROOTFS_MOUNT/shared
```

### 9.3 自动创建设备

```bash
#!/bin/bash
# lxc.hook.autodev = /etc/lxc/hooks/create-devices.sh
# 在 /dev 填充后创建额外设备
mknod $LXC_ROOTFS_MOUNT/dev/custom_device c 100 0
chmod 666 $LXC_ROOTFS_MOUNT/dev/custom_device
```

### 9.4 克隆后修改主机名

```bash
#!/bin/bash
# lxc.hook.clone = /etc/lxc/hooks/fix-hostname.sh
# 克隆后更新新容器的 hostname
echo "$LXC_NAME" > $LXC_ROOTFS_MOUNT/etc/hostname
sed -i "s/$LXC_SRC_NAME/$LXC_NAME/g" $LXC_ROOTFS_MOUNT/etc/hosts
```

### 9.5 停止时清理资源

```bash
#!/bin/bash
# lxc.hook.stop = /etc/lxc/hooks/cleanup.sh
# 容器停止后清理网络
if [ "$LXC_TARGET" = "stop" ]; then
    ip link del veth-${LXC_NAME} 2>/dev/null
    iptables -t nat -D POSTROUTING -s 10.0.3.0/24 -j MASQUERADE
fi
```

---

## 10. 错误处理与安全

### 10.1 错误处理

- 钩子脚本返回非零 → `run_lxc_hooks()` 返回 `-1`
- 调用方根据返回值决定是否中止：
  - `pre-start` 失败 → 容器启动中止
  - `mount` 失败 → lxc_setup() 失败 → 启动中止
  - `stop`/`post-stop` 失败 → 仅记录警告，不影响停止流程

### 10.2 安全考虑

- **执行上下文隔离**：宿主侧钩子有完整权限，容器侧钩子受 namespace 限制
- **脚本路径验证**：路径必须绝对路径，由 root 拥有
- **环境清理**：仅传递 LXC_* 前缀变量，不泄露宿主环境
- **Namespace fd 安全**：通过 `/proc/<pid>/fd/<N>` 路径传递，受 procfs 权限控制
