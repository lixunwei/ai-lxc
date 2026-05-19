# 03-A 网络子系统深入分析：数据结构与 Netlink 通信

## 1. 完整数据结构详解

### 1.1 网络类型枚举

```c
enum {
    LXC_NET_EMPTY,      // 仅 loopback
    LXC_NET_VETH,       // 虚拟以太网对
    LXC_NET_MACVLAN,    // MAC 地址虚拟化
    LXC_NET_IPVLAN,     // IP 地址虚拟化
    LXC_NET_PHYS,       // 物理网卡直通
    LXC_NET_VLAN,       // VLAN 子接口
    LXC_NET_NONE,       // 无网络
    LXC_NET_MAXCONFTYPE // 哨兵值
};
```

### 1.2 IPv4/IPv6 地址结构

**`struct lxc_inetdev`** — IPv4 地址配置项：

| 字段 | 类型 | 说明 |
|------|------|------|
| `addr` | `struct in_addr` | IPv4 地址 |
| `bcast` | `struct in_addr` | 广播地址 |
| `prefix` | `unsigned int` | 前缀长度（如 24 → /24） |
| `head` | `struct list_head` | 链表节点 |

**`struct lxc_inet6dev`** — IPv6 地址配置项：

| 字段 | 类型 | 说明 |
|------|------|------|
| `addr` | `struct in6_addr` | IPv6 地址 |
| `mcast` | `struct in6_addr` | 多播地址 |
| `acast` | `struct in6_addr` | 任播地址 |
| `prefix` | `unsigned int` | 前缀长度 |
| `head` | `struct list_head` | 链表节点 |

### 1.3 类型专属结构

#### `struct ifla_veth` — veth 宿主端信息

| 字段 | 类型 | 说明 |
|------|------|------|
| `pair[IFNAMSIZ]` | `char` | 用户指定的宿主端名称 |
| `veth1[IFNAMSIZ]` | `char` | 自动生成的宿主端名称 |
| `ifindex` | `int` | 宿主端 ifindex |
| `ipv4_routes` | `struct list_head` | 宿主侧 IPv4 路由列表 |
| `ipv6_routes` | `struct list_head` | 宿主侧 IPv6 路由列表 |
| `mode` | `int` | 模式：bridge / router |
| `n_rxqueues` | `int` | 接收队列数 |
| `n_txqueues` | `int` | 发送队列数 |
| `vlan_id` | `short` | VLAN ID |
| `vlan_id_set` | `bool` | 是否设置了 VLAN ID |
| `vlan_tagged_ids` | `struct lxc_list` | VLAN tagged ID 列表 |

#### `struct ifla_macvlan` — macvlan 专属

| 字段 | 类型 | 说明 |
|------|------|------|
| `mode` | `int` | 模式：private / vepa / bridge / passthru |

#### `struct ifla_ipvlan` — ipvlan 专属

| 字段 | 类型 | 说明 |
|------|------|------|
| `mode` | `int` | 模式：l2 / l3 / l3s |
| `isolation` | `int` | 隔离模式：bridge / private / vepa |

#### `struct ifla_phys` — 物理网卡信息

| 字段 | 类型 | 说明 |
|------|------|------|
| `ifindex` | `int` | 宿主侧原始 ifindex |
| `altnames` | `char **` | 别名列表 |
| `mtu` | `int` | 原始 MTU（用于恢复） |

#### `struct ifla_vlan` — VLAN 属性

| 字段 | 类型 | 说明 |
|------|------|------|
| `flags` | `unsigned int` | VLAN 标志 |
| `fmask` | `unsigned int` | 标志掩码 |
| `vid` | `unsigned short` | VLAN ID |
| `pad` | `unsigned short` | 填充 |

#### `union netdev_p` — 类型联合体

根据 `lxc_netdev.type` 解释为对应结构体：

```c
union netdev_p {
    struct ifla_macvlan macvlan_attr;
    struct ifla_ipvlan  ipvlan_attr;
    struct ifla_phys    phys_attr;
    struct ifla_veth    veth_attr;
    struct ifla_vlan    vlan_attr;
};
```

### 1.4 核心结构：`struct lxc_netdev`

