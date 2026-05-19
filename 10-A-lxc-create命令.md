# LXC 命令深度分析：lxc-create

## 1. 概述

`lxc-create` 用于创建一个新的 LXC 容器，包括：创建目录结构、初始化存储后端、运行模板脚本填充 rootfs、生成配置文件。

**典型用法**：
```bash
lxc-create -t download -n mycontainer
lxc-create -t download -n mycontainer -B btrfs -- -d ubuntu -r jammy -a amd64
lxc-create -t local -n mycontainer -- -m /path/to/metadata -f /path/to/rootfs.tar.xz
```

---

## 2. CLI 入口

**文件**：`src/lxc/tools/lxc_create.c:199-334`

### 2.1 参数解析

```c
lxc_create_main()                               [lxc_create.c:199]
  +-- lxc_arguments_parse(&my_args, argc, argv)
  `-- my_parser() handles create-specific options [lxc_create.c:95-139]
```

| 选项 | 含义 |
|------|------|
| `-t <template>` | 模板名（如 download, local, busybox） |
| `-n <name>` | 容器名 |
| `-B <backend>` | 存储后端（dir, btrfs, lvm, zfs, loop, overlay） |
| `-f <config>` | 指定配置文件 |
| `--fstype` | 文件系统类型（用于 loop 后端） |
| `--fssize` | 文件系统大小 |
| `--lvname` | LVM 逻辑卷名 |
| `--vgname` | LVM 卷组名 |
| `--thinpool` | LVM 精简池名 |
| `--zfsroot` | ZFS 数据集根路径 |
| `--dir` | 自定义 rootfs 目录 |
| `--` | 之后的参数传递给模板脚本 |

### 2.2 模板名规范化

```c
// lxc_create.c:220-249
if template == "none":   不运行模板
if template == "_unset":  报错，必须指定 -t
if template == "best":   使用最佳可用后端
```

### 2.3 构建 bdev_specs

```c
// lxc_create.c:280-318
struct bdev_specs spec = {
    .fstype  = my_args.fstype,
    .fssize  = my_args.fssize,
    .dir     = my_args.dir,
    .lvm.lv  = my_args.lvname,
    .lvm.vg  = my_args.vgname,
    .lvm.thinpool = my_args.thinpool,
    .zfs.zfsroot  = my_args.zfsroot,
};
```

### 2.4 调用创建 API

```c
// lxc_create.c:326-330
c->create(c, my_args.template, my_args.bdevtype, &spec, flags, &argv[optind])
```

---

## 3. API 层

**文件**：`src/lxc/lxccontainer.c:1755-1928`

### 3.1 入口函数

```
lxcapi_create() / lxcapi_createl()              [lxccontainer.c:2136-2165]
  └── __lxcapi_create()                          [lxccontainer.c:1755-1920]
```

`lxcapi_createl()` 将可变参数转换为 `argv` 数组后调用 `__lxcapi_create()`。

### 3.2 创建主流程

```
__lxcapi_create(c, t, bdevtype, specs, flags, argv)
  |
  +--1-- create container directory
  |      create_container_dir(c)                [lxccontainer.c:1201-1218]
  |      -> mkdir(<lxcpath>/<name>)
  |
  +--2-- load default config (if not specified)
  |      c->load_config(c, lxc_global_config_value("lxc.default_conf"))
  |                                              [lxccontainer.c:1784-1787]
  |
  +--3-- create partial marker file
  |      create_partial(c)                       [lxccontainer.c:1828-1833]
  |      -> write a temp marker in the container dir to prevent concurrent create
  |
  +--4-- fork child to create storage
  |      pid = fork()                            [lxccontainer.c:1845-1873]
  |      child:
  |        +-- if ID mapping exists, switch to mapped identity
  |        `-- storage_create(c, bdevtype, specs)
  |      parent: wait for child exit
  |
  +--5-- reload config
  |      c->load_config(c, fname)                [lxccontainer.c:1878-1883]
  |
  +--6-- run template script
  |      create_run_template(c, t, !!(flags & LXC_CREATE_QUIET), argv)
  |                                              [lxccontainer.c:1885-1886]
  |
  +--7-- write config header
  |      prepend_lxc_header(c->configfile, tpath, argv)
  |                                              [lxccontainer.c:1893-1898]
  |      -> insert template info and SHA1 checksum at file start
  |
  +--8-- final config reload
  |      c->load_config(c, fname)                [lxccontainer.c:1900]
  |
  `--9-- remove partial marker
         remove_partial(c)                       [lxccontainer.c:1902]
         -> container creation complete
```

### 3.3 错误处理

创建失败时的清理（`lxccontainer.c:1907-1918`）：

```c
if (创建失败 && 存储是新创建的) {
    c->destroy(c);  // 清理已创建的存储和目录
}
```

---

## 4. 存储后端创建

**文件**：`src/lxc/storage/storage.c:170-280`

### 4.1 后端分发

```
storage_create(c, bdevtype, specs)               [storage.c:258-280]
  +-- choose backend type:
  |   +-- bdevtype == NULL -> "best" (auto-pick best)
  |   +-- bdevtype == "best" -> try the btrfs -> dir fallback chain
  |   `-- explicit type -> use directly
  +-- bdev = storage_init(conf)                  -> create backend instance
  `-- bdev->ops->create(bdev, name, specs)       -> call the concrete backend
