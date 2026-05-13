# LXC Cgroup 子系统深度分析（二）：运行时操作

## 1. 概述

本文档分析 cgfsng 驱动在容器运行期间的核心操作，包括进程进入 cgroup、资源限制设置、读写控制、冻结/解冻、权限管理以及 cgroup 销毁等。

所有实现位于 `src/lxc/cgroups/cgfsng.c`（约 3600 行），冻结相关还涉及 `src/lxc/freezer.c`。

---

## 2. 进程进入 Cgroup

### 2.1 Monitor 进入

`cgfsng_monitor_enter()`，`cgfsng.c:1555-1620`：

```
cgfsng_monitor_enter(ops, handler)
  for each hierarchy:
      // 将 monitor PID 写入 cgroup.procs
      write(hierarchy->dfd_mon + "/cgroup.procs", monitor_pid)

      // 如果有 transient PID，也写入
      if (transient_pid)
          write(hierarchy->dfd_mon + "/cgroup.procs", transient_pid)

      // 非 unified 层级的 fd 用完即关闭
      if (!unified)
          close(hierarchy->dfd_mon)
```

### 2.2 Payload 进入

`cgfsng_payload_enter()`，`cgfsng.c:1622-1660`：

```
cgfsng_payload_enter(ops, handler)
  for each hierarchy:
      // 如果使用了 CLONE_INTO_CGROUP 且是 unified 层级
      // 则跳过（进程已通过 clone3 直接进入）
      if (unified && CLONE_INTO_CGROUP)
          continue;

      // 将 payload PID 写入 cgroup.procs
      write(hierarchy->dfd_con + "/cgroup.procs", payload_pid)
```

**CLONE_INTO_CGROUP 优化**：Linux 5.7+ 的 `clone3()` 系统调用支持直接将子进程放入目标 cgroup，无需额外的 `cgroup.procs` 写入。LXC 会检测此特性并自动利用。

---

## 3. 资源限制

### 3.1 批量设置：`cgfsng_setup_limits()`

`cgfsng.c:2905-2976`，在容器启动时批量应用所有 `lxc.cgroup2.*` 配置：

```
cgfsng_setup_limits(ops, conf)
  // 仅对纯 unified（cgroup v2）配置生效
  // legacy/v1 配置在其他路径处理
  if (!pure_unified_layout)
      跳过 cgroup2 配置项

  for each cgroup2 配置项:
      if (subsystem == "devices"):
          // 设备控制走 BPF 路径
          bpf_device_cgroup_prepare(ops, conf, key, value)
      else:
          // 普通控制器：直接写文件
          // 写入路径：hierarchy->path_lim/<subsystem>
          lxc_write_to_file(path_lim/subsystem, value)
```

关键点：
- **资源限制写入 limit cgroup**（`path_lim`），不是 container cgroup
- **设备控制不写文件**，而是收集 BPF 规则，后续通过 `devices_activate` 一次性挂载

### 3.2 单项读取：`cgfsng_get()`

`cgfsng.c:2602-2637`：

```
cgfsng_get(filename, value, len, path, lxcpath)
  1. 从 filename 提取 controller（去掉第一个 '.' 后面的部分）
     例如："memory.max" → controller = "memory"
  2. 通过命令接口获取运行中容器的 cgroup 路径
  3. 拼出完整路径：<cgroup_path>/<filename>
  4. lxc_read_from_file() 读取
```

### 3.3 单项写入：`cgfsng_set()`

`cgfsng.c:2749-2799`：

```
cgfsng_set(filename, value, path, lxcpath)
  1. 解析 controller 和 path
  2. if (unified && controller == "devices"):
       // cgroup v2 设备控制
       解析规则 → lxc_cmd_add_bpf_device_cgroup()
       // 通过命令通道发送给运行中的容器管理进程
  3. else:
       // 直接写文件
       lxc_write_to_file(cgroup_path/filename, value)
```

---

## 4. 冻结与解冻

### 4.1 cgroup v2 路径（cgfsng.c 内实现）

`cgfsng.c:3806-3899`：

```
cgroup_freeze(ops, timeout)
  // 获取 unified cgroup 的 fd
  fd = ops->unified->dfd_con

  // 写入 cgroup.freeze = 1
  write(fd + "/cgroup.freeze", "1")

  // 可选：通过 cgroup.events + mainloop 等待状态变化
  if (timeout > 0):
      监听 cgroup.events 中 "frozen 1" 状态

cgroup_unfreeze(ops, timeout)
  write(fd + "/cgroup.freeze", "0")
  // 等待 "frozen 0" 状态
```

### 4.2 cgroup v1 路径（freezer.c）

`freezer.c:28-87`：

```
do_freeze_thaw(name, lxcpath, freeze)
  1. cgroup_init(conf)  // 初始化 cgroup 后端
  2. 写入 freezer.state = "FROZEN" 或 "THAWED"
  3. 循环读取 freezer.state 直到状态生效
     // 防止中间态 "FREEZING"

lxc_freeze(name, lxcpath)
  1. 通知监听者 → FREEZING
  2. do_freeze_thaw(freeze=true)
  3. 成功 → 通知 FROZEN
     失败 → 通知 RUNNING

lxc_unfreeze(name, lxcpath)
  1. 通知监听者 → THAWED
  2. do_freeze_thaw(freeze=false)
  3. 成功 → 通知 RUNNING
     失败 → 通知 FROZEN
```

---

## 5. 权限管理（chown）

### 5.1 非特权容器 chown

`cgfsng_chown()`，`cgfsng.c:1746-1758`：

