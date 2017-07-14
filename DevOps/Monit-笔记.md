Monit 笔记
===
### 0. 开始之前
- 本文主要基于 Monit [官方文档][monit-doc], 因此跟偏向于速查手册
- 整理 Monit 相关知识用于公司内部技术分享
- 生产环境上主要用于进程监控和系统资源监控, 报警使用 `邮件` 和 `shell + bearychat`
- 个人能力有限, 如有错误欢迎指出


### 1. Monit 是什么

- Monit 是一个管理和监控 Unix 系统的小型开源组件.
- Monit 可以在出现错误的情况下, 自动维护, 修复和做一些有意义的行为


### 2. 为什么选择 Monit

除了 Monit 还有一些其他的第三方监控方案(eg. Supervisor), 我们考虑选择额 Monit 作为监控的原因有
- 超轻量, 稳定, 高可用
- 依赖少, 安装配置方便, 尽量减少运维及学习成本(即使没有任何 Monit 基础的人, 都能轻易的读懂大部分监控文件)
- 非侵入式, 被监控的程序可以不用知道监控程序的存在(如果使用 Supervisor 监控, 则服务必须从 Supervisor 启动)
- 基本功能完备(9 种类型监控, 邮件报警, 支持用户自定义 shell 扩展)


### 3. Monit 使用

#### 3.1 概述

想要让 Monit 可靠的为我们工作, 学习成本非常低, 只需要学习一些 Monit 命令行和配置文件写法

#### 3.2 命令行

##### 3.2.1 常用命令

```sh
# options - 选项
- monit
- monit -t
- monit -c /var/monit/monitrc  # 指定配置文件
- monit -g <groupname> start/stop # Monit 可以对各个监控分组, 如果需要对某个分组统一操作, 可以用这个命令

# arguments - 参数
- monit reload
- monit quit
- monit start/stop/restart/monitor/unmonitor <name>/all  # <name>: 每个监控都有一个独一无二的名字, 具体后面会提到; all: 所有监控服务
```

##### 3.2.2 全部命令文档

- [options][monit-doc-options]
- [arguments][monit-doc-args]

#### 3.3 配置文件

##### 3.3.1 概述

###### 1. 配置文件 monitrc 优先级

```sh
# Monit 查找配置文件的优先级按如下顺序

# 命令行指定
command-line

# 配置文件
~/.monitrc
/etc/monitrc
@sysconfdir@/monitrc  # 编译安装指定配置文件路径  eg: ./configure --sysconfdir /var/monit/etc
./monitrc
```

###### 2. 配置文件权限

- 不能超过 `0700` (u=xrw,g=,o=) 权限, 否则 Monit 会警告并退出

###### 3. 配置文件支持字符

- 关键字不区分大小写
- 3 种: 语法字段, 十进制数字, 字符串
- 字符串: 可以用双引号(可以包含空格)或者不用引号, 由字符和数字构成

###### 4. 配置文件分层

- 全局的 set 段
- 全局的 include 段
- 一个或多个 具体服务配置 段


##### 3.3.2 服务监控配置文件格式

详细配置, 共计 9 种, 所有配置中, 都符合以下规则

- 如果指定的 path 不存在, 而且配置块里包含 start 方法, 会调用这个 start 方法
- 如果 path 指定的文件类型不对, Monit 不能监控这个项目

###### 1. Process

```
CHECK PROCESS <unique name> <PIDFILE <path> | MATCHING <regex>>

<path> pid-file 的绝对路径. 不存在 pid-file 文件或者 pid-file 文件没有对应的正在运行的程序, Monit 会执行 start 方法

<regex> 进程名称的正则表达来监控进程, 可以通过命令行测试正则是否写对了: monit procmatch "regex-pattern"
```

###### 2. File

```
CHECK FILE <unique name> PATH <path>

<path> file 的绝对路径.
```

###### 3. Fifo

```
CHECK FIFO <unique name> PATH <path>
<path> fifo 的绝对路径.
```

###### 4. Filesystem

```
CHECK FILESYSTEM <unique name> PATH <path>
<path> 设备/磁盘, 挂载点的路径 或 NFS/CIFS/FUSE 链接字符串. 如果文件系统不可用, Monit 会执行 start 方法
```