| 字段 | 类型 | 说明 |
|------|------|------|
| `idx` | `ssize_t` | 网络配置序号（lxc.net.**0**, lxc.net.**1** ...） |
| `ifindex` | `int` | 容器 netns 内的 ifindex |
| `type` | `int` | 网络类型（`LXC_NET_*`） |
| `flags` | `int` | 接口标志（如 `IFF_UP`） |
| `link[IFNAMSIZ]` | `char` | 关联的宿主侧接口/桥名 |
| `l2proxy` | `bool` | 是否启用 L2 代理 |
| `name[IFNAMSIZ]` | `char` | 容器内最终接口名 |
| `created_name[IFNAMSIZ]` | `char` | 创建时的临时名 |
| `transient_name[IFNAMSIZ]` | `char` | 过渡名称（避免命名冲突） |
| `hwaddr` | `char *` | MAC 地址 |
| `mtu` | `char *` | MTU 值 |
| `priv` | `union netdev_p` | 类型专属私有数据 |
| `ipv4_addresses` | `struct list_head` | IPv4 地址链表 |
| `ipv6_addresses` | `struct list_head` | IPv6 地址链表 |
| `ipv4_gateway_auto` | `bool` | 自动获取 IPv4 网关 |
| `ipv4_gateway_dev` | `bool` | IPv4 网关作为设备路由 |
| `ipv4_gateway` | `struct in_addr *` | IPv4 网关地址 |
| `ipv6_gateway_auto` | `bool` | 自动获取 IPv6 网关 |
| `ipv6_gateway_dev` | `bool` | IPv6 网关作为设备路由 |
| `ipv6_gateway` | `struct in6_addr *` | IPv6 网关地址 |
| `upscript` | `char *` | 接口创建后执行的脚本 |
| `downscript` | `char *` | 接口销毁前执行的脚本 |
| `head` | `struct list_head` | 链表节点 |

---

## 2. Netlink 通信层

### 2.1 核心数据结构

**`struct nl_handler`**（`nl.h:24-39`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `fd` | `int` | netlink socket fd |
| `seq` | `__u32` | 序列号（初始化为 `time(NULL)`） |
| `local` | `struct sockaddr_nl` | 本端地址 |
| `peer` | `struct sockaddr_nl` | 对端地址（内核：pid=0） |

**`struct nlmsg`**（`nl.h:43-54`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `nlmsghdr` | `struct nlmsghdr *` | netlink 消息头指针 |
| `cap` | `ssize_t` | 缓冲区总容量 |

### 2.2 消息构造 API

#### 分配与预留

| 函数 | 说明 |
|------|------|
| `nlmsg_alloc(size)` | 分配消息缓冲区，初始 `nlmsg_len = NLMSG_HDRLEN` |
| `nlmsg_alloc_reserve(size)` | 分配并设 `nlmsg_len = cap`（用于接收缓冲） |
| `nlmsg_reserve(nlmsg, len)` | 在尾部预留空间（含对齐检查） |
| `nlmsg_free(nlmsg)` | 释放消息 |
| `nlmsg_data(nlmsg)` | 获取 payload 指针 |

#### 属性操作

| 函数 | 说明 |
|------|------|
| `nla_put(nlmsg, type, data, len)` | 核心函数：构造 `rtattr`，写入 type/len/data |
| `nla_put_string(nlmsg, type, str)` | 写入字符串属性（含 `\0`） |
| `nla_put_u32(nlmsg, type, val)` | 写入 32 位整数属性 |
| `nla_put_u16(nlmsg, type, val)` | 写入 16 位整数属性 |
| `nla_put_attr(nlmsg, type)` | 写入空属性头（用于嵌套容器） |
| `nla_begin_nested(nlmsg, type)` | 开始嵌套属性（记录起始位置） |
| `nla_end_nested(nlmsg, attr)` | 结束嵌套属性（回填 `rta_len`） |

#### 嵌套属性工作原理

```
nla_begin_nested(nlmsg, IFLA_LINKINFO)
  +-- save current tail ptr -> attr
  +-- insert empty attr hdr { rta_type=IFLA_LINKINFO, rta_len=RTA_HDRLEN }

  ... add child attrs inside ...

nla_end_nested(nlmsg, attr)
  +-- attr->rta_len = current tail - attr start
```

### 2.3 Socket 操作

| 函数 | 说明 |
|------|------|
| `netlink_open(handler, protocol)` | 创建 `AF_NETLINK` socket，设置缓冲区，bind，初始化 seq |
| `netlink_close(handler)` | 关闭 socket |
| `netlink_send(handler, nlmsg)` | 发送消息（`sendmsg` + `MSG_NOSIGNAL`） |
| `netlink_rcv(handler, nlmsg)` | 接收消息（`recvmsg`，`EINTR` 自动重试） |
| `netlink_transaction(handler, req, resp)` | 完整事务：发送 → 接收 → 解析 `NLMSG_ERROR` |

#### 事务错误处理

```
netlink_transaction()
  +-- netlink_send(request)
  +-- netlink_rcv(response)
  +-- check response:
      +-- nlmsg_type == NLMSG_ERROR?
      |   +-- nlmsgerr.error == 0 -> success
      |   +-- nlmsgerr.error < 0 -> return -errno
      +-- else -> return raw response
```

