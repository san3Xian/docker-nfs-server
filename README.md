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

### 方式 5：Kubernetes 中让 NFSv3 和 NFSv4 使用相同挂载路径

如果你希望客户端无论走 `NFSv3` 还是 `NFSv4`，都统一挂载：

```bash
server:/exports/model
```

那么在 Kubernetes 里，更稳的做法是：

- 用一个单独的 `emptyDir` 作为 `NFSv4` pseudo-root
- 把同一个 PVC 挂两次
- 一次挂到 `NFSv4` 根下面，给 `NFSv4` 看
- 一次挂到真实路径 `/exports`，给 `NFSv3` 看

示例片段如下：

```yaml
hostNetwork: true
containers:
  - name: nfs-server
    image: ghcr.io/san3xian/docker-nfs-server:latest
    imagePullPolicy: IfNotPresent
    securityContext:
      privileged: true
      capabilities:
        add:
          - SYS_ADMIN
    env:
      - name: NFS_EXPORT_0
        value: /nfs *(ro,sync,no_subtree_check,no_root_squash,fsid=0,crossmnt)
      - name: NFS_EXPORT_1
        value: /nfs/exports *(rw,sync,no_subtree_check,no_root_squash)
      - name: NFS_EXPORT_2
        value: /exports *(rw,sync,no_subtree_check,no_root_squash)
      - name: MOUNTD_PORT
        value: "20048"
    ports:
      - containerPort: 2049
        name: nfs-tcp
        protocol: TCP
      - containerPort: 2049
        name: nfs-udp
        protocol: UDP
      - containerPort: 111
        name: rpcbind-tcp
        protocol: TCP
      - containerPort: 111
        name: rpcbind-udp
        protocol: UDP
      - containerPort: 20048
        name: mountd-tcp
        protocol: TCP
    volumeMounts:
      - name: nfsv4-pseudo-root
        mountPath: /nfs
        readOnly: true
      - name: nfs-storage
        mountPath: /nfs/exports
      - name: nfs-storage
        mountPath: /exports
volumes:
  - name: nfsv4-pseudo-root
    emptyDir:
      sizeLimit: 128Mi
  - name: nfs-storage
    persistentVolumeClaim:
      claimName: nfsdata
```

这套配置要求 `nfsdata` 这个 PVC 的根目录下直接就有 `model` 目录。这样：

- `NFSv3` 客户端挂载 `server:/exports/model`
- `NFSv4` 客户端挂载 `server:/exports/model`

两边最终访问的是同一份数据。

路径映射关系如下：

```txt
NFSv4 fsid=0 根         /nfs
NFSv4 可见路径          /exports/model
服务端真实路径          /nfs/exports/model
NFSv3 真实导出路径      /exports/model
```

#### 为什么 Kubernetes 里通常要这样做

一般来说，Pod 里的容器根文件系统通常是 `overlayfs`。所以如果你只是把一个卷挂到：

```txt
/nfs/data
```

然后再尝试把：

```txt
/nfs
```

作为 `fsid=0` 导出，往往会有问题。因为这时：

- `/nfs/data` 是 PVC
- 但 `/nfs` 本身仍然是容器根文件系统上的目录
- 它很可能就是 `overlayfs`
- `overlayfs` 往往不能作为稳定的 NFS 导出根(会遇到not support NFS export的错误)

所以 Kubernetes 里更常见、更稳的做法就是：

- 单独准备一个 `emptyDir` 或其他可写目录作为 pseudo-root
- 再把真正的数据卷挂到 pseudo-root 下面

#### 不用 `emptyDir` 也可以，但前提要看清楚

如果你不想用 `emptyDir`，也可以把一个可导出的卷直接挂到 `fsid=0` 根本身，例如 `/data`。但这里有一个很容易写错的点：

- 如果 `fsid=0` 是 `/data`
- 那么 `NFSv4` 客户端看到的是“相对于 `/data` 的路径”
- 不是服务器上的绝对路径 `/data/...`

也就是说，如果你只是把卷挂到 `/data`，然后期望客户端通过 `NFSv4` 挂载：

```txt
server:/data/nfs
```

这通常是不成立的；对 `NFSv4` 来说，更自然的可见路径会是：

```txt
server:/nfs
```

如果你真的想在“不用 `emptyDir`”的情况下，仍然让 `NFSv3` 和 `NFSv4` 都统一成：

```txt
server:/exports/nfs
```

那你仍然需要同时满足两件事：

