# docker-nfs-server

一个轻量 NFS Server 容器镜像，基于上游 `erichough/nfs-server` 调整。

- 镜像地址：`ghcr.io/san3xian/docker-nfs-server`
- 自动发布：`push`、手动触发 GitHub Actions、发布 Release
- 支持架构：`linux/amd64`、`linux/arm64`

## 这个镜像适合什么场景

- 想快速起一个容器化 NFS Server
- 需要支持 `NFSv3`、`NFSv4`，或者同时支持两者
- 希望主要通过环境变量配置导出目录，而不是手改容器内文件

## 前置条件

1. 宿主机内核需要支持并加载这些模块：
   `nfs`、`nfsd`

   如果要用 Kerberos，还需要：
   `rpcsec_gss_krb5`

2. 容器需要额外权限，至少要有：
   `--cap-add SYS_ADMIN`

   实际使用里更省事的方式通常是：
   `--privileged`

3. 容器内部必须能看到你要导出的目录。
   常见做法就是把宿主机目录通过 `-v` 挂进去。

4. 本 README 下面的命令示例使用 `nerdctl`，如果你用的是 `docker`，命令格式基本一样。

## 快速开始

### 方式 1：默认同时支持 NFSv3 和 NFSv4

如果你想先用最少配置把服务跑起来，同时让服务端默认提供 `NFSv3` 和 `NFSv4`，可以直接这样启动：

```bash
nerdctl run --net host --privileged \
  -v /data:/data \
  -e NFS_EXPORT_0='/data *(rw,sync,no_root_squash,all_squash,anonuid=0,anongid=0,no_subtree_check,fsid=0)' \
  ghcr.io/san3xian/docker-nfs-server
```

这里没有额外设置 `NFS_VERSION` 或 `NFS_DISABLE_VERSION_3`，所以会按默认行为同时提供 `NFSv3` 和 `NFSv4`。

客户端挂载方式：

```bash
# NFSv3
mount -t nfs -o nfsvers=3 127.0.0.1:/data /mnt

# NFSv4
mount -t nfs -o nfsvers=4.1 127.0.0.1:/ /mnt
```

注意：这套配置里，`/data` 被作为 `NFSv4` 根导出，所以 `NFSv3` 和 `NFSv4` 的客户端挂载路径不一样：

- `NFSv3` 挂 `server:/data`
- `NFSv4` 挂 `server:/`

如果你希望 `NFSv4` 客户端也挂载 `server:/data`，看下面的方式 3。

### 方式 2：只提供 NFSv3，最简单

如果你只需要 `NFSv3`，并且希望客户端直接挂载 `server:/data`，这是最直观的方式：

```bash
nerdctl run --net host --privileged \
  -v /data:/data \
  -e NFS_VERSION=3 \
  -e NFS_EXPORT_0='/data *(rw,sync,no_root_squash,all_squash,anonuid=0,anongid=0,no_subtree_check)' \
  ghcr.io/san3xian/docker-nfs-server
```

客户端挂载：

```bash
mount -t nfs -o nfsvers=3 127.0.0.1:/data /mnt
```

### 方式 3：只提供 NFSv4，并保留客户端挂载路径 `/data`

如果你希望客户端继续挂载：

```bash
127.0.0.1:/data
```

那么推荐显式构造一个 `NFSv4` 根：

```bash
nerdctl run --net host --privileged \
  -v /data:/exports/data \
  -e NFS_DISABLE_VERSION_3=1 \
  -e NFS_EXPORT_0='/exports *(ro,fsid=0,crossmnt,no_subtree_check)' \
  -e NFS_EXPORT_1='/exports/data *(rw,sync,no_root_squash,all_squash,anonuid=0,anongid=0,no_subtree_check)' \
  ghcr.io/san3xian/docker-nfs-server
```

客户端挂载：

```bash
mount -t nfs -o nfsvers=4.1 127.0.0.1:/data /mnt
```

### 方式 4：只提供 NFSv4，把 `/data` 直接当根

如果你可以接受客户端挂载 `server:/`，配置会更短：

