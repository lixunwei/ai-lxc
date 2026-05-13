# LXC 安全子系统深度分析（一）：LSM 抽象与实现

## 1. 概述

LXC 通过 **LSM（Linux Security Module）** 抽象层统一管理 AppArmor 和 SELinux 两种强制访问控制系统。该抽象层采用编译时/运行时自动检测的方式选择后端，优先级为：**AppArmor > SELinux > nop**。

核心文件：

| 文件 | 职责 |
|------|------|
| `src/lxc/lsm/lsm.h` | 操作表定义 |
| `src/lxc/lsm/lsm.c` | 后端选择逻辑 |
| `src/lxc/lsm/apparmor.c` | AppArmor 后端（~1300 行） |
| `src/lxc/lsm/selinux.c` | SELinux 后端（~200 行） |
| `src/lxc/lsm/nop.c` | 空操作后端 |

---

## 2. 抽象层设计

### 2.1 操作表：`struct lsm_ops`

`lsm.h:16-37`：

```c
struct lsm_ops {
    /* 检查 LSM 是否启用 */
    bool (*enabled)(void);

    /* 获取/设置进程安全标签 */
    char *(*process_label_get)(pid_t pid);
    int   (*process_label_set)(const char *label, struct lxc_conf *conf,
                               bool on_exec, bool use_default);

    /* 通过 fd 获取/设置标签（跨 namespace） */
    int   (*process_label_fd_get)(pid_t pid, bool on_exec);
    char *(*process_label_get_at)(int label_fd);
    int   (*process_label_set_at)(int label_fd, const char *label, bool on_exec);

    /* keyring 标签设置（SELinux） */
    int   (*keyring_label_set)(const char *label);

    /* 生命周期钩子 */
    int   (*prepare)(struct lsm_ops *ops, struct lxc_conf *conf,
                     const char *lxcpath);
    void  (*cleanup)(struct lsm_ops *ops, struct lxc_conf *conf,
                     const char *lxcpath);
};
```

### 2.2 后端选择

`lsm.c:18-39`，`lsm_init_static()`：

```c
struct lsm_ops *lsm_init_static(void) {
    struct lsm_ops *ops;

    // 优先级 1：AppArmor
    ops = lsm_apparmor_ops_init();
    if (ops && ops->enabled()) {
        INFO("Using AppArmor LSM backend");
        return ops;
    }

    // 优先级 2：SELinux
    ops = lsm_selinux_ops_init();
    if (ops && ops->enabled()) {
        INFO("Using SELinux LSM backend");
        return ops;
    }

    // 兜底：nop（无操作）
    ops = lsm_nop_ops_init();
    INFO("Using nop LSM backend");
    return ops;
}
```

没有动态注册机制，选择逻辑是硬编码的优先级链。

---

## 3. AppArmor 后端

### 3.1 启用检测

`apparmor.c:393-421`：

```c
apparmor_enabled() {
    // 1. 检查 /sys/module/apparmor/parameters/enabled 是否为 "Y"
    // 2. 检查 /sys/kernel/security/apparmor 是否已挂载
    // 3. 检查 AppArmor 特性（mount, stacking, namespaces 等）
}
```

### 3.2 Profile 模板系统

AppArmor 后端使用宏模板动态生成容器安全策略。模板定义在 `apparmor.c` 上半部分：

| 模板 | 用途 |
|------|------|
| `AA_PROFILE_BASE` | 基础规则：文件、网络、capability 权限 |
| `AA_PROFILE_NESTING` | 嵌套容器规则：允许内层容器的挂载操作 |
| `AA_PROFILE_STACKING` | LSM stacking 规则：多 LSM 共存时的权限 |
| `AA_PROFILE_UNPRIVILEGED` | 非特权容器附加限制 |

### 3.3 Profile 生成与加载

`apparmor.c:779+`：

