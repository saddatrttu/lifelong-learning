# 4. 权限与用户管理

> Linux 是多用户系统，权限模型是安全的基础。理解权限位、属主属组、sudo 提权与用户管理。

---

## 目录

1. [权限模型](#权限模型)
2. [chmod 修改权限](#chmod-修改权限)
3. [chown 修改属主属组](#chown-修改属主属组)
4. [umask 默认权限](#umask-默认权限)
5. [用户与组管理](#用户与组管理)
6. [sudo 提权](#sudo-提权)
7. [特殊权限位](#特殊权限位)
8. [常见问题](#常见问题)

---

## 权限模型

`ls -l` 第一列就是权限，形如 `drwxr-xr--`：

```
d rwx r-x r--
│ │││ │││ │││
│ │││ │││ └┴┴── 其他用户(o)
│ │││ └┴┴─────── 属组(g)
│ └┴┴─────────── 属主(u)
└─────────────── 类型(d目录 / -文件 / l链接)
```

每一位字符含义：`r` 读、`w` 写、`x` 执行（目录则是"可进入/列出"），`-` 表示无权限。

**三组对象**：`u` 属主（user）、`g` 属组（group）、`o` 其他（other）。

### 数字表示法

| 权限 | 数字 |
|------|------|
| r | 4 |
| w | 2 |
| x | 1 |

`rwxr-xr--` = 7（u）5（g）4（o）= **754**。即 `r`=4、`w`=2、`x`=1，相加得到。

---

## chmod 修改权限

```bash
# 数字法（最常用）
chmod 755 script.sh        # 属主 rwx，属组/其他 r-x
chmod 644 file.txt         # 属主 rw-，属组/其他 r--

# 符号法
chmod u+x script.sh        # 属主加执行权限
chmod g-w file.txt         # 属组去掉写权限
chmod o=r file.txt         # 其他只读
chmod a+x script.sh        # 所有人加执行（a=all）

# 递归
chmod -R 755 mydir/        # 目录及内部全部修改
```

> 日常建议：**目录 755、文件 644、脚本 755**。用 `ls -l` 验证结果。

---

## chown 修改属主属组

```bash
chown user file.txt            # 修改属主
chown user:group file.txt      # 同时修改属主和属组
chown :group file.txt          # 只修改属组
chown -R user:group mydir/     # 递归
```

> `chown` 通常需要 root 权限，前面加 `sudo`。

---

## umask 默认权限

新建文件的默认权限由 umask 决定：默认全权限 `666/777` 减去 umask 值。

```bash
umask                # 查看，通常 022
umask 022            # 设置：文件=666-022=644，目录=777-022=755
```

永久生效需写入 `~/.bashrc`。

---

## 用户与组管理

```bash
# 用户
useradd -m -s /bin/bash alice     # 建用户+家目录+shell
passwd alice                      # 设置/修改密码
usermod -aG sudo alice            # 把 alice 加入 sudo 组
userdel -r alice                  # 删除用户及家目录
id alice                          # 查看用户 uid/gid/所属组
whoami                            # 当前用户名

# 组
groupadd devops                   # 建组
usermod -aG devops alice          # 加入组（注意 -aG 保留原有组）
groups alice                      # 查看用户所在组
```

> ⚠️ `usermod -G` 不加 `-a` 会把用户**踢出所有其他组**，务必用 `-aG`。

### 配置文件

| 文件 | 内容 |
|------|------|
| `/etc/passwd` | 用户账号信息 |
| `/etc/shadow` | 密码哈希（root 可读） |
| `/etc/group` | 组信息 |
| `/etc/sudoers` | sudo 权限配置（必须用 `visudo` 编辑） |

---

## sudo 提权

```bash
sudo ls /root            # 以 root 执行单条命令
sudo -i                  # 切换为 root shell
sudo -u www curl localhost  # 以指定用户执行
```

> sudo 安全：普通用户必须有 sudo 权限（在 sudo 组或 `/etc/sudoers` 配置）。`visudo` 编辑 sudoers 时会校验语法，防止把自己锁死。

---

## 特殊权限位

| 权限 | 表示 | 作用 |
|------|------|------|
| SUID | `-rwsr-xr-x` | 执行时以属主身份运行（如 `/usr/bin/passwd`） |
| SGID | 目录 `drwxrwsr-x` | 目录内新建文件继承目录属组 |
| Sticky | 目录 `drwxrwxrwt` | `/tmp` 等目录，仅属主/root 可删自己文件 |

```bash
chmod u+s file          # 加 SUID
chmod g+s dir           # 加 SGID
chmod +t dir            # 加 Sticky
```

> ⚠️ 慎用 SUID，配置不当是提权漏洞的常见来源。

---

## 常见问题

**Q：`Permission denied`，`sudo` 也不行？**
A：你的用户不在 sudo 组。用 root 登录执行 `usermod -aG sudo 用户名`，或让管理员加你。

**Q：执行脚本提示 `Permission denied`？**
A：脚本没有执行权限。`chmod +x script.sh` 再执行 `./script.sh`。

**Q：`./` 开头执行和直接写文件名有什么区别？**
A：`./script.sh` 明确指定当前目录；直接写 `script.sh` 系统会在 `PATH` 里找，当前目录默认不在 PATH。

**Q：为什么 `/tmp` 下的文件别人也能删？**
A：Sticky 位限制：只有属主/root 能删。你自己的文件请放 `~` 或 `/home`。

---

## 下一步

- 进程与系统管理 → [[5-Linux-进程与系统管理]]
- 网络与远程管理 → [[6-Linux-网络与远程管理]]
- 常见坑排查 → [[9-Linux-常见问题与实战]]
- 完整手册导航 → [[0-Linux 使用指南]]