```bash
nerdctl run --net host --privileged \
  -v /data:/data \
  -e NFS_DISABLE_VERSION_3=1 \
  -e NFS_EXPORT_0='/data *(rw,sync,no_root_squash,all_squash,anonuid=0,anongid=0,no_subtree_check,fsid=0)' \
  ghcr.io/san3xian/docker-nfs-server
```

客户端挂载：

```bash
mount -t nfs -o nfsvers=4.1 127.0.0.1:/ /mnt
```

## 常用环境变量

| 变量名 | 作用 |
| --- | --- |
| `NFS_EXPORT_0`, `NFS_EXPORT_1`... | 每个变量对应一行 `/etc/exports` |
| `NFS_VERSION` | 指定协议版本，可用值：`3`、`4`、`4.1`、`4.2` |
| `NFS_DISABLE_VERSION_3` | 设为非空值后禁用 `NFSv3` |
| `NFS_LOG_LEVEL` | `INFO` 或 `DEBUG` |
| `NFS_SERVER_THREAD_COUNT` | 指定 `rpc.nfsd` 线程数 |

## FAQ

### 为什么 `showmount -e` 正常，通过NFS v3 也能挂载，但 `NFSv4` 挂载 `:/data` 失败？(提示: mounting 127.0.0.1:/xxxx failed, reason given by server: No such file or directory)

这是 `NFSv3` 和 `NFSv4` 的路径语义不同，不一定是服务没起来。

- `NFSv3` 更接近“按导出路径直接挂”
- `NFSv4` 走的是 pseudo-root，客户端看到的路径是相对于 `fsid=0` 根来的

例如下面这组导出：

```bash
-e NFS_EXPORT_0='/exports *(ro,fsid=0,crossmnt,no_subtree_check)'
-e NFS_EXPORT_1='/exports/data *(rw,sync,no_root_squash,all_squash,anonuid=0,anongid=0,no_subtree_check)'
```

客户端看到的路径关系其实是：

```txt
服务器真实路径            客户端看到的路径
/exports                  /
/exports/data             /data
```

所以客户端挂载 `127.0.0.1:/data` 时，实际访问的是服务器里的 `/exports/data`，不是 `/data`。

如果你只是导出了：

```bash
-e NFS_EXPORT_0='/data *(rw,...)'
```

那它对 `NFSv3` 通常没问题，但对 `NFSv4` 不一定成立。

PS: 根据man page说明，showmount 对 NFSv4 的支持也不太完善，所以它的输出不一定能反映实际的导出状态。
> BUGS  
> 
>       The completeness and accuracy of the information that showmount
>       displays varies according to the NFS server's implementation.
>
>       Because showmount sorts and uniqs the output, it is impossible to
>       determine from the output whether a client is mounting the same
>       directory more than once.
>
>       showmount works by contacting the server's MNT service directly.
>       NFSv4-only servers have no need to advertise their exported root
>       filehandles via this method, and may not expose their MNT service
>       to clients.
也就是说，在 NFSv4-only 的情况下，showmount 可能无法列出任何导出目录。

### 为什么会报 `exportfs: /nfs does not support NFS export`？

这通常不是目录名的问题，而是这个目录所在的底层文件系统不支持被 NFS 导出。

最常见的容器场景是：

```bash
nerdctl run --net host --privileged \
  -v /data:/nfs/data \
  -e NFS_EXPORT_0='/nfs *(ro,fsid=0,crossmnt,no_subtree_check)' \
  -e NFS_EXPORT_1='/nfs/data *(rw,sync,no_root_squash,all_squash,anonuid=0,anongid=0,no_subtree_check)' \
  ghcr.io/san3xian/docker-nfs-server
```

这里：

- `/nfs/data` 是宿主机挂进来的目录
- `/nfs` 本身往往还是容器根文件系统上的目录
- 容器根文件系统在很多运行时里通常是 `overlayfs`

而 `overlayfs` 往往不能直接作为 NFS 导出根，所以 `exportfs` 会报：

