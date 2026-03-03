---
layout: post
title: Tailscale 自建 DERP 节点全教程：解决国内中继高延迟问题
date: 2026-03-03 19:58:42
tags:  
    - Tailscale
    - DERP
    - WireGuard
    - AI辅助
---

## 为什么需要 Tailscale？

AI推动了硬件销售和软件迭代，很多人家里都部署了一下自己玩的或者用的服务，然后可能有些人为了方便随时能用就直接把服务暴露在公网上，这就导致人为增加了一堆入侵点。从安全角度来说，漏洞修复永远赶不上被利用的速度。一个 0day 出来，几小时之内全网扫描工具就开始扫了。

最近的飞牛 NAS OS 被 0day 打穿，详情看[飞牛社区公告](https://club.fnnas.com/forum.php?mod=viewthread&tid=53420&extra=page%3D1)，一堆暴露在公网的设备变成了肉鸡；还有[超22万OpenClaw部署实例暴露公网](https://www.163.com/dy/article/KN3U5CL10511D3QS.html),基本是裸奔部署，甚至未启用身份验证。

所以结论很简单：**别把东西直接丢公网上**。你需要一个安全的内网隧道，让设备之间像在同一个局域网一样互通，对外面的人来说你的服务压根不存在，这样能很大程度给你增加容错空间，让你发现安全问题时能有更多时间去修复。

## 选型：为什么是 Tailscale？

市面上组网方案不少，简单列一下：

| 特性 | Tailscale | WireGuard | OpenVPN |
|------|-----------|-----------|---------|
| 配置复杂度 | ⭐ 极低 | ⭐⭐ 中等 | ⭐⭐⭐ 较高 |
| NAT 穿透 | ✅ 自动打洞 | ❌ 需手动配置 | ❌ 需中心服务器 |
| 网络拓扑 | Mesh（去中心化） | 点对点 / 星型 | 星型（中心化） |
| 性能 | 高（基于 WireGuard） | 高 | 中等 |
| 多平台支持 | 全平台 | 全平台 | 全平台 |
| MagicDNS | ✅ 内置 | ❌ 需自建 | ❌ 需自建 |
| 上手难度 | 注册即用 | 需要理解密钥交换 | 需要管理证书 |

WireGuard 性能没话说，但你得自己搞密钥交换、自己配路由、自己处理 NAT 穿透，折腾完一圈头都大了。OpenVPN 就更不用提了，证书管理那一套，光配就能劝退一半人。

Tailscale 底层就是 WireGuard，性能和加密都继承了，但上面做了一整套自动化——密钥交换、NAT 穿透、路由管理全给你包了。装上就能用，真的省心。

## 为什么要自建 DERP 节点？

Tailscale 大部分时候能直接打洞成功，两台设备点对点直连，体验很好。但总有打洞失败的时候，这时候流量就得走 **DERP（Designated Encrypted Relay for Packets）** 中继服务器绕一下。

问题来了：**官方的 DERP 节点基本全在境外**。

国内的网络环境大家都懂，中继流量绕一圈海外再回来，延迟上百毫秒是常态，还不稳定，丢包也多。这体验能好才怪。

所以如果你在国内有台服务器，自建一个 DERP 节点是很有必要的。打洞失败的时候走国内中继，延迟直接从上百毫秒降到个位数，体验天差地别。

---

## 部署环境

先交代一下本文的环境：

- **操作系统**：Debian 13 (Trixie)
- **Go 版本**：1.26.0
- **示例域名**：`derper.example.com`（请替换为你自己的域名）

> ⚠️ **注意**：本文中所有涉及域名的地方均使用 `derper.example.com` 作为示例，请在实际部署时替换为你自己的域名。

---

## 第一步：安装 Go 语言环境

derper 是 Go 写的，安装需要 Go 环境。一般来说 derper 都依赖最新版本的 Go，而各大发行版包管理器里的版本基本都是老的，所以老老实实用官方的二进制包装吧。

参考 [Go 官方安装文档](https://go.dev/doc/install)：

```bash
# 下载 Go 最新版本（以 1.26.0 为例，请根据实际情况调整版本号）
wget https://go.dev/dl/go1.26.0.linux-amd64.tar.gz

# 删除旧版本并解压新版本
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.26.0.linux-amd64.tar.gz

# 配置环境变量（添加到 ~/.bashrc 或 ~/.zshrc）
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc

# 验证安装
go version
# 输出：go version go1.26.0 linux/amd64
```

## 第二步：安装 DERP 服务端

Go 装好了，装 derper 就一条命令的事：

```bash
go install tailscale.com/cmd/derper@latest
```

> 💡 **国内加速**：国内服务器拉 Go 模块可能连不上，配个 [GOPROXY.CN](https://goproxy.cn) 代理就好了：
>
> ```bash
> export GOPROXY=https://goproxy.cn,direct
> go install tailscale.com/cmd/derper@latest
> ```

装完之后 derper 二进制文件在 `$HOME/go/bin/derper`。

## 第三步：安装 Tailscale 客户端（安全加固）

这步很关键。

默认情况下，**只要有人知道你 DERP 服务器的 IP，就可以把它加到自己的 DERP map 里，然后白嫖你的带宽做中继**。你辛辛苦苦搭的节点，结果给别人当免费中转站，想想就亏。

解决办法是在 DERP 服务器上同时跑一个 `tailscaled`，然后启动 derper 的时候加 `--verify-clients` 参数。这样 derper 会检查过来连接的客户端是不是你自己 tailnet 里的设备，不是的话直接拒绝。

装 Tailscale：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

然后加入你的 tailnet：

```bash
sudo tailscale up
```

会给你一个 URL，浏览器打开登录授权就行。去 [Tailscale 管理面板](https://login.tailscale.com/admin/machines) 看到这台机器了，就说明成了。

## 第四步：配置 TLS 证书

DERP 是走 HTTPS 的，所以需要 TLS 证书。我们用 Let's Encrypt 的免费证书，工具用 certbot。

### 安装 Certbot

```bash
sudo apt update
sudo apt install -y certbot
```

### 申请证书

先确保 `derper.example.com` 的 DNS 已经解析到这台服务器，然后：

```bash
# 使用 standalone 模式申请证书（需要临时占用 80 端口）
sudo certbot certonly --standalone -d derper.example.com
```

跟着提示填邮箱、同意条款就行。搞定后证书在这：

- 证书：`/etc/letsencrypt/live/derper.example.com/fullchain.pem`
- 私钥：`/etc/letsencrypt/live/derper.example.com/privkey.pem`

### 创建证书软链接

这里有个坑：**derper 只能指定证书目录，没法直接指定证书文件**。它会在目录里按 `<hostname>.crt` 和 `<hostname>.key` 的规则去找。但 Let's Encrypt 生成出来的叫 `fullchain.pem` 和 `privkey.pem`，名字对不上。

建个软链接就好了：

```bash
# 创建证书目录
sudo mkdir -p /var/lib/derper/certs/

# 创建软链接
sudo ln -s /etc/letsencrypt/live/derper.example.com/fullchain.pem \
    /var/lib/derper/certs/derper.example.com.crt
sudo ln -s /etc/letsencrypt/live/derper.example.com/privkey.pem \
    /var/lib/derper/certs/derper.example.com.key
```

## 第五步：启动 DERP 服务

准备工作搞定了，来启动 derper。先看下每个参数啥意思：

| 参数 | 说明 |
|------|------|
| `-hostname` | DERP 服务器的域名 |
| `-certdir` | TLS 证书所在目录 |
| `-certmode=manual` | 手动管理证书（我们用 certbot，不让 derper 自己申请） |
| `-http-port=-1` | 关掉 HTTP 端口（只用 HTTPS） |
| `-a=:4501` | HTTPS 监听端口 |
| `-stun-port=4502` | STUN 端口（辅助 NAT 穿透） |
| `-verify-clients` | 只允许你自己 tailnet 的设备用 |

启动：

```bash
/root/go/bin/derper \
    -hostname=derper.example.com \
    -certdir=/var/lib/derper/certs/ \
    -certmode=manual \
    -http-port=-1 \
    -a=:4501 \
    -stun-port=4502 \
    -verify-clients
```

### 验证

浏览器打开 `https://derper.example.com:4501`，看到下面这行字就成了：

> **This is a Tailscale DERP server.**

---

## 第六步：自动化运维

总不能一直手动跑 derper 吧，搞成系统服务，再加上自动更新。

### 创建 Systemd Service

我们使用 systemd 的模板实例功能。

创建 `/etc/systemd/system/tailscale-derp@.service`：

```ini
[Unit]
Description=Tailscale DERP Server
After=network.target tailscaled.service
Requires=tailscaled.service

[Service]
Type=simple
ExecStart=/root/go/bin/derper \
    -hostname=%i \
    -certdir=/var/lib/derper/certs/ \
    -certmode=manual \
    -http-port=-1 \
    -a=:4501 \
    -stun-port=4502 \
    -verify-clients
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

启用：

```bash
# 启动服务（将 derper.example.com 替换为你的域名）
sudo systemctl daemon-reload
sudo systemctl enable --now tailscale-derp@derper.example.com
    
# 检查运行状态
sudo systemctl status tailscale-derp@derper.example.com
```

### 创建自动更新脚本

derper 和 tailscale 都在持续更新，为了不落后版本太多，写个自动更新脚本。

创建 `/usr/local/bin/update-tailscale-derp.sh`：

```bash
#!/bin/bash
set -euo pipefail

LOG_TAG="tailscale-derp-updater"
GOPROXY="https://goproxy.cn,direct"
export GOPROXY

log() {
    logger -t "$LOG_TAG" "$@"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

update_derper() {
    log "检查 derper 更新..."
    
    local current_version
    current_version=$(/root/go/bin/derper --version 2>&1 || echo "unknown")
    
    if timeout 60 go install tailscale.com/cmd/derper@latest 2>&1; then
        local new_version
        new_version=$(/root/go/bin/derper --version 2>&1 || echo "unknown")
        
        if [ "$current_version" != "$new_version" ]; then
            log "derper 已从 $current_version 更新至 $new_version"
            return 0
        else
            log "derper 已是最新版本: $current_version"
        fi
    else
        log "derper 更新失败"
    fi
    return 1
}

update_tailscale() {
    log "检查 Tailscale 更新..."
    
    local current_version
    current_version=$(tailscale version 2>&1 | head -1)
    
    if timeout 60 bash -c 'apt-get update && apt-get install -y --only-upgrade tailscale' 2>&1; then
        local new_version
        new_version=$(tailscale version 2>&1 | head -1)
        
        if [ "$current_version" != "$new_version" ]; then
            log "Tailscale 已从 $current_version 更新至 $new_version"
            return 0
        else
            log "Tailscale 已是最新版本: $current_version"
        fi
    else
        log "Tailscale 更新失败"
    fi
    return 1
}

main() {
    log "===== 开始自动更新 ====="
    
    if update_derper; then
        log "重启 derper 服务..."
        systemctl restart 'tailscale-derp@*'
    fi
    
    if update_tailscale; then
        log "重启 tailscaled 服务..."
        systemctl restart tailscaled
    fi
    
    log "===== 自动更新完成 ====="
}

main "$@"
```

赋予执行权限：

```bash
sudo chmod +x /usr/local/bin/update-tailscale-derp.sh
```

### 配置定时任务

用 systemd timer 每天凌晨 4 点跑一次更新。

创建 `/etc/systemd/system/update-tailscale-derp.service`：

```ini
[Unit]
Description=Update Tailscale and DERP Server

[Service]
Type=oneshot
ExecStart=/usr/local/bin/update-tailscale-derp.sh
Environment="PATH=/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="HOME=/root"
```


创建 `/etc/systemd/system/update-tailscale-derp.timer`：

```ini
[Unit]
Description=Daily Update for Tailscale and DERP Server

[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=false
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now update-tailscale-derp.timer

# 验证定时器状态
sudo systemctl list-timers update-tailscale-derp.timer
```

---

## 防火墙配置

别忘了放行端口：

```bash
# 如果使用 ufw
sudo ufw allow 4501/tcp   # HTTPS (DERP)
sudo ufw allow 4502/udp   # STUN

# 如果使用 iptables
sudo iptables -A INPUT -p tcp --dport 4501 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 4502 -j ACCEPT
```

## 在 Tailscale 中配置自定义 DERP 节点

DERP 跑起来了还不够，得告诉 Tailscale 它的存在。

去 [Tailscale ACL 配置页面](https://login.tailscale.com/admin/acls)，加上 `derpMap`：

```json
{
    "derpMap": {
        "OmitDefaultRegions": false,
        "Regions": {
            "900": {
                "RegionID": 900,
                "RegionCode": "myderp",
                "RegionName": "My DERP Server",
                "Nodes": [
                    {
                        "Name": "myderp-1",
                        "RegionID": 900,
                        "HostName": "derper.example.com",
                        "DERPPort": 4501,
                        "STUNPort": 4502
                    }
                ]
            }
        }
    }
}
```

`OmitDefaultRegions` 设成 `false`，保留官方节点当备选。`RegionID` 用 900 以上的数字，别跟官方的撞了。

保存后在任意客户端上跑一下：

```bash
tailscale netcheck
```

能看到你的自定义节点和延迟信息就说明配置生效了。

---

## 未来展望

说实话现在这套部署流程还是偏复杂了——装 Go、编译、搞证书、写一堆 systemd 文件，步骤不少。理想情况应该是 Docker 一把梭，再丢到 Nginx 后面，对外只开 80 和 443。

但目前 Tailscale 官方不太推荐反代 derper，因为 DERP 协议涉及 WebSocket 升级和长连接，反代容易出问题。所以这事暂时先放着，后面有新进展再更新。