###### 5. Directory

```
CHECK DIRECTORY <unique name> PATH <path>

<path> 目录问价的绝对路径
```

###### 6. Remote host

```
CHECK HOST <unique name> ADDRESS <host>

<host> 可以是域名或者 IP 地址. eg: "tildeslash.com" or "64.87.72.95".
```

###### 7. System

```
CHECK SYSTEM <unique name>

<unique name> 通常来说是本机名称(可以用 $HOST), 也可以是其他名称. 用于邮件报警或者 M/Monit 的初始化名称
这类配置可以监控系统资源(CPU, memory, load average...)
```

###### 8. Program

```
CHECK PROGRAM <unique name> PATH <executable file> [TIMEOUT <number> SECONDS]

<path> 可执行程序或脚本的绝对路径. 允许检查程序退出状态.如果程序没能在 <number> 秒内执行完成, Monit 会终结这个程序, 默认是 300s
程序的输出会被记录, 用于用户界面或者报警, 默认 512 bytes(可以通过 set limits 修改)
```

###### 9. Network

```
CHECK NETWORK <unique name> <ADDRESS <ipaddress> | INTERFACE <name>>

# <ipaddress> 是被监控的 IPv4/IPv6 网卡地址. 用 eth0 也是可以的
```

##### 3.3.3 全局配置

###### 1. 设置日志路径:
```
SET LOGFILE
```

###### 2. 守护进程模式:
```
SET DAEMON <seconds>
    [[WITH] START DELAY <seconds>]

第一个 <seconds> 监控周期
第二个 <seconds> 多少时间后开始监控 - 开机启动时候比较有用

命令行:
    monit - 如果已经有后台守护 Monit 进程, 发送唤醒信号给守护进程的 Monit, 立刻开始检查
    monit quit - 关闭后台守护 Monit 进程
```

###### 3. HTTPD

    - 可以通过过页面做一些操作(查看状态/控制监控服务)
        默认: http://localhost:2812/

    - 如果关闭可能影响某些功能
        monit status

    - 强烈建议开启
        如果安全敏感性比较高, 绑定本机访问
        可以设定只读账号 `read-only`

###### 4. 报警
Monit 提供邮件报警, 如果有其他报警方案, 可以通过自己实现 shell 脚本来扩展功能, 比如我们就通过 shell 脚本向第三方实时通讯软件 bearychat 发消息, 实现手机端的推送, 这里主要描述邮件报警的相关配置

- 设定/取消报警
```
set alert foo@bar  # 默认报警邮箱, 如果有多个, 可以写多个
set alert foo1@bar

check ....
    noalert foo@bar
```

- 自定义邮件格式
```
set mail-format {
    from: Monit Support <monit@foo.bar>
    reply-to: support@domain.com
    subject: $SERVICE $EVENT at $DATE
    message: Monit $ACTION $SERVICE at $DATE on $HOST: $DESCRIPTION.
             Yours sincerely,
             monit
}
```
可用变量
```
$EVENT
$SERVICE
$DATE
$HOST
$ACTION
$DESCRIPTION
```

- 配置邮件服务器和队列
```
需要配置邮件服务器和队列

SET MAILSERVER
<hostname|ip-address>
[PORT number]
[USERNAME string] [PASSWORD string]
[using SSL [with options {...}]
[CERTIFICATE CHECKSUM [MD5|SHA1] <hash>],
...
[with TIMEOUT X SECONDS]
[using HOSTNAME hostname]
```
多个服务器可以用逗号隔开, Monit 会依次尝试, 直到找到可用的
如果没有可用的邮件服务器, 会存在本地文件系统的队列里, 等待下一次尝试(队列内部信息是持久化的)

```
SET EVENTQUEUE BASEDIR <path> [SLOTS <number>]
```
<path> 存储队列的目录
<number> 限定队列的长度

**如果一台机器上运行多个 Monit 实例, 请确保队列使用不同的文件目录**

###### 5. 服务内 start/stop 方法

