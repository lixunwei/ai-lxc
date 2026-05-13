# LXC Cgroup 子系统深度分析（三）：BPF 设备控制与工具函数

## 1. 概述

在 cgroup v2 中，设备访问控制不再使用 `devices.allow/devices.deny` 伪文件，而是通过 **BPF 程序**（`BPF_PROG_TYPE_CGROUP_DEVICE`）实现。LXC 在用户态动态生成 BPF 字节码，加载到内核并附加到容器的 cgroup 上。

核心文件：
- `src/lxc/cgroups/cgroup2_devices.c`（~700 行）— BPF 程序构建与管理
- `src/lxc/cgroups/cgroup2_devices.h`（~60 行）— 数据结构定义
- `src/include/bpf.h` — BPF 系统调用相关常量
- `src/lxc/cgroups/cgroup_utils.c/h` — cgroup 工具函数
- `src/lxc/freezer.c` — 冻结实现

---

## 2. BPF 设备控制数据结构

### 2.1 `struct bpf_program`

`cgroup2_devices.h:39-50`：

```c
struct bpf_program {
    /* 默认策略：ALLOWLIST（默认拒绝）或 DENYLIST（默认允许） */
    int device_list_type;

    /* BPF 程序加载后的内核 fd */
    int kernel_fd;

    /* 程序类型：BPF_PROG_TYPE_CGROUP_DEVICE */
    __u32 prog_type;

    /* 当前指令数 */
    size_t n_instructions;

    /* 用户态构造的 BPF 指令数组 */
    struct bpf_insn *instructions;

    /* 已 attach 到的 cgroup fd */
    int fd_cgroup;

    /* attach 类型：BPF_CGROUP_DEVICE */
    __u32 attached_type;

    /* attach 标志：如 BPF_F_ALLOW_MULTI */
    __u32 attached_flags;
};
```

### 2.2 `struct bpf_cgroup_dev_ctx`

`bpf.h:4930-4933`，内核传给 BPF 程序的上下文：

```c
struct bpf_cgroup_dev_ctx {
    __u32 access_type;  // 编码：(BPF_DEVCG_ACC_* << 16) | BPF_DEVCG_DEV_*
    __u32 major;        // 主设备号
    __u32 minor;        // 次设备号
};
```

`access_type` 字段编码方式：
- **低 16 位**：设备类型（`BPF_DEVCG_DEV_BLOCK` = 1, `BPF_DEVCG_DEV_CHAR` = 2）
- **高 16 位**：访问权限（`BPF_DEVCG_ACC_MKNOD` = 1, `BPF_DEVCG_ACC_READ` = 2, `BPF_DEVCG_ACC_WRITE` = 4）

---

## 3. BPF 程序生成

### 3.1 前置指令：解码上下文

`bpf_program_init()`，`cgroup2_devices.c:178-200`：

每个 BPF 程序开头都有固定的解码指令，从 `ctx` 中提取设备信息：

```asm
; 输入：r1 = ctx (struct bpf_cgroup_dev_ctx *)

r2 = *(u32 *)(r1 + 0)    ; r2 = ctx->access_type
r2 &= 0xFFFF             ; r2 = 设备类型（低 16 位）

r3 = *(u32 *)(r1 + 0)    ; r3 = ctx->access_type
r3 >>= 16                ; r3 = 访问权限（高 16 位）

r4 = *(u32 *)(r1 + 4)    ; r4 = ctx->major
r5 = *(u32 *)(r1 + 8)    ; r5 = ctx->minor
```

执行完后的寄存器分配：
| 寄存器 | 内容 |
|--------|------|
| r2 | 设备类型（block/char） |
| r3 | 访问权限掩码（r/w/m） |
| r4 | 主设备号 |
| r5 | 次设备号 |

### 3.2 规则编译：条件跳转链

`bpf_program_append_device()`，`cgroup2_devices.c:202-281`：

每条设备规则被编译为一系列条件跳转指令：

