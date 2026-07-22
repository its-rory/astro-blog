---
author: Luoverse
description: 美国 VPS 精准解锁 Gemini & Google AI 资产规范指引 (WARP 分流篇)
draft: false
pubDatetime: '2026-07-22'
tags:
- Blog
title: 美国 VPS 精准解锁 Gemini & Google AI 资产规范指引 (WARP 分流篇)
---
## 一、 系统初始化与补全基础依赖

首先升级系统软件包，并安装网络和运行所需的基础依赖。

Bash

```
# 更新本地软件包缓存并升级系统
sudo apt update && sudo apt upgrade -y

# 安装后续配置所需的全部核心依赖工具
sudo apt install curl wget socat git nano ufw lsb-release gnupg2 ca-certificates apt-transport-https software-properties-common -y

```

## 二、 修改默认 SSH 端口与防火墙安全配置

为了防止公网爆破脚本天天扫描 22 端口，将其修改为一个自定义的高位端口。

1. 打开 SSH 配置文件：

Bash

```
sudo nano /etc/ssh/sshd_config

```

2. 找到 [\#Port](#Port) 22（或 Port 22），将数字修改为你自定义的端口（例如：8956）：

```
Port 8956

```

保存并退出（快捷键：Ctrl + O \rightarrow Enter \rightarrow Ctrl + X）。

3. **必须先配置防火墙放行新端口**，然后再重启 SSH 服务，否则会被锁在系统外面：

Bash

```
# 放行你自定义的 SSH 端口（极其重要）
sudo ufw allow 8956/tcp

# 放行 Reality 伪装服务所需的公网 443 端口（由于是基于 UDP 的 QUIC 伪装，建议同时放行 TCP 和 UDP）
sudo ufw allow 443/tcp
sudo ufw allow 443/udp

# 启动防火墙（遇到提示输入 y 确认即可）
sudo ufw enable

# 重启 SSH 服务使新端口生效
sudo systemctl restart ssh

```

_提示：此时先不要关闭当前的窗口。新开一个终端窗口，尝试使用 ssh -p 8956 root@你的VPS_IP 登录，验证新端口畅通无阻后，再关闭旧窗口。

4. 请输入以下命令：

Bash

```
# 1. 彻底停止并禁用这个锁死 22 端口的托管服务
sudo systemctl stop ssh.socket
sudo systemctl disable ssh.socket

# 2. 重新加载系统守护进程
sudo systemctl daemon-reload

# 3. 重启传统的 SSH 主服务（它会去老老实实读你的 8956 端口配置）
sudo systemctl restart ssh

```

执行完这三行后，我们再来验证一下系统现在到底听谁的：

Bash

```
sudo ss -tlnp | grep ssh

```

此时你在屏幕上应该能清晰地看到输出变为了 *:8956，这就说明 22 端口已经被彻底释放，新端口 8956 正式对外营业了。_

## 三、 开启 Linux 内核 BBR 加速

对于 RackNerd 这种非优化直连线路，BBR 是对抗丢包、强行提升传输速率的绝对利器。

Bash

```
# 注入 BBR 内核参数
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf

# 重载内核参数使其立即生效
sudo sysctl -p

```

## 四、 安装 Xray 并全手动生成你的公私钥与 UUID

1. 执行 Xray 官方一键脚本安装最新内核核心：

Bash

```
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install

```

2. **【核心步骤：生成并查看你的 Reality 公私钥】**

   在终端中执行以下命令，Xray 会在屏幕上为你直接计算并打印出一对崭新的非对称加密密钥：

Bash

```
xray x25519

```

执行后，你会看到类似下面的输出：

- Private key: gA3...（这里是一串长字符串，这就是你的私钥，稍后填入服务端配置）\
- Public key: 7fB...（这里是一串长字符串，这就是你的公钥，稍后填入客户端配置） **请把这两串字符复制保存到你的本地记事本里，别混淆了。**\

3. 运行以下命令，获取一个随机生成的客户端用户 UUID：

Bash

```
xray uuid

```

*把这串 36 位的 UUID 字符串也复制保存下来。*

## 五、 配置 Xray 服务端（含 Gemini 专属分流规则）

1. 打开并编辑 Xray 的默认配置文件：

Bash

```
sudo nano /usr/local/etc/xray/config.json

```

2. **彻底清空**里面的原有内容，完整替换为以下精简、无错误的 VLESS-Reality 全套双路分流配置：

JSON

```
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "port": 443,
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "替换为你刚才生成的_UUID",
                        "flow": "xtls-rprx-vision"
                    }
                ],
                "decryption": "none"
            },
            "streamSettings": {
                "network": "tcp",
                "security": "reality",
                "realitySettings": {
                    "show": false,
                    "dest": "www.target.com:443",
                    "xver": 0,
                    "serverNames": [
                        "www.target.com"
                    ],
                    "privateKey": "替换为你刚才生成的_PrivateKey",
                    "shortIds": [
                        "a1b2c3d4e5f6"
                    ]
                }
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom",
            "tag": "direct"
        },
        {
            "protocol": "socks",
            "tag": "gemini-warp",
            "settings": {
                "servers": [
                    {
                        "address": "127.0.0.1",
                        "port": 40000
                    }
                ]
            }
        }
    ],
    "routing": {
        "domainStrategy": "AsIs",
        "rules": [
            {
                "type": "field",
                "outboundTag": "gemini-warp",
                "domain": [
                    "domain:ai.google.dev",
                    "domain:alkalimakersuite-pa.clients6.google.com",
                    "domain:makersuite.google.com",
                    "domain:bard.google.com",
                    "domain:deepmind.com",
                    "domain:deepmind.google",
                    "domain:gemini.google.com",
                    "domain:generativeai.google",
                    "domain:proactivebackend-pa.googleapis.com",
                    "domain:apis.google.com",
                    "keyword:colab",
                    "keyword:developerprofiles",
                    "keyword:generativelanguage",
                    "domain:geller-pa.googleapis.com",
                    "domain:geller-pa.clients6.google.com",
                    "domain:assistant.googleapis.com",
                    "domain:speech.googleapis.com",
                    "domain:googleapis.com",
                    "domain:notebooklm.google",
                    "keyword:notebooklm", 
                    "domain:notebooklm.google.com",
                    "domain:tunnel.google.com",
                    "domain:clients6.google.com",
                    "domain:alkalimakersuite-pa.googleapis.com",
                    "domain:one.google.com",
                    "domain:accounts.google.com",
                    "domain:ssl.gstatic.com",
                    "domain:www.gstatic.com",
                    "domain:fonts.gstatic.com",
                    "keyword:googleusercontent"
                ]
            },
            {
                "type": "field",
                "outboundTag": "direct",
                "network": "tcp,udp"
            }
        ]
    }
}

```

*注意：请仔细核对并替换第 11 行的 id（填入 UUID）和第 27 行的 privateKey（填入刚才生成的 Private key）。这里的被偷域名我们选择[www.target.com，极为安全稳定。](http://www.target.com%EF%BC%8C%E6%9E%81%E4%B8%BA%E5%AE%89%E5%85%A8%E7%A8%B3%E5%AE%9A%E3%80%82)*

保存并退出编辑器（Ctrl + O \rightarrow Enter \rightarrow Ctrl + X）。

## 六、 安装 Cloudflare WARP 并配置 Proxy 模式

配置本地无污染的 Socks5 代理管道（运行在 40000 端口），用来专门承接并洗白去往 Google Gemini 的网络请求。

Bash

```
# 引入官方 WARP 存储库并安装客户端
curl -fsSL https://pkg.cloudflareclient.com/pubkey.gpg | sudo gpg --yes --dearmor --output /usr/share/keyrings/cloudflare-warp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/cloudflare-warp-archive-keyring.gpg] https://pkg.cloudflareclient.com/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/cloudflare-client.list
sudo apt update && sudo apt install cloudflare-warp -y

# 注册并强制切换为只代理本地 40000 端口的模式（绝对不能开全局 TUN，否则远程 SSH 会失联断开）
warp-cli registration new
warp-cli mode proxy
warp-cli proxy port 40000
warp-cli connect
sudo systemctl enable warp-svc

# 验证 WARP 分流管道是否正常通畅
curl --socks5-hostname 127.0.0.1:40000 https://www.cloudflare.com/cdn-cgi/trace

```

*检查输出结果，如果看到返回的内容里带有 warp=on，说明 WARP 本地管道已经彻底打通。*

如果

```
curl --socks5-hostname 127.0.0.1:40000 https://www.cloudflare.com/cdn-cgi/trace

```

报错，尝试使用以下命令：

```
warp-cli registration delete
warp-cli registration new
warp-cli mode proxy
warp-cli connect
warp-cli proxy port 40000

```

成功后再次验证：

```
curl --socks5-hostname 127.0.0.1:40000 https://www.cloudflare.com/cdn-cgi/trace

```

## 七、 启动服务与最终校验

最后，重启 Xray 核心服务使其接管机器的 443 端口：

Bash

```
sudo systemctl restart xray
sudo systemctl enable xray

```

你可以运行以下命令检查 Xray 的运行状态，确保它没有报错崩溃：

Bash

```
sudo systemctl status xray

```

如果看到绿色的 active (running)，说明一切就绪。

如果最后一步提示：

```
/etc/systemd/system/xray.service:7: Special user nobody configured, this is not safe!

```

别慌！这是一个**系统警告（Warning）**，并不是阻碍程序运行的**致命错误（Error）**。即使看到这行提示，你的 Xray 其实也已经成功启动并正常在后台跑着了，不会影响你连接网络。

### 为什么会有这个提示？

这完全是因为你用了最新版的 **Ubuntu 24.04**。

新版 Ubuntu 引入了更严格的系统安全审计机制。Xray 官方安装脚本为了安全，默认指定了让 Xray 运行在系统自带的 nobody（无特权用户）身份下。

但是现代 Linux 安全规范认为：nobody 是一个全系统共享的兜底账户。如果很多没有特权的软件都混用 nobody，万一其中一个软件被黑了，黑客就能顺藤摸瓜去窥探其他同样运行在 nobody 下的程序。

systemd 觉得这“不够隔离”，于是强迫症发作，弹出了这个警告，意思是：*“你用通用的 nobody 风险太高啦，最好给它单独建个专属账户！”*

### 怎么解决？

既然你前面部署了 apt-fast，说明你也是个有“完美主义”的操作者，那咱们就花 10 秒钟写三行命令，给 Xray 办一个专属的身份证明，彻底堵上系统日志的嘴。

请在终端里依次复制运行以下命令：

Bash

```
# 1. 为 Xray 创建一个独立的、没有任何登录权限的专用系统用户（名叫 xray）
sudo useradd -r -s /bin/false xray

# 2. 用 sed 命令，把 Xray 服务配置文件里的 User=nobody 替换为 User=xray
sudo sed -i 's/User=nobody/User=xray/g' /etc/systemd/system/xray.service

# 3. 让系统重载配置文件，并重启 Xray 释放隐患
sudo systemctl daemon-reload
sudo systemctl restart xray

```

### 验证最终状态

搞定之后，我们来检查一下 Xray 的最新状态和日志：

Bash

```
sudo systemctl status xray

```

你会发现之前的黄色警告彻底消失了，取而代之的是纯净、绿色的 active (running)。由于 Xray 配置文件里自带了 CAP_NET_BIND_SERVICE 特权借调机制，即使换成了我们新建立的无特权独立用户 xray，它依然能稳稳地秒开 443 端口，安全系数直接拉满！

## 八、 客户端节点参数对照表

在你的手机或电脑客户端（如 v2rayN、v2rayNG、Clash Verge Rev 或 sing-box）中，手动添加一个 **VLESS** 节点。由于改为了 Reality 架构，请务必严格对照填入以下参数：

- **地址 (Address)**: 你的VPS实际公网IP地址 （不需要填域名，直接填 IP）\
- **端口 (Port)**: 443\
- **用户 ID (UUID)**: 刚才由 xray uuid 生成的那串 36 位字符串\
- **传输协议 (Network)**: tcp\
- **安全传输 (Security)**: **reality** （注意：这里从 tls 改选为 reality）\
- **流控 (Flow)**: xtls-rprx-vision\
- **SNI / 目标域名**: [www.target.com](http://www.target.com) （这里填写我们“偷”的域名）\
- **Fingerprint (指纹)**: chrome （必须伪装成主流浏览器的 TLS 握手特征）\
- **PublicKey (公钥)**: **填入你在第四步里由命令生成的那个 Public key** （这就是对暗号的关键，不能填错成私钥）\
- **ShortId**: a1b2c3d4e5f6 （需要与服务端的短 ID 一致）
