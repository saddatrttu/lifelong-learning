# 7. 文本处理三剑客

> grep、sed、awk 是 Linux 文本处理的三大神器，配合 sort/uniq/cut/xargs 可完成几乎所有数据加工任务。

---

## 目录

1. [grep 文本搜索](#grep-文本搜索)
2. [sed 流式编辑](#sed-流式编辑)
3. [awk 列处理](#awk-列处理)
4. [排序与去重](#排序与去重)
5. [截取与拼接](#截取与拼接)
6. [xargs 批量执行](#xargs-批量执行)
7. [实战组合](#实战组合)
8. [常见问题](#常见问题)

---

## grep 文本搜索

按正则匹配行，输出整行。

```bash
grep "ERROR" app.log            # 基本搜索
grep -i "error" app.log         # 忽略大小写
grep -n "ERROR" app.log         # 显示行号
grep -v "DEBUG" app.log         # 反选（不含 DEBUG 的行）
grep -r "TODO" src/             # 递归搜索目录
grep -c "ERROR" app.log         # 统计匹配行数
grep -A 2 -B 2 "ERROR" app.log  # 前后各 2 行上下文
grep -E "ERROR|FATAL" app.log   # 扩展正则（多个关键词）
grep "^2026-08" app.log         # 行首匹配
grep -l "secret" *.conf         # 只列匹配的文件名
```

> 日志排查经典：`grep -i "error\|exception" app.log | tail -20`。

---

## sed 流式编辑

流式文本编辑器，逐行处理，不改原文件（除非 `-i`）。

```bash
# 打印/删除
sed -n '10,20p' file.txt      # 只打印 10-20 行（-n 静默）
sed '5d' file.txt             # 删除第 5 行
sed '/ERROR/d' file.txt       # 删除含 ERROR 的行

# 替换（核心功能）
sed 's/old/new/' file.txt     # 每行第一个 old → new
sed 's/old/new/g' file.txt    # 全部替换（g=global）
sed 's/old/new/g' file.txt > new.txt   # 输出到新文件
sed -i 's/old/new/g' file.txt # 直接改原文件
sed -i.bak 's/old/new/g' file.txt      # 改前备份为 .bak
sed -E 's/(a)(b)/\2\1/g' file  # 扩展正则 + 分组引用

# 插入/追加
sed '3i\INSERTED' file.txt    # 第 3 行前插入
sed '3a\APPENDED' file.txt    # 第 3 行后追加
```

> `-E` 启用扩展正则；分组用 `()`，引用用 `\1`、`\2`。sed 最适合"简单替换 + 按行号操作"。

---

## awk 列处理

按列处理文本（默认按空白分列），编程能力最强。

```bash
awk '{print $1}' file.txt      # 打印第一列
awk '{print $2, $NF}' file.txt # 第二列和最后一列
awk -F',' '{print $1}' data.csv  # 按逗号分列

# 条件过滤
awk '$3 > 100 {print $1}' file.txt    # 第三列>100 的打印第一列
awk '/ERROR/ {print $1}' app.log      # 匹配 ERROR 的行

# 内置变量与运算
awk '{sum += $1} END {print sum}' nums.txt   # 求和
awk 'NR==1, NR==10' file.txt         # 前 10 行（NR=行号）
awk '{print NR": "$0}' file.txt      # 带行号输出整行

# 字符串与统计
awk '{count[$2]++} END {for (k in count) print k, count[k]}' file.txt
```

| awk 内置变量 | 含义 |
|--------------|------|
| `$0` | 整行 |
| `$1`..`$N` | 第 N 列 |
| `NF` | 列数（NF=最后一列） |
| `NR` | 当前行号 |
| `FS` | 列分隔符（`-F','` 等价 `-v FS=","`） |

> awk 的通用模式是 `awk '条件 {动作}'`。`END{}` 块在所有行处理完后执行，适合统计。

---

## 排序与去重

```bash
sort file.txt                 # 按字典序排序
sort -n file.txt              # 按数值排序
sort -rn file.txt             # 数值降序
sort -t: -k3 -n passwd        # 按冒号分列，第 3 列数值排序
sort -u file.txt              # 排序并去重

uniq file.txt                 # 去除相邻重复行（须先排序）
uniq -c file.txt              # 统计每行出现次数
sort file.txt | uniq -c | sort -rn   # 词频统计经典组合
```

---

## 截取与拼接

```bash
cut -d',' -f1,3 data.csv      # 按逗号取第 1、3 列
cut -c1-10 file.txt           # 取每行前 10 个字符

paste a.txt b.txt             # 按行拼接两文件
paste -d',' a.txt b.txt       # 指定分隔符

tr 'a-z' 'A-Z' < file.txt     # 大小写转换
tr -d '\r' < win.txt > unix.txt   # 去除 Windows 换行符
```

---

## xargs 批量执行

把前一个命令的输出，逐批传给后一个命令作参数。解决"find/grep 输出不能直接当参数"的问题。

```bash
find . -name "*.tmp" | xargs rm        # 删除所有 .tmp
cat urls.txt | xargs curl -O           # 批量下载
echo "a b c" | xargs mkdir             # 批量建目录
find . -name "*.log" | xargs grep "ERROR"   # 批量搜日志
```

> 文件名含空格时用 `xargs -0`（配合 find 的 `-print0`）：`find . -name "*.tmp" -print0 | xargs -0 rm`。

---

## 实战组合

```bash
# 1. 统计日志中最常出现的 IP
grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" access.log | sort | uniq -c | sort -rn | head

# 2. 查看系统最耗内存的前 5 个进程
ps aux | sort -rk 4 | head -n 6

# 3. 提取 CPU 使用率最高的进程名
top -b -n1 | sed -n '8,15p' | awk '{print $NF}'

# 4. 找出所有文件里包含 TODO 的行，带文件名行号
grep -rn "TODO" src/ | awk -F: '{print $1" 行"$2": "$3}'
```

---

## 常见问题

**Q：`sed -i` 后文件没了？**
A：sed 语法错误时 `-i` 会覆盖原文件（保留不完整内容）。先不带 `-i` 测试，或用 `-i.bak` 备份。

**Q：awk 中文乱码/字段错乱？**
A：列分隔符没配对。先 `head` 看原始分隔符（空格？逗号？），再指定 `-F`。

**Q：`uniq` 没去重？**
A：uniq 只去**相邻**重复行。先 `sort` 再 `uniq`。

**Q：grep 匹配 `[`、`.` 等特殊字符？**
A：加转义 `\` 或用 `grep -F`（固定字符串，不做正则解析）。

---

## 下一步

- 用 Shell 脚本把这些命令组合成自动化工具 → [[8-Linux-Shell脚本编程]]
- 文件操作基础 → [[2-Linux-文件与目录操作]]
- 日志与排障实战 → [[9-Linux-常见问题与实战]]
- 完整手册导航 → [[0-Linux 使用指南]]
