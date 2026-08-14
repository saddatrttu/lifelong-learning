# 9. 常见问题与实战

> 汇总使用 Linux 时最高频的坑与排查思路，附完整速查表，供日常查表使用。

---

## 目录

1. [权限问题](#权限问题)
2. [路径与命令问题](#路径与命令问题)
3. [进程与资源问题](#进程与资源问题)
4. [网络问题](#网络问题)
5. [脚本常见坑](#脚本常见坑)
6. [实战场景](#实战场景)
7. [完整速查表](#完整速查表)

---

## 权限问题

**`Permission denied`**

```bash
ls -l file           # 看权限，判断缺 r/w/x
chmod +x file        # 缺执行权限
sudo <cmd>           # 权限不够直接提权
```

**sudo 无法使用**

```bash
su -                  # 切到 root
usermod -aG sudo 你的用户名   # 把自己加进 sudo 组
```

---

## 路径与命令问题

**`command not found`**

```bash
which python3        # 命令存在吗？在哪个路径
echo $PATH           # 该路径在 PATH 里吗？
export PATH=$PATH:/usr/local/bin    # 临时加入
# 永久生效写进 ~/.bashrc
```

**`No such file or directory` 但文件存在？**
- 路径打错：用 `pwd` 确认当前目录，`find . -name "文件名"` 确认位置
- 软链接失效：`ls -l` 看 `->` 指向

**文件名有空格/特殊字符**
```bash
cat "my file.txt"
cat my\ file.txt
rm -i -- -weird-name   # 以 - 开头的文件用 -- 分隔
```

**Shell 换了不生效？**
```bash
source ~/.bashrc       # 立即生效（或 . ~/.bashrc）
```

---

## 进程与资源问题

**杀不掉进程**

```bash
kill -9 PID
pkill -9 -f 命令名
```

**磁盘满了**

```bash
df -h                     # 看哪个分区满
du -sh /home/* 2>/dev/null | sort -rh | head   # 找大目录
lsof | grep deleted       # 找被占用但已删除的文件，重启进程释放
```

**内存不足**

```bash
free -h                   # 看内存
top -o %MEM               # 按内存排序找大户
ps aux --sort=-%mem | head
```

**CPU 100%**

```bash
top -o %CPU               # 找 PID
ps -fp PID                # 看是什么程序
```

---

## 网络问题

| 报错 | 含义 | 排查 |
|------|------|------|
| `Connection refused` | 端口没服务/被拒 | `systemctl status`、`ss -tlnp` |
| `Connection timed out` | 网络不通/防火墙丢包 | ping、检查安全组/防火墙 |
| `Name or service not known` | DNS 解析失败 | `nslookup 域名`、检查 `/etc/resolv.conf` |
| `No route to host` | 路由不通 | `ip route`、`traceroute` |

**端口被占用**

```bash
ss -tlnp | grep 8080      # 找 PID
kill -9 PID               # 或换端口
```

**防火墙放行**

```bash
# ufw (Ubuntu/Debian)
sudo ufw allow 8080/tcp
# firewalld (CentOS/RHEL)
sudo firewall-cmd --permanent --add-port=8080/tcp && sudo firewall-cmd --reload
```

---

## 脚本常见坑

| 症状 | 原因 | 解决 |
|------|------|------|
| `command not found` | 脚本 PATH 不全 | 脚本开头 `export PATH=/usr/local/bin:/usr/bin:/bin` |
| `syntax error` | Windows 换行符 `\r` | `sed -i 's/\r$//' script.sh` |
| `set -e` 提前退出 | 没配 pipefail | `set -euo pipefail` |
| 条件判断一直成立 | `[ ]` 空格错误 | `[ -f file ]` 空格不能少 |
| 文件名含空格出错 | 变量没加引号 | `"$var"` |

---

## 实战场景

### 场景一：排查服务为什么挂了

```bash
systemctl status myapp          # 1. 服务状态
journalctl -u myapp -n 100      # 2. 服务日志（末尾 100 行）
tail -f /var/log/myapp.log      # 3. 实时日志
ss -tlnp | grep 8080            # 4. 端口还在吗
ps aux | grep myapp             # 5. 进程还在吗
```

### 场景二：日志里统计错误最多的类型

```bash
grep -E "ERROR|Exception" app.log \
  | sed -E 's/.*(ERROR|Exception)[^:]*:.*/\1/' \
  | sort | uniq -c | sort -rn
```

### 场景三：一键部署

```bash
#!/bin/bash
set -euo pipefail
cd /opt/app
git pull
pip install -r requirements.txt
systemctl restart myapp
curl -sf http://localhost:8080/health && echo "部署成功"
```

---

## 完整速查表

### 文件与目录

| 命令 | 作用 |
|------|------|
| `ls -la` | 列出（含隐藏、长格式） |
| `cd ~ / cd -` | 回家 / 回上目录 |
| `pwd` | 当前路径 |
| `mkdir -p` | 递归建目录 |
| `touch f` | 建空文件 |
| `cp -r` `mv` `rm -rf` | 复制 / 移动 / 删除 |
| `find . -name "x"` | 查找 |
| `ln -s` | 软链接 |
| `tar -czvf` / `-xzvf` | 压缩 / 解压 |

### 查看内容

| 命令 | 作用 |
|------|------|
| `cat` / `less` | 全文 / 分页 |
| `head -n` / `tail -n` / `tail -f` | 头 / 尾 / 实时 |
| `wc -l` | 行数 |
| `grep -r` | 递归搜索 |
| `vim` | 编辑 |

### 权限用户

| 命令 | 作用 |
|------|------|
| `chmod 755 f` | 改权限 |
| `chown u:g f` | 改属主属组 |
| `sudo cmd` | 提权 |
| `useradd` / `passwd` | 建用户 / 设密码 |
| `id` | 查看身份 |

### 进程系统

| 命令 | 作用 |
|------|------|
| `ps aux` | 查看进程 |
| `top` | 实时监控 |
| `kill -9 PID` | 杀进程 |
| `systemctl start/restart/status nginx` | 管理服务 |
| `free -h` / `df -h` | 内存 / 磁盘 |
| `crontab -e` | 定时任务 |

### 网络

| 命令 | 作用 |
|------|------|
| `ip a` | 查看 IP |
| `ping -c 4` | 测连通 |
| `curl -I URL` | 测 HTTP |
| `ssh user@host` | 远程登录 |
| `scp` / `rsync` | 传输文件 |
| `ss -tlnp` | 查看端口 |

### 文本处理

| 命令 | 作用 |
|------|------|
| `grep` | 搜索 |
| `sed 's/a/b/g'` | 替换 |
| `awk '{print $1}'` | 取列 |
| `sort` / `uniq -c` | 排序 / 统计 |
| `cut -d, -f1` | 截取 |
| `xargs` | 批量执行 |

### Make / gmake 构建

| 命令 | 作用 |
|------|------|
| `make` | 构建默认目标 |
| `make <target>` | 构建指定目标（如 `make clean`） |
| `make -j$(nproc)` | 并行构建（按 CPU 核数） |
| `make -B` | 强制重建（忽略时间戳） |
| `make -n` | 预览命令，不真正执行 |
| `make -f my.mk` | 指定 Makefile 文件名 |
| `make -p` | 打印所有变量/规则（调试） |
| `make --debug=v` | 详细输出，看谁触发重建 |
| `gmake` | GNU make（macOS/BSD 上代替 make） |
| `gmake --version` / `make --version` | 查看 make 版本/是否 GNU |
| `make -C <dir>` | 进入子目录执行 make |

### 其他高频

| 命令 | 作用 |
|------|------|
| `man cmd` / `cmd --help` | 查帮助 |
| `history` | 历史命令 |
| `echo` / `read` | 输出 / 输入 |
| `date +%F` | 当前日期 |
| `|` 管道 / `>` 重定向 | 组合命令 |

---

## 参考资料

- GNU 核心工具手册：https://www.gnu.org/software/coreutils/manual/
- Bash 参考手册：https://www.gnu.org/software/bash/manual/
- 在线练习：https://overthewire.org/wargames/bandit（闯关式学命令）

## 下一步

- 入门从头学起 → [[1-Linux-入门与环境]]
- Shell 脚本进阶 → [[8-Linux-Shell脚本编程]]
- 相关：[[0-Git 使用说明]]、[[0-Claude Code 使用指南]]、[[0-自动化测试方法论]]
- 完整手册导航 → [[0-Linux 使用指南]]
