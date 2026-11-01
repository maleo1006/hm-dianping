---
tags:
  - 黑马点评
  - Linux
  - Redis
  - 部署
---
# Day01 - Redis安装（Linux系统）

## 一、SQL和NoSQL

MySQL是关系型数据库，Redis是非关系型。

**七个维度对比：**

| 维度 | SQL（MySQL） | NoSQL（Redis） |
|------|-------------|---------------|
| 数据结构 | 结构化（表/行/列） | 非结构化（key-value） |
| 数据关联 | 关联的（外键/多表） | 无关联的 |
| 查询方式 | SQL 查询 | 非 SQL |
| 事务特性 | ACID | BASE |
| 存储方式 | 磁盘 | 内存 |
| 扩展性 | 垂直扩展（加机器配置） | 水平扩展（加机器数量） |
| 使用场景 | 结构固定、对数据安全一致性要求高 | 结构不固定、对一致性安全性要求不高、对性能要求高 |

**NoSQL 的四种类型：**

| 类型 | 代表 | 场景 |
|------|------|------|
| 键值类型 | Redis | 缓存、计数器 |
| 文档类型 | MongoDB | 灵活结构的业务数据 |
| 列类型 | HBase | 海量结构化数据 |
| Graph 类型 | Neo4j | 关系图谱、社交网络 |

## 二、Redis安装

之前安装的是非官方Windows版本，本项目里是用Linux系统来安装Redis，Linux版本为CentOS 7。所以我们需要先安装**虚拟机**、然后在虚拟机上安装**CentOS7**，然后可以安装**远程连接软件**来在Windows上敲Linux命令。
虚拟机 VMware Workstation
远程连接 FinalShell

**完整安装流程：**

```
装虚拟机+CentOS7 → 换阿里云源（官方源失效）→ 装编译环境 → 上传解压源码包 → make编译安装 → 改配置 → 启动验证
```

**① 换阿里云源（我踩的坑）**

Linux安装Redis
```yum install -y gcc tcl```我安装的时候报错了，原因是CentOS7 官方 yum 源失效。问AI换的阿里云源。
```
# 1.备份旧的源文件
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak

# 2.下载阿里云CentOS7源配置 
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

#3.清理旧缓存 
yum clean all

#4.生成新缓存 
yum makecache

# 安装
yum install -y gcc tcl
```

**② 解压编译**
将压缩包放到任意目录（规范 /usr/local/src ）
解压缩```tar -xzf redis-6.2.6.tar.gz```
进入redis目录：```cd redis-6.2.6``` 
运行编译命令安装：```make && make install```

**③ 修改配置后台启动**

修改配置后台启动

```conf
# 允许访问的地址，默认127.0.0.1只能在本地访问；0.0.0.0任意IP可访问，生产环境不要设
bind 0.0.0.0
# 守护进程，yes后即可后台运行
daemonize yes
# 密码，设置后访问Redis必须输入密码
requirepass 123321
```

## 三、启动与验证

**启动：**
```bash
redis-server /usr/local/src/redis-6.2.6/redis.conf   # 指定配置文件启动
```

**本机验证：**
```bash
redis-cli -p 6379
auth 123321      # 设置了密码必须先 auth
ping             # 返回 PONG 说明启动成功
```

**查看进程：**
```bash
ps -ef | grep redis    # 看 redis-server 有没有在跑
```

## 四、今天用到的 Linux 命令（明天学命令的铺垫）

| 命令 | 作用 |
|------|------|
| `yum install -y xxx` | 安装软件包 |
| `mv a b` | 移动/重命名（换源时备份用） |
| `curl -o` | 下载文件到指定路径 |
| `tar -xzf xxx.tar.gz` | 解压 .tar.gz 压缩包 |
| `cd` | 切换目录 |
| `make && make install` | 编译 + 安装（&& 表示成功才执行后面） |
| `redis-server 配置文件` | 启动 redis |
| `redis-cli` | 连接 redis 客户端 |
| `ps -ef \| grep redis` | 查看 redis 进程 |

> 明天确实要专门补一补 Linux 命令啊，今天先把安装流程吃透。
---
## ⚠️ 避坑 Tips

### 1. 环境的坑
- **CentOS7 官方 yum 源已失效**，新装的系统直接 `yum install` 会报错，先换阿里云源（备份 → 下载 → clean → makecache）
- **gcc 和 tcl 都要装**：gcc 编译 C 代码，tcl 是 Redis 测试框架要用的，缺一个 make 可能出问题
### 2. 编译安装的坑
- **make 报 jemalloc 错误**：有些环境编译报 `jemalloc/jemalloc.h` 找不到，用 `make MALLOC=libc` 绕过
- **编译很慢**：正常，C 项目全量编译要几分钟，别以为卡死了
- **make install 装到哪**：默认装到 `/usr/local/bin`，这样 `redis-server` 命令全局可用
### 3. 配置启动的坑
- **关掉终端 Redis 就停了**：`daemonize yes` 没开，改成后台运行
- **远程连不上三件套**：① bind 是不是 0.0.0.0 ② protected-mode 是不是 no ③ 防火墙有没有放行 6379
- **改了配置必须重启 Redis 才生效**：改完 redis.conf 要重启进程
---
## 🎯 面试题
### Q1：SQL 和 NoSQL 的区别？什么场景用什么？
> **回答框架：**
> 1. **结构**：SQL 结构化（表），NoSQL 非结构化（key-value/文档/列/图）
> 2. **关系**：SQL 支持关联查询（多表 JOIN），NoSQL 一般无关联
> 3. **事务**：SQL 满足 ACID，NoSQL 一般 BASE（最终一致）
> 4. **存储**：SQL 磁盘，NoSQL 内存（Redis）
> 5. **扩展**：SQL 垂直扩展，NoSQL 水平扩展
> 6. **场景**：数据固定+强一致用 SQL（订单、账户）；热点数据+高性能用 NoSQL（缓存、排行榜）
> 7. **项目例子**：苍穹外卖业务数据存 MySQL（订单、用户），热点数据（店铺状态、菜品缓存）放 Redis
### Q2：Redis 生产环境部署要注意什么？
> **回答框架：**
> 1. **环境**：部署在 Linux，不用 Windows（官方只对 Linux 深度优化）
> 2. **配置**：daemonize yes 后台运行、bind 0.0.0.0、requirepass 设密码
> 3. **安全**：必须设密码，生产环境 bind 不要设 0.0.0.0（只绑内网 IP），防火墙只放行必要端口
> 4. **持久化**：配置好 RDB/AOF 和 dir 目录
> 5. **高可用**：生产一般主从 + 哨兵，或 Redis Cluster（后面黑马点评会学）

记录第十八天，开启新的篇章。
黑马点评前置day1，今天想着是按计划快速过一遍基础的，但是没想到是Linux系统安装redis没想到这么麻烦又是虚拟机又是远程连接又是一堆命令，直接懵了，后悔当时没好好学了（是不是就算学了该忘的也都忘了哈哈哈），今天就有点儿耽误进度了，决定变一下吧，明天先补一补基本的 Linux 命令。