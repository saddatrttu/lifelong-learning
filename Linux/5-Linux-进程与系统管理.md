# 5. 进程与系统管理

> 查看和控制进程、管理系统服务、监控资源占用，是运维与排障的核心能力。

---

## 目录

1. [查看进程](#查看进程)
2. [控制进程](#控制进程)
3. [系统服务 systemctl](#系统服务-systemctl)
4. [系统资源监控](#系统资源监控)
5. [定时任务 cron](#定时任务-cron)
6. [系统信息](#系统信息)
7. [常见问题](#常见问题)

---

## 查看进程

### ps（静态快照）

```bash
ps aux                # 全部进程（a 所有用户、u 详细信息、x 含无终端）
ps -ef                # 完整格式（兼容风格）
ps aux | grep nginx   # 查找某进程
ps -ef --sort=-%cpu   # 按 CPU 排序
```

`ps aux` 输出字段：`USER PID %CPU %MEM VSZ RSS TTY STAT START TIME COMMAND`。PID 是进程号，杀进程靠它。

### top / htop（动态实时）

```bash
top               # 实时刷新，按 q 退出
top -p 1234       # 只看某 PID
```

top 交互按键：`M` 按内存排序、`P` 按 CPU 排序、`k` 输入 PID 杀进程、`q` 退出。

> `htop` 是 top 的友好替代（需安装），支持鼠标、树状视图。

---

## 控制进程

### kill / killall / pkill

```bash
kill 1234              # 发送 TERM 信号，请求正常退出
kill -9 1234           # SIGKILL 强制杀死（最后手段）
kill -l                # 列出所有信号
killall nginx          # 按名字杀所有匹配进程
pkill -f "python app"  # 按完整命令行匹配
```

| 信号 | 数字 | 作用 |
|------|------|------|
| SIGTERM | 15 | 默认，礼貌终止 |
| SIGKILL | 9 | 强制杀死，不可拦截 |
| SIGHUP | 1 | 终端断开/重载配置 |

### 前后台

```bash
long_command &          # 后台运行
Ctrl+Z                  # 挂起当前进程
jobs                    # 列出后台任务
fg                      # 调到前台
bg                      # 后台继续运行
nohup command &         # 退出终端后仍运行（脱机）
```

> 让服务在 SSH 断开后继续运行的标准做法：`nohup python app.py > app.log 2>&1 &`。

---

## 系统服务 systemctl

现代发行版用 systemd 管理服务：

```bash
systemctl status nginx     # 查看服务状态
systemctl start nginx      # 启动
systemctl stop nginx       # 停止
systemctl restart nginx    # 重启
systemctl enable nginx     # 开机自启
systemctl disable nginx    # 取消自启
systemctl list-units --type=service   # 列出所有服务
journalctl -u nginx -f     # 查看服务日志（实时）
```

> 传统 sysvinit 时代的命令 `service nginx start` 在部分系统仍兼容。

---

## 系统资源监控

```bash
free -h                # 内存使用
df -h                  # 磁盘分区使用
du -sh /home/*         # 各目录占用
uptime                 # 负载（1/5/15 分钟均值）
vmstat 1               # 每 1 秒刷新一次系统指标
iostat                 # 磁盘 IO（需安装 sysstat）
```

**排查系统卡顿的思路**：`top` 看 CPU/内存 → `free` 看内存 → `df` 看磁盘 → `tail -f` 看日志。

---

## 定时任务 cron

```bash
crontab -e         # 编辑当前用户定时任务
crontab -l         # 列出
crontab -r         # 清空
```

格式：`分 时 日 月 周 命令`

```bash
# 每天凌晨 2 点备份
0 2 * * * tar -czf /backup/daily.tar.gz /data

# 每小时跑一次
0 * * * * /opt/check.sh

# 每 5 分钟
*/5 * * * * curl -s http://localhost/health

# 每周一早上 9 点
0 9 * * 1 /opt/weekly_report.sh
```

| 字段 | 取值 | 含义 |
|------|------|------|
| 分 | 0-59 | 分钟 |
| 时 | 0-23 | 小时 |
| 日 | 1-31 | 日期 |
| 月 | 1-12 | 月份 |
| 周 | 0-7 | 星期（0/7 都指周日） |

> 调试技巧：cron 环境变量与终端不同（无完整 PATH），脚本内尽量用绝对路径。日志看 `/var/log/cron` 或系统日志。

---

## 系统信息

```bash
uname -a              # 内核版本信息
uname -r              # 内核版本号
cat /etc/os-release   # 发行版信息
hostname              # 主机名
lscpu                 # CPU 信息
```

---

## 常见问题

**Q：`kill` 杀不死进程？**
A：进程在等待 IO 或忽略信号。先 `kill -9` 强杀，或用 `pkill -9 -f 命令名`。

**Q：SSH 断开后进程就停了？**
A：命令在前台运行，终端断开会收到 SIGHUP。用 `nohup 命令 &` 或 systemd 管理。

**Q：`crontab -e` 里执行脚本不生效？**
A：脚本里用绝对路径、cron 环境 PATH 不全。先在脚本开头 `export PATH=/usr/local/bin:/usr/bin:/bin`。

**Q：磁盘满了但 `df` 显示还有空间？**
A：可能有已删除但被进程占用的文件：`lsof | grep deleted` 找到进程重启即可释放。

---

## 下一步

- 网络与远程管理 → [[6-Linux-网络与远程管理]]
- 日志与文本处理 → [[7-Linux-文本处理三剑客]]
- 常见坑与实战 → [[9-Linux-常见问题与实战]]
- 完整手册导航 → [[0-Linux 使用指南]]
