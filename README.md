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