```asm
; 规则示例：c 1:5 rwm → 允许字符设备 1:5 的读写创建

; === 检查访问权限（如果不是全权限） ===
r1 = r3                    ; r1 = 实际访问权限
r1 &= access_mask          ; r1 &= 请求的权限掩码
if r1 != r3: goto next_rule ; 权限不匹配，跳过

; === 检查设备类型（如果不是 'a' 全部） ===
if r2 != device_type: goto next_rule  ; 类型不匹配

; === 检查主设备号（如果指定了具体值） ===
if r4 != major: goto next_rule        ; 主设备号不匹配

; === 检查次设备号（如果指定了具体值） ===
if r5 != minor: goto next_rule        ; 次设备号不匹配

; === 全部匹配，返回结果 ===
r0 = allow  ; 1 = 允许, 0 = 拒绝
exit

next_rule:
; ... 下一条规则的指令 ...
```

规则中的通配符处理：
- 类型为 `'a'`（all）→ 跳过类型检查
- `major < 0` → 跳过主设备号检查
- `minor < 0` → 跳过次设备号检查
- 全权限（rwm 都有）→ 跳过权限检查

### 3.3 尾部：默认策略

`bpf_program_finalize()`，`cgroup2_devices.c:284-299`：

```asm
; 所有规则都不匹配时的默认返回
r0 = device_list_type  ; ALLOWLIST(0)=拒绝 / DENYLIST(1)=允许
exit
```

### 3.4 生成的完整程序结构

```
┌─────────────────────────┐
│  解码上下文（5 条指令）    │  从 ctx 提取 r2-r5
├─────────────────────────┤
│  规则 1：条件跳转链       │  匹配则返回 allow/deny
├─────────────────────────┤
│  规则 2：条件跳转链       │
├─────────────────────────┤
│  ...                    │
├─────────────────────────┤
│  规则 N：条件跳转链       │
├─────────────────────────┤
│  默认策略返回（2 条指令）  │  返回 ALLOWLIST/DENYLIST
└─────────────────────────┘
```

---

## 4. Allowlist vs Denylist 策略

### 4.1 策略切换逻辑

`bpf_list_add_device()`，`cgroup2_devices.c:451-477`：

```c
// 遇到 "a *:* rwm" 这样的全局规则时，切换策略
if (device_type == 'a' && major == -1 && minor == -1) {
    if (allow) {
        // "允许所有" → 默认策略变为 DENYLIST（只编译拒绝规则）
        device_list_type = DENYLIST;
        清空所有现有规则;
    } else {
        // "拒绝所有" → 默认策略变为 ALLOWLIST（只编译允许规则）
        device_list_type = ALLOWLIST;
        清空所有现有规则;
    }
    return;
}
```

### 4.2 优化：只编译必要规则

`__bpf_cgroup_devices()`，`cgroup2_devices.c:556-593`：

```c
for (device in device_list) {
    // 只有与默认策略"相反"的规则才编译进 BPF
    // ALLOWLIST 默认拒绝 → 只编译 allow 规则
    // DENYLIST 默认允许 → 只编译 deny 规则
    if (device->allow == device_list_type)
        continue;  // 与默认行为相同，无需编译

    bpf_program_append_device(prog, device);
}
```

这大幅减少了 BPF 程序的指令数，提升性能。

---

## 5. BPF 程序加载与 Attach

### 5.1 加载到内核

`bpf_program_load_kernel()`，`cgroup2_devices.c:301-337`：

```c
union bpf_attr attr = {
    .prog_type  = BPF_PROG_TYPE_CGROUP_DEVICE,
    .insns      = prog->instructions,
    .insn_cnt   = prog->n_instructions,
    .license    = "GPL",
    // 可选：verifier log buffer（调试用）
};

prog->kernel_fd = bpf(BPF_PROG_LOAD, &attr, sizeof(attr));
```

### 5.2 Attach 到 cgroup

`bpf_program_cgroup_attach()`，`cgroup2_devices.c:339-389`：

