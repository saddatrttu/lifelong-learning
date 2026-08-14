# 2. 文件与目录操作

> 文件操作是 Linux 使用频率最高的场景：增删改查、查找定位、软硬链接与归档压缩。

---

## 目录

1. [目录操作](#目录操作)
2. [文件操作](#文件操作)
3. [查找文件](#查找文件)
4. [软链接与硬链接](#软链接与硬链接)
5. [归档与压缩](#归档与压缩)
6. [磁盘与文件信息](#磁盘与文件信息)
7. [常见问题](#常见问题)

---

## 目录操作

```bash
mkdir mydir                # 创建目录
mkdir -p a/b/c             # 递归创建多级目录
rmdir mydir                # 删除空目录
rm -r mydir                # 递归删除非空目录
cp -r src dst              # 复制整个目录
mv src dst                 # 移动/重命名目录
```

---

## 文件操作

### 增 / 查

```bash
touch new.txt              # 创建空文件（或更新时间戳）
ls -l                      # 长格式：权限 链接 属主 属组 大小 时间 名
ls -lh                     # -h 人类可读大小
ls -lt                     # 按修改时间排序（最新在前）
file a.txt                 # 识别文件真实类型
du -sh mydir               # 查看目录占用大小
```

### 复制 / 移动

```bash
cp a.txt b.txt             # 复制
cp -i a.txt b.txt          # 覆盖前询问
cp -p a.txt b.txt          # 保留属性（时间/权限）
mv a.txt b.txt             # 重命名
mv a.txt /tmp/             # 移动
mv -i a.txt b.txt          # 覆盖前询问
```

> ⚠️ `cp`/`mv` 默认会**静默覆盖**已存在文件。生产环境建议习惯性加 `-i` 或先检查。

### 删除

```bash
rm a.txt                   # 删除文件
rm -i a.txt                # 删除前确认
rm -f a.txt                # 强制删除（不询问）
rm -rf mydir               # 递归强制删除目录
```

> ⚠️ **`rm -rf` 没有回收站，删除即永久丢失**。高危操作前先 `ls` 确认路径，或用 `mv 到 /tmp` 代替删除。

---

## 查找文件

### find（条件查找，最强大）

```bash
find . -name "*.log"             # 按名字
find /var -size +100M            # 大于 100M 的文件
find /home -mtime -7             # 7 天内修改过的
find . -type f                   # 只找普通文件（-type d 目录）
find . -name "*.tmp" -delete     # 找到并删除
find . -name "*.py" -exec grep -l "TODO" {} \;
```

> `find ... -exec 命令 {} \;`：对每个结果执行命令，`{}` 是结果占位符，`\;` 表示结束。更简单的方式是用管道 `xargs`（见 [[7-Linux-文本处理三剑客]]）。

### locate / which / whereis

```bash
locate nginx.conf        # 基于数据库快速查找（需先 updatedb）
which python3            # 找可执行命令路径
whereis ls               # 找命令、源码、手册位置
```

---

## 软链接与硬链接

| 类型 | 命令 | 特点 |
|------|------|------|
| 软链接（符号链接） | `ln -s 目标 链接名` | 指向路径，跨分区/目录，目标删了链接失效 |
| 硬链接 | `ln 目标 链接名` | 指向同一 inode，只能同分区，目标删了内容还在 |

```bash
ln -s /usr/bin/python3 ~/py      # 软链接
ln /data/a.txt /data/b.txt       # 硬链接

ls -l       # 软链接显示 -> 目标
```

> Windows 用户可以把软链接理解为"快捷方式"，硬链接理解为"同一个文件的多个名字"。

---

## 归档与压缩

### tar（打包 + 压缩，最常用）

```bash
tar -czvf archive.tar.gz mydir/    # 打包并 gzip 压缩
tar -tzf archive.tar.gz            # 查看压缩包内容
tar -xzvf archive.tar.gz           # 解压
tar -xzvf archive.tar.gz -C /opt   # 解压到指定目录
```

选项速记：`c` 创建、`x` 解压、`t` 查看、`z` gzip、`j` bzip2、`v` 显示过程、`f` 指定文件名。

### zip / unzip

```bash
zip -r my.zip mydir/
unzip my.zip
unzip my.zip -d /tmp/
```

---

## 磁盘与文件信息

```bash
df -h                 # 各分区磁盘占用（人类可读）
du -sh /home/*        # 各目录占用
ls -i file            # 查看 inode 编号
stat file             # 文件详细信息（时间、权限、大小）
```

---

## 常见问题

**Q：`find` 找不到刚创建的文件？**
A：`locate` 走数据库有延迟，需 `sudo updatedb`；`find` 是实时扫描不会漏，优先用 find。

**Q：软链接失效（显示红底闪烁）？**
A：目标文件被删或移动了。用 `ls -l` 看 `->` 指向的路径是否存在，重新 `ln -s` 即可。

**Q：`rm -rf` 误删了重要文件怎么办？**
A：无法恢复（无回收站）。平时用 `rm -i`、重要文件用 Git 管理（见 [[0-Git 使用说明]]），或备份。

**Q：`Permission denied`？**
A：权限不足，用 `sudo`，或 `chmod` 调整权限（见 [[4-Linux-权限与用户管理]]）。

---

## 下一步

- 查看文件内容与编辑 → [[3-Linux-文件内容查看与编辑]]
- 权限管理 → [[4-Linux-权限与用户管理]]
- 用 grep/sed/awk 加工数据 → [[7-Linux-文本处理三剑客]]
- 完整手册导航 → [[0-Linux 使用指南]]