```txt
exportfs: /nfs does not support NFS export
```

要点是：不是“有的目录名不行”，而是“有的目录所在文件系统不支持导出”。

你可以在容器里这样检查：

```bash
stat -f -c %T /nfs
stat -f -c %T /nfs/data
```

常见结果会是：

- `/nfs` 是 `overlayfs`
- `/nfs/data` 是宿主机上的本地文件系统，例如 `ext4`、`xfs`、`btrfs`

#### 解决方法 1：把真正的数据目录直接作为 `NFSv4` 根

如果你能接受客户端挂载 `server:/`，最简单的做法就是不要导出 `/nfs`，而是直接导出 `/nfs/data`：

```bash
nerdctl run --net host --privileged \
  -v /data:/nfs/data \
  -e NFS_DISABLE_VERSION_3=1 \
  -e NFS_EXPORT_0='/nfs/data *(rw,sync,no_root_squash,all_squash,anonuid=0,anongid=0,no_subtree_check,fsid=0)' \
  ghcr.io/san3xian/docker-nfs-server
```

客户端挂载：

```bash
mount -t nfs -o nfsvers=4.1 127.0.0.1:/ /mnt
```

#### 解决方法 2：如果你想让客户端继续挂载 `:/data`

那就要保证 `fsid=0` 的父目录本身也来自一个可导出的宿主机文件系统，而不是容器里的 `overlayfs`：

```bash
mkdir -p /srv/nfs-root

nerdctl run --net host --privileged \
  -v /srv/nfs-root:/nfs \
  -v /data:/nfs/data \
  -e NFS_DISABLE_VERSION_3=1 \
  -e NFS_EXPORT_0='/nfs *(ro,fsid=0,crossmnt,no_subtree_check)' \
  -e NFS_EXPORT_1='/nfs/data *(rw,sync,no_root_squash,all_squash,anonuid=0,anongid=0,no_subtree_check)' \
  ghcr.io/san3xian/docker-nfs-server
```

这样客户端就可以继续挂载：

```bash
mount -t nfs -o nfsvers=4.1 127.0.0.1:/data /mnt
```

前提是 `/srv/nfs-root` 自己也在可导出的本地文件系统上。

#### 解决方法 3：如果你只需要 `NFSv3`

那就不必强行构造 `NFSv4` 根，直接导出实际目录会更简单：

```bash
nerdctl run --net host --privileged \
  -v /data:/data \
  -e NFS_VERSION=3 \
  -e NFS_EXPORT_0='/data *(rw,sync,no_root_squash,all_squash,anonuid=0,anongid=0,no_subtree_check)' \
  ghcr.io/san3xian/docker-nfs-server
```

### 为什么这里大多示例都用了 `--net host`？

因为这样最省事，尤其是 `NFSv3`。

如果不用 `host network`，你就需要自己处理端口映射。`NFSv4` 相对简单，通常至少要开放 TCP `2049`；`NFSv3` 还会涉及 `rpcbind`、`mountd`、`statd` 等附加端口。对大多数排障场景来说，先用 `--net host` 更直接。

## 更多文档

- [日志与调试](doc/feature/logging.md)
- [自动加载内核模块](doc/feature/auto-load-kernel-modules.md)
- [Kerberos](doc/feature/kerberos.md)
- [NFSv4 用户 ID 映射](doc/feature/nfs4-user-id-mapping.md)
- [AppArmor](doc/feature/apparmor.md)
- [自定义 NFS 版本](doc/advanced/nfs-versions.md)
- [自定义端口](doc/advanced/ports.md)
- [性能调优](doc/advanced/performance-tuning.md)

## 本地构建

```bash
nerdctl build -t docker-nfs-server:test .
```

## 致谢

- [f-u-z-z-l-e/docker-nfs-server](https://github.com/f-u-z-z-l-e/docker-nfs-server)
- [sjiveson/nfs-server-alpine](https://github.com/sjiveson/nfs-server-alpine)
- [erichough/nfs-server](https://github.com/ehough/docker-nfs-server)
