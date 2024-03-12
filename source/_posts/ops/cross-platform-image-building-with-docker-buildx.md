---
title: 跨平台构建docker镜像
date: 2024-03-12 14:47:39
tags:
    - docker
    - buildx
categories:
  - [运维]
---

业务需要同时运行在 x86_64 和 arm64 两种架构的机器上，贫穷让公司选择了跨平台构建。

## 安装 buildx

buildx 是作为 docker cli 的一个插件存在的，所以如果已经在系统中配置了 docker 仓库的话，直接用系统的包管理器安装 `docker-buildx-plugin` 即可，这也是最推荐的方案。

如果非要手动安装，可以自行从 [ docker buildx github releases ](https://github.com/docker/buildx/releases/latest) 进行下载，然后拷贝到 `/usr/local/lib/docker/cli-plugins` 目录下即可。仅当前用户使用的话也可以放到 `$HOME/.docker/cli-plugins` 目录。

## 初始化构建器

docker buildx 跨平台打包镜像有两种方案，第一种使用不同架构的机器来做 worker ，但是如果有这种资源我还是比较倾向用 [kaniko](https://github.com/GoogleContainerTools/kaniko)，所以这里暂不赘叙，有需求可以看一下[官方文档](https://docs.docker.com/build/building/multi-platform/#multiple-native-nodes)。这里主要介绍一下第二种，通过 binfmt_misc 和 QEMU 来实现跨平台构建。

buildx 的跨平台构建，实际上是通过 QEMU 作为构建器后端，然后再利用 binfmt_misc 模块注册 QEMU 作为其他CPU架构可执行文件的处理程序来实现的。我们首先需要先启用 binfmt_misc，这里可以通过 [tonistiigi/binfmt](https://github.com/tonistiigi/binfmt) 这个脚本镜像直接实现：

```shell
# 这里最后的 --install all 是安装所有支持的架构，如果只有特定需求也可以手动指定
# 例如 --install arm64,riscv64,arm
$ docker run --privileged --rm tonistiigi/binfmt --install all
```

执行完毕之后，可以通过这两种方法验证是否已经成功开启 binfmt_misc ：

```shell
# 该命令可以查看当前所有可用构建器，如未启用则 PLATFORMS 应该仅支持 amd64 和 386 相关的架构
$ docker buildx ls
NAME/NODE     DRIVER/ENDPOINT   STATUS    BUILDKIT   PLATFORMS
default*      docker
 \_ default    \_ default       running   v0.12.5    linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/386, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/mips64le, linux/mips64, linux/arm/v7, linux/arm/v6


# 该命令可以查看 binfmt_misc 的开启状态
$ cat /proc/sys/fs/binfmt_misc/status
```

确认无误后，即可开始创建构建器：

```shell
$ docker buildx create --name mybuilder --bootstrap --use
```

创建完之后就有了一个可用的，名称叫 mybuilder 构建器，这里再详细介绍一些参数：

```shell
$ docker buildx create \
  --name mybuilder \
  --driver=docker-container \
  --driver-opt "network=host" \
  --config /etc/buildkitd.toml \
  --bootstrap \
  --use
```

第一个参数 --name 就不用讲了，我们从第二个开始。

> --driver=docker-container

其实在安装 docker buildx 之后，就可以通过 docker buildx ls 看到一个 default 构建器，但是这个构建器并不可用到跨平台构建上，即使启用了 binfmt_misc 。原因就是因为这个 driver。docker driver 的构建器只会有一个 default ，无法单独指定创建，不使用 buildx 的时候的 docker 操作，其实都是在使用这个构建器。

docker driver 考虑的更多的是简单和易用两个方面，而我们这里指定的 docker-container driver 比 dokcer driver 要灵活很多，可以指定 BuildKit 版本、通过 QEMU 进行跨平台构建以及更高级的缓存导出功能（ps.这里的更高级指的是相对于 docker drvier 只能将缓存导出到镜像中而言）。同时也是 `docker buildx create` 默认 driver，所以也基本不用指定。

除了 docker 和 docker-container 之外还有 kubernetes 和 remote 两个 driver，主要差异在于使用场景上，这里不多缀述，有兴趣可以看一下[官方文档](https://docs.docker.com/build/drivers/)。

> --driver-opt "network=host"

构建器实际上是打开一个 buildkit 容器进行构建，如果对这个容器有任何的配置需求，基本都是写在这个参数中，包括资源限制、启动的镜像、重启策略、环境变量以及我们这里写的网络的配置，具体可选配置可参考[该表格](https://docs.docker.com/build/drivers/docker-container/#synopsis)。

这里单独使用了一个 network=host 是因为遇到过构建时网络情况和预期不一致的情况。譬如拥有两个dns，一个用于内网一个用于外网，如果不使用 host 网络，则构建时就会使用 docker 内部 dns。作为一个构建镜像的基础服务，为了规避可能存在的干扰，还是建议使用 host 网络。

> --config /etc/buildkitd.toml

与所有服务一样的，buildkit 也是有配置文件的，具体的配置内容可以参考[官方的配置文件](https://docs.docker.com/build/buildkit/toml-configuration/)。当然了，这里的大多数配置都是无须更改的，这里单独将 config 拎出来，是因为有一种比较常见的情况是，构建的 base image 来自私有仓库，使用 http 或者自签 SSL 证书。

因为构建时并不在物理机而是在 buildkit 镜像内，所以它也没办法读 `/etc/docker/daemon.json` 的配置，必须要单独配置，例如：

```toml
[registry."192.168.189.102:5000"]
http = true
insecure = true
```

> --bootstrap --use

bootstrap 就是创建的同时启动这个构建器使用的 buildkit 容器，use 就是创建的同时切换到这个构建器。

## 开始构建

前置准备虽然漫长的，但是实践总是很简单的，首先准备一个Dockerfile：

```Dockerfile
FROM alpine:3.19
RUN apk add curl
```

然后执行构建命令：
```shell
$ docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 -t ccr.ccs.tencentyun.com/openimage/alpine:3.19 --push .

...
(以下省略一堆输出)
 => => pushing manifest for ccr.ccs.tencentyun.com/openimage/alpine:3.19@sha256:d2ca9689374079757822ac8a9bd1e298a96fd10fc7648c4a5043499f61a1e432                                                            2.1s
 => [auth] openimage/alpine:pull,push token for ccr.ccs.tencentyun.com
```

命令执行成功，构建完成，当你使用 `--push` 参数的时候，buildx 甚至自动创建了 manifest ，保证了多个架构镜像共用一个 tag，可以通过 `docker buildx imagetools inspect` 进行验证。

```shell
$ docker buildx imagetools inspect ccr.ccs.tencentyun.com/openimage/alpine:3.19

Name:      ccr.ccs.tencentyun.com/openimage/alpine:3.19
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:d2ca9689374079757822ac8a9bd1e298a96fd10fc7648c4a5043499f61a1e432

Manifests:
  Name:        ccr.ccs.tencentyun.com/openimage/alpine:3.19@sha256:dc6b015793c2ac537f61ec80348a7f34739cfa6580813f24e813d653fc37faf3
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/amd64

  Name:        ccr.ccs.tencentyun.com/openimage/alpine:3.19@sha256:a3df735a618653a5bb73c1f09fe73feb30f352c19760326155bebf29172a7de5
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/arm64

  Name:        ccr.ccs.tencentyun.com/openimage/alpine:3.19@sha256:1cf066565331d3feb598c1d7fb5a181a151bd6a965ffd891f9ab459e0a708afb
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    linux/arm/v7

  Name:        ccr.ccs.tencentyun.com/openimage/alpine:3.19@sha256:d43c80d6216bc643f283163c89b6f3d5813d02e8fa76f5a86115bcf8b16d10a2
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    unknown/unknown
  Annotations:
    vnd.docker.reference.digest: sha256:dc6b015793c2ac537f61ec80348a7f34739cfa6580813f24e813d653fc37faf3
    vnd.docker.reference.type:   attestation-manifest

  Name:        ccr.ccs.tencentyun.com/openimage/alpine:3.19@sha256:c2e29cae1de1f6d4f285ed5055c871c745862054c8693b2e4d51fa9c3cb49954
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    unknown/unknown
  Annotations:
    vnd.docker.reference.digest: sha256:a3df735a618653a5bb73c1f09fe73feb30f352c19760326155bebf29172a7de5
    vnd.docker.reference.type:   attestation-manifest

  Name:        ccr.ccs.tencentyun.com/openimage/alpine:3.19@sha256:b593bc641ba6ea8d046ff4d6e78f73ac4be1422c1ad3db0fc23dfc07950d049b
  MediaType:   application/vnd.oci.image.manifest.v1+json
  Platform:    unknown/unknown
  Annotations:
    vnd.docker.reference.digest: sha256:1cf066565331d3feb598c1d7fb5a181a151bd6a965ffd891f9ab459e0a708afb
    vnd.docker.reference.type:   attestation-manifest
```