```c
bpf_program_cgroup_attach(prog, attach_type, fd_cgroup, flags)
  // 1. 验证 flags 合法性
  只允许：BPF_F_ALLOW_OVERRIDE | BPF_F_ALLOW_MULTI | BPF_F_REPLACE

  // 2. 复制 cgroup fd（避免外部关闭）
  prog->fd_cgroup = dup_cloexec(fd_cgroup)

  // 3. 加载 BPF 程序
  bpf_program_load_kernel(prog)

  // 4. Attach
  union bpf_attr attr = {
      .attach_type   = BPF_CGROUP_DEVICE,
      .target_fd     = cgroup_fd,
      .attach_bpf_fd = prog->kernel_fd,
      .attach_flags  = flags,
  };
  bpf(BPF_PROG_ATTACH, &attr, sizeof(attr))

  // 5. 记录状态
  prog->attached_type  = attach_type;
  prog->attached_flags = flags;
```

### 5.3 首次 Attach

`bpf_cgroup_devices_attach()`，`cgroup2_devices.c:595-614`：

```
bpf_cgroup_devices_attach(ops, bpf_devices)
  1. __bpf_cgroup_devices()  → 构建完整 BPF 程序
  2. attach 到 ops->unified->dfd_lim（limit cgroup）
  3. flags = BPF_F_ALLOW_MULTI
  4. swap(prog, ops->cgroup2_devices)  → 保存程序引用
```

### 5.4 动态更新

`bpf_cgroup_devices_update()`，`cgroup2_devices.c:616-705`：

```
bpf_cgroup_devices_update(ops, bpf_devices, new_rule)
  1. bpf_list_add_device()        → 更新规则集合
  2. if (之前没有挂过程序):
       bpf_cgroup_devices_attach()  → 首次挂载
       return
  3. 重新生成 BPF 程序
  4. bpf_program_load_kernel()     → 加载新程序
  5. 尝试原子替换：
       attr.attach_flags = BPF_F_REPLACE | BPF_F_ALLOW_MULTI
       attr.replace_bpf_fd = old->kernel_fd
       bpf(BPF_PROG_ATTACH, ...)
  6. if (内核不支持 REPLACE):
       退化为 BPF_F_ALLOW_MULTI（追加）
  7. 更新 ops->cgroup2_devices
```

**BPF_F_REPLACE** 是 Linux 5.5+ 引入的特性，允许原子替换已挂载的 BPF 程序，避免短暂的策略缺失窗口。

---

## 6. 特性检测

### 6.1 `bpf_devices_cgroup_supported()`

`cgroup2_devices.c:520-554`：

```
bpf_devices_cgroup_supported()
  1. 检查 geteuid() == 0（必须 root）
  2. 检查 bpf 系统调用可用：
     bpf(BPF_PROG_LOAD, NULL, sizeof(attr))
     若 errno == ENOSYS → 不支持
  3. 构造最小测试程序：
     r0 = 1; exit    // 最简单的合法 BPF 程序
  4. 尝试加载：
     bpf_program_init()
     bpf_program_add_instructions(test_insns)
     bpf_program_load_kernel()
  5. 成功加载 → 支持
```

---

## 7. 工具函数

### 7.1 cgroup 文件系统检测

`cgroup_utils.c:22-33`：

```c
bool unified_cgroup_fd(int fd) {
    struct statfs fs;
    fstatfs(fd, &fs);
    return fs.f_type == CGROUP2_SUPER_MAGIC;
}
```

### 7.2 cgroup namespace 检测

`cgroup_utils.h:16-24`：

```c
static inline bool cgns_supported(void) {
    static int supported = -1;
    if (supported == -1)
        supported = file_exists("/proc/self/ns/cgroup");
    return supported;
}
```

使用 `static` 变量缓存结果，避免重复系统调用。

### 7.3 递归删除 cgroup 树

`cgroup_utils.c:35-94`，`cgroup_tree_prune()`：

```
cgroup_tree_prune(dfd, path)
  dirfd = openat(dfd, path, O_DIRECTORY)
  dir = fdopendir(dirfd)

  for entry in readdir(dir):
      if entry.type == DT_DIR:
          cgroup_tree_prune(dirfd, entry.name)  // 递归

  closedir(dir)
  unlinkat(dfd, path, AT_REMOVEDIR)  // 删除空目录
```

