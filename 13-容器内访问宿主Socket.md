# 容器内访问宿主 Unix Domain Socket

## 1. 问题描述

容器内的进程需要与宿主端的服务通信（例如通过 Unix Domain Socket），但 LXC 容器默认使用独立的 network namespace，导致宿主的 socket 不可见。

## 2. Abstract vs 路径型 Unix Socket

| 类型 | 绑定位置 | 容器可见性 |
|------|----------|-----------|
| Abstract (`\0name`) | network namespace | ❌ 独立 netns 下不可见 |
| 路径型 (`/run/x.sock`) | 文件系统 | ✅ 可通过 bind mount 暴露 |

**关键原理**：Abstract unix domain socket 存在于 network namespace 的内核数据结构中，而非文件系统。当容器拥有独立的 CLONE_NEWNET 时，宿主端的 abstract socket 对容器完全不可见，反之亦然。

## 3. 解决方案

### 3.1 方案 1：共享网络命名空间（放弃网络隔离）

```ini
# 容器直接使用宿主网络栈
lxc.net.0.type = none
```

或指定共享特定进程的 netns：

```ini
lxc.namespace.share.net = /proc/1/ns/net
```

**优点**：最简单，容器可直接访问宿主所有 abstract socket  
**缺点**：完全失去网络隔离，容器与宿主共享 IP 和端口

### 3.2 方案 2：Bind Mount 路径型 Socket（推荐）

前提：宿主服务必须使用**路径型** socket（如 `/run/myservice.sock`）。

```ini
# LXC 配置：将宿主 socket 文件挂载到容器内
lxc.mount.entry = /run/myservice.sock run/myservice.sock none bind,create=file 0 0
```

容器内进程连接 `/run/myservice.sock` 即可通信。

**工作原理**：
1. 宿主服务 `bind()` 到 `/run/myservice.sock`，创建 socket inode
2. LXC 的 `lxc.mount.entry` 将该 inode bind mount 到容器 rootfs 内
3. 容器进程 `connect()` 该路径时，内核通过 inode 找到 socket，直接建立连接
4. 通信不经过 network namespace，而是通过 VFS 层的 inode 引用

**注意事项**：
- 使用 `create=file` 确保容器内目标路径存在
- socket 文件必须在 mount 之前由宿主服务创建
- 权限：容器内进程的 UID 需要有权限连接该 socket

### 3.3 方案 3：TCP/IP 通过 veth 通信

```ini
lxc.net.0.type = veth
lxc.net.0.link = lxcbr0
lxc.net.0.flags = up
lxc.net.0.ipv4.address = 10.0.3.100/24
lxc.net.0.ipv4.gateway = 10.0.3.1
```

宿主服务监听 `10.0.3.1:PORT` 或 `0.0.0.0:PORT`，容器通过网关 IP 连接。

**优点**：标准网络通信，服务无需修改  
**缺点**：需要 TCP 开销，需要管理端口

### 3.4 方案 4：socat 代理中转

当宿主服务只提供 abstract socket 且无法修改时：

```bash
# 宿主端：创建代理，将 abstract socket 转为路径型
socat UNIX-LISTEN:/run/proxy.sock,fork ABSTRACT-CONNECT:myservice
```

然后使用方案 2 将 `/run/proxy.sock` bind mount 到容器内。

### 3.5 方案 5：使用 nsenter 反向代理

```bash
# 在宿主端运行代理，连接容器 netns 中的 abstract socket
nsenter --net=/proc/<container-pid>/ns/net socat ABSTRACT-CONNECT:myservice ...
```

适用于从宿主访问容器内 abstract socket 的反向场景。

## 4. 方案对比

| 方案 | 网络隔离 | 性能 | 适用场景 |
|------|----------|------|----------|
| 共享 netns | ❌ | 最高 | 不需要网络隔离的环境 |
| Bind mount socket | ✅ | 高（零拷贝 IPC） | 服务使用路径型 socket |
| TCP/IP via veth | ✅ | 中等 | 通用 RPC、HTTP 服务 |
| socat 代理 | ✅ | 中等 | 不可修改的 abstract socket 服务 |
| nsenter 反向 | ✅ | 中等 | 宿主访问容器内服务 |

## 5. LXC 中 bind mount socket 的内核机制

### 5.1 为什么 bind mount 可以跨 network namespace

Unix domain socket 的连接不通过 network namespace 的路由/协议栈，而是通过 VFS inode：

```
Container process                        Host service
     |                                        |
     | connect("/run/svc.sock")               | bind("/run/svc.sock")
     |      |                                 |      |
     |      v                                 |      v
     |  VFS lookup -> inode (bind-mounted)    |  VFS -> same inode
     |      |                                 |      |
     |      +---------> kernel socket <-------+      |
     |                  (unix_sock)                   |
     |                                               |
     | <------------- data transfer --------------> |
```

bind mount 让容器内路径指向**同一个 inode**，内核的 unix socket 连接逻辑通过 inode 匹配，不检查 network namespace。

### 5.2 LXC mount entry 处理流程

```
lxc_setup() -> lxc_mount_auto_mounts() / mount_file_entries()
  -> mount(source, target, NULL, MS_BIND, NULL)
  -> 如果 socket 文件未创建：create=file 先 touch 目标路径
```

## 6. 安全考虑

| 风险 | 缓解措施 |
|------|----------|
| 容器内进程访问敏感宿主服务 | 仅 mount 必要的 socket |
| 权限逃逸 | socket 文件权限限制（chmod 0600） |
| 容器内 root 可连接 | 配合 user namespace + ID mapping |
| DoS 宿主服务 | 服务端做连接数限制 |

## 7. 实践示例

### 7.1 容器访问宿主 Docker daemon

```ini
# Docker 使用路径型 socket
lxc.mount.entry = /var/run/docker.sock var/run/docker.sock none bind,create=file 0 0
```

### 7.2 容器访问宿主 D-Bus 系统总线

```ini
# D-Bus system bus
lxc.mount.entry = /run/dbus/system_bus_socket run/dbus/system_bus_socket none bind,create=file 0 0
```

### 7.3 容器访问宿主 GPU 运行时

```ini
# NVIDIA container runtime socket
lxc.mount.entry = /run/nvidia-container-runtime.sock run/nvidia-container-runtime.sock none bind,create=file 0 0
```