```
<START | STOP | RESTART> [PROGRAM] = "program"
    [[AS] UID <number | string>]
    [[AS] GID <number | string>]
    [[WITH] TIMEOUT <number> SECOND(S)]

check process mmonit with pidfile /usr/local/mmonit/mmonit/logs/mmonit.pid
    start program = "/usr/local/mmonit/bin/mmonit" as uid "mmonit" and gid "mmonit" with timeout 60 seconds
```
默认是 30s 的尝试 start/stop, 失败后会放弃尝试并打印错误信息, 全局设定是通过 `SET LIMITS`

###### 6. 监控分组

```
monit -g <groupname> start/stop/restart

check process mmonit with pidfile /usr/local/mmonit/mmonit/logs/mmonit.pid
    GROUP groupname
    GROUP groupname1
```

###### 7. 报警模式

```
MODE <ACTIVE | PASSIVE>
```
ACTIVE: 默认, 尝试重启服务, 发报警
PASSIVE: 不会尝试重启服务, 只会发报警

###### 8. 开机行为

```
ONREBOOT <START | NOSTART | LASTSTATE>
```
START: Monit 总是启动所有监控, 不论服务在重启前是否是停止的状态
NOSTART: 永远不会自动启动监控服务. 用于一些高可用场景(XXX)
```
这块理解描述可能不准, 原文:

In nostart mode, the service is never started automatically after reboot. This mode is intended for a high-availability solutions with active/passive clusters. For example, a service group HA, consisting of e.g. a mobile IP alias and an application server, is started on host H1, host H2 is backup and heartbeat is in place between both hosts. The service group HA must be started on one node only. If H1 dies, H2 takes over the HA group. If H1 reboots, it is important that it won't try to start the HA group also. Even though the group was active on H1 before it crashed, as HA is running on H2 now.
```
LASTSTATE: 保持之前状态

###### 9. 尝试服务重启重试次数

```
IF <number> RESTART <number> CYCLE(S) THEN <action>

if 2 restarts within 3 cycles then unmonitor
if 5 restarts within 5 cycles then exec "/foo/bar"
```

###### 10. 依赖

```
DEPENDS on service[, service [,...]]

WEB-SERVER(a) -> APPLICATION-SERVER(b) -> DATABASE(c) -> FILESYSTEM(d)

当前没有服务启动
    启动顺序: d, c, b, a

当前所有服务都启动
    (monit stop all) 停止顺序: a, b, c, d;
    (monit stop d) a, b, c 也会被停止, 因为依赖 d

如果 a 没有启动
    Monit 会启动 a

如果 b 没有启动
    Monit 会停止 a, 启动 b, 最后启动 a

如果依赖包含循环或者不包含被依赖的服务, 会通告并退出
```
可以同时依赖多个服务
在 stop/start/monitor/unmonitor 的时候会检查 DEPENDS 语句里的服务

如果服务 stop/unmonitor, 则会 stop/unmonitor 任何依赖这个服务的服务
如果服务 start, 所有该服务依赖的服务都会先 started, 然后在 start 这个服务
如果服务 restart, 关闭依赖该服务的所有服务, 在这个服务正常启动后, 会启动 restart 之前是活跃状态的服务

###### 11. Monit 本身相关的配置 LIMITS

```
SET LIMITS {
    PROGRAMOUTPUT:     <number> <unit>,
    SENDEXPECTBUFFER:  <number> <unit>,
    FILECONTENTBUFFER: <number> <unit>,
    HTTPCONTENTBUFFER: <number> <unit>,
    NETWORKTIMEOUT:    <number> <timeunit>
    PROGRAMTIMEOUT:    <number> <timeunit>
    STOPTIMEOUT:       <number> <timeunit>
    STARTTIMEOUT:      <number> <timeunit>
    RESTARTTIMEOUT:    <number> <timeunit>
}

unit is "B" (byte), "kB" (kilobyte) or "MB" (megabyte) timeunit is "MS" (millisecond) or "S" (second)
```

###### 12. ACTION

Monit 监控之后可以做的行为