- `fsid=0` 根下面存在 `/exports/nfs`
- `NFSv3` 视角下也存在真实导出路径 `/exports/nfs`

也就是说，本质上还是要为 `NFSv4` 的 pseudo-root 和 `NFSv3` 的真实导出路径同时准备好对应的目录结构，而不是只改一个挂载点名字就够了。

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

这通常不是服务没起来，也不一定是 `rpc.nfsd` 参数有问题，更常见的原因是 `NFSv3` 和 `NFSv4` 的挂载路径语义本来就不同。

可以先把结论记住：

- `NFSv3` 更接近“按导出路径直接挂”
- `NFSv4` 不是按 `mountd` 看到的“真实导出路径”来挂
- `NFSv4` 走的是内核 `nfsd` 提供的 pseudo-root
- 客户端看到的 `:/data`，其实是“相对于 `fsid=0` 根的路径”

#### 为什么 `NFSv4` 会和 `NFSv3` 不一样

官方 `exports(5)` 里明确说明：`NFSv4` 有一个特殊的根文件系统，也就是 “distinguished filesystem”，需要用 `fsid=root` 或 `fsid=0` 指定。客户端访问的路径，都是相对于这个根来的。

同时，`rpc.mountd(8)` 也明确说明：`NFSv4` 不使用单独的 `MOUNT` 协议，挂载动作是通过普通 NFS 请求直接交给内核里的 `nfsd` 处理的。

这两点叠在一起，就导致：

1. `showmount -e` 看到的内容，更多反映的是 `mountd` / `MOUNT` 协议这一侧的信息
1. `NFSv4` 客户端真正挂载时，走的却是内核 `nfsd` 的 pseudo-root 视角
1. 所以“`showmount -e` 正常”并不等于“`NFSv4` 的挂载路径一定写对了”

参考：

- `exports(5)`：`NFSv4` 需要 `fsid=root` / `fsid=0`  
  https://man7.org/linux/man-pages/man5/exports.5.html
- `rpc.mountd(8)`：`NFSv4` 挂载不使用单独的 `MOUNT` 协议  
  https://man7.org/linux/man-pages/man8/mountd.8.html

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

所以这时：

- 客户端挂载 `127.0.0.1:/data`
- 实际访问的是服务器里的 `/exports/data`
- 不是服务器里的 `/data`

这就是最容易混淆的地方。

如果你只是导出了：

```bash
-e NFS_EXPORT_0='/data *(rw,...)'
```

那么：

- 对 `NFSv3` 来说，客户端挂载 `127.0.0.1:/data` 往往没问题
- 对 `NFSv4` 来说，客户端挂载 `127.0.0.1:/data` 就不一定成立

因为这时候你并没有明确告诉 `NFSv4`：“哪一个导出应该作为 pseudo-root 的根”。

#### 用一句话理解这个现象

`NFSv3` 更像是在问服务器：“把你导出的 `/data` 给我挂过来。”

`NFSv4` 更像是在问服务器：“从你的 `fsid=0` 根开始，给我找一个叫 `/data` 的路径。”

如果你的 `fsid=0` 根本不是 `/`，或者根本没有正确设置，那么 `NFSv4` 客户端请求 `:/data` 时，就会收到：

```txt
No such file or directory
```

#### 对应到实际排查，通常可以这样理解

1. `showmount -e` 能看到 `/data`
   这只能说明 `NFSv3` / `mountd` 这一侧看起来是通的
1. `mount -t nfs -o nfsvers=3 server:/data /mnt` 成功
   说明按 `NFSv3` 语义，导出本身基本没问题
1. `mount -t nfs -o nfsvers=4.1 server:/data /mnt` 失败
   往往说明失败点在 `NFSv4` 的 pseudo-root 路径映射，而不是服务进程没启动

#### 该怎么改

常见做法有两种：

1. 把真实目录直接当成 `NFSv4` 根

   例如：

   ```bash
   -e NFS_EXPORT_0='/data *(rw,...,fsid=0)'
   ```

   这时客户端应该挂载：

   ```bash
   server:/
   ```

1. 显式构造一个 `NFSv4` 根，再把数据目录放到它下面

   例如：

   ```bash
   -e NFS_EXPORT_0='/exports *(ro,fsid=0,crossmnt,no_subtree_check)'
   -e NFS_EXPORT_1='/exports/data *(rw,...)'
   ```

   这时客户端挂载：

   ```bash
   server:/data
   ```

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
