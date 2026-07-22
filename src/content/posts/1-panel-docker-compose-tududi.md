---
author: Luoverse
description: 基于 1Panel 与 Docker Compose 极速部署轻量级待办清单 Tududi
draft: false
pubDatetime: '2026-07-22'
tags:
- Blog
title: 基于 1Panel 与 Docker Compose 极速部署轻量级待办清单 Tududi
---
Tududi 是一款美观、清爽且易于自托管的轻量级个人/团队任务管理及知识库应用。本教程将介绍如何在 **1Panel** 面板环境下，使用 **Docker Compose** 快速搭建并配置 Nginx 反向代理与 HTTPS，带你避开容器权限和端口映射的“常见坑点”。

---

## 🚀 项目特性

- **技术栈轻量**：前端采用 React + TypeScript，后端使用 Express + Sequelize，默认使用 SQLite 数据库。
- **资源占用极低**：RAM 占用约 100MB - 150MB，CPU 仅在请求处理时有微量开销。
- **功能完备**：支持 Area -> Project -> Task -> Subtask 的多层级管理；内置 Markdown 笔记、便签系统、Telegram 机器人同步创建任务等。

---

## 🛠️ 部署前置准备

1. **服务器环境**：已安装 Docker 和 1Panel 面板的 Linux VPS。
1. **网络与域名**：提前准备好一个子域名（例如 `todo.yourdomain.com`），并将解析指向你的 VPS IP。
1. **1Panel 基础服务**：确保已在 1Panel 应用商店安装并运行了 **OpenResty (Nginx)**。

---

## 📝 详细部署步骤

### 第一步：创建项目目录与分配权限

Tududi 容器默认运行在非 root 用户（uid `1000`）下，因此如果挂载的本地目录由 root 用户创建，会导致 SQLite 数据库写入失败（报错 `SQLITE_BUSY: database is locked` 或 `Permission Denied`）。

在服务器终端执行以下命令创建目录并赋予相应权限：

```
# 创建项目存储根目录及数据库、上传文件夹
mkdir -p /opt/1panel/docker/compose/tududi/db /opt/1panel/docker/compose/tududi/uploads

# 将目录所有者改为容器内对应的应用用户（uid 1000）
chown -R 1000:1000 /opt/1panel/docker/compose/tududi/db /opt/1panel/docker/compose/tududi/uploads

```

### 第二步：编写 Docker Compose 配置文件

在项目根目录 `/opt/1panel/docker/compose/tududi/` 下，创建 `docker-compose.yml` 配置文件：

```
version: '3.8'

services:
  tududi:
    image: chrisvel/tududi:latest
    container_name: tududi
    restart: unless-stopped
    ports:
      - "9292:3002"  # 关键：将宿主机的 9292 端口映射到容器内服务实际监听的 3002 端口
    environment:
      # 初始化管理员账号
      - TUDUDI_USER_EMAIL=admin@yourdomain.com
      - TUDUDI_USER_PASSWORD=your-secure-password
      
      # 会话加密密钥（建议使用 openssl rand -hex 64 生成一串强随机字符）
      - TUDUDI_SESSION_SECRET=your-random-64-character-session-secret
      
      # 是否启用 CalDAV 同步，默认设为 false，如需同步可设为 true 并补全加密 Key
      - CALDAV_ENABLED=false
    volumes:
      - ./db:/app/db
      - ./uploads:/app/uploads

```

Important

这里的 `ports` 必须是 `"9292:3002"`。Tududi 容器内部固定在 `3002` 端口监听，不要配置成 `"9292:9292"`，否则将无法建立连接。

### 第三步：启动容器项目

在配置文件所在目录下运行命令启动容器：

```
docker compose up -d

```

你可以通过 `docker logs -f tududi` 查看控制台输出。当看到以下日志时，说明项目已成功初始化完毕：

```
Database connection successful
Running database migrations...
Migrations completed successfully
Server running on port 3002
Server listening on http://localhost:3002

```

---

## 🌐 1Panel 反向代理与 HTTPS 配置

为了使用准备好的域名访问，我们需要在 1Panel 中将流量反向代理到本地的 `9292` 端口。

1. **创建反代规则**：
   - 登录 1Panel 后台，进入 **【网站】** -> **【创建网站】**。
   - 选择 **【反向代理】**。
   - **主域名** 输入：`todo.yourdomain.com`。
   - **代理地址** 输入：`127.0.0.1:9292`。
   - 点击确定。
1. **配置 SSL 安全证书**：
   - 创建完成后，点击该网站右侧的 **【配置】**。
   - 切换到 **【HTTPS】** 标签页。
   - 申请/选择你的 SSL 证书并启用，建议开启 **【强制 HTTPS】** 确保通信安全。

---

## 🔒 生产环境安全建议

1. **应用内修改密码**：首次使用 `admin@yourdomain.com` 和你在环境变量中设置的密码登录后，应立即在系统设置（User Settings）中修改为一个更为安全且不暴露在 `docker-compose.yml` 中的密码。
1. **备份机制**：Tududi 会在每次更新和启动时在 `/app/db/` 目录下自动生成 SQLite 备份文件。建议在宿主机定时将 `/opt/1panel/docker/compose/tududi/db/` 中的数据库备份归档。
