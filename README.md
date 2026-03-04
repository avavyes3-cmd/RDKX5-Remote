# RDK X5 VNC（x11vnc）从 0 配置到开机自启动完整教程

本教程适用于 RDK X5（Ubuntu 22.04 类似环境），使用 **x11vnc** 共享 **真实桌面会话**（显示 `:0`），实现：

- 开机自动启动 VNC 服务（systemd）
- 断电重启后仍可远程连接
- 不接显示器（无 HDMI）也可连接（依赖系统/X 会话策略，本方案包含常见无屏兼容处理）
- 端口固定为 `5900`
- 客户端使用 RealVNC Viewer / TigerVNC Viewer 等均可

---

## 目录

- [01-目标](#01-目标)
- [02-准备](#02-准备)
- [03-安装](#03-安装)
- [04-设置密码](#04-设置密码)
- [05-创建服务](#05-创建服务)
- [06-启用自动启动](#06-启用自动启动)
- [07-禁用冲突服务](#07-禁用冲突服务)
- [08-验证](#08-验证)
- [09-客户端连接](#09-客户端连接)
- [10-加固建议](#10-加固建议)
- [11-排查](#11-排查)
- [12-安全](#12-安全)
- [13-附录](#13-附录)

---

## 01-目标

我们要配置的是：

- **服务端（板子上）**：`x11vnc`（共享真实桌面 `:0`）
- **客户端（电脑上）**：VNC Viewer（RealVNC / TigerVNC / TightVNC 均可）
- **连接方式**：`<板子IP>:5900` 或 `<板子IP>:0`

> 注意：VNC 协议必须有客户端连接服务端。你电脑上用 VNC Viewer 是正常且推荐的方式。

## 02-准备

### 2.1 确认系统版本（可选）

```bash
lsb_release -a
uname -a
```

### 2.2 确认网络与 IP

```bash
ip a
```

记下板子 IP，例如：`192.168.8.109`

### 2.3 确认 5900 端口未被占用（可选）

```bash
sudo ss -lntp | grep 5900 || true
```

若已有程序占用，后续需要先停掉占用者。

## 03-安装

安装 `x11vnc`：

```bash
sudo apt update
sudo apt install -y x11vnc
```

确认安装成功：

```bash
which x11vnc
x11vnc -version
```

## 04-设置密码

将 VNC 密码文件统一放在 `/etc/vnc/passwd`：

```bash
sudo mkdir -p /etc/vnc
sudo x11vnc -storepasswd /etc/vnc/passwd
sudo chmod 600 /etc/vnc/passwd
```

说明：

- `-storepasswd` 会交互式让你设置密码
- `chmod 600` 防止密码文件被其他用户读取

## 05-创建服务

创建 systemd 服务文件 `/etc/systemd/system/x11vnc.service`：

```bash
sudo tee /etc/systemd/system/x11vnc.service > /dev/null <<'EOF'
[Unit]
Description=Start x11vnc at startup
After=multi-user.target

[Service]
Type=simple

# 无 HDMI 时尝试强制某些设备保持输出为 on（适配部分板子/驱动）
# 如果你的设备不需要这段，可以删除 ExecStartPre
ExecStartPre=/bin/bash -c 'for dev in /sys/class/drm/card*-HDMI-A-*; do if [ -f "$dev/status" ] && [ "$(cat "$dev/status")" != "connected" ]; then echo "on" > "$dev/status"; fi; done'

ExecStart=/usr/bin/x11vnc \
  -auth guess \
  -forever \
  -loop \
  -capslock \
  -nomodtweak \
  -noxdamage \
  -repeat \
  -rfbauth /etc/vnc/passwd \
  -rfbport 5900 \
  -shared

[Install]
WantedBy=multi-user.target
EOF
```

### 5.1 参数说明（方便你以后改）

- `-auth guess`：自动猜测正确的 Xauthority（共享真实桌面时常用）
- `-forever`：客户端断开后服务不退出
- `-shared`：允许多个客户端共享连接
- `-rfbport 5900`：固定端口 `5900`
- `-rfbauth /etc/vnc/passwd`：启用密码认证
- `ExecStartPre ... HDMI ...`：无 HDMI 时做兼容处理（部分设备需要）

## 06-启用自动启动

加载 systemd 并启动 + 设置开机自启：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now x11vnc.service
```

## 07-禁用冲突服务

如果系统里还有 `vncserver.service`（tightvnc/tigervnc 一类），可能出现 `:1 already running` / `pid not found` 等冲突报错。建议禁用并屏蔽：

```bash
sudo systemctl disable --now vncserver.service 2>/dev/null || true
sudo systemctl mask vncserver.service 2>/dev/null || true
sudo systemctl reset-failed vncserver.service 2>/dev/null || true
```

## 08-验证

### 8.1 检查服务状态（应为 active/running）

```bash
systemctl status x11vnc.service
```

### 8.2 检查是否开机自启（应为 enabled）

```bash
systemctl is-enabled x11vnc.service
```

### 8.3 检查端口监听（应看到 5900）

```bash
sudo ss -lntp | grep 5900
```

### 8.4 一键健康检查（推荐收藏）

```bash
echo "== enable =="; systemctl is-enabled x11vnc.service; \
echo "== status =="; systemctl --no-pager -l status x11vnc.service; \
echo "== listen =="; sudo ss -lntp | egrep ':(5900|5901)\b' || true; \
echo "== processes =="; ps -ef | egrep 'x11vnc|Xtightvnc|Xvnc|vncserver' | grep -v grep; \
echo "== last logs =="; journalctl -u x11vnc.service -n 50 --no-pager
```

### 8.5 断电/重启验证（最硬核）

```bash
sudo reboot
```

设备重启后，用 VNC 客户端连接验证即可。

## 09-客户端连接

在电脑端打开 VNC 客户端（RealVNC Viewer / TigerVNC Viewer 等）：

- 地址：`<板子IP>:5900`
- 或：`<板子IP>:0`（等价于 5900）
- 示例：`192.168.8.109:5900`

## 10-加固建议

### 10.1 服务异常退出自动拉起（强烈推荐）

使用 override（不改原服务文件）：

```bash
sudo systemctl edit x11vnc.service
```

在打开的编辑器中粘贴：

```ini
[Service]
Restart=always
RestartSec=2
```

保存退出后：

```bash
sudo systemctl daemon-reload
sudo systemctl restart x11vnc.service
```

确认 override 生效：

```bash
systemctl cat x11vnc.service
```

### 10.2 如果遇到开机偶尔连不上，等待桌面服务启动

```bash
sudo systemctl edit x11vnc.service
```

加入：

```ini
[Unit]
After=display-manager.service
Wants=display-manager.service
```

然后：

```bash
sudo systemctl daemon-reload
sudo systemctl restart x11vnc.service
```

## 11-排查

### 11.1 连不上：先看端口 / 服务 / 日志

```bash
sudo ss -lntp | grep 5900
systemctl status x11vnc.service
journalctl -u x11vnc.service -n 100 --no-pager
```

### 11.2 是否有其他 VNC 进程冲突

```bash
ps -ef | egrep 'x11vnc|Xtightvnc|Xvnc|vncserver' | grep -v grep
```

如存在 tightvnc/tigervnc 残留（例如 `Xtightvnc`），清理：

```bash
sudo pkill -f Xtightvnc || true
sudo pkill -f Xvnc || true
sudo pkill -f vncserver || true
```

必要时清理锁文件（谨慎执行）：

```bash
sudo rm -f /tmp/.X1-lock
sudo rm -f /tmp/.X11-unix/X1
```

## 12-安全

- 强烈建议只在内网使用 VNC，或限制来源 IP
- 不建议把 `5900` 直接暴露公网

UFW 示例：仅允许 `192.168.8.0/24` 访问 `5900`：

```bash
sudo ufw allow from 192.168.8.0/24 to any port 5900 proto tcp
sudo ufw enable
sudo ufw status
```

## 13-附录

### 13.1 查看服务文件内容

```bash
systemctl cat x11vnc.service
```

### 13.2 停止 / 启动 / 重启服务

```bash
sudo systemctl stop x11vnc.service
sudo systemctl start x11vnc.service
sudo systemctl restart x11vnc.service
```

### 13.3 卸载（可选）

```bash
sudo systemctl disable --now x11vnc.service
sudo rm -f /etc/systemd/system/x11vnc.service
sudo systemctl daemon-reload
sudo apt remove -y x11vnc
```

### 13.4操作过程中遇到的小问题

你尝试上传 README 时发现：

SFTP 上传一直 0%

touch /home/sunrise/test_write 报 只读文件系统

findmnt 显示根分区 / 的挂载选项为 ro

一度甚至出现 /usr/bin/sudo: 输入/输出错误

这说明系统把根分区挂成了只读（ro），通常原因是：

ext4 检测到文件系统错误（常见于突然断电）

或底层存储（eMMC/SD）发生 I/O 错误
Linux 为了保护数据，会把 / 自动 remount 成只读，避免继续写导致更大损坏。

✅ 关键现象：只读时无法写文件、无法上传、无法 git commit、甚至可能连 sudo 都读不出来。

3）为什么“重启后又莫名其妙好了”

你重启后再次检查：

/ 变为 rw

写入测试成功（WRITE_OK）

sudo 恢复正常（SUDO_OK）

这种情况常见：重启时文件系统重新挂载，有时错误没有立刻触发，于是又能正常写入。
但要注意：这不代表问题彻底消失，如果根因是存储不稳或文件系统有隐患，它可能复发。

这次涉及到的核心知识点
1）什么是“根分区 / 根文件系统（root filesystem）”

Linux 的所有文件都挂在一个树形目录里，最顶层是 /，叫 根目录。

/ 目录所在的那个分区（例如 /dev/mmcblk1p2）就是 根分区（root partition）。

根分区里通常包含：

/bin /usr/bin（命令，如 sudo、ls）

/etc（配置）

/home（用户目录）

/var（日志/缓存）

以及系统运行所需的大量文件

所以一旦根分区变成只读：

你不仅不能写 /home，很多系统服务也会异常

甚至会出现 “sudo 读不到” 这种 I/O 错误

如何查看根分区是谁
findmnt -no SOURCE,FSTYPE,OPTIONS /

例：输出 /dev/mmcblk1p2 ext4 rw,relatime
说明根分区设备是 /dev/mmcblk1p2，文件系统 ext4，当前可写（rw）。

2）rw / ro 是什么

rw：read-write，可读可写（正常状态）

ro：read-only，只读（异常保护状态）

查看挂载状态常用：

mount | grep ' on / '
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS
3）为什么系统会自动变 ro

ext4 或底层存储遇到问题时，内核可能打印日志并把分区 remount 为只读，例如：

EXT4-fs error ...

Buffer I/O error ...

mmc timeout ...

这是保护机制：宁可只读，也不要继续写导致数据结构进一步损坏。

查看内核日志（需要 sudo）：

sudo dmesg -T | egrep -i "ext4|mmc|i/o error|timeout" | tail -n 80
4）SFTP 上传 0% 卡住的真正原因

当远端目录不可写时（尤其根分区 ro）：

SFTP 仍然能建立连接、列目录

但创建临时文件/写入会失败，表现为“卡住/0%/一直不动”

所以排查上传问题时，第一优先不是网络，而是先测写入：

touch ~/rw_test && echo OK && rm ~/rw_test
排查与修复方法（这次用到的）
1）快速确认是否只读
findmnt -no SOURCE,FSTYPE,OPTIONS /
touch ~/rw_test && echo WRITE_OK && rm ~/rw_test
2）尝试临时 remount（不一定成功）
sudo mount -o remount,rw /
3）最终修复：fsck（文件系统检查/修复）

如果 ro 反复出现，通常需要 fsck 修复 ext4：

fsck -n：只检查，不修改

fsck -fy：自动修复（风险更大，但常用）

⚠️ 重要：根分区在运行中不适合直接修复，最稳做法是在恢复模式/救援系统下对分区执行：

sudo fsck -fy /dev/mmcblk1p2
预防建议（非常实用）

尽量不要直接拔电
突然断电是 ext4 损坏最常见原因。尽量：

sudo shutdown -h now

如果经常断电场景，考虑：

更稳的电源

可靠的 eMMC/SD 卡（劣质卡很容易 I/O error）

关键设备用 UPS 或电源管理

若未来再次出现只读，第一时间保存重要数据，然后安排离线 fsck 或重刷。


