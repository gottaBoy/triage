# Bash 脚本快速入门指南

## 命令解析：`SCRIPTS_DIR=$(cd $(dirname $0); pwd)`

### 命令分解

```bash
SCRIPTS_DIR=$(cd $(dirname $0); pwd)
```

**逐步解析**：

1. **`$0`** - 当前脚本的文件名（包括路径）
   ```bash
   # 如果脚本是 /home/user/scripts/myscript.sh
   # 那么 $0 = /home/user/scripts/myscript.sh
   ```

2. **`dirname $0`** - 获取脚本所在的目录路径
   ```bash
   # dirname /home/user/scripts/myscript.sh
   # 结果：/home/user/scripts
   ```

3. **`$(dirname $0)`** - 命令替换，执行命令并获取输出
   ```bash
   # 等价于先执行 dirname $0，然后用结果替换
   ```

4. **`cd $(dirname $0)`** - 切换到脚本所在目录
   ```bash
   # 切换到 /home/user/scripts 目录
   ```

5. **`pwd`** - 打印当前工作目录的绝对路径
   ```bash
   # 输出：/home/user/scripts
   ```

6. **`$(cd $(dirname $0); pwd)`** - 组合命令
   ```bash
   # 先切换到脚本目录，然后获取绝对路径
   ```

7. **`SCRIPTS_DIR=...`** - 将结果赋值给变量
   ```bash
   # SCRIPTS_DIR 现在包含脚本目录的绝对路径
   ```

### 完整示例

```bash
#!/bin/bash
# 获取脚本所在目录的绝对路径
SCRIPTS_DIR=$(cd $(dirname $0); pwd)

echo "脚本目录：$SCRIPTS_DIR"
# 输出：脚本目录：/home/user/scripts

# 现在可以使用这个路径
CONFIG_FILE="$SCRIPTS_DIR/config.conf"
LOG_FILE="$SCRIPTS_DIR/logs/app.log"
```

### 为什么需要这个命令？

**问题场景**：
- 脚本可能从任何目录被调用
- 需要访问脚本同目录下的文件
- 需要确保路径是绝对路径

**示例**：
```bash
# 用户在不同目录执行脚本
cd /tmp
/home/user/scripts/myscript.sh  # 脚本需要找到同目录的 config.conf

cd /var
/home/user/scripts/myscript.sh  # 同样需要找到 config.conf
```

**解决方案**：
```bash
SCRIPTS_DIR=$(cd $(dirname $0); pwd)
CONFIG_FILE="$SCRIPTS_DIR/config.conf"  # 总是能找到
```

---

## Bash 脚本基础语法

### 1. 变量

```bash
# 定义变量（等号两边不能有空格）
NAME="张三"
AGE=25

# 使用变量
echo $NAME
echo ${NAME}  # 推荐使用大括号

# 只读变量
readonly PI=3.14

# 删除变量
unset NAME
```

### 2. 命令替换

```bash
# 方式1：使用 $()
CURRENT_DIR=$(pwd)
DATE=$(date +%Y-%m-%d)

# 方式2：使用反引号（不推荐，已废弃）
CURRENT_DIR=`pwd`
```

### 3. 字符串操作

```bash
STR="Hello World"

# 获取长度
echo ${#STR}  # 输出：11

# 截取子串
echo ${STR:0:5}    # 输出：Hello（从0开始，长度5）
echo ${STR:6}      # 输出：World（从索引6开始）

# 字符串替换
echo ${STR/World/Bash}  # 输出：Hello Bash（替换第一个）
echo ${STR//l/L}        # 输出：HeLLo WorLd（替换所有）

# 删除前缀/后缀
FILE="app.log"
echo ${FILE%.log}      # 输出：app（删除 .log 后缀）
echo ${FILE#app}       # 输出：.log（删除 app 前缀）
```

### 4. 数组

```bash
# 定义数组
ARR=("apple" "banana" "orange")

# 访问元素
echo ${ARR[0]}      # 输出：apple
echo ${ARR[@]}      # 输出所有元素
echo ${#ARR[@]}     # 输出数组长度：3

# 遍历数组
for item in "${ARR[@]}"; do
    echo $item
done
```

### 5. 条件判断

