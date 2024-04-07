---
title: 镜像迁移
date: 2024-04-07 10:52:35
tags:
    - docker
    - harbor
categories:
  - [运维]
---

迁移镜像有三种场景 ：

1. **在线迁移**：通常是有一台机器能同时连通源镜像仓库和目标镜像仓库就算在线迁移
2. **离线迁移**：只要源镜像仓库和目标镜像仓库无法在一台机器上同时联通就算离线迁移
3. **备份**：没错，备份也是一种迁移

## 前置准备

迁移之前首先需要有一份迁移的目标，这里我会以部署 k8s 所需要的容器为例。首先需要获取一份所有的镜像清单 `images.list`，这里通过 kubespray 的[离线脚本](https://github.com/kubernetes-sigs/kubespray/blob/release-2.24/contrib/offline/generate_list.sh)自动生成。

**另外需要提前声明一点**

**以下的所有迁移，包括备份，都是以一次性迁移所有架构的镜像为目标。**


```shell
$ cat images.list
docker.io/mirantis/k8s-netchecker-server:v1.2.2
docker.io/mirantis/k8s-netchecker-agent:v1.2.2
quay.io/coreos/etcd:v3.5.10
quay.io/cilium/cilium:v1.13.4
quay.io/cilium/operator:v1.13.4
quay.io/cilium/hubble-relay:v1.13.4
quay.io/cilium/certgen:v0.1.8
quay.io/cilium/hubble-ui:v0.11.0
quay.io/cilium/hubble-ui-backend:v0.11.0
docker.io/envoyproxy/envoy:v1.22.5
ghcr.io/k8snetworkplumbingwg/multus-cni:v3.8
docker.io/flannel/flannel:v0.22.0
docker.io/flannel/flannel-cni-plugin:v1.1.2
quay.io/calico/node:v3.26.4
quay.io/calico/cni:v3.26.4
quay.io/calico/pod2daemon-flexvol:v3.26.4
quay.io/calico/kube-controllers:v3.26.4
quay.io/calico/typha:v3.26.4
quay.io/calico/apiserver:v3.26.4
docker.io/weaveworks/weave-kube:2.8.1
docker.io/weaveworks/weave-npc:2.8.1
docker.io/kubeovn/kube-ovn:v1.11.5
docker.io/cloudnativelabs/kube-router:v2.0.0
registry.k8s.io/pause:3.9
ghcr.io/kube-vip/kube-vip:v0.5.12
docker.io/library/nginx:1.25.2-alpine
docker.io/library/haproxy:2.8.2-alpine
registry.k8s.io/coredns/coredns:v1.10.1
registry.k8s.io/dns/k8s-dns-node-cache:1.22.28
registry.k8s.io/cpa/cluster-proportional-autoscaler:v1.8.8
docker.io/library/registry:2.8.1
registry.k8s.io/metrics-server/metrics-server:v0.6.4
registry.k8s.io/sig-storage/local-volume-provisioner:v2.5.0
quay.io/external_storage/cephfs-provisioner:v2.1.0-k8s1.11
quay.io/external_storage/rbd-provisioner:v2.1.1-k8s1.11
docker.io/rancher/local-path-provisioner:v0.0.24
registry.k8s.io/ingress-nginx/controller:v1.9.4
docker.io/amazon/aws-alb-ingress-controller:v1.1.9
quay.io/jetstack/cert-manager-controller:v1.13.2
quay.io/jetstack/cert-manager-cainjector:v1.13.2
quay.io/jetstack/cert-manager-webhook:v1.13.2
registry.k8s.io/sig-storage/csi-attacher:v3.3.0
registry.k8s.io/sig-storage/csi-provisioner:v3.0.0
registry.k8s.io/sig-storage/csi-snapshotter:v5.0.0
registry.k8s.io/sig-storage/snapshot-controller:v4.2.1
registry.k8s.io/sig-storage/csi-resizer:v1.3.0
registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.4.0
docker.io/k8scloudprovider/cinder-csi-plugin:v1.22.0
docker.io/amazon/aws-ebs-csi-driver:v0.5.0
docker.io/kubernetesui/dashboard:v2.7.0
docker.io/kubernetesui/metrics-scraper:v1.0.8
quay.io/metallb/speaker:v0.13.9
quay.io/metallb/controller:v0.13.9
registry.k8s.io/kube-apiserver:v1.28.6
registry.k8s.io/kube-controller-manager:v1.28.6
registry.k8s.io/kube-scheduler:v1.28.6
registry.k8s.io/kube-proxy:v1.28.6
```

## 在线迁移

在线迁移是最简单的场景，首先我们确认源镜像，这里已经准备好了，就是上一步中生成的 `images.list`，其次我们确定目标镜像仓库，迁移目标都是按具体需求来定，这里我们以 `uhub.service.ucloud.cn/k8s-use/` 为例。

在线迁移最简单的工具就是 `skopeo`，安装可以参考[官方文档](https://github.com/containers/skopeo/blob/main/install.md)，这里推荐使用包管理器安装，如果包管理器默认版本太老或者包管理器版本有 bug，就需要[编译安装](https://github.com/containers/skopeo/blob/main/install.md#building-from-source)。

安装完 `skopeo` 之后，一个 for 循环即可解决。

```shell
for src in $(cat images.list); do
    image=${src#*/}
    dest="uhub.service.ucloud.cn/k8s-use/${image}"
    skopeo --insecure-policy copy --all  docker://${src} docker://${dest}
done
```

这里用到的两个参数：

`--insecure-policy` 是 `skopeo` 的参数，因为我本地没有配置 `policy`，必须要手动指定一个。

`--all` 是 `skopeo copy` 的参数，作用是迁移所有架构的镜像。如果需要迁移非当前系统架构，则需要使用参数 `--override-os`,`--override-arch`来指定。例如: `--override-os linux --override-arch arm64`。注意，只要不是使用 `--all` 参数，都只支持一种架构，要么是当前默认使用当前系统的架构，要么使用通过 `--override-*` 所指定的架构。只同步部分架构的需求，skoepo官方[尚未实现，也不在规划中](https://github.com/containers/skopeo/issues/1694)，如果有这种需求的话目前还只能通过自己编写 manifest 实现。

迁移到其他自建的 harbor 也都是一样的操作，但是如果你的 harbor 证书不可信的话还需要在 `copy` 命令后面加上 `--dest-tls-verify=false` 来关闭 tls 验证，如果源仓库的证书也不可信的话还需要使用`--src-tls-verify=false`。

如果是迁移到其他云服务商的话，还需要注意一个三级路径的问题，多数云服务商的镜像仓库都需要企业版才提供多级路径的功能。所以要么用企业版，要么就需要对镜像重命名。以上面的 for 循环为例，可以把所有的 `/` 都替换为 `-`：`image=$(echo ${src}| sed 's#^[^/]*/##;s/\//-/g')`。

另外，在线迁移同样可以使用 `skopeo sync` 来解决，为了能多水点字，sync 就放到下个场景介绍了。

## 离线迁移

在私有化场景中通常都是需要离线同步镜像的，第一步，需要将镜像转成文件，然后再传入离线环境中再导入，这里同样可以用 `skopeo`。

`skopeo` 的安装方法同在线迁移场景中一样，这里不多缀叙。离线迁移用到的是 `skopeo sync` 命令。离线迁移的步骤会稍微复杂一点，所以我们先写一个脚本：

```bash
#!/bin/bash

save_images() {
  local image_file=$1
  local src
  for src in $(cat ${image_file}); do
      image=${src#*/}
      [[ "$image" =~ "/" ]] && image_path=${image%%/*} || image_path="root"
      skopeo --insecure-policy sync --all --src docker --dest dir ${src} images/${image_path}
  done
}


load_images() {
  local image_file=$1
  local src
  dest_repo=$2
  for src in $(cat ${image_file}); do
    image=${src#*/}
    [[ "$image" =~ "/" ]] && image_path=${image%%/*} || image_path="root"
    [[ "$image_path" == "root" ]] && unset dest_name || dest_name=${image_path}
    skopeo --insecure-policy sync --all --src dir --dest docker images/${image_path} ${dest_repo}/${dest_name}
  done
}

case "$1" in
  "save")
    shift
    save_images "$@"
    ;;
  "load")
    shift
    load_images "$@"
    ;;
  *)
    echo "Invalid command: $1" >&2
    exit 1
    ;;
esac

```

这个脚本也很简单，只有两个函数，一个 `save_images()`,一个 `load_images()`,首先我们需要做的就是将镜像下载到本地，我们将脚本保存为 `images.sh` 并且赋予可执行权限，然后执行以下命令：

```shell
$ ./images.sh save images.list

# 等待执行完成之后即同步成功了，下载下来的目录如下
$ tree -d
.
└── images
    ├── amazon
    │   ├── aws-alb-ingress-controller:v1.1.9
    │   └── aws-ebs-csi-driver:v0.5.0
    ├── calico
    │   ├── apiserver:v3.26.4
    │   ├── cni:v3.26.4
    │   ├── kube-controllers:v3.26.4
    │   ├── node:v3.26.4
    │   ├── pod2daemon-flexvol:v3.26.4
    │   └── typha:v3.26.4
    ├── cilium
    │   ├── certgen:v0.1.8
    │   ├── cilium:v1.13.4
    │   ├── hubble-relay:v1.13.4
    │   ├── hubble-ui-backend:v0.11.0
    │   ├── hubble-ui:v0.11.0
    │   └── operator:v1.13.4
    ├── cloudnativelabs
    │   └── kube-router:v2.0.0
    ├── coredns
    │   └── coredns:v1.10.1
    ├── coreos
    │   └── etcd:v3.5.10
    ├── cpa
    │   └── cluster-proportional-autoscaler:v1.8.8
    ├── dns
    │   └── k8s-dns-node-cache:1.22.28
    ├── envoyproxy
    │   └── envoy:v1.22.5
    ├── external_storage
    │   ├── cephfs-provisioner:v2.1.0-k8s1.11
    │   └── rbd-provisioner:v2.1.1-k8s1.11
    ├── flannel
    │   ├── flannel-cni-plugin:v1.1.2
    │   └── flannel:v0.22.0
    ├── ingress-nginx
    │   └── controller:v1.9.4
    ├── jetstack
    │   ├── cert-manager-cainjector:v1.13.2
    │   ├── cert-manager-controller:v1.13.2
    │   └── cert-manager-webhook:v1.13.2
    ├── k8scloudprovider
    │   └── cinder-csi-plugin:v1.22.0
    ├── k8snetworkplumbingwg
    │   └── multus-cni:v3.8
    ├── kubeovn
    │   └── kube-ovn:v1.11.5
    ├── kubernetesui
    │   ├── dashboard:v2.7.0
    │   └── metrics-scraper:v1.0.8
    ├── kube-vip
    │   └── kube-vip:v0.5.12
    ├── library
    │   ├── haproxy:2.8.2-alpine
    │   ├── nginx:1.25.2-alpine
    │   └── registry:2.8.1
    ├── metallb
    │   ├── controller:v0.13.9
    │   └── speaker:v0.13.9
    ├── metrics-server
    │   └── metrics-server:v0.6.4
    ├── mirantis
    │   ├── k8s-netchecker-agent:v1.2.2
    │   └── k8s-netchecker-server:v1.2.2
    ├── rancher
    │   └── local-path-provisioner:v0.0.24
    ├── root
    │   ├── kube-apiserver:v1.28.6
    │   ├── kube-controller-manager:v1.28.6
    │   ├── kube-proxy:v1.28.6
    │   ├── kube-scheduler:v1.28.6
    │   └── pause:3.9
    ├── sig-storage
    │   ├── csi-attacher:v3.3.0
    │   ├── csi-node-driver-registrar:v2.4.0
    │   ├── csi-provisioner:v3.0.0
    │   ├── csi-resizer:v1.3.0
    │   ├── csi-snapshotter:v5.0.0
    │   ├── local-volume-provisioner:v2.5.0
    │   └── snapshot-controller:v4.2.1
    └── weaveworks
        ├── weave-kube:2.8.1
        └── weave-npc:2.8.1

85 directories

```

下载之后就可以进行推送了，执行以下命令：

```shell

$ ./images.sh load images.list uhub.service.ucloud.cn/k8s-use

# 注意，这里的 uhub.service.ucloud.cn/k8s-use 仅是我的示例
# 等待执行完毕之后，即代表推送成功

```

打开推送的目标仓库进行验证。

![uhub](https://oem-1251690846.cos.ap-guangzhou.myqcloud.com/img/uhub.png)

## 备份

备份其实是一种很小众的场景，某种意义上来说上面的离线迁移中，将目标镜像全部存到本地就是一种备份了，所以这种常规备份也没有单独介绍的必要了，这里讲一下 `harbor` 的迁移时将目标全部备份到本地的操作。

首先放脚本：

```bash
#!/bin/bash

username="yourname"
password="yourpassword"
harbor_url="harbor.your.url"

images_dir="images"
images_list="images.list"

project_images_url="https://${harbor_url}/api/v2.0/repositories?page=1&page_size=100"
images=$(curl -k -s -X GET -H "Authorization: Basic $(echo -n "$username:$password" | base64)" "$project_images_url" | jq -r '.[] | .name')

mkdir -p "$images_dir"
>"$images_dir/$images_list"

cd "$images_dir"
for image in $images; do
    project_name=$(echo ${image} | sed 's/\/.*//')
    repository_name=$(echo ${image} | sed 's/^[^\/]*\///')
    repository_name_recode=$(echo ${repository_name} | sed 's/\//%252F/g')
    artifacts_url="https://${harbor_url}/api/v2.0/projects/${project_name}/repositories/${repository_name_recode}/artifacts?page=1&page_size=100&with_tag=true"
    tags=$(curl -k -s -X GET -H "Authorization: Basic $(echo -n "$username:$password" | base64)" "${artifacts_url}" | jq -r '.[].tags[].name')
    for tag in $tags; do
        image_name=${harbor_url}/${project_name}/${repository_name}:${tag}
        docker pull ${image_name}
        image_arch=$(docker inspect ${image_name} | jq -r '.[0].Architecture')
        docker save ${image_name} >$(echo ${image_name} | awk '{gsub(/[.\/:]/,"_")}1').${image_arch}.tar.xz
        echo ${image_name} >>images.list
    done
done

cd -
tar -c --use-compress-program="xz -9 -T0" -f images.tar.xz images/

```

简单讲一下备份的思路，首先通过 api 获取所有的仓库，然后遍历所有仓库的 tag，arch，按照最小单位 arch 进行下载镜像打包，最后压缩一下。

这里之所以没有用上一个场景离线迁移中使用的 `skopeo sync` ，其实是因为最开始的时候项目被推着上线，就直接弄了一个 arm64 harbor ，一个 x86 harbor。等演示完毕之后有喘息时间了我突然想要把两个 harbor 合成一个，于是就开始这个场景的操作，先将 x86 harbor 的镜像统一备份下来，然后再构建 manifest 往 arm64 的 harbor 上推。这个场景小众到几乎不可能重复，但是这个脚本甚至是我整篇博客最先完成的部分，我几乎是为了这碟醋包了盘饺子🌚，所以还是必须水出来了。

正好也为下一篇水一下如何手动构建 manifest 做一个铺垫，完美！