```
get_apparmor_profile_content(conf)
  ├── 拼接基础模板 AA_PROFILE_BASE
  ├── if (allow_nesting):
  │     └── 追加 AA_PROFILE_NESTING 模板
  ├── if (aa_can_stack):
  │     └── 追加 AA_PROFILE_STACKING 模板
  ├── if (unprivileged):
  │     └── 追加 AA_PROFILE_UNPRIVILEGED 模板
  ├── 追加用户自定义原始规则（lxc.apparmor.raw）
  └── 返回完整 profile 文本
```

加载流程（`load_apparmor_profile()`，`apparmor.c:980`）：

```
load_apparmor_profile()
  ├── 通过管道将 profile 文本传给 apparmor_parser
  ├── apparmor_parser --add --replace --quiet
  └── 标记 conf->lsm_aa_profile_created = true
```

### 3.4 Namespace 支持

`apparmor.c:896`，`make_apparmor_namespace()`：

当 AppArmor 支持命名空间时，LXC 为每个容器创建独立的 AppArmor namespace：

```c
// 命名空间路径：lxc-<container_name>-<random>
// 完整 profile 名：:lxc-name-XXXX://lxc-name-XXXX_<-container-profile>
```

这提供了：
- 容器间的 profile 隔离
- 容器内无法查看或修改其他容器的 profile
- 嵌套容器可拥有自己的 AppArmor namespace

### 3.5 Prepare 流程

`apparmor_prepare()`，`apparmor.c:1099`：

```
apparmor_prepare(ops, conf, lxcpath)
  1. 确定最终 profile 名称
     ├── 用户指定了 lxc.apparmor.profile → 使用它
     └── 未指定 → 使用默认 "generated" 模式
  2. 检查 allow_incomplete 配置
     // 当 AppArmor 功能不完整时是否允许降级运行
  3. 检查 apparmor_parser 版本
  4. 生成并加载 profile
  5. 决定 transition 还是 stacking 行为
     ├── 支持 stacking → 使用 "stack" 模式（多 LSM 共存）
     └── 不支持 → 使用传统 "transition" 模式
```

### 3.6 标签应用

`apparmor_process_label_set()`，`apparmor.c:1242`：

```c
// 通过写入 /proc/self/attr/exec 或 /proc/self/attr/current
// 在容器进程 exec 之前切换到目标 profile

// 使用 changehat / changeprofile / stack 等机制
// 取决于 stacking 支持情况
```

### 3.7 后端注册

`apparmor.c:1274-1294`：

```c
static struct lsm_ops apparmor_ops = {
    .enabled              = apparmor_enabled,
    .process_label_get    = apparmor_process_label_get,
    .process_label_set    = apparmor_process_label_set,
    .process_label_fd_get = apparmor_process_label_fd_get,
    .process_label_get_at = apparmor_process_label_get_at,
    .process_label_set_at = apparmor_process_label_set_at,
    .prepare              = apparmor_prepare,
    .cleanup              = apparmor_cleanup,
    // stacking 状态字段
    .aa_can_stack         = ...,
    .aa_is_stacked        = ...,
};
```

---

## 4. SELinux 后端

### 4.1 核心实现

SELinux 后端相对简单（~200 行），主要包装 `libselinux` API：

**获取上下文**（`selinux.c:34-68`）：
```c
selinux_process_label_get(pid) {
    getpidcon_raw(pid, &ctx);
    return ctx;
}
```

**设置上下文**（`selinux.c:82-105`）：
```c
selinux_process_label_set(label, conf, on_exec, use_default) {
    if (on_exec)
        setexeccon_raw(label);    // exec 时切换
    else
        setcon_raw(label);        // 立即切换
}
```

**通过 fd 操作**（`selinux.c:128-160`）：
```c
selinux_process_label_fd_get(pid, on_exec) {
    // 打开 /proc/<pid>/attr/current 或 /proc/<pid>/attr/exec
    return fd;
}

selinux_process_label_set_at(label_fd, label, on_exec) {
    // 通过 fd 写入标签（跨 namespace 场景）
    write(label_fd, label, strlen(label));
}
```

**Keyring 上下文**（`selinux.c:114-117`）：
```c
selinux_keyring_label_set(label) {
    setkeycreatecon_raw(label);  // 设置后续创建的 keyring 的安全上下文
}
```

