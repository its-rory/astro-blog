---
title: 媒体自动化监控、转码与重命名服务架构与运维指南
description: FileBrowser无法识别Heic文件，本脚本实现自动监控特定文件夹，自动将Heic文件转码，源文件归档
pubDatetime: '2026-09-02'
author: Luoverse
featured: false
draft: false
tags: []
---
本文档记录了基于&nbsp;Linux&nbsp;`inotify`&nbsp;机制实现的媒体自动化处理服务（`auto-rename.service`）。该服务用于在用户通过&nbsp;WebDAV（iOS&nbsp;原生文件管理/FE&nbsp;File&nbsp;Explorer）或&nbsp;FileBrowser&nbsp;Quantum&nbsp;网页端上传文件时，自动完成格式检测、HEIC&nbsp;转码、原图归档隔离以及按父目录序列化重命名。

## 1.&nbsp;架构总览

整个处理链路与系统依赖关系如下：

```
[iOS 原生文件/WebDAV/Web 上传]
            │
            ▼
   /opt/my_files/storage/（递归监听）
            │
            ├─► 触发 Linux inotify 事件 (on_closed / on_moved / on_created)
            │
            ▼
    [文件状态锁定检测: wait_for_upload_finish]
            │  (通过 lsof 确认无写入句柄，且文件大小连续稳定 > 2s)
            ▼
      [媒体格式判断]
      ├─► 静态 HEIC  ──► [Pillow-HEIF] ──► 转换为高质量 JPG
      ├─► 实况 MOV   ──► [FFmpeg]      ──► 提取剥离为 MOV
      └─► 常规文件   ──► 直接进入排号流程
            │
            ▼
     [原文件隔离归档]
     原 HEIC 移入: /opt/my_files/storage/temp/
            │
            ▼
     [序列化重命名]
     提取直接父文件夹名称，以 `文件夹名_序号.ext` 递增命名（如 Test_01.jpg）
            │
            ▼
     [权限主动放行: fix_perms]
     重置 UID:GID 为 1000:1000，赋予 777 读写删权限


```

---

## 2. 核心技术选型与避坑记录

{% table %}
- **核心组件**
- **选用方案**
- **曾遇问题与技术根因**
---
- **文件系统监听**
- Python `watchdog`
- 仅监听 `on_closed` 会遗漏客户端临时缓存后重命名的事件；扩展覆盖 `on_moved` 与 `on_created` 实现全兼容。
---
- **上传并发保护**
- `lsof` + 文件体积轮询对比
- iOS/WebDAV 上传大文件存在延时，直接转码会因文件未写完触发截断错误。引入双重锁确保写入完成后再处理。
---
- **HEIC 静态图转码**
- `pillow-heif` (Python)
- Ubuntu 24.04 预编译 FFmpeg 未开启 `--enable-libheif`，会将单张静态 HEIC 当作视频流解析，报 `moov atom not found`。必须改用专用的静态图像解码器。
---
- **视频流剥离**
- `ffmpeg -c copy`
- iOS 实况照片（Live Photo）的伴生视频容器保留用 FFmpeg 无损抽取，速度极快。
---
- **权限控制**
- Root 运行 + 代码级 `umask(0)` / `chown(1000:1000)`
- 避免 Systemd 以纯数字 `User=1000` 运行时触发 `status=217/USER` 退出，同时避免 root 生成的文件导致 FileBrowser（UID 1000）无权删除。
{% /table %}

---

## 3. 环境准备与依赖安装

在 Ubuntu 24.04 宿主机上执行以下命令安装依赖：

```
# 1. 更新系统包并安装基础工具
sudo apt-get update
sudo apt-get install -y ffmpeg lsof python3-pip python3-watchdog

# 2. 安装 Python 高保真图像解码库
pip3 install pillow pillow-heif --break-system-packages


```

---

## 4. 守护脚本完整源码

脚本存放路径：`/opt/scripts/auto_rename.py`

---

