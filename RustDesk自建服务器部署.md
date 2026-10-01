# RustDesk 自建服务器部署教程

简述：本文是搭建rustdesk中转服务器的笔记，不提供确切方法指引，仅供参考

## 1. 部署环境

### 1.1 服务器环境

- 云服务器：阿里云 ECS
- 操作系统：Ubuntu 24.04.2 LTS
- CPU 架构：x86_64
- RustDesk Server：1.1.16

### 1.2 部署方式

本教程使用 RustDesk 官方 Linux 二进制文件部署：

- `hbbs`：RustDesk ID / Rendezvous Server
- `hbbr`：RustDesk Relay Server

不使用 Docker 部署。

---

## 2. 安装 RustDesk Server

### 2.1 创建安装目录

```bash
sudo mkdir -p /opt/rustdesk
cd /opt/rustdesk
```

### 2.2 下载 RustDesk Server

下载官方 Linux AMD64 压缩包：

```text
rustdesk-server-linux-amd64.zip
```

将压缩包上传到服务器后解压：

```bash
unzip rustdesk-server-linux-amd64.zip
```

解压后得到：

```text
hbbs
hbbr
```

将两个程序放到：

```text
/opt/rustdesk/
```

### 2.3 添加执行权限

```bash
cd /opt/rustdesk

chmod +x hbbs
chmod +x hbbr
```

检查程序是否正常：

```bash
./hbbs -h
./hbbr -h
```

如果能够正常显示帮助信息，并显示版本号：

```text
1.1.16
```

说明程序可以正常运行。

---

## 3. 配置 systemd 服务

使用 systemd 可以让 RustDesk Server：

- 开机自动启动
- 后台运行
- 异常退出后自动重启
- 使用 `systemctl` 统一管理

### 3.1 创建 hbbs 服务

创建：

```bash
sudo nano /etc/systemd/system/rustdesk-hbbs.service
```

写入：

```ini
[Unit]
Description=RustDesk HBBS Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/rustdesk
ExecStart=/opt/rustdesk/hbbs
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3.2 创建 hbbr 服务

创建：

```bash
sudo nano /etc/systemd/system/rustdesk-hbbr.service
```

写入：

```ini
[Unit]
Description=RustDesk HBBR Relay Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/rustdesk
ExecStart=/opt/rustdesk/hbbr
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3.3 重新加载 systemd

```bash
sudo systemctl daemon-reload
```

### 3.4 启动并设置开机自启

启动 `hbbs`：

```bash
sudo systemctl enable --now rustdesk-hbbs
```

启动 `hbbr`：

```bash
sudo systemctl enable --now rustdesk-hbbr
```

---

## 4. 检查 RustDesk 服务

### 4.1 检查 hbbs

```bash
systemctl status rustdesk-hbbs
```

正常情况下应该看到：

```text
Active: active (running)
```

### 4.2 检查 hbbr

```bash
systemctl status rustdesk-hbbr
```

同样应该显示：

```text
Active: active (running)
```

### 4.3 检查监听端口

```bash
sudo ss -lntup | grep -E '21115|21116|21117|21118|21119'
```

可以看到 RustDesk 服务正在监听相关端口。

---

## 5. 配置云服务器安全组

RustDesk Server 对外使用的核心端口：

| 端口    | 协议  | 用途   |
| ----- | --- | ---- |
| 21115 | TCP | hbbs |
| 21116 | TCP | hbbs |
| 21116 | UDP | hbbs |
| 21117 | TCP | hbbr |

因此在阿里云 ECS 安全组中开放：

```text
TCP 21115
TCP 21116
UDP 21116
TCP 21117
```

---

## 6. RustDesk 密钥

### 6.1 自动生成密钥

第一次运行 `hbbs` 后，RustDesk Server 会在工作目录生成密钥文件。

目录：

```text
/opt/rustdesk/
```

主要文件：

```text
id_ed25519
id_ed25519.pub
```

### 6.2 公钥

```text
id_ed25519.pub
```

这是客户端配置 RustDesk Server 时使用的公钥。

可以查看：

```bash
cat /opt/rustdesk/id_ed25519.pub
```

复制输出内容保存下来。

---

## 7. 配置 RustDesk 客户端

打开 RustDesk 客户端的网络设置。

### 7.1 ID Server

填写服务器公网 IP：

```text
服务器公网IP
```

如果客户端要求明确指定端口，可以填写：

```text
服务器公网IP:21116
```

### 7.2 Relay Server

填写：

```text
服务器公网IP
```

如果客户端要求明确指定端口：

```text
服务器公网IP:21117
```

### 7.3 Key

填写服务器上的：

