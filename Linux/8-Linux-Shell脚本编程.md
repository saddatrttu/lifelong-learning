# 8. Shell 脚本编程

> 把常用命令串成脚本，配合变量、判断、循环、函数，实现自动化。这是运维与自动化测试的核心技能。

---

## 目录

1. [脚本基础](#脚本基础)
2. [变量](#变量)
3. [输入输出](#输入输出)
4. [条件判断](#条件判断)
5. [循环](#循环)
6. [函数](#函数)
7. [常用内置变量与技巧](#常用内置变量与技巧)
8. [脚本实战示例](#脚本实战示例)
9. [常见问题](#常见问题)

---

## 脚本基础

```bash
#!/bin/bash
# 上面的 shebang 声明用哪个解释器执行本脚本

# 给执行权限
chmod +x deploy.sh
# 运行
./deploy.sh
```

### 变量引用技巧

| 写法 | 说明 |
|------|------|
| `set -e` | 任一命令失败立即退出（推荐） |
| `set -u` | 使用未定义变量时报错 |
| `set -x` | 打印每条执行的命令（调试用） |
| `set -o pipefail` | 管道中任一段失败则整体失败 |

```bash
#!/bin/bash
set -euo pipefail
echo "脚本开始"
```

> 脚本第一行加 `set -euo pipefail` 是行业惯例，能避免大量"命令失败脚本还继续跑"的坑。

---

## 变量

```bash
# 定义（注意等号两边不能有空格）
name="huan"
count=3

# 使用（建议加大括号）
echo "hello, $name"
echo "count=$count"
echo "path=${name}_dir"

# 数字运算（$(( ))）
sum=$(( count + 2 ))
echo $sum          # 5

# 命令替换：把命令输出存入变量
today=$(date +%F)
files=$(ls /etc | wc -l)
echo "today=$today, files=$files"

# 字符串变量操作
msg="hello world"
echo ${#msg}            # 长度 11
echo ${msg:0:5}         # 子串 hello
echo ${msg/hello/hi}    # 替换 hi world
```

---

## 输入输出

```bash
# 命令行参数
# $0=脚本名  $1=第1个参数  $2=第2个  $#=参数个数  $@=全部参数
echo "脚本名: $0"
echo "第一个参数: $1"
echo "参数个数: $#"

# 运行：./deploy.sh dev build
# $1=dev  $2=build

# 交互输入
read -p "请输入名字: " name
echo "你好, $name"

read -s -p "密码: " pass    # -s 隐藏输入
```

---

## 条件判断

### if / elif / else

```bash
if [ -f /etc/passwd ]; then
    echo "文件存在"
elif [ -d /etc ]; then
    echo "目录存在"
else
    echo "都不存在"
fi
```

### 常用判断

| 判断 | 含义 | 判断 | 含义 |
|------|------|------|------|
| `-f file` | 是普通文件 | `-z str` | 字符串为空 |
| `-d dir` | 是目录 | `-n str` | 字符串非空 |
| `-e file` | 存在（文件或目录） | `a = b` | 字符串相等 |
| `-x file` | 可执行 | `a != b` | 字符串不等 |
| `-r file` | 可读 | `n1 -eq n2` | 数字相等 |
| `-w file` | 可写 | `n1 -gt n2` | 数字大于 |

### 数值比较

```bash
count=10
if [ "$count" -gt 5 ]; then
    echo "count 大于 5"
fi

# 复合条件
if [ -f a.txt ] && [ -r a.txt ]; then
    echo "a.txt 存在且可读"
fi

if [ "$1" = "prod" ] || [ "$1" = "dev" ]; then
    echo "合法环境"
fi

# 新版写法 [[ ]] 更安全，支持正则
if [[ "$str" == *.log ]]; then
    echo "是日志文件"
fi
```

> `[ ]` 内**每个空格都重要**（`[ -f file ]` 不是 `[-f file]`）。`[[ ]]` 是 bash 扩展，更健壮。

---

## 循环

### for

```bash
# 遍历列表
for env in dev test prod; do
    echo "部署 $env"
done

# 数字范围
for i in {1..5}; do
    echo "第 $i 次"
done

# 遍历文件
for f in *.log; do
    echo "处理 $f"
done

# C 风格
for ((i=0; i<5; i++)); do
    echo $i
done
```

### while

```bash
count=0
while [ $count -lt 5 ]; do
    echo "count=$count"
    count=$((count + 1))
done

# 逐行读取文件
while IFS= read -r line; do
    echo "行: $line"
done < hosts.txt

# break / continue
for i in {1..10}; do
    [ $i -eq 3 ] && continue    # 跳过 3
    [ $i -eq 8 ] && break       # 到 8 停止
    echo $i
done
```

---

## 函数

```bash
# 定义
log_info() {
    echo "[$(date '+%F %T')] [INFO] $1"
}

deploy() {
    local env=$1          # local 声明局部变量
    echo "部署到 $env"
    log_info "deploy to $env"
}

# 调用
log_info "开始部署"
deploy prod
```

```bash
# 函数返回值（0=成功，非0=失败）
check_dir() {
    [ -d "$1" ] && return 0
    return 1
}
if check_dir /etc; then
    echo "/etc 存在"
fi
```

---

## 常用内置变量与技巧

```bash
echo "脚本路径: $0"
echo "当前目录: $PWD"
echo "上条命令退出码: $?"        # 0=成功
echo "当前用户: $USER"
echo "主机名: $HOSTNAME"

# 变量默认值
echo "${PORT:-8080}"          # PORT 为空时用 8080

# 多条命令
cd /tmp && echo "进入成功"     # 前一条成功才执行后一条
cd /tmp || exit 1              # 失败则退出
```

---

## 脚本实战示例

**自动备份脚本 `backup.sh`：**

```bash
#!/bin/bash
set -euo pipefail

# 配置
SRC="/data"
BACKUP_DIR="/backup"
DAYS_KEEP=7

# 1. 创建备份目录
mkdir -p "$BACKUP_DIR"

# 2. 打包压缩，文件名带日期
STAMP=$(date +%Y%m%d_%H%M%S)
tar -czf "$BACKUP_DIR/data_$STAMP.tar.gz" -C "$SRC" .

# 3. 清理 7 天前的备份
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$DAYS_KEEP -delete

# 4. 输出结果
echo "[$(date)] 备份完成: data_$STAMP.tar.gz"
```

**批量部署脚本 `deploy.sh`：**

```bash
#!/bin/bash
set -euo pipefail

ENV=${1:-dev}      # 默认 dev

echo "=== 部署到 $ENV ==="
cd /opt/app

git pull || { echo "拉取失败"; exit 1; }

if [ "$ENV" = "prod" ]; then
    echo "生产环境：跳过测试"
else
    pytest tests/ || { echo "测试失败"; exit 1; }
fi

echo "=== 部署完成 ==="
```

> 更多可组合的命令与思路见 [[7-Linux-文本处理三剑客]] 与 [[5-Linux-进程与系统管理]]。

---

## 常见问题

**Q：`command not found` 但命令明明存在？**
A：脚本里 PATH 不完整。脚本开头加 `export PATH=/usr/local/bin:/usr/bin:/bin`，或命令用绝对路径。

**Q：`syntax error near unexpected token`？**
A：多因 Windows 换行符 `\r`。用 `sed -i 's/\r$//' script.sh` 去除，或用 `dos2unix script.sh`。

**Q：`set -e` 后脚本在判断处退出？**
A：`set -e` 对条件判断里的命令不生效（if/while 中是安全的）。但管道中命令失败要加 `set -o pipefail`。

**Q：变量加不加引号有什么区别？**
A：`"$var"` 保留空格为整体（推荐），`$var` 会被按空白拆词。路径含空格时必须加引号。

---

## 下一步

- 实战排障速查 → [[9-Linux-常见问题与实战]]
- 与 Git 自动化工作流结合 → [[0-Git 使用说明]]
- 文本处理三剑客 → [[7-Linux-文本处理三剑客]]
- 完整手册导航 → [[0-Linux 使用指南]]