### 2.4 rtnetlink 封装（rtnl.c）

`rtnl.c` 是对 `nl.c` 的薄封装，协议固定为 `NETLINK_ROUTE`：

```c
rtnetlink_open(handler)  → netlink_open(handler, NETLINK_ROUTE)
rtnetlink_close(handler) → netlink_close(handler)
rtnetlink_send(handler, msg) → netlink_send(handler, (struct nlmsg *)msg)
rtnetlink_rcv(handler, msg)  → netlink_rcv(handler, (struct nlmsg *)msg)
rtnetlink_transaction(h, req, resp) → netlink_transaction(h, req, resp)
```

### 2.5 Unix Socket 工具（af_unix.c）

用于进程间传递文件描述符和凭证（不构造 netlink 消息）：

| 函数 | 说明 |
|------|------|
| `lxc_abstract_unix_open()` | 打开抽象 Unix 域 socket |
| `lxc_abstract_unix_connect()` | 连接到 Unix 域 socket |
| `lxc_abstract_unix_send_fds()` | 通过 `SCM_RIGHTS` 发送文件描述符 |
| `lxc_abstract_unix_recv_fds()` | 接收文件描述符 |
| `lxc_abstract_unix_recv_one_fd()` | 接收单个 fd |
| `send_credential()` / `recv_credential()` | 发送/接收进程凭证 |

---

## 3. Netlink 消息详解

### 3.1 消息结构总览

所有 netlink 消息遵循以下结构：

```
+------------------------------+
| struct nlmsghdr               |  <- msg hdr
|   nlmsg_len, nlmsg_type,     |     RTM_NEWLINK / RTM_NEWADDR / ...
|   nlmsg_flags, nlmsg_seq     |     NLM_F_REQUEST | NLM_F_ACK | ...
+------------------------------+
| Payload (struct ifinfomsg    |  <- protocol hdr
|   / ifaddrmsg / rtmsg / ndmsg)|
+------------------------------+
| Attributes (rtattr chain)    |  <- attr chain
|   IFLA_IFNAME, IFLA_MTU,    |
|   IFLA_LINKINFO { nested }, |
|   ...                        |
+------------------------------+
```

### 3.2 各操作的 Netlink 消息构造

#### 创建 veth pair（`lxc_veth_create`，`network.c:2372-2458`）

```
RTM_NEWLINK | NLM_F_REQUEST | NLM_F_CREATE | NLM_F_EXCL | NLM_F_ACK

payload: struct ifinfomsg { ifi_family = AF_UNSPEC }

attributes:
  IFLA_LINKINFO (nested)
  +-- IFLA_INFO_KIND = "veth"
  +-- IFLA_INFO_DATA (nested)
      +-- VETH_INFO_PEER (nested)
          +-- struct ifinfomsg { ... }    <- peer info hdr
          +-- IFLA_IFNAME = name2         <- peer ifname
          +-- IFLA_NUM_RX_QUEUES = n_rxqueues  (optional)
          +-- IFLA_NUM_TX_QUEUES = n_txqueues  (optional)
          +-- IFLA_MTU = mtu              (optional)
          +-- IFLA_NET_NS_PID = pid       (optional, create directly in target netns)
  IFLA_IFNAME = name1                    <- local ifname
  IFLA_NUM_RX_QUEUES = n_rxqueues        (optional)
  IFLA_NUM_TX_QUEUES = n_txqueues        (optional)
```

#### 创建 macvlan（`lxc_macvlan_create`）

```
RTM_NEWLINK | NLM_F_REQUEST | NLM_F_CREATE | NLM_F_EXCL | NLM_F_ACK

payload: struct ifinfomsg { ifi_family = AF_UNSPEC }

attributes:
  IFLA_IFNAME = name
  IFLA_LINK = if_nametoindex(parent)     <- parent if
  IFLA_LINKINFO (nested)
  +-- IFLA_INFO_KIND = "macvlan"
  +-- IFLA_INFO_DATA (nested)
      +-- IFLA_MACVLAN_MODE = mode       <- bridge/vepa/private/passthru
```

#### 创建 ipvlan

```
RTM_NEWLINK | NLM_F_REQUEST | NLM_F_CREATE | NLM_F_EXCL | NLM_F_ACK

payload: struct ifinfomsg { ifi_family = AF_UNSPEC }

attributes:
  IFLA_IFNAME = name
  IFLA_LINK = if_nametoindex(parent)
  IFLA_LINKINFO (nested)
  +-- IFLA_INFO_KIND = "ipvlan"
  +-- IFLA_INFO_DATA (nested)
      +-- IFLA_IPVLAN_MODE = mode        <- l2/l3/l3s
      +-- IFLA_IPVLAN_ISOLATION = isolation  (optional)
```

