# RDK X5 / Ubuntu 上配置 x11vnc（共享桌面）并开机自启动

> 目标：在 RDK X5（Ubuntu 22.04 类似环境）上使用 **x11vnc** 共享系统真实桌面（`:0`），并实现 **断电重启 / 无显示器（无 HDMI）也能通过 VNC 连接**。
>
> 客户端可使用 RealVNC Viewer / TigerVNC Viewer 等连接 `IP:5900`。

---

## 目录

- [环境与说明](#环境与说明)
- [1. 安装 x11vnc](#1-安装-x11vnc)
- [2. 设置 VNC 密码](#2-设置-vnc-密码)
- [3. 创建 systemd 服务并配置开机自启](#3-创建-systemd-服务并配置开机自启)
- [4. 禁用可能冲突的 vncserver 服务](#4-禁用可能冲突的-vncserver-服务)
- [5. 启动并验证](#5-启动并验证)
- [6. 客户端连接方式](#6-客户端连接方式)
- [7. 加固建议（推荐）](#7-加固建议推荐)
- [8. 常见问题排查](#8-常见问题排查)
- [9. 安全建议](#9-安全建议)

---

## 环境与说明

- 系统：Ubuntu 22.04 / arm64（RDK X5 常见）
- 方案：`x11vnc` 共享 **真实桌面会话**（显示 `:0`）
- 端口：默认 `5900`
- 适合场景：
  - 远程看到的就是板子本机桌面
  - 希望断电重启后自动可连
  - 甚至不接显示器也能连（取决于系统/X 会话策略，本方案包含常用无屏兼容处理）

---

## 1. 安装 x11vnc

```bash
sudo apt update
sudo apt install -y x11vnc

sudo mkdir -p /etc/vnc
sudo x11vnc -storepasswd /etc/vnc/passwd
sudo chmod 600 /etc/vnc/passwd