```text
id_ed25519.pub
```

内容。

可以通过：

```bash
cat /opt/rustdesk/id_ed25519.pub
```

查看。

### 7.4 API Server

本次部署不需要 API Server，因此保持：

```text
空白
```

即可。

---

## 8. 客户端连接测试

### 8.1 Windows 客户端

Windows RustDesk 客户端配置：

```text
ID Server:     服务器公网IP
Relay Server:  服务器公网IP
Key:           id_ed25519.pub 内容
API Server:    留空
```

配置完成后可以正常连接服务器。

### 8.2 Android 客户端

Android RustDesk 使用同样的服务器配置。

如果出现：

```text
Key mismatch
```

首先检查客户端的 `Key` 是否与服务器：

```text
/opt/rustdesk/id_ed25519.pub
```

完全一致。

修改为正确的公钥后即可正常连接。

---

## 9. 测试 P2P 与 Relay

RustDesk 并不是所有连接都会经过服务器转发。

### 9.1 相同网络环境

例如：

```text
Android 平板
      ↓
Windows 电脑热点
      ↓
同一局域网
```

这种情况下，如果两台设备可以直接建立连接，RustDesk 可以使用：

```text
P2P / 直接连接
```

此时 HBBR 服务器不一定会产生明显的中继流量。

### 9.2 不同网络环境

例如：

```text
Android 平板
      ↓
移动数据网络

        Internet

      ↓
RustDesk VPS
      ↓
Windows 电脑
```

如果两个客户端无法建立直接连接，则可以通过：

```text
HBBR Relay Server
```

进行中继。

因此可以通过观察服务器上的 HBBR 活动来判断是否正在使用 Relay。

---

## 10. RustDesk 的连接逻辑

整体可以理解为：

```text
                RustDesk Server
               ┌───────────────┐
               │     HBBS      │
               │ ID / 会合服务 │
               └───────┬───────┘
                       │
             尝试建立直接连接
                       │
                 ┌─────┴─────┐
                 │           │
              成功           失败
                 │           │
                 ↓           ↓
                P2P         HBBR
                            Relay
```

也就是说：

```text
HBBS
↓
帮助客户端找到彼此并建立连接

P2P
↓
两个客户端直接通信

HBBR
↓
无法直接连接时进行中继
```

---

## 11. 服务维护

### 11.1 查看服务状态

查看 HBBS：

```bash
systemctl status rustdesk-hbbs
```

查看 HBBR：

```bash
systemctl status rustdesk-hbbr
```

### 11.2 重启服务

重启 HBBS：

```bash
sudo systemctl restart rustdesk-hbbs
```

重启 HBBR：

```bash
sudo systemctl restart rustdesk-hbbr
```

### 11.3 查看日志

查看 HBBS：

```bash
journalctl -u rustdesk-hbbs
```

查看 HBBR：

```bash
journalctl -u rustdesk-hbbr
```

实时查看日志：

```bash
journalctl -u rustdesk-hbbs -f
```

或者：

```bash
journalctl -u rustdesk-hbbr -f
```

### 11.4 检查端口

```bash
sudo ss -lntup | grep -E '21115|21116|21117'
```

---

## 12. 服务器目录结构

最终 `/opt/rustdesk/` 目录主要包含：

```text
/opt/rustdesk/
├── hbbs
├── hbbr
├── id_ed25519
├── id_ed25519.pub
├── db_v2.sqlite3
├── db_v2.sqlite3-shm
├── db_v2.sqlite3-wal
└── data/
```

其中：

```text
hbbs
```

负责 ID / 会合服务。

```text
hbbr
```

负责 Relay 中继服务。

```text
id_ed25519
```

服务器私钥，必须妥善保存。

```text
id_ed25519.pub
```

服务器公钥，用于客户端配置。

---

## 13. 最终配置总结

### 13.1 服务器

```text
系统：Ubuntu 24.04.2 LTS
架构：x86_64
RustDesk Server：1.1.16
安装目录：/opt/rustdesk/
```

### 13.2 公网开放端口

```text
TCP 21115
TCP 21116
UDP 21116
TCP 21117
```

### 13.3 客户端

```text
ID Server：服务器公网 IP
Relay Server：服务器公网 IP
Key：id_ed25519.pub
API Server：留空
```

### 13.4 服务

```text
rustdesk-hbbs.service
rustdesk-hbbr.service
```

均设置为：

```text
开机自动启动
异常自动重启
systemd 管理
```

### 13.5 工作方式

```text
客户端
   │
   ├── 可以直接连接 → P2P
   │
   └── 无法直接连接 → HBBR Relay
```