```bash
# if-else
if [ $AGE -gt 18 ]; then
    echo "成年人"
elif [ $AGE -gt 0 ]; then
    echo "未成年人"
else
    echo "无效年龄"
fi

# 常用比较运算符
# 数值比较：-eq, -ne, -gt, -lt, -ge, -le
# 字符串比较：=, !=, -z（空）, -n（非空）
# 文件测试：-f（文件存在）, -d（目录存在）, -r（可读）

# 示例
if [ -f "$CONFIG_FILE" ]; then
    echo "配置文件存在"
fi

if [ -d "$SCRIPTS_DIR" ]; then
    echo "目录存在"
fi
```

### 6. 循环

```bash
# for 循环
for i in {1..5}; do
    echo $i
done

# C 风格 for 循环
for ((i=1; i<=5; i++)); do
    echo $i
done

# while 循环
COUNT=0
while [ $COUNT -lt 5 ]; do
    echo $COUNT
    COUNT=$((COUNT + 1))
done

# 遍历文件
for file in *.txt; do
    echo "处理文件：$file"
done
```

### 7. 函数

```bash
# 定义函数
function greet() {
    local name=$1  # 局部变量
    echo "Hello, $name!"
}

# 调用函数
greet "张三"

# 返回值（通过 $? 获取，0表示成功）
function check_file() {
    if [ -f "$1" ]; then
        return 0  # 成功
    else
        return 1  # 失败
    fi
}

check_file "test.txt"
if [ $? -eq 0 ]; then
    echo "文件存在"
fi
```

### 8. 输入输出

```bash
# 读取用户输入
read -p "请输入姓名：" NAME
echo "你好，$NAME"

# 读取文件
while IFS= read -r line; do
    echo "$line"
done < file.txt

# 输出重定向
echo "Hello" > output.txt        # 覆盖
echo "World" >> output.txt       # 追加

# 错误重定向
command 2> error.log            # 错误输出到文件
command > output.log 2>&1       # 所有输出到文件
```

### 9. 特殊变量

```bash
$0      # 脚本文件名
$1, $2, ...  # 位置参数（第1个、第2个参数）
$@      # 所有参数（数组）
$*      # 所有参数（字符串）
$#      # 参数个数
$$      # 当前进程ID
$?      # 上一个命令的退出状态（0=成功）
```

### 10. 常用命令

```bash
# 文件操作
ls -la                    # 列出文件
cp source dest           # 复制
mv source dest           # 移动/重命名
rm file                  # 删除
mkdir dir                # 创建目录
find . -name "*.txt"     # 查找文件

# 文本处理
grep "pattern" file      # 搜索
sed 's/old/new/g' file   # 替换
awk '{print $1}' file    # 文本处理
sort file                # 排序
uniq file                # 去重

# 系统信息
date                     # 日期时间
whoami                   # 当前用户
pwd                      # 当前目录
ps aux                   # 进程列表
```

---

## 实用脚本示例

### 示例1：获取脚本目录（标准写法）

```bash
#!/bin/bash
# 获取脚本所在目录的绝对路径
SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
echo "脚本目录：$SCRIPT_DIR"
```

**说明**：
- `"${BASH_SOURCE[0]}"` 比 `$0` 更可靠（在 source 时也能正确工作）
- 使用双引号防止路径中有空格

### 示例2：检查文件是否存在

```bash
#!/bin/bash
FILE="$1"

if [ -z "$FILE" ]; then
    echo "用法：$0 <文件名>"
    exit 1
fi

if [ -f "$FILE" ]; then
    echo "文件存在：$FILE"
else
    echo "文件不存在：$FILE"
    exit 1
fi
```

### 示例3：日志函数

```bash
#!/bin/bash
LOG_FILE="app.log"

log() {
    local level=$1
    shift
    local message="$@"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] [$level] $message" | tee -a "$LOG_FILE"
}

log "INFO" "脚本开始执行"
log "ERROR" "发生错误"
log "DEBUG" "调试信息"
```

### 示例4：错误处理

```bash
#!/bin/bash
set -e  # 遇到错误立即退出
set -u  # 使用未定义变量时报错
set -o pipefail  # 管道命令失败时也退出

# 或者手动检查
if ! command -v python3 &> /dev/null; then
    echo "错误：未找到 python3"
    exit 1
fi

# 执行命令并检查结果
if ! python3 script.py; then
    echo "错误：脚本执行失败"
    exit 1
fi
```