**默认回退标签**：`unconfined_t`（`selinux.c:20`）

### 4.2 后端注册

`selinux.c:168-194`：

```c
static struct lsm_ops selinux_ops = {
    .enabled              = selinux_enabled,     // is_selinux_enabled()
    .process_label_get    = selinux_process_label_get,
    .process_label_set    = selinux_process_label_set,
    .process_label_fd_get = selinux_process_label_fd_get,
    .process_label_get_at = selinux_process_label_get_at,
    .process_label_set_at = selinux_process_label_set_at,
    .keyring_label_set    = selinux_keyring_label_set,
    .prepare              = NULL,   // SELinux 无需 prepare
    .cleanup              = NULL,
};
```

---

## 5. nop 后端

`nop.c:9-77`，所有操作均为空实现：

```c
static struct lsm_ops nop_ops = {
    .enabled              = nop_enabled,           // return false
    .process_label_get    = nop_process_label_get,  // return NULL
    .process_label_set    = nop_process_label_set,  // return 0
    // ... 其余均为无操作
};
```

在没有任何 LSM 可用时使用，确保代码路径不需要条件判断。

---

## 6. 容器启动集成

LSM 在容器启动流程中的调用点（`start.c`）：

```
容器启动序列:
  ├── start.c:970   handler->lsm_ops = lsm_init_static()
  │                  // 选择并初始化 LSM 后端
  │
  ├── start.c:1067  handler->lsm_ops->prepare(ops, conf, lxcpath)
  │                  // 生成/加载安全策略
  │                  // AppArmor: 生成 profile + apparmor_parser 加载
  │                  // SELinux: 无操作
  │
  ├── start.c:1471  handler->lsm_ops->process_label_set(label, conf, true, false)
  │                  // 在容器进程 exec 之前应用标签
  │                  // on_exec=true → 标签在 exec 时生效
  │
  └── start.c:1139  handler->lsm_ops->cleanup(ops, conf, lxcpath)
                     // 清理临时资源
                     // AppArmor: 清理临时 profile/namespace
```

---

## 7. 配置项

### 7.1 AppArmor 配置

`confile.c:193-196`：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `lxc.apparmor.profile` | `generated` | 安全策略名称，`unconfined` 禁用，`generated` 自动生成 |
| `lxc.apparmor.allow_incomplete` | `0` | AppArmor 功能不完整时是否继续启动 |
| `lxc.apparmor.allow_nesting` | `0` | 是否允许嵌套容器（追加挂载规则） |
| `lxc.apparmor.raw` | _(空)_ | 追加到 profile 的原始 AppArmor 规则 |

存储字段（`conf.h`）：
```c
conf->lsm_aa_profile          // profile 名称
conf->lsm_aa_allow_incomplete // 允许不完整
conf->lsm_aa_allow_nesting    // 允许嵌套
conf->lsm_aa_raw              // 原始规则列表
conf->lsm_aa_profile_created  // 是否已创建 profile（内部标志）
```

### 7.2 SELinux 配置

`confile.c:266-267`：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `lxc.selinux.context` | `unconfined_t` | 容器进程的 SELinux 上下文 |
| `lxc.selinux.context.keyring` | _(空)_ | 容器 keyring 的安全上下文 |

---

## 8. 设计总结

1. **策略模式**：通过 `struct lsm_ops` 虚函数表实现 LSM 后端的可插拔替换。

2. **AppArmor 动态 Profile**：相比 SELinux 的静态策略文件，AppArmor 后端在运行时动态生成 profile，根据容器配置（嵌套、特权级别等）组合不同模板。

3. **fd-based 跨 namespace 操作**：`process_label_fd_get/set_at` 允许通过文件描述符操作其他 namespace 中进程的安全标签，是 attach 机制的基础。

4. **LSM Stacking 感知**：当内核支持多 LSM 同时运行时，AppArmor 后端自动适配 stacking 模式。

5. **零开销兜底**：nop 后端确保无 LSM 环境下代码路径一致，无需散布条件检查。