### 7.4 init.scope 路径裁剪

`cgroup_utils.c:96-123`，`prune_init_scope()`：

```c
char *prune_init_scope(char *path) {
    // 如果路径末尾是 "/init.scope"，裁掉这部分
    // 例如："/user.slice/user-1000.slice/init.scope"
    //     → "/user.slice/user-1000.slice"
    char *slash = strrchr(path, '/');
    if (slash && strcmp(slash, "/init.scope") == 0)
        *slash = '\0';
    return path;
}
```

这在 systemd 环境下很常见，`init.scope` 是 systemd 为 PID 1 创建的默认 scope。

---

## 8. 冻结实现细节

### 8.1 cgroup v1 冻结（freezer.c）

`freezer.c:28-67`：

v1 冻结使用 `freezer.state` 伪文件：

```
状态机：
  THAWED ──写"FROZEN"──→ FREEZING ──（内核完成）──→ FROZEN
  FROZEN ──写"THAWED"──→ THAWED
```

LXC 写入目标状态后，需要循环轮询直到实际状态生效：

```c
do_freeze_thaw(name, lxcpath, freeze) {
    cgroup_init(conf);  // 初始化 cgroup

    // 写入目标状态
    ops->set(freeze ? "FROZEN" : "THAWED",
             "freezer.state", name, lxcpath);

    // 轮询等待状态生效
    for (int i = 0; i < 120; i++) {  // 最多等 120 次
        usleep(50000);  // 每次等 50ms
        ops->get("freezer.state", value, ...);
        if (strcmp(value, target_state) == 0)
            return 0;  // 成功
    }
    return -1;  // 超时
}
```

### 8.2 冻结事件通知

```c
lxc_freeze() {
    lxc_set_state(FREEZING);   // 通知监听者
    do_freeze_thaw(freeze);
    if (success)
        lxc_set_state(FROZEN);  // 冻结成功
    else
        lxc_set_state(RUNNING); // 冻结失败，恢复
}
```

---

## 9. 与 cgfsng 的集成关系

```
                    ┌──────────────────┐
                    │   confile.c      │
                    │ 配置解析         │
                    │ lxc.cgroup2.*   │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │   cgfsng.c       │
                    │ setup_limits()   │
                    │ devices_activate │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼────┐  ┌──────▼──────┐  ┌───▼──────────┐
    │ 普通控制器    │  │ BPF 设备控制 │  │ commands.c   │
    │ 直接写文件    │  │ 编译+加载    │  │ 运行时更新    │
    │ memory.max  │  │ BPF 程序     │  │ 命令通道      │
    │ cpu.max     │  │ attach       │  │ BPF_F_REPLACE │
    └──────────────┘  └─────────────┘  └──────────────┘
```

**启动时**：cgfsng 通过 `setup_limits()` + `devices_activate()` 一次性编译并挂载 BPF 程序。

**运行时**：通过 LXC 命令通道（`commands.c`）接收设备规则更新请求，调用 `bpf_cgroup_devices_update()` 原子替换 BPF 程序。

---

## 10. 关键设计总结

1. **用户态 BPF 生成**：LXC 在用户态直接构造 BPF 字节码指令，无需外部编译器（如 clang/llvm），减少运行时依赖。

2. **最小指令集优化**：只编译与默认策略相反的规则，减少 BPF 程序大小和内核验证时间。

3. **原子更新**：优先使用 `BPF_F_REPLACE` 原子替换旧程序，内核不支持时退化为 `BPF_F_ALLOW_MULTI` 追加。

4. **策略自动切换**：遇到 `a *:* rwm` 全局规则时，自动在 ALLOWLIST/DENYLIST 之间切换，并清空旧规则。

5. **fd-based 设计一致性**：与 cgfsng 其他部分一致，BPF attach 也使用 cgroup fd 而非路径。

6. **特性检测**：运行时探测 BPF 支持，gracefully 处理不支持 BPF 的旧内核。
