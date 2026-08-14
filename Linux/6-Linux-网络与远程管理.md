# 6. 网络与远程管理

> 查看网络状态、连通性测试、HTTP 请求工具，以及最核心的 SSH 远程登录与文件传输。

---

## 目录

1. [查看网络状态](#查看网络状态)
2. [连通性与域名](#连通性与域名)
3. [HTTP 请求工具](#http-请求工具)
4. [SSH 远程登录](#ssh-远程登录)
5. [文件传输 scp/rsync](#文件传输-scprsync)
6. [端口与监听](#端口与监听)
7. [常见问题](#常见问题)

---

## 查看网络状态

```bash
ip addr          # 查看网卡与 IP（现代命令）
ip a             # 简写
ifconfig         # 老命令，部分系统需 net-tools

ip route         # 查看路由表（默认网关）
ip link set eth0 up    # 启用网卡
```

> `ip` 是 iproute2 新命令，`ifconfig` 是老工具。优先学 `ip`。

---

## 连通性与域名

```bash
ping -c 4 8.8.8.8        # 测连通性（-c 次数，Linux 默认无限发）
ping google.com          # 测 DNS 是否解析
curl -I https://example.com   # 测 HTTP 服务连通
telnet host 80           # 测端口是否开放（Ctrl+] 后 quit 退出）
nc -vz host 80           # nc 测端口
```

> Windows 的 ping 默认 4 次后退出，Linux 的 ping 要加 `-c` 限制次数，否则 Ctrl+C。

---

## HTTP 请求工具

### curl（最常用）

```bash
curl -I https://example.com        # 只看响应头
curl -s URL > file                 # 静默下载
curl -o out.html URL               # 指定输出文件
curl -O https://x.com/file.zip     # 按原文件名保存
curl -X POST -d "name=huan" URL    # POST 请求
curl -H "Authorization: Bearer xxx" URL   # 带请求头
curl -k https://self-signed.com    # 跳过证书校验（测试用）
```

### wget（下载器）

```bash
wget URL                     # 下载
wget -c URL                  # 断点续传
wget -r -np URL              # 递归下载目录
```

---

## SSH 远程登录

```bash
ssh user@host                      # 登录远程主机
ssh -p 2222 user@host              # 指定端口
ssh -i ~/.ssh/id_rsa user@host     # 用密钥登录
```

### 配置免密登录（SSH 密钥）

```bash
# 1. 本地生成密钥对（一路回车）
ssh-keygen -t ed25519 -C "your@email.com"

# 2. 复制公钥到服务器（会要求输入一次密码）
ssh-copy-id user@host
# 手动方式：把 ~/.ssh/id_ed25519.pub 内容追加到服务器 ~/.ssh/authorized_keys

# 3. 之后即可免密登录
ssh user@host
```

### SSH 配置优化（~/.ssh/config）

```
Host myserver
    HostName 192.168.1.10
    User huan
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

之后直接 `ssh myserver` 即可。

### SSH 隧道（端口转发）

```bash
# 本地 8080 → 远程 80（访问本地即访问远程网站）
ssh -L 8080:localhost:80 user@host

# 动态代理（SOCKS5，本地 1080 端口）
ssh -D 1080 user@host
```

> 排查 SSH 问题：`ssh -v user@host` 查看详细握手过程。

---

## 文件传输 scp/rsync

### scp（加密复制）

```bash
scp local.txt user@host:/home/user/    # 上传
scp user@host:/home/user/a.txt .       # 下载
scp -r mydir/ user@host:/tmp/          # 目录递归
scp -P 2222 file user@host:/tmp/       # 指定端口（大写 P）
```

### rsync（增量同步，更强）

```bash
rsync -avz mydir/ user@host:/backup/        # 增量同步
rsync -avz --delete src/ dst/               # 删除目标端多余文件
rsync -e "ssh -p 2222" src/ user@host:dst/  # 指定 ssh 参数
```

> rsync 只传差异部分，适合大目录同步、备份。

---

## 端口与监听

```bash
ss -tlnp            # 查看监听端口及进程（现代命令）
netstat -tlnp       # 老命令
lsof -i :8080       # 谁占用了 8080 端口
```

| 选项 | 含义 |
|------|------|
| `-t` | TCP |
| `-l` | 只显示监听 |
| `-n` | 不解析域名（快） |
| `-p` | 显示进程 |

> 端口被占的排查标准流程：`ss -tlnp` 找到 PID → `ps -fp PID` 看进程 → 决定 kill 或换端口。

---

## 常见问题

**Q：`Connection refused`？**
A：目标端口没服务监听。检查服务是否启动（`systemctl status`）、防火墙是否放行。

**Q：`Connection timed out`？**
A：网络不通或被防火墙丢弃。先 ping 通不通，再检查安全组/防火墙规则。

**Q：SSH 登录很慢（卡几秒）？**
A：多为 DNS 反向解析。服务器 `/etc/ssh/sshd_config` 设 `UseDNS no` 后重启 sshd。

**Q：如何快速开放防火墙端口？**
```bash
# ufw（Ubuntu）
sudo ufw allow 8080
# firewalld（CentOS）
sudo firewall-cmd --permanent --add-port=8080/tcp && sudo firewall-cmd --reload
```

---

## 下一步

- 文本处理三剑客 → [[7-Linux-文本处理三剑客]]
- Shell 脚本编程 → [[8-Linux-Shell脚本编程]]
- 进程与系统管理 → [[5-Linux-进程与系统管理]]
- 完整手册导航 → [[0-Linux 使用指南]]
