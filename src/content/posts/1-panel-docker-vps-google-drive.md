---
author: Luoverse
description: 1Panel + Docker 环境下 VPS 备份至 Google Drive
draft: false
pubDatetime: '2026-07-22'
tags:
- Blog
title: 1Panel + Docker 环境下 VPS 备份至 Google Drive
---
本技术 WIKI 记录了将基于 1Panel 运维面板及 Docker 容器化部署的 Linux 服务器（Ubuntu/Debian），完整备份并离线传输至 Google Drive 的标准化作业流程。适用于服务器跨机房迁移、离线灾备等技术场景。

## 一、 架构与迁移逻辑简述

在容器化环境中，**“完整备份”的核心在于备份数据的“灵魂”（数据卷、配置文件、数据库），而不是“肉体”（Docker 镜像、系统底层程序）**。

- **高效性**：仅备份核心文本与源码，体积小（如几百 MB），压缩率高，传输极快。\
- **纯净性**：新服务器拉起环境时会自动从官方仓库下载全新镜像，避免了直接复制系统镜像导致的底层硬件与网络配置冲突。\

## 二、 旧服务器端：数据打包与离线准备

在提取备份前，必须使业务进入**静态（停机）状态**，防止因异步写入导致数据库或文件损坏。

### 1. 停止业务容器

使用 SSH 工具（如 Termius）登录旧服务器，运行以下命令停止所有容器：

Bash

```
docker stop $(docker ps -a -q)

```

### 2. 打包 1Panel 核心数据

1Panel 的网站源码、数据库、持久化数据卷（Volumes）及面板配置默认集中在 `/opt/1panel`。将其整体打包并使用 Gzip 算法压缩：

Bash

```
tar -czvf /root/cloudcone_mo_backup.tar.gz /opt/1panel

```

## 三、 环境部署：Rclone 安装与云盘挂载

Rclone 是 Linux 端管理网盘的行业标准命令行工具，支持在无浏览器（Headless）环境下的远程授权。

### 1. 在服务器端安装 Rclone

Bash

```
curl https://rclone.org/install.sh | sudo bash

```

### 2. 初始化配置向导

在服务器端运行以下命令进入交互式配置：

Bash

```
rclone config

```

按照以下交互指引逐步操作：

1. 输入 **`n`**（新建远程连接）。
1. 输入网盘别名，例如：**`googledrive`**。
1. 在存储类型列表中找到并输入 **`drive`**（代表 Google Drive）。
1. `client_id` 和 `client_secret`：直接**连续按两次回车**（留空使用默认值）。
1. `scope`（权限范围）：输入 **`1`**（获取完全读写权限）。
1. `service_account_file`：直接**回车留空**。
1. `Edit advanced config?`：输入 **`n`**（跳过高级配置）。
1. `Use auto config?`：**务必输入 `n`**（声明当前处于无浏览器远程服务器环境）。

### 3. 跨端无线授权（Headless Authorization）

此时，服务器终端会暂停，并提示你在一台**有浏览器**的本地电脑上获取 Token。

1. **本地电脑操作**：在你的本地电脑（Mac 或 Windows）终端中同样安装并运行：

   Bash

   ```
   rclone authorize "drive"
   
   ```

1. 本地计算机会自动弹窗并调用浏览器，登录你的 Google 账号，点击 **“允许 / 授权”**。

1. 授权成功后，本地电脑终端会打印出一串很长的 JSON 格式代码（Token）：

   JSON

   ```
   {"access_token":"xxxx","token_type":"Bearer","refresh_token":"xxxx","expiry":"2026-..."}
   
   ```

1. **回传服务器**：复制整段 JSON 代码，回到旧服务器的 SSH 终端，粘贴到 `config_token>` 提示符后，按回车。

1. `Configure this as a team drive?`：输入 **`n`**（若为个人云盘）。

1. 确认配置无误，输入 **`y`** 保存。

1. 输入 **`q`** 退出配置向导。

## 四、 核心操作：数据直传与状态校验

### 1. 启动可视进度条传输

直接将本地打包好的文件“秒传”至 Google Drive 指定的文件夹中。**必须带上 `-P` 参数以观测实时进度和健康度**：

Bash

```
rclone copy /root/cloudcone_mo_backup.tar.gz googledrive:VPSBACKUP/ -P

```

> **运维注意**：当进度达到 100% 时，由于 Google Drive 云端需要进行文件块拼接、MD5 哈希校验及目录索引更新，终端可能会在 `Transferring: ... 100%` 状态下静止 30 秒至 2 分钟。**此时切勿人为中断（Control+C），属于完全合规的收尾现象。**

### 2. 云端目录双重校验

当终端重新跳出 `root@rory:~#` 提示符后，运行以下命令列出云端文件，肉眼比对文件大小，形成闭环确认：

Bash

```
rclone lsf googledrive:VPSBACKUP/

```

## 五、 新服务器端：数据恢复与环境复活

当新机房的服务器开通并交付全新公网 IP 后，执行以下割接复活流程。

### 1. 恢复文件准备（SFTP 拖拽法）

打开 Termius 的图形化 **SFTP 面板**：

1. 左侧面板定位到 Windows 本地留存的备份文件。
1. 右侧面板连接**新服务器 IP**，进入 `/root/` 目录。
1. 将本地的 `cloudcone_mo_backup.tar.gz` 拖拽上传至新服务器。

### 2. 部署基础 1Panel 环境

在新服务器干净的系统（建议与旧系统版本一致）中，执行官方一键安装脚本（此处以 Ubuntu 为例）：

Bash

```
curl -sSL https://resource.fit2cloud.com/1panel/package/quick_start.sh -o quick_start.sh && sudo bash quick_start.sh

```

### 3. 数据完全覆盖

安装完毕后，切勿直接登录新面板。在 SSH 中强制停止新服务并用备份覆盖：

Bash

```
# 1. 停止新面板服务
1pctl stop

# 2. 解压备份并强制覆盖回系统根目录
tar -xzvf /root/cloudcone_mo_backup.tar.gz -C /

# 3. 重新拉起面板服务
1pctl start

```

> **关键结论**：覆盖完成后，**新服务器面板的登录端口、安全入口、登录用户名和密码已完全恢复为旧服务器的状态。**

### 4. 容器重建与业务上线

1. 使用原本的账号密码登录新服务器的 1Panel 网页端。
1. 进入“容器”模块，1Panel 会自动识别残留的 Compose 编排图纸，并自动从 Docker Hub 重新拉取（Pull）对应的干净镜像（包括任何大型的第三方 Agent 容器）。
1. 如果部分容器未自动运行，在列表中手动点击 **“启动”** 或 **“重建”** 即可。

## 六、 迁移善后 checklist

- **DNS 解析重定向**：立刻前往域名服务商（如 Cloudflare），将所有相关 A 记录修改为**新服务器的洛杉矶 IP**。\
- **全局硬编码替换**：如果网站代码、反向代理（OpenResty）配置文件中曾硬编码写入了旧的公网 IP，必须在面板“文件”管理中全局搜索并修改为新 IP。\
- **SSL 证书续签**：由于公网 IP 变更，进入 1Panel “证书”管理中，对现有域名的证书进行一次手动续签，防止新机房 443 端口握手失败。
