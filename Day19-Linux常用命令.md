---
tags:
  - 黑马点评前置
  - Linux
  - 命令速查
---
# Day02 - Linux 常用命令速查

## 一、命令使用技巧

| 技巧 | 操作 | 说明 |
|------|------|------|
| Tab 自动补全 | 输入前几个字符按 `Tab` | 自动补全文件/目录/命令名 |
| 双 Tab 提示 | 连按两次 `Tab` | 有多个候选时列出所有选项 |
| 历史命令 | 上下箭头 | 快速调出用过的命令 |
| 清屏 | `clear` 或 `Ctrl + L` | 界面干净 |

## 二、命令格式

```
command [-options] [parameter]
```

| 部分 | 说明 |
|------|------|
| command | 命令名 |
| [-options] | 选项，控制行为，可省略 |
| [parameter] | 参数，零个/一个/多个 |

**注意：** 命令名、选项、参数之间用**空格**分隔；`[]` 代表可选。

## 三、文件目录操作

### ls — 查看目录内容

```bash
ls [-al] [dir]
```
- `-a`：显示所有文件（含 `.` 开头的隐藏文件）
- `-l`：详细列出（类型 d=目录，-=文件、权限、拥有者、大小）
- **`ll` = `ls -l` 的简写**

| 用法 | 作用 |
|------|------|
| `ls -al` | 当前目录所有文件及详细信息 |
| `ls -al /etc` | 查看 /etc 目录 |
| `ll` | 当前目录详细信息 |

### cd — 切换目录

```bash
cd [dirName]
```

| 符号 | 含义 |
|------|------|
| `~` | 用户 home 目录（root: /root，其他: /home/xxx） |
| `.` | 当前目录 |
| `..` | 上级目录 |
| `../..` | 上上级目录 |

### cat — 一次性查看文件

```bash
cat [-n] fileName    # -n：显示行号
cat /etc/profile
```

### more — 分页查看（大文件）

```bash
more fileName
```

| 按键 | 作用 |
|------|------|
| 回车 | 向下滚动一行 |
| 空格 | 向下滚动一屏 |
| `b` | 返回上一屏 |
| `q` / `Ctrl+C` | 退出 |

### tail — 查看文件尾部（日志必备）

```bash
tail [-f] fileName
```

| 用法 | 作用 |
|------|------|
| `tail /etc/profile` | 末尾 10 行（默认） |
| `tail -20 /etc/profile` | 末尾 20 行 |
| `tail -f my.log` | **动态读取**（实时看日志，Ctrl+C 结束） |

> **tail -f 是看日志的核心命令**：一个窗口 `tail -f` 盯着，另一个窗口操作，实时看到新内容。部署排查全靠它。

### mkdir — 创建目录

```bash
mkdir [-p] dirName
```
- `-p`：父目录不存在自动创建，支持**多级**创建

```bash
mkdir demo          # 建子目录
mkdir -p demo/test  # 多级创建
```

### rmdir — 删除空目录（了解）

```bash
rmdir [-p] dirName
```
- `*` 是通配符：`rmdir demo*` 删以 demo 开头的空目录
- **只能删空目录**，非空要用 rm

### rm — 删除文件/目录

```bash
rm [-rf] name
```
- `-r`：递归删除目录及所有内容
- `-f`：无需确认直接删

| 用法 | 作用 |
|------|------|
| `rm -r demo/` | 删目录（会确认） |
| `rm -rf demo/` | 强制删目录 |
| `rm -f hello.txt` | 强制删文件 |

> ⚠️ **`rm -rf` 慎用！** 执行前确认路径，误删不可恢复。

## 四、拷贝移动

### cp — 复制

```bash
cp [-r] source dest    # 复制目录必须加 -r
```

| 用法 | 作用 |
|------|------|
| `cp hello.txt demo/` | 复制到目录 |
| `cp hello.txt ./hi.txt` | 复制并改名 |
| `cp -r demo/ ./demo2/` | 复制整个目录 |
| `cp -r demo/* ./demo2/` | 复制目录下所有文件 |

### mv — 改名/移动

```bash
mv source dest
```

| 用法 | 作用 |
|------|------|
| `mv hello.txt hi.txt` | 改名 |
| `mv hi.txt demo2/` | 移动到目录 |
| `mv hi.txt demo2/hello.txt` | 移动+改名 |
| `mv demo/ demo2/` | dest 不存在=改名；存在=移动 |

> 判断规则：**第二个参数是已存在的目录 → 移动；否则 → 改名**。

## 五、打包压缩（重要）

```bash
tar [-zcxvf] fileName [files]
```
- `.tar`：只打包没压缩；`.tar.gz`：打包+压缩（gzip）

| 选项 | 含义 |
|------|------|
| `-z` | gzip 压缩/解压 |
| `-c` | create 创建包 |
| `-x` | extract 解包 |
| `-v` | 显示执行过程 |
| `-f` | 指定包文件名（必须放最后） |

**打包压缩：**
```bash
tar -cvf hello.tar ./*          # 打包（不压缩）
tar -zcvf hello.tar.gz ./*      # 打包+压缩 ★常用
tar -zcvf hello.tar.gz hello    # 打包指定目录
```