```
import os
import re
import time
import shutil
import subprocess
from PIL import Image
import pillow_heif
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# 注册 HEIF 解码插件到 Pillow
pillow_heif.register_heif_opener()

# 全局权限掩码：放开文件创建默认权限
os.umask(0)

# 配置目录
WATCH_DIR = "/opt/my_files/storage"
TEMP_DIR = os.path.join(WATCH_DIR, "temp")
TARGET_UID = 1000
TARGET_GID = 1000

def fix_perms(path):
    """自动将文件或目录的属主赋予 1000:1000 并放开读写执行权限"""
    try:
        os.chown(path, TARGET_UID, TARGET_GID)
        os.chmod(path, 0o777)
    except Exception:
        pass

# 确保隔离归档目录存在
os.makedirs(TEMP_DIR, exist_ok=True)
fix_perms(TEMP_DIR)

def wait_for_upload_finish(file_path, timeout=30):
    """
    确保文件已完全上传完毕并释放写入句柄：
    1. 使用 lsof 检测是否有进程（如 Nginx/WebDAV/FileBrowser）仍占用该文件写入
    2. 持续对比文件体积，要求体积大于0且至少稳定 2 秒以上未改变
    """
    start_time = time.time()
    last_size = -1
    stable_count = 0

    while time.time() - start_time < timeout:
        if not os.path.exists(file_path):
            return False

        try:
            res = subprocess.run(['lsof', file_path], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
            if res.returncode == 0:
                time.sleep(1)
                continue
        except Exception:
            pass

        try:
            curr_size = os.path.getsize(file_path)
        except OSError:
            time.sleep(1)
            continue

        if curr_size > 0 and curr_size == last_size:
            stable_count += 1
            if stable_count >= 2:
                return True
        else:
            stable_count = 0
            last_size = curr_size

        time.sleep(1)

    return False

def get_file_mime(file_path):
    """识别容器内部真实 MIME 结构"""
    try:
        res = subprocess.run(['file', '-b', file_path], stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
        return res.stdout
    except Exception:
        return ""

def process_heic(file_path):
    """
    HEIC 处理流水线：
    - QuickTime 容器转换为 .mov
    - 静态图像转换为高保真 .jpg
    - 原图移至 temp 目录
    """
    directory, filename = os.path.split(file_path)
    base_name, _ = os.path.splitext(filename)
    mime_info = get_file_mime(file_path)

    # 分支 1：如果是 QuickTime 视频容器（Live Photo 伴生轨）
    if "QuickTime" in mime_info:
        temp_target = os.path.join(directory, f"__proc_{base_name}.mov")
        cmd = ["ffmpeg", "-v", "error", "-y", "-i", file_path, "-c", "copy", temp_target]
        try:
            subprocess.run(cmd, check=True)
            if os.path.isfile(temp_target) and os.path.getsize(temp_target) > 0:
                fix_perms(temp_target)
                archive_original(file_path, filename)
                return temp_target
        except Exception as e:
            print(f"[{time.strftime('%H:%M:%S')}] 视频剥离失败 {filename}: {e}", flush=True)
        if os.path.exists(temp_target):
            os.remove(temp_target)
        return None

    # 分支 2：标准静态 HEIC 图像
    temp_target = os.path.join(directory, f"__proc_{base_name}.jpg")
    try:
        with Image.open(file_path) as img:
            rgb_img = img.convert('RGB')
            rgb_img.save(temp_target, "JPEG", quality=95)

        if os.path.isfile(temp_target) and os.path.getsize(temp_target) > 0:
            fix_perms(temp_target)
            archive_original(file_path, filename)
            print(f"[{time.strftime('%H:%M:%S')}] HEIC 成功转为 JPG: {filename}", flush=True)
            return temp_target
        else:
            if os.path.exists(temp_target):
                os.remove(temp_target)
            return None
    except Exception as e:
        print(f"[{time.strftime('%H:%M:%S')}] HEIC 图像解码失败 {filename}: {e}", flush=True)
        if os.path.exists(temp_target):
            os.remove(temp_target)
        return None

def archive_original(file_path, filename):
    """将原图平移至 temp 目录归档，并修复所有权"""
    dest_heic = os.path.join(TEMP_DIR, filename)
    if os.path.exists(dest_heic):
        dest_heic = os.path.join(TEMP_DIR, f"{int(time.time())}_{filename}")
    shutil.move(file_path, dest_heic)
    fix_perms(dest_heic)
    print(f"[{time.strftime('%H:%M:%S')}] 原图已移至 temp: {filename} -> {os.path.basename(dest_heic)}", flush=True)

def rename_to_next_seq(file_path):
    """根据父目录名称和现有最大序号重命名文件"""
    if not os.path.exists(file_path):
        return

    directory, filename = os.path.split(file_path)
    folder_name = os.path.basename(directory)
    _, ext = os.path.splitext(filename)

    existing_files = [f for f in os.listdir(directory) if os.path.isfile(os.path.join(directory, f))]
    max_idx = 0
    pattern = rf"^{re.escape(folder_name)}_(\d+)"
    for f in existing_files:
        match = re.match(pattern, f)
        if match:
            idx = int(match.group(1))
            if idx > max_idx:
                max_idx = idx

    next_idx = max_idx + 1
    new_filename = f"{folder_name}_{next_idx:02d}{ext.lower()}"
    new_path = os.path.join(directory, new_filename)

    if not os.path.exists(new_path) and os.path.exists(file_path):
        try:
            os.rename(file_path, new_path)
            fix_perms(new_path)
            print(f"[{time.strftime('%H:%M:%S')}] 重命名完成: {filename} -> {new_filename}", flush=True)
        except Exception as e:
            print(f"重命名失败: {e}", flush=True)

class MediaAutoHandler(FileSystemEventHandler):
    def handle_event(self, file_path):
        if not os.path.isfile(file_path):
            return

        directory, filename = os.path.split(file_path)

        # 过滤机制：跳过根目录直属文件、temp 目录变更、隐藏文件及临时中间件
        if os.path.abspath(directory) == os.path.abspath(WATCH_DIR):
            return
        if os.path.commonpath([directory, TEMP_DIR]) == os.path.abspath(TEMP_DIR):
            return
        if filename.startswith('.') or filename.startswith('__tmp_') or filename.startswith('__proc_'):
            return

        folder_name = os.path.basename(directory)
        _, ext = os.path.splitext(filename)
        ext_lower = ext.lower()

        # 已经规范命名的文件忽略，避免自触发死循环
        if re.match(rf"^{re.escape(folder_name)}_(\d+){re.escape(ext)}$", filename, re.IGNORECASE):
            return

        # 等待文件完全落盘
        if not wait_for_upload_finish(file_path):
            return

        if ext_lower in ['.heic', '.heif']:
            print(f"\n[{time.strftime('%H:%M:%S')}] 捕获到 HEIC: {filename}，开始处理...", flush=True)
            conv_path = process_heic(file_path)
            if conv_path:
                rename_to_next_seq(conv_path)
        else:
            print(f"\n[{time.strftime('%H:%M:%S')}] 捕获到常规文件: {filename}，自动排号...", flush=True)
            rename_to_next_seq(file_path)

    def on_closed(self, event):
        if not event.is_directory:
            self.handle_event(event.src_path)

    def on_moved(self, event):
        if not event.is_directory:
            self.handle_event(event.dest_path)

    def on_created(self, event):
        if not event.is_directory:
            self.handle_event(event.src_path)

if __name__ == "__main__":
    event_handler = MediaAutoHandler()
    observer = Observer()
    # recursive=True 自动监听已有及未来新建的任意层级子文件夹
    observer.schedule(event_handler, path=WATCH_DIR, recursive=True)
    observer.start()
    print(f"自动化服务已启动，监控目录: {WATCH_DIR}", flush=True)

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()
```