```
cgfsng_chown(ops, conf)
  // 在新的 user namespace 中执行 chown
  userns_exec_1(chown_cgroup_wrapper, ...)
```

### 5.2 chown 实际逻辑

`chown_cgroup_wrapper()`，`cgfsng.c:1687-1743`：

```
chown_cgroup_wrapper(data)
  // 切换到目标 namespace 的身份
  setresgid(target_gid)
  setresuid(target_uid)

  for each hierarchy:
      // chown + chmod cgroup 目录本身
      fchownat(dfd, "", uid, gid)
      fchmodat(dfd, "", 0775)

      // Legacy 下还处理 tasks 文件
      if (legacy):
          chown(tasks)

      // 总是处理 cgroup.procs
      chown(cgroup.procs)

      // Unified 下处理所有 delegated 文件
      if (unified):
          for file in delegated_files:
              chown(file)
```

这确保非特权容器（unprivileged）能正确操作自己的 cgroup 目录和控制文件。

---

## 6. CRIU 支持

### 6.1 Escape 到根 cgroup

`cgfsng_criu_escape()`，`cgfsng.c:2038-2065`：

```
cgfsng_criu_escape(ops, conf)
  // 仅 root 用户且非 relative 模式
  if (geteuid() != 0 || conf->cgroup_meta.relative)
      return;

  for each hierarchy:
      // 将 "0"（当前进程 PID）写入根 cgroup 的 cgroup.procs
      write(hierarchy->at_base + "/cgroup.procs", "0")
```

CRIU 执行 checkpoint/restore 时需要进程先"逃回"根 cgroup，以避免 cgroup 路径冲突。

### 6.2 统一层级移动

`move_and_delegate_unified()`，`cgfsng.c:1223-1246`：

```
move_and_delegate_unified()
  // 创建 parent_cgroup/init 子目录
  mkdir(parent_cgroup + "/init")

  // 将当前进程移入
  write(parent_cgroup + "/init/cgroup.procs", self_pid)

  // 用于 delegation/escape 场景
```

---

## 7. Cgroup 销毁

### 7.1 Payload 销毁

`cgfsng_payload_destroy()`，`cgfsng.c:398-449`：

```
cgfsng_payload_destroy(ops, handler)
  // 1. 卸载 BPF 设备程序
  if (ops->cgroup2_devices)
      bpf_program_cgroup_detach(ops->cgroup2_devices)

  // 2. 删除容器 cgroup 树
  if (has_userns):
      // 非特权模式：在新 userns 中执行删除
      userns_exec_full(cgroup_tree_remove_wrapper, ...)
  else:
      // 特权模式：直接删除
      cgroup_tree_remove(container_limit_cgroup)
```

### 7.2 Monitor 销毁

`cgfsng_monitor_destroy()`，`cgfsng.c:596-668`：

```
cgfsng_monitor_destroy(ops, handler)
  // 1. 先将 monitor 进程移到 pivot cgroup
  //    避免删除正在使用的 cgroup（内核禁止）
  write(CGROUP_PIVOT + "/init/cgroup.procs", monitor_pid)

  // 2. 删除监控 cgroup 树
  cgroup_tree_prune(monitor_cgroup)
```

### 7.3 递归删除

`cgroup_tree_prune()`，`cgroup_utils.c:35-94`：

```
cgroup_tree_prune(dfd, path)
  // 打开目标目录
  dirfd = openat(dfd, path)

  // 遍历子目录，递归删除
  for entry in readdir(dirfd):
      if (entry.is_dir):
          cgroup_tree_prune(dirfd, entry.name)

  // 删除空目录
  unlinkat(dfd, path, AT_REMOVEDIR)
```

---

## 8. 设备控制流程总结

cgroup v2 下的设备控制涉及多个组件协作：

```
=== 容器启动 ===

配置解析（confile.c）
    ↓ lxc.cgroup2.devices.allow = ...
cgfsng_setup_limits()
    ↓ 收集规则
bpf_device_cgroup_prepare()
    ↓ 构建规则列表
cgfsng_devices_activate()
    ↓ 编译 BPF 程序
bpf_cgroup_devices_attach()
    ↓ attach 到 limit cgroup
内核执行 BPF 程序进行设备访问控制

=== 运行时动态更新 ===

lxc_cmd_add_bpf_device_cgroup()
    ↓ 通过命令通道
bpf_cgroup_devices_update()
    ↓ 更新规则列表
重新编译 BPF 程序
    ↓ BPF_F_REPLACE（原子替换）或 BPF_F_ALLOW_MULTI（追加）
内核切换到新 BPF 程序
```

---

## 9. 完整生命周期调用链

```
容器启动:
  cgroup_init()
    → cgroup_ops_init()           # 探测环境
    → data_init()                 # 读取配置

  monitor_create()                # 创建 monitor cgroup
  monitor_enter()                 # monitor 进程进入
  monitor_delegate_controllers()  # 启用 v2 controller delegation

  payload_create()                # 创建 payload cgroup
  payload_delegate_controllers()  # 启用 v2 controller delegation
  chown()                         # 非特权容器权限设置

  setup_limits()                  # 设置资源限制
  devices_activate()              # 挂载 BPF 设备程序

  payload_enter()                 # 容器进程进入 cgroup

容器运行:
  get() / set()                   # 读写 cgroup 参数
  freeze() / unfreeze()           # 冻结/解冻
  bpf_cgroup_devices_update()     # 动态更新设备规则

容器停止:
  payload_destroy()               # 卸载 BPF + 删除 payload cgroup
  monitor_destroy()               # pivot + 删除 monitor cgroup
  cgroup_exit()                   # 释放所有资源
```