```
ALERT: 发报警
RESTART: 重启并发报警(注册的 restart 方法, 如果没有, 则先 stop 再 start)
START: 启动并发报警(注册的 start 方法)
STOP: 关闭服务并发报警, 关闭之后不会再被 Monit 检查, 重启 Monit 也不会监控这个服务, 只能从网页或者控制台再次开启 (注册的 stop 方法)
EXEC: 执行指定的脚本并报警, 可以指定用户(需要以 root 权限启动), 可以设定多次检查周期作为一个周期
    if failed <test> then exec "/usr/local/bin/sms.sh"
        as uid nobody and gid nobody
        repeat every 5 cycles
UNMONITOR: 不再监控并发报警, 关闭之后不会再被 Monit 检查, 重启 Monit 也不会监控这个服务, 只能从网页或者控制台再次开启
```

###### 13.允许一定的监控公差

```
FOR

<X> CYCLES ...
or
<Y> [TIMES WITHIN] <Z> CYCLES ...


if failed
    port 80
    for 3 cycles
then alert

if failed
    port 80
    for 3 times within 5 cycles
then alert
```

<X> X 个周期连续符合条件
<Y/Z> Z 个周期内有 Y 次符合条件

**cycles 最大值为 64**

###### 14. 一些判断条件的语法, 写在 CHECK 中

```
# process/file/directory/filesystem/fifo
IF [DOES] NOT EXIST THEN <action>
IF [DOES] EXIST THEN <action>

# system/process
IF <resource> <operator> <value> THEN <action>
    <resource>:  "CPU", "TOTAL CPU", "CPU([user|system|wait])", "MEMORY", "SWAP", "THREADS", "CHILDREN", "TOTAL MEMORY", "LOADAVG([1min|5min|15min])"
    <operator>:  "<", ">", "!=", "==" in C notation, "gt", "lt", "eq", "ne" in shell sh notation and "greater", "less", "equal", "notequal" in human readable form (if not specified, default is EQUAL)

# process
IF DISK READ [RATE] <operator> <number> <unit>/S THEN action
IF DISK READ <operator> <number> operations/S THEN action
IF DISK WRITE <operator> <number> <unit>/S THEN action
IF DISK WRITE <operator> <number> operations/S THEN action

# file
IF FAILED [MD5|SHA1] CHECKSUM [EXPECT checksum] THEN action
IF CHANGED [MD5|SHA1] CHECKSUM THEN action
# test the content of a text file
IF CONTENT <operator> <regex|path> THEN action  # 默认只有 511 字符被监控, 可以用 limit 设置该值的大小
IGNORE CONTENT <operator> <regex|path>  # 被 IGNORE 命中的行不被监控

# file/fifo/directory
IF TIMESTAMP [[operator] value [unit]] THEN action
IF CHANGED TIMESTAMP THEN action

# file
IF SIZE [[operator] value [unit]] THEN action
IF CHANGED SIZE THEN action

# filesystem
IF CHANGED FSFLAGS THEN action
IF SPACE operator value unit THEN action
IF SPACE FREE operator value unit THEN action

# filesystem
IF INODE(S) operator value [unit] THEN action
IF INODE(S) FREE operator value [unit] THEN action

# filesystem
IF READ [RATE] <operator> <number> <unit>/S THEN action
IF READ [RATE] <operator> <number> operations/S THEN action
IF WRITE [RATE] <operator> <number> <unit>/S THEN action
IF WRITE [RATE] <operator> <number> operations/S THEN action

# Service Time is the time taken to complete a read or a write operation
IF SERVICE TIME <operator> <number> <unit> THEN action

# file/fifo/directory/filesystem
IF FAILED PERM(ISSION) octalnumber THEN action
IF CHANGED PERM(ISSION) THEN action

IF FAILED [E]UID user THEN action
IF FAILED GID group THEN action
IF CHANGED PID THEN action
IF CHANGED PPID THEN action

# process/system
IF UPTIME [[operator] value [unit]] THEN action

# program
IF STATUS operator value THEN action
IF CHANGED STATUS THEN action

# network
IF FAILED LINK THEN action
IF CHANGED LINK [CAPACITY] THEN action
IF SATURATION operator value% THEN action
IF UPLOAD operator value unit/S THEN action
IF DOWNLOAD operator value unit/S THEN action
IF TOTAL UPLOADED operator value unit IN LAST number time-unit THEN action
IF TOTAL DOWNLOADED operator value unit IN LAST number time-unit THEN action
IF UPLOAD operator value PACKETS/S THEN action
IF DOWNLOAD operator value PACKETS/S THEN action
IF TOTAL UPLOADED operator value PACKETS IN LAST number time-unit THEN action
IF TOTAL DOWNLOADED operator value PACKETS IN LAST number time-unit THEN action
IF FAILED PING[4|6]
    [COUNT number]
    [SIZE number]
    [TIMEOUT number SECONDS]
    [ADDRESS string]
THEN action

# process/host
## TCP/UDP
    IF FAILED
    [HOST string]
    <PORT number>
    [ADDRESS string]
    [IPV4 | IPV6]
    [TYPE <TCP|UDP>]
    [<SSL|TLS> [with options {...}]
    [CERTIFICATE CHECKSUM [MD5|SHA1] string]
    [CERTIFICATE VALID for number DAYS]
    [PROTOCOL protocol | <SEND|EXPECT> "string",...]
    [TIMEOUT number SECONDS]
    [RETRY number]
THEN action
## SOCKET
IF FAILED
    <UNIXSOCKET path>
    [TYPE <TCP|UDP>]
    [PROTOCOL protocol | <SEND|EXPECT> "string",...]
    [TIMEOUT number SECONDS]
    [RETRY number]
THEN action
## Specific protocol test options
[<SEND|EXPECT> "string"]+
PROTO(COL) HTTP
    [USERNAME "string"]
    [PASSWORD "string"]
    [REQUEST "string"]
    [STATUS operator number]
    [CHECKSUM checksum]
    [HTTP HEADERS list of headers]
    [CONTENT < "=" | "!=" > STRING]
## MySQL
PROTOCOL MYSQL [USERNAME string PASSWORD string]
## SIP
PROTOCOL SIP [TARGET valid@uri] [MAXFORWARD n]
## SMTP
PROTOCOL SMTP[S] [USERNAME string PASSWORD string]
PROTOCOL WEBSOCKET
[REQUEST string]
    [HOST string]
    [ORIGIN string]
    [VERSION number]
```

##### 3.3.4 include 配置

```
INCLUDE <globstring>
```

<globstring> 按照 glob(7) 规范, 如果匹配到目录而不是文件, 会忽略
被 include 的文件也可以 include 其他文件
如果 <globstring> 能匹配到多个, 引入没有特定的顺序, 如果你真的需要按照某些顺序, 请按顺序 include 每一条记录

#### 3.4 高级用法

##### 3.4.1 环境变量

你自己实现 shell 扩展的时候, 可以通过以下环境变量获取当前 Monit 监控到的问题的详细信息

```
- MONIT_SERVICE

# only available for service

- MONIT_DESCRIPTION
- MONIT_DATE
- MONIT_HOST

# only available for process

- MONIT_PROCESS_PID
- MONIT_PROCESS_MEMORY
- MONIT_PROCESS_CHILDREN
- MONIT_PROCESS_CPU_PERCENT

# only available for program

- MONIT_PROGRAM_STATUS
```

##### 3.4.2 信号量

- SIGUSR1 唤醒并检查所有检查项目
- SIGTERM/SIGINT 平滑关闭守护进程
- SIGHUP 重新初始化 -重新读取配置文件, 关闭并重开日志


### 4 更多

- [首页地址(包含简介 ppt)][monit-home]
- [官方文档地址][monit-doc]
- [官方样例地址][monit-examples]
- 本地官方样例:
    - Ubuntu 通过 apt 安装 Monit 后, 在 /etc/monit/conf-available 目录下有全部样例
    - CentOS 截止目前, yum 安装后没有样例

[monit-home]: https://mmonit.com/monit/
[monit-doc]: https://mmonit.com/monit/documentation/
[monit-doc-options]: https://mmonit.com/monit/documentation/monit.html#Options
[monit-doc-args]: https://mmonit.com/monit/documentation/monit.html#Arguments
[monit-examples]: https://mmonit.com/wiki/Monit/ConfigurationExamples