---

## 5. Systemd 服务部署与配置

配置文件路径：[`/etc/systemd/system/auto-rename.service`](https://colab.research.google.com/drive/1vzlNCVvh5U0G_bLNaPt0e9HuHuxLJz7U#)

```
[Unit]
Description=Auto Rename and Convert Uploaded Media Service
After=network.target

[Service]
Type=simple
User=root
Group=root
WorkingDirectory=/opt/scripts
ExecStart=/usr/bin/python3 /opt/scripts/auto_rename.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target


```

### 常用管理命令：

```
# 重载服务定义
sudo systemctl daemon-reload

# 启动并设置开机自启
sudo systemctl enable --now auto-rename.service

# 重启服务
sudo systemctl restart auto-rename.service

# 查看服务状态
sudo systemctl status auto-rename.service

# 实时查看业务日志
journalctl -u auto-rename.service -f


```

---

## 6. WebDAV 客户端接入规范（iOS 端）

iOS 原生「文件」App 内置的“连接服务器”仅支持 SMB 协议。要无缝连接 FileBrowser Quantum，需借助支持 File Provider 规范的客户端（如 FE File Explorer、Documents）。

### 连接参数核对表：

- **服务器 URL**：`https://storage.luoverse.cn/dav/<source-name>/` *(注意：必须保留 `/dav/` 前缀和数据源名称，末尾带 `/`)*
- **用户名**：任意字符（服务端会自动忽略，例如填 `admin`）
- **密码**：**必须使用后台生成的 API Token**，登录密码无效 *(获取路径：FileBrowser Web 端 -> 设置 -> API Tokens -> 新建非定制 Token)*
- **集成至系统文件管理**：打开 iOS「文件」App -> 右上角 `...` -> 编辑边栏 -> 打开对应客户端开关。

---

## 7. 维护与清理建议

由于 HEIC 原图会被持续归档至 [`/opt/my_files/storage/temp`](https://colab.research.google.com/drive/1vzlNCVvh5U0G_bLNaPt0e9HuHuxLJz7U#)`/`，为防止磁盘空间耗尽，建议在 1Panel 的「计划任务」中添加定期清理 Shell 任务：

```
# 查找并删除 temp 目录下修改时间超过 30 天的原图文件
find /opt/my_files/storage/temp -type f -mtime +30 -delete

```