#### 重命名接口（`lxc_netdev_rename_by_index`，`network.c:2034-2071`）

```
RTM_NEWLINK | NLM_F_REQUEST | NLM_F_ACK

payload: struct ifinfomsg { ifi_family = AF_UNSPEC, ifi_index = ifindex }

attributes:
  IFLA_IFNAME = newname
```

#### 启用/关闭接口（`netdev_set_flag`，`network.c:2089-2129`）

```
RTM_NEWLINK | NLM_F_REQUEST | NLM_F_ACK

payload: struct ifinfomsg {
    ifi_family = AF_UNSPEC
    ifi_index = ifindex
    ifi_change = IFF_UP       <- change mask
    ifi_flags = IFF_UP / 0    <- UP or DOWN
}
```

#### 设置 MTU（`lxc_netdev_set_mtu`，`network.c:2320-2359`）

```
RTM_NEWLINK | NLM_F_REQUEST | NLM_F_ACK

payload: struct ifinfomsg { ifi_family = AF_UNSPEC, ifi_index = ifindex }

attributes:
  IFLA_MTU = mtu
```

#### 移动接口到其他 netns

```
RTM_NEWLINK | NLM_F_REQUEST | NLM_F_ACK

payload: struct ifinfomsg { ifi_family = AF_UNSPEC, ifi_index = ifindex }

attributes:
  IFLA_NET_NS_PID = pid       <- move by PID
  or
  IFLA_NET_NS_FD = fd         <- move by fd
  IFLA_IFNAME = newname       <- optional, rename on move
```

#### 添加 IP 地址（`ip_addr_add`，`network.c:2738-2790`）

```
RTM_NEWADDR | NLM_F_REQUEST | NLM_F_ACK | NLM_F_CREATE | NLM_F_EXCL

payload: struct ifaddrmsg {
    ifa_family = AF_INET / AF_INET6
    ifa_prefixlen = prefix
    ifa_index = ifindex
    ifa_scope = 0
}

attributes:
  IFA_LOCAL = addr
  IFA_ADDRESS = addr
  IFA_BROADCAST = bcast       <- IPv4 only
```

#### 添加路由（`lxc_ip_route_dest`，`network.c:123-168`）

```
RTM_NEWROUTE | NLM_F_REQUEST | NLM_F_ACK | NLM_F_CREATE | NLM_F_EXCL

payload: struct rtmsg {
    rtm_family = family
    rtm_table = RT_TABLE_MAIN
    rtm_scope = RT_SCOPE_LINK
    rtm_protocol = RTPROT_BOOT
    rtm_type = RTN_UNICAST
    rtm_dst_len = netmask
}

attributes:
  RTA_DST = dest_addr
  RTA_OIF = ifindex
```

#### 添加邻居代理（`lxc_ip_neigh_proxy`，`network.c:261-299`）

```
RTM_NEWNEIGH | NLM_F_REQUEST | NLM_F_ACK | NLM_F_CREATE | NLM_F_EXCL

payload: struct ndmsg {
    ndm_ifindex = ifindex
    ndm_flags = NTF_PROXY
    ndm_type = NDA_DST
    ndm_family = AF_INET / AF_INET6
}

attributes:
  NDA_DST = dest_addr
```

#### 桥端口 VLAN 配置（`lxc_bridge_vlan_add`，`network.c:324-375`）

```
RTM_SETLINK | NLM_F_REQUEST | NLM_F_ACK

payload: struct ifinfomsg {
    ifi_family = AF_BRIDGE
    ifi_index = ifindex
}

attributes:
  IFLA_AF_SPEC (nested)
  +-- IFLA_BRIDGE_FLAGS = BRIDGE_FLAGS_MASTER
  +-- IFLA_BRIDGE_VLAN_INFO = {
        vid = vlan_id,
        flags = BRIDGE_VLAN_INFO_PVID | BRIDGE_VLAN_INFO_UNTAGGED  (untagged)
              or 0  (tagged)
      }
```

#### 桥接口附加（`lxc_bridge_attach`，`network.c:3111-3145`）

> **注意**：桥接口附加使用 `ioctl`，不是 netlink：

```c
ioctl(SIOCBRADDIF, &ifr)     // ifr.ifr_ifindex = 目标接口 ifindex
```

OVS 桥则调用 `lxc_ovs_attach_bridge()` 执行 `ovs-vsctl` 命令。