**解包解压：**
```bash
tar -xvf hello.tar              # 解包到当前目录
tar -zxvf hello.tar.gz          # 解压到当前目录 ★常用
tar -zxvf hello.tar.gz -C /usr/local   # 解压到指定目录
```

> 记忆口诀：`-c` 打包、`-x` 解压、`-z` 压缩、`-v` 啰嗦、`-f` 文件名。解压到指定目录加 `-C`。

## 六、文本编辑

### touch — 创建空文件

```bash
touch 文件名
```

> ⚠️ **乱码解决**：执行命令提示乱码时，改编码：
> ```bash
> echo 'LANG="en_US.UTF-8"' >> /etc/profile
> source /etc/profile
> ```

### vim — 文本编辑器（重点）

```bash
vim fileName        # 文件存在打开，不存在新建
yum install vim     # 先安装（vi 系统自带，vim 功能更强带语法着色）
```

**三种模式（使用 vim 必须知道当前在哪个模式）：**

```
命令模式 ←→ 插入模式
   ↕
 底行模式
```

**① 命令模式**（打开文件默认进入；i/a/o 进插入，:/ 进底行）

| 指令 | 含义 |
|------|------|
| `gg` / `G` | 定位到第一行 / 最后一行 |
| `dd` / `ndd` | 删除光标所在行 / 当前行及后 n 行 |
| `u` | 撤销 |
| `shift + zz` | 保存并退出 |
| `i` / `a` / `o` | 进入插入模式 |

**② 插入模式**（可以编辑内容，按 `ESC` 回命令模式，下方显示 `insert`）

**③ 底行模式**（命令模式按 `:` 或 `/` 进入）

| 指令 | 含义 |
|------|------|
| `:wq` | 保存并退出 |
| `:q!` | 不保存退出 |
| `:set nu` / `:set nonu` | 显示/取消行号 |
| `:10` | 定位到第 10 行 |
| `/关键词` | 查找内容 |

> 最常用组合：`vim 文件` → `i` 编辑 → `ESC` → `:wq` 保存退出。

## 七、查找命令

### find — 按文件名查找

```bash
find dirName -option fileName
find . -name "*.java"              # 当前目录及子目录找 java 文件
find /demo -name "*.java"       # 指定目录找
```

### grep — 查文件内容（重要）

```bash
grep word fileName
grep Hello HelloWorld.java          # 文件里找字符串
grep hello *.java                   # 批量文件里找
cat 1.txt | grep password           # 配合管道筛选
```

> `|` 管道：把左边命令的输出交给右边命令处理，`ps -ef | grep redis` 就是这么来的。

## 八、进程与端口

### top — 实时监控进程

```bash
top               # 实时显示所有进程（CPU/内存占用）
top -c            # 显示完整命令
top -p PID        # 监控指定进程
```

### ps — 查看进程快照

```bash
ps -A                         # 所有进程
ps -ef                        # 所有进程（完整信息）★常用
ps -u 用户名                  # 指定用户进程
ps -ef | grep tomcat          # 找含 tomcat 的进程 ★常用
```

### kill — 杀进程

```bash
kill PID              # 正常杀死
kill -9 PID           # 强制杀死 ★常用
killall -u 用户名     # 杀指定用户所有进程
```

### netstat — 端口占用

```bash
netstat -anp | grep 80        # 80 端口是否被占用
netstat -nultp                # 查看已使用的端口信息
```

> 部署服务报"端口被占用"，就 `netstat -anp | grep 端口` 找到 PID，`kill -9` 强杀。

---

## ⚠️ 避坑 Tips

### 1. 安全的坑
- **`rm -rf` 一定要确认路径**。误删不可恢复，公司里手滑删库的案例不少
- **`kill -9` 是最后手段**。正常服务先 `kill`，不行再 `-9`
- 不要在 root 下乱跑高危命令，养成先 `ls` 确认再动手的习惯

### 2. 操作的坑
- **tar 的 `-f` 必须放最后**。`tar -zxvf hello.tar.gz` 是对的，`-f` 后面跟文件名
- **vim 保存：命令模式 `shift+zz`，底行模式 `:wq`**，别在插入模式里瞎按
- **grep 配管道**：`cat 文件 | grep 关键词`、`ps -ef | grep 进程名`，筛选信息最常用
- **编码乱码**：执行命令提示乱码，改 `LANG="en_US.UTF-8"` 并 source

### 3. 装 Redis 时用到的命令（回看）
| 命令                       | 对应知识点           |
| ------------------------ | --------------- |
| `yum install -y gcc tcl` | 软件安装            |
| `mv a b`                 | 改名（换源备份）        |
| `curl -o`                | 下载              |
| `tar -xzf`               | 解压（就是本节 tar 命令） |
| `make && make install`   | 编译              |
| `vi redis.conf`          | 文本编辑（本节vi命令）    |
| `ps -ef \| grep redis`   | 进程查看            |

记录第十九天，计划稍微改一下吧。
黑马点评前置day2，国庆回来事有点儿多啊，是要放弃些东西。今天简单学了学Linux的常用命令，也就是有个印象吧，整理了一下，用的时候记不起来看看笔记，这东西就得多敲才行，不用肯定忘。然后下载了黑马点评实战的资料。