```

### 4.2 各后端实现

| 后端 | 文件 | 创建操作 |
|------|------|----------|
| **dir** | `storage/dir.c:57-93` | `mkdir(rootfs_path)`，设置 `bdev->src = "dir:<path>"` |
| **btrfs** | `storage/btrfs.c:907+` | 创建 btrfs 子卷 |
| **lvm** | `storage/lvm.c:95-170` | 运行 `lvcreate`（支持普通卷和精简池） |
| **zfs** | `storage/zfs.c:688+` | 创建 ZFS 数据集 |
| **loop** | `storage/loop.c:113-184` | 创建磁盘镜像文件 + mkfs + losetup |
| **overlay** | `storage/overlay.c:253+` | 创建 overlay 元数据目录结构 |

### 4.3 dir 后端详解

最常用的后端，流程最简单：

```c
dir_create(bdev, name, specs)                    [dir.c:57-93]
  +-- determine rootfs path: <lxcpath>/<name>/rootfs
  +-- mkdir_p(rootfs_path, 0755)
  +-- bdev->src = strdup("dir:<rootfs_path>")
  `-- bdev->dest = strdup(rootfs_path)
```

### 4.4 lvm 后端详解

```c
do_lvm_create(path, size, thinpool)              [lvm.c:95-169]
  +-- parse /dev/<vg>/<lv> path
  +-- if thinpool:
  |   `-- execlp("lvcreate", "--thinpool", thinpool, "-V", size, vg, "-n", lv)
  `-- else:
      `-- execlp("lvcreate", "-L", size, vg, "-n", lv)
```

---

## 5. 模板脚本执行

**文件**：`src/lxc/lxccontainer.c:1288-1605`

### 5.1 模板调用流程

```
create_run_template(c, tpath, quiet, argv)       [lxccontainer.c:1288-1605]
  +-- fork()
  |   `-- child:
  |       +-- build template args:
  |       |   template --path=<lxcpath>/<name>
  |       |            --name=<name>
  |       |            --rootfs=<rootfs>
  |       |            [user custom args]
  |       |                                  [lxccontainer.c:1390-1449]
  |       |
  |       +-- if user namespace mapping exists:
  |       |   wrap with lxc-usernsexec
  |       |   lxc-usernsexec --mapped-uid <uid>
  |       |                  --mapped-gid <gid>
  |       |                  -- template [args...]
  |       |                                  [lxccontainer.c:1450-1592]
  |       |
  |       `-- execvp(template, argv)          [lxccontainer.c:1594-1596]
  |
  `-- parent: waitpid() for template completion
```

### 5.2 模板脚本接口

模板脚本（shell 或任意可执行程序）接收以下标准参数：

| 参数 | 含义 |
|------|------|
| `--path` | 容器配置目录（`<lxcpath>/<name>`） |
| `--name` | 容器名 |
| `--rootfs` | rootfs 路径 |
| `--mapped-uid` | UID 映射（用户命名空间场景） |
| `--mapped-gid` | GID 映射 |
| 用户参数 | `--` 之后的所有参数 |

### 5.3 download 模板

**文件**：`templates/lxc-download.in:1-501`

```
lxc-download
  +-- parse args (-d distro, -r release, -a arch) [140-165]
  +-- download image cache                         [304-405]
  |   +-- fetch index from the image server
  |   +-- download the rootfs tarball
  |   `-- extract into $LXC_ROOTFS
  +-- generate container config                    [407-450]
  |   `-- write lxc.uts.name = $LXC_NAME
  `-- adjust ownership (ID-mapped case)           [452-493]
```

---

## 6. 配置头部写入

**文件**：`src/lxc/lxccontainer.c:1608-1727`

```
prepend_lxc_header(configfile, tpath, argv)      [lxccontainer.c:1681-1701]
  +-- calculate config file SHA1 checksum
  +-- insert a comment header at file start:
  |     # Template used to create this container: <template>
  |     # Parameters passed to template: <args>
  |     # Template script checksum (SHA-1): <sha1>
  |     # For additional config options, please look at lxc.container.conf(5)
  `-- append original config content
```

---

## 7. 完整流程图

```
lxc-create -t download -n mycontainer -B dir -- -d ubuntu -r jammy -a amd64
  |
  +-- CLI parse -> template=download, name=mycontainer, bdevtype=dir
  |                                              [lxc_create.c:199-334]
  |
  +-- c->create(c, "download", "dir", &spec, flags, ["-d","ubuntu","-r","jammy","-a","amd64"])
  |                                              [lxccontainer.c:1755]
  |
  +-- create /var/lib/lxc/mycontainer/           [lxccontainer.c:1201]
  |
  +-- load /etc/lxc/default.conf                 [lxccontainer.c:1784]
  |
  +-- create partial marker                      [lxccontainer.c:1828]
  |
  +-- fork -> storage_create("dir")
  |   `-- mkdir /var/lib/lxc/mycontainer/rootfs  [dir.c:57]
  |
  +-- fork -> run template
  |   `-- execvp("lxc-download",
  |              "--path=/var/lib/lxc/mycontainer",
  |              "--name=mycontainer",
  |              "--rootfs=/var/lib/lxc/mycontainer/rootfs",
  |              "--", "-d", "ubuntu", "-r", "jammy", "-a", "amd64")
  |       -> download image -> extract to rootfs -> write config
  |                                              [lxc-download.in:304-501]
  |
  +-- prepend_lxc_header() -> write template info to config header
  |                                              [lxccontainer.c:1681]
  |
  +-- reload final config
  |
  `-- remove partial marker -> container ready
```