### 示例5：参数解析

```bash
#!/bin/bash
# 用法：script.sh -f file.txt -v -n 10

VERBOSE=false
FILE=""
NUMBER=0

while [[ $# -gt 0 ]]; do
    case $1 in
        -f|--file)
            FILE="$2"
            shift 2
            ;;
        -v|--verbose)
            VERBOSE=true
            shift
            ;;
        -n|--number)
            NUMBER="$2"
            shift 2
            ;;
        *)
            echo "未知参数：$1"
            exit 1
            ;;
    esac
done

echo "文件：$FILE"
echo "详细模式：$VERBOSE"
echo "数字：$NUMBER"
```

---

## 最佳实践

### 1. 脚本头部

```bash
#!/bin/bash
# 脚本描述
# 作者：Your Name
# 日期：2024-01-01

set -euo pipefail  # 严格模式
IFS=$'\n\t'        # 设置内部字段分隔符
```

### 2. 变量命名

```bash
# 使用大写字母和下划线
CONFIG_FILE="/etc/app.conf"
MAX_RETRY=3
LOG_LEVEL="INFO"
```

### 3. 路径处理

```bash
# 总是使用引号
FILE="$HOME/documents/file.txt"

# 使用绝对路径或相对脚本的路径
SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
CONFIG_FILE="$SCRIPT_DIR/config.conf"
```

### 4. 错误处理

```bash
# 检查命令是否成功
if ! command; then
    echo "错误：command 执行失败"
    exit 1
fi

# 检查文件是否存在
if [ ! -f "$FILE" ]; then
    echo "错误：文件不存在：$FILE"
    exit 1
fi
```

### 5. 调试技巧

```bash
# 启用调试模式
set -x  # 显示执行的命令
# 或运行时：bash -x script.sh

# 添加调试输出
DEBUG=true
if [ "$DEBUG" = true ]; then
    echo "调试：变量值 = $VAR"
fi
```

---

## 快速参考表

| 语法 | 说明 | 示例 |
|------|------|------|
| `$VAR` | 变量值 | `echo $NAME` |
| `${VAR}` | 变量值（推荐） | `echo ${NAME}` |
| `$(cmd)` | 命令替换 | `DIR=$(pwd)` |
| `$0` | 脚本名 | `echo $0` |
| `$1, $2` | 位置参数 | `echo $1` |
| `$#` | 参数个数 | `echo $#` |
| `$?` | 退出状态 | `if [ $? -eq 0 ]` |
| `[ ]` | 条件测试 | `[ -f file ]` |
| `[[ ]]` | 扩展测试 | `[[ $str == *pattern* ]]` |
| `-f` | 文件存在 | `[ -f file ]` |
| `-d` | 目录存在 | `[ -d dir ]` |
| `-z` | 字符串为空 | `[ -z "$str" ]` |
| `-n` | 字符串非空 | `[ -n "$str" ]` |
| `-eq` | 等于（数值） | `[ $a -eq $b ]` |
| `-gt` | 大于（数值） | `[ $a -gt $b ]` |

---

## 学习资源

1. **在线文档**：
   - [Bash 参考手册](https://www.gnu.org/software/bash/manual/)
   - [Bash 脚本教程](https://www.runoob.com/linux/linux-shell.html)

2. **练习**：
   - 编写简单的文件操作脚本
   - 实现日志记录功能
   - 创建自动化部署脚本

3. **常用工具**：
   - `shellcheck` - 脚本语法检查工具
   - `bashdb` - Bash 调试器

---

## 总结

**`SCRIPTS_DIR=$(cd $(dirname $0); pwd)` 的作用**：
- 获取脚本所在目录的**绝对路径**
- 无论从哪个目录执行脚本，都能正确获取脚本目录
- 常用于访问脚本同目录下的配置文件、日志文件等

**快速记忆**：
- `$0` = 脚本路径
- `dirname` = 获取目录
- `cd` = 切换目录
- `pwd` = 获取绝对路径
- `$()` = 执行命令并获取结果

