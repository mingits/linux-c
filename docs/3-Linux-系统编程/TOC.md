# 部分 III. Linux 系统编程

目录

28. 文件与 I/O
    1. 汇编程序的 Hello world
    2. C 标准 I/O 库函数与 Unbuffered I/O 函数
    3. open/close
    4. read/write
    5. lseek
    6. fcntl
    7. ioctl
    8. mmap
29. 文件系统
    1. 引言
    2. ext2 文件系统
        1. 总体存储布局
        2. 实例剖析
        3. 数据块寻址
        4. 文件和目录操作的系统函数
    3. VFS
        1. 内核数据结构
        2. dup 和 dup2 函数
30. 进程
    1. 引言
    2. 环境变量
    3. 进程控制
        1. fork 函数
        2. exec 函数
        3. wait 和 waitpid 函数
    4. 进程间通信
        1. 管道
        2. 其它 IPC 机制
    5. 练习：实现简单的 Shell
31. Shell 脚本
    1. Shell 的历史
    2. Shell 如何执行命令
        1. 执行交互式命令
        2. 执行脚本
    3. Shell的基本语法
        1. 变量
        2. 文件名代换（Globbing）： * ? []
        3. 命令代换： ` 或 $()
        4. 算术代换： $(())
        5. 转义字符 \
        6. 单引号
        7. 双引号
    4. bash 启动脚本
        1. 作为交互登录 Shell 启动，或者使用 --login 参数启动
        2. 以交互非登录 Shell 启动
        3. 非交互启动
        4. 以 sh 命令启动
    5. Shell 脚本语法
        1. 条件测试： test [
        2. if/then/elif/else/fi
        3. case/esac
        4. for/do/done
        5. while/do/done
        6. 位置参数和特殊变量
        7. 函数
    6. Shell 脚本的调试方法
32. 正则表达式
    1. 引言
    2. 基本语法
    3. sed
    4. awk
    5. 练习：在 C 语言中使用正则表达式
33. 信号
    1. 信号的基本概念
    2. 产生信号
        1. 通过终端按键产生信号
        2. 调用系统函数向进程发信号
        3. 由软件条件产生信号
    3. 阻塞信号
        1. 信号在内核中的表示
        2. 信号集操作函数
        3. sigprocmask
        4. sigpending
    4. 捕捉信号
        1. 内核如何实现信号的捕捉
        2. sigaction
        3. pause
        4. 可重入函数
        5. sig_atomic_t 类型与 volatile 限定符
        6. 竞态条件与 sigsuspend 函数
        7. 关于 SIGCHLD 信号
34. 终端、作业控制与守护进程
    1. 终端
        1. 终端的基本概念
        2. 终端登录过程
        3. 网络登录过程
    2. 作业控制
        1. Session 与进程组
        2. 与作业控制有关的信号
    3. 守护进程
35. 线程
    1. 线程的概念
    2. 线程控制
        1. 创建线程
        2. 终止线程
    3. 线程间同步
        1. mutex
        2. Condition Variable
        3. Semaphore
        4. 其它线程间同步机制
    4. 编程练习
36. TCP/IP 协议基础
    1. TCP/IP 协议栈与数据包封装
    2. 以太网 (RFC 894) 帧格式
    3. ARP 数据报格式
    4. IP 数据报格式
    5. IP 地址与路由
    6. UDP 段格式
    7. TCP 协议
        1. 段格式
        2. 通讯时序
        3. 流量控制
37. socket 编程
    1. 预备知识
        1. 网络字节序
        2. socket 地址的数据类型及相关函数
    2. 基于 TCP 协议的网络程序
        1. 最简单的 TCP 网络程序
        2. 错误处理与读写控制
        3. 把 client 改为交互式输入
        4. 使用 fork 并发处理多个 client 的请求
        5. setsockopt
        6. 使用 select
    3. 基于 UDP 协议的网络程序
    4. UNIX Domain Socket IPC
    5. 练习：实现简单的 Web 服务器
        1. 基本 HTTP 协议
        2. 执行 CGI 程序