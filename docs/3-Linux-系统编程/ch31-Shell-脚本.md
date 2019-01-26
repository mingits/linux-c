# 第 31 章 Shell 脚本

## 1. Shell 的历史

Shell 的作用是解释执行用户的命令，用户输入一条命令，Shell 就解释执行一条，这种方式称为交互式（Interactive），Shell 还有一种执行命令的方式称为批处理（Batch），用户事先写一个 Shell 脚本（Script），其中有很多条命令，让 Shell 一次把这些命令执行完，而不必一条一条地敲命令。Shell 脚本和编程语言很相似，也有变量和流程控制语句，但 Shell 脚本是解释执行的，不需要编译，Shell 程序从脚本中一行一行读取并执行这些命令，相当于一个用户把脚本中的命令一行一行敲到 Shell 提示符下执行。

由于历史原因，UNIX 系统上有很多种 Shell：

1. `sh`（Bourne Shell）：由 Steve Bourne 开发，各种 UNIX 系统都配有 `sh`。
2. `csh`（C Shell）：由 Bill Joy 开发，随 BSD UNIX 发布，它的流程控制语句很像 C 语言，支持很多 Bourne Shell 所不支持的功能：作业控制，命令历史，命令行编辑。
3. `ksh`（Korn Shell）：由 David Korn 开发，向后兼容 `sh` 的功能，并且添加了 `csh` 引入的新功能，是目前很多 UNIX 系统标准配置的 Shell，在这些系统上 `/bin/sh` 往往是指向 `/bin/ksh` 的符号链接。
4. `tcsh`（TENEX C Shell）：是 `csh` 的增强版本，引入了命令补全等功能，在 FreeBSD、Mac OS X 等系统上替代了 `csh`。
5. `bash`（Bourne Again Shell）：由 GNU 开发的 Shell，主要目标是与 POSIX 标准保持一致，同时兼顾对 `sh` 的兼容，`bash` 从 `csh` 和 `ksh` 借鉴了很多功能，是各种 Linux 发行版标准配置的 Shell，在 Linux 系统上 `/bin/sh` 往往是指向 `/bin/bash` 的符号链接<sup>[38]</sup>。虽然如此，`bash` 和 `sh` 还是有很多不同的，一方面，`bash` 扩展了一些命令和参数，另一方面，`bash` 并不完全和 `sh` 兼容，有些行为并不一致，所以 `bash` 需要模拟 `sh` 的行为：当我们通过 `sh` 这个程序名启动 `bash` 时，`bash` 可以假装自己是 `sh`，不认扩展的命令，并且行为与 `sh` 保持一致。

> <sup>[38]</sup> 最新的发行版有一些变化，例如 Ubuntu 7.10 的 `/bin/sh` 是指向 `/bin/dash` 的符号链接，`dash` 也是一种类似 `bash` 的 Shell。
> 
> ```bash
> $ ls /bin/sh /bin/dash -l
> -rwxr-xr-x 1 root root 79988 2008-03-12 19:22 /bin/dash
> lrwxrwxrwx 1 root root     4 2008-07-04 05:58 /bin/sh -> dash
> ```

文件 `/etc/shells` 给出了系统中所有已知（不一定已安装）的 Shell，除了上面提到的 Shell 之外还有很多变种。

```bash
# /etc/shells: valid login shells
/bin/csh
/bin/sh
/usr/bin/es
/usr/bin/ksh
/bin/ksh
/usr/bin/rc
/usr/bin/tcsh
/bin/tcsh
/usr/bin/esh
/bin/dash
/bin/bash
/bin/rbash
/usr/bin/screen
```

用户的默认 Shell 设置在 `/etc/passwd` 文件中，例如下面这行对用户 mia 的设置：

```bash
mia:L2NOfqdlPrHwE:504:504:Mia Maya:/home/mia:/bin/bash
```

用户 mia 从字符终端登录或者打开图形终端窗口时就会自动执行 `/bin/bash`。如果要切换到其它 Shell，可以在命令行输入程序名，例如：

```bash
~$ sh（在 bash 提示符下输入 sh 命令）
$（出现 sh 的提示符）
$（按 Ctrl-d 或者输入 exit 命令）
~$（回到 bash 提示符）
~$（再次按 Ctrl-d 或者输入 exit 命令会退出登录或者关闭图形终端窗口）
```

本章只介绍 `bash` 和 `sh` 的用法和相关语法，不介绍其它 Shell。所以下文提到 Shell 都是指 `bash` 或 `sh`。

## 2. Shell 如何执行命令

### 2.1. 执行交互式命令

用户在命令行输入命令后，一般情况下 Shell 会 `fork` 并 `exec` 该命令，但是 Shell 的内建命令例外，执行内建命令相当于调用Shell进程中的一个函数，并不创建新的进程。以前学过的 `cd`、`alias`、`umask`、`exit` 等命令即是内建命令，凡是用 `which` 命令查不到程序文件所在位置的命令都是内建命令，内建命令没有单独的 man 手册，要在 man 手册中查看内建命令，应该

```bash
$ man bash-builtins
```

本节会介绍很多内建命令，如 `export`、`shift`、`if`、`eval`、`[`、`for`、`while` 等等。内建命令虽然不创建新的进程，但也会有 Exit Status，通常也用 0 表示成功非零表示失败，虽然内建命令不创建新的进程，但执行结束后也会有一个状态码，也可以用特殊变量 `$?` 读出。

#### 习题

1. 在完成[第 30 章第 5 节 「练习：实现简单的 Shell」](3-Linux-系统编程/ch30-进程#_5-练习：实现简单的-shell)时也许有的读者已经试过了，在自己实现的 Shell 中不能执行 `cd` 命令，因为 `cd` 是一个内建命令，没有程序文件，不能用 `exec` 执行。现在请完善该程序，实现 `cd` 命令的功能，用 `chdir(2)` 函数可以改变进程的当前工作目录。
2. 思考一下，为什么 `cd` 命令要实现成内建命令？可不可以实现一个独立的 `cd` 程序，例如 `/bin/cd`，就像 `/bin/ls` 一样？

### 2.2. 执行脚本

首先编写一个简单的脚本，保存为 `script.sh`：

<p id="e31-1">例 31.1. 简单的 Shell 脚本</p>

```bash
#! /bin/sh

cd ..
ls
```

Shell 脚本中用 `#` 表示注释，相当于 C 语言的 `//` 注释。但如果 `#` 位于第一行开头，并且是 `#!`（称为 Shebang）则例外，它表示该脚本使用后面指定的解释器 `/bin/sh` 解释执行。如果把这个脚本文件加上可执行权限然后执行：

```bash
$ chmod +x script.sh
$ ./script.sh
```

Shell 会 `fork` 一个子进程并调用 `exec` 执行 `./script.sh` 这个程序，`exec` 系统调用应该把子进程的代码段替换成 `./script.sh` 程序的代码段，并从它的 `_start` 开始执行。然而 `script.sh` 是个文本文件，根本没有代码段和 `_start` 函数，怎么办呢？其实 `exec` 还有另外一种机制，如果要执行的是一个文本文件，并且第一行用 Shebang 指定了解释器，则用解释器程序的代码段替换当前进程，并且从解释器的 `_start` 开始执行，而这个文本文件被当作命令行参数传给解释器。因此，执行上述脚本相当于执行程序

```bash
$ /bin/sh ./script.sh
```

以这种方式执行不需要 `script.sh` 文件具有可执行权限。再举个例子，比如某个 `sed` 脚本的文件名是 `script`，它的开头是

```bash
#! /bin/sed -f
```

执行 `./script` 相当于执行程序

```bash
$ /bin/sed -f ./script.sh
```

以上介绍了两种执行 Shell 脚本的方法：

```bash
$ ./script.sh
$ sh ./script.sh
```

这两种方法本质上是一样的，执行上述脚本的步骤为：

<p id="c31-1">图 31.1. Shell脚本的执行过程</p>

![Shell 脚本的执行过程](../images/shellscript.shellexec.png)

1. 交互 Shell（`bash`）`fork`/`exec` 一个子 Shell（`sh`）用于执行脚本，父进程 `bash` 等待子进程 `sh` 终止。
2. `sh` 读取脚本中的 `cd ..` 命令，调用相应的函数执行内建命令，改变当前工作目录为上一级目录。
3. `sh` 读取脚本中的 `ls` 命令，`fork`/`exec` 这个程序，列出当前工作目录下的文件，`sh` 等待 `ls` 终止。
4. `ls` 终止后，`sh` 继续执行，读到脚本文件末尾，`sh` 终止。
5. `sh` 终止后，`bash` 继续执行，打印提示符等待用户输入。

如果将命令行下输入的命令用 `()` 括号括起来，那么也会 `fork` 出一个子 Shell 执行小括号中的命令，一行中可以输入由分号;隔开的多个命令，比如：

```bash
$ (cd ..;ls -l)
```

和上面两种方法执行 Shell 脚本的效果是相同的，`cd ..` 命令改变的是子 Shell 的 `PWD`，而不会影响到交互式 Shell。然而命令

```bash
$ cd ..;ls -l
```

则有不同的效果，`cd ..` 命令是直接在交互式 Shell 下执行的，改变交互式 Shell 的 `PWD`，然而这种方式相当于这样执行 Shell 脚本：

```bash
$ source ./script.sh
```

或者

```bash
$ . ./script.sh
```

`source` 或者 `.` 命令是 Shell 的内建命令，这种方式也不会创建子 Shell，而是直接在交互式 Shell 下逐行执行脚本中的命令。

#### 习题

1. 解释如下命令的执行过程：

```bash
$ (exit 2)
$ echo $?
2
```

## 3. Shell 的基本语法

### 3.1. 变量

按照惯例，Shell 变量由全大写字母加下划线组成，有两种类型的 Shell 变量：

- 环境变量：在[第 30 章「进程」第 2 节「环境变量」](3-Linux-系统编程/ch30-进程#_2-环境变量)中讲过，环境变量可以从父进程传给子进程，因此 Shell 进程的环境变量可以从当前 Shell 进程传给 `fork` 出来的子进程。用 `printenv` 命令可以显示当前 Shell 进程的环境变量。

- 本地变量：只存在于当前 Shell 进程，用 `set` 命令可以显示当前 Shell 进程中定义的所有变量（包括本地变量和环境变量）和函数。

环境变量是任何进程都有的概念，而本地变量是 Shell 特有的概念。在 Shell 中，环境变量和本地变量的定义和用法相似。在 Shell 中定义或赋值一个变量：

```bash
$ VARNAME=value
```

注意等号两边都不能有空格，否则会被 Shell 解释成命令和命令行参数。

一个变量定义后仅存在于当前 Shell 进程，它是本地变量，用 `export` 命令可以把本地变量导出为环境变量，定义和导出环境变量通常可以一步完成：

```bash
$ export VARNAME=value
```

也可以分两步完成：

```bash
$ VARNAME=value
$ export VARNAME
```

用 `unset` 命令可以删除已定义的环境变量或本地变量。

```bash
$ unset VARNAME
```

如果一个变量叫做 `VARNAME`，用 `${VARNAME}` 可以表示它的值，在不引起歧义的情况下也可以用 `$VARNAME` 表示它的值。通过以下例子比较这两种表示法的不同：

```bash
$ echo $SHELL
$ echo $SHELLabc
$ echo $SHELL abc
$ echo ${SHELL}abc
```

注意，在定义变量时不用$，取变量值时要用 `$`。和 C 语言不同的是，Shell 变量不需要明确定义类型，事实上 Shell 变量的值都是字符串，比如我们定义 `VAR=45`，其实 `VAR` 的值是字符串 `45` 而非整数。Shell 变量不需要先定义后使用，如果对一个没有定义的变量取值，则值为空字符串。

### 3.2. 文件名代换（Globbing）：* ? []

这些用于匹配的字符称为通配符（Wildcard），具体如下：

<p id="">表 31.1. 通配符</p>

| *          | 匹配 0 个或多个任意字符              |
| ---------- | ---------------------------------- |
| ?          | 匹配一个任意字符                   |
| [若干字符] | 匹配方括号中任意一个字符的一次出现 |

```bash
$ ls /dev/ttyS*
$ ls ch0?.doc
$ ls ch0[0-2].doc
$ ls ch[012][0-9].doc
```

注意，Globbing 所匹配的文件名是由 Shell 展开的，也就是说在参数还没传给程序之前已经展开了，比如上述 `ls ch0[012].doc` 命令，如果当前目录下有 `ch00.doc` 和 `ch02.doc`，则传给 `ls` 命令的参数实际上是这两个文件名，而不是一个匹配字符串。

### 3.3. 命令代换：` 或 $()

由反引号括起来的也是一条命令，Shell 先执行该命令，然后将输出结果立刻代换到当前命令行中。例如定义一个变量存放 `date` 命令的输出：

```bash
$ DATE=`date`
$ echo $DATE
```

命令代换也可以用 `$()` 表示：

```bash
$ DATE=$(date)
```

### 3.4. 算术代换：$(())

用于算术计算，`$(())` 中的 Shell 变量取值将转换成整数，例如：

```bash
$ VAR=45
$ echo $(($VAR+3))
```

`$(())` 中只能用 `+-*/` 和 `()` 运算符，并且只能做整数运算。

### 3.5. 转义字符 \

和 C 语言类似，\ 在 Shell 中被用作转义字符，用于去除紧跟其后的单个字符的特殊意义（回车除外），换句话说，紧跟其后的字符取字面值。例如：

```bash
$ echo $SHELL
/bin/bash
$ echo \$SHELL
$SHELL
$ echo \\
\
```

比如创建一个文件名为「$ $」的文件可以这样：

```bash
$ touch \$\ \$
```

还有一个字符虽然不具有特殊含义，但是要用它做文件名也很麻烦，就是 - 号。如果要创建一个文件名以 - 号开头的文件，这样是不行的：

```bash
$ touch -hello
touch: invalid option -- h
Try `touch --help' for more information.
```

即使加上 `\` 转义也还是报错：

```bash
$ touch \-hello
touch: invalid option -- h
Try `touch --help' for more information.
```

因为各种 UNIX 命令都把 - 号开头的命令行参数当作命令的选项，而不会当作文件名。如果非要处理以 - 号开头的文件名，可以有两种办法：

```bash
$ touch ./-hello
```

或者

```bash
$ touch -- -hello
```

\ 还有一种用法，在 \ 后敲回车表示续行，Shell 并不会立刻执行命令，而是把光标移到下一行，给出一个续行提示符 >，等待用户继续输入，最后把所有的续行接到一起当作一个命令执行。例如：

```bash
$ ls \
> -l
（ls -l命令的输出）
```

### 3.6. 单引号

和 C 语言不一样，Shell 脚本中的单引号和双引号一样都是字符串的界定符（双引号下一节介绍），而不是字符的界定符。单引号用于保持引号内所有字符的字面值，即使引号内的 \ 和回车也不例外，但是字符串中不能出现单引号。如果引号没有配对就输入回车，Shell 会给出续行提示符，要求用户把引号配上对。例如：

```bash
$ echo '$SHELL'
$SHELL
$ echo 'ABC\（回车）
> DE'（再按一次回车结束命令）
ABC\
DE
```

### 3.7. 双引号

双引号用于保持引号内所有字符的字面值（回车也不例外），但以下情况除外：

- $ 加变量名可以取变量的值
- 反引号仍表示命令替换
- \\$ 表示 $ 的字面值
- \\\` 表示 ` 的字面值
- \" 表示 " 的字面值
- \\\\ 表示 \ 的字面值
- 除以上情况之外，在其它字符前面的 \ 无特殊含义，只表示字面值

```bash
$ echo "$SHELL"
/bin/bash
$ echo "`date`"
Sun Apr 20 11:22:06 CEST 2003
$ echo "I'd say: \"Go for it\""
I'd say: "Go for it"
$ echo "\"（回车）
>"（再按一次回车结束命令）
"

$ echo "\\"
\
```

## 4. bash 启动脚本

启动脚本是 `bash` 启动时自动执行的脚本。用户可以把一些环境变量的设置和 `alias`、`umask` 设置放在启动脚本中，这样每次启动 Shell 时这些设置都自动生效。思考一下，`bash` 在执行启动脚本时是以 `fork` 子 Shell 方式执行的还是以 `source` 方式执行的？

启动 bash 的方法不同，执行启动脚本的步骤也不相同，具体可分为以下几种情况。

### 4.1. 作为交互登录 Shell 启动，或者使用 --login 参数启动

交互 Shell 是指用户在提示符下输命令的 Shell 而非执行脚本的 Shell，登录 Shell 就是在输入用户名和密码登录后得到的 Shell，比如从字符终端登录或者用 `telnet`/`ssh` 从远程登录，但是从图形界面的窗口管理器登录之后会显示桌面而不会产生登录 Shell（也不会执行启动脚本），在图形界面下打开终端窗口得到的 Shell 也不是登录 Shell。

这样启动 `bash` 会自动执行以下脚本：

1. 首先执行 `/etc/profile`，系统中每个用户登录时都要执行这个脚本，如果系统管理员希望某个设置对所有用户都生效，可以写在这个脚本里
2. 然后依次查找当前用户主目录的 `~/.bash_profile`、`~/.bash_login` 和 `~/.profile` 三个文件，找到第一个存在并且可读的文件来执行，如果希望某个设置只对当前用户生效，可以写在这个脚本里，由于这个脚本在 `/etc/profile` 之后执行，`/etc/profile` 设置的一些环境变量的值在这个脚本中可以修改，也就是说，当前用户的设置可以覆盖（Override）系统中全局的设置。`~/.profile` 这个启动脚本是 `sh` 规定的，`bash` 规定首先查找以 `~/.bash_` 开头的启动脚本，如果没有则执行 `~/.profile`，是为了和 `sh` 保持一致。
3. 顺便一提，在退出登录时会执行 `~/.bash_logout` 脚本（如果它存在的话）。

### 4.2. 以交互非登录 Shell 启动

比如在图形界面下开一个终端窗口，或者在登录 Shell 提示符下再输入 `bash` 命令，就得到一个交互非登录的 Shell，这种 Shell 在启动时自动执行 `~/.bashrc` 脚本。

为了使登录 Shell 也能自动执行 `~/.bashrc`，通常在 `~/.bash_profile` 中调用 `~/.bashrc`：

```bash
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```

这几行的意思是如果 `~/.bashrc` 文件存在则 `source` 它。多数 Linux 发行版在创建帐户时会自动创建 `~/.bash_profile` 和 `~/.bashrc` 脚本，`~/.bash_profile` 中通常都有上面这几行。所以，如果要在启动脚本中做某些设置，使它在图形终端窗口和字符终端的 Shell 中都起作用，最好就是在 `~/.bashrc` 中设置。

下面做一个实验，在 `~/.bashrc` 文件末尾添加一行（如果这个文件不存在就创建它）：

```bash
export PATH=$PATH:/home/akaedu
```

然后关掉终端窗口重新打开，或者从字符终端 `logout` 之后重新登录，现在主目录下的程序应该可以直接输程序名运行而不必输入路径了，例如：

```bash
~$ a.out
```

就可以了，而不必

```bash
~$ ./a.out
```

为什么登录 Shell 和非登录 Shell 的启动脚本要区分开呢？最初的设计是这样考虑的，如果从字符终端或者远程登录，那么登录 Shell 是该用户的所有其它进程的父进程，也是其它子 Shell 的父进程，所以环境变量在登录 Shell 的启动脚本里设置一次就可以自动带到其它非登录 Shell 里，而 Shell 的本地变量、函数、`alias` 等设置没有办法带到子 Shell 里，需要每次启动非登录 Shell 时设置一遍，所以就需要有非登录 Shell 的启动脚本，所以一般来说在 `~/.bash_profile` 里设置环境变量，在 `~/.bashrc` 里设置本地变量、函数、`alias` 等。如果你的 Linux 带有图形系统则不能这样设置，由于从图形界面的窗口管理器登录并不会产生登录 Shell，所以环境变量也应该在 `~/.bashrc` 里设置。 

### 4.3. 非交互启动

为执行脚本而 `fork` 出来的子 Shell 是非交互 Shell，启动时执行的脚本文件由环境变量 `BASH_ENV` 定义，相当于自动执行以下命令：

```bash
if [ -n "$BASH_ENV" ]; then . "$BASH_ENV"; fi
```

如果环境变量 `BASH_ENV` 的值不是空字符串，则把它的值当作启动脚本的文件名，`source` 这个脚本。

### 4.4. 以 sh 命令启动

如果以 `sh` 命令启动 `bash`，`bash` 将模拟 `sh` 的行为，以 `~/.bash_` 开头的那些启动脚本就不认了。所以，如果作为交互登录 Shell 启动，或者使用 `--login` 参数启动，则依次执行以下脚本：

1. `/etc/profile`
2. `~/.profile`

如果作为交互 Shell 启动，相当于自动执行以下命令：

```bash
if [ -n "$ENV" ]; then . "$ENV"; fi
```

如果作为非交互 Shell 启动，则不执行任何启动脚本。通常我们写的 Shell 脚本都以 `#! /bin/sh` 开头，都属于这种方式。

## 5. Shell 脚本语法

### 5.1. 条件测试：test [

命令 `test` 或 `[` 可以测试一个条件是否成立，如果测试结果为真，则该命令的 Exit Status 为 0，如果测试结果为假，则命令的 Exit Status 为 1（注意与 C 语言的逻辑表示正好相反）。例如测试两个数的大小关系：

```bash
$ VAR=2
$ test $VAR -gt 1
$ echo $?
0
$ test $VAR -gt 3
$ echo $?
1
$ [ $VAR -gt 3 ]
$ echo $?
1
```

**虽然看起来很奇怪，但左方括号 `[` 确实是一个命令的名字，传给命令的各参数之间应该用空格隔开**，比如，`$VAR`、`-gt`、`3`、`]` 是 `[` 命令的四个参数，它们之间必须用空格隔开。命令 `test` 或 `[` 的参数形式是相同的，只不过 `test` 命令不需要 `]` 参数。以 `[` 命令为例，常见的测试命令如下表所示：

<p id="t31.2">表 31.2. 测试命令</p>

| `[ -d DIR ]`             | 如果 `DIR` 存在并且是一个目录则为真                            |
| ------------------------ | ------------------------------------------------------------ |
| `[ -f FILE ]`            | 如果 `FILE` 存在且是一个普通文件则为真                         |
| `[ -z STRING ]`          | 如果 `STRING` 的长度为零则为真                                 |
| `[ -n STRING ]`          | 如果 `STRING` 的长度非零则为真                                 |
| `[ STRING1 = STRING2 ]`  | 如果两个字符串相同则为真                                     |
| `[ STRING1 != STRING2 ]` | 如果字符串不相同则为真                                       |
| `[ ARG1 OP ARG2 ]`       | `ARG1` 和 `ARG2` 应该是整数或者取值为整数的变量，`OP` 是 `-eq`（等于）`-ne`（不等于）`-lt`（小于）`-le`（小于等于）`-gt`（大于）`-ge`（大于等于）之中的一个 |

和 C 语言类似，测试条件之间还可以做与、或、非逻辑运算：

<p id="t31.3">表 31.3. 带与、或、非的测试命令</p>

| `[ ! EXPR ]`         | `EXPR` 可以是上表中的任意一种测试条件，! 表示逻辑反            |
| -------------------- | ------------------------------------------------------------ |
| `[ EXPR1 -a EXPR2 ]` | `EXPR1` 和 `EXPR2` 可以是上表中的任意一种测试条件，`-a` 表示逻辑与 |
| `[ EXPR1 -o EXPR2 ]` | `EXPR1` 和 `EXPR2` 可以是上表中的任意一种测试条件，`-o` 表示逻辑或 |

例如：

```bash
$ VAR=abc
$ [ -d Desktop -a $VAR = 'abc' ]
$ echo $?
0
```

注意，如果上例中的 `$VAR` 变量事先没有定义，则被 Shell 展开为空字符串，会造成测试条件的语法错误（展开为 `[ -d Desktop -a  = 'abc' ]`），作为一种好的 Shell 编程习惯，应该总是把变量取值放在双引号之中（展开为 `[ -d Desktop -a "" = 'abc' ]`）：

```bash
$ unset VAR
$ [ -d Desktop -a $VAR = 'abc' ]
bash: [: too many arguments
$ [ -d Desktop -a "$VAR" = 'abc' ]
$ echo $?
1
```

### 5.2. if/then/elif/else/fi

和 C 语言类似，在 Shell 中用 `if`、`then`、`elif`、`else`、`fi` 这几条命令实现分支控制。这种流程控制语句本质上也是由若干条 Shell 命令组成的，例如先前讲过的

```bash
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```

其实是三条命令，`if [ -f ~/.bashrc ]` 是第一条，`then . ~/.bashrc` 是第二条，`fi` 是第三条。如果两条命令写在同一行则需要用 `;` 号隔开，一行只写一条命令就不需要写 `;` 号了，另外，`then` 后面有换行，但这条命令没写完，Shell 会自动续行，把下一行接在 `then` 后面当作一条命令处理。和 `[` 命令一样，要注意命令和各参数之间必须用空格隔开。`if` 命令的参数组成一条子命令，如果该子命令的 Exit Status 为 0（表示真），则执行 `then` 后面的子命令，如果 Exit Status 非 0（表示假），则执行 `elif`、`else` 或者 `fi` 后面的子命令。`if` 后面的子命令通常是测试命令，但也可以是其它命令。Shell 脚本没有 `{}`v括号，所以用 `fi` 表示 `if` 语句块的结束。见下例：

```bash
#! /bin/sh

if [ -f /bin/bash ]
then echo "/bin/bash is a file"
else echo "/bin/bash is NOT a file"
fi
if :; then echo "always true"; fi
```

`:` 是一个特殊的命令，称为空命令，该命令不做任何事，但 Exit Status 总是真。此外，也可以执行 `/bin/true` 或 `/bin/false` 得到真或假的 Exit Status。再看一个例子：

```bash
#! /bin/sh

echo "Is it morning? Please answer yes or no."
read YES_OR_NO
if [ "$YES_OR_NO" = "yes" ]; then
  echo "Good morning!"
elif [ "$YES_OR_NO" = "no" ]; then
  echo "Good afternoon!"
else
  echo "Sorry, $YES_OR_NO not recognized. Enter yes or no."
  exit 1
fi
exit 0
```

上例中的 `read` 命令的作用是等待用户输入一行字符串，将该字符串存到一个 Shell 变量中。

此外，Shell 还提供了 `&&` 和 `||` 语法，和 C 语言类似，具有 Short-circuit 特性，很多 Shell 脚本喜欢写成这样：

```bash
test "$(whoami)" != 'root' && (echo you are using a non-privileged account; exit 1)
```

`&&` 相当于 `if...then...`，而 `||` 相当于 `if not...then...`。`&&` 和 `||` 用于连接两个命令，而上面讲的 `-a` 和 `-o` 仅用于在测试表达式中连接两个测试条件，要注意它们的区别，例如，

```bash
test "$VAR" -gt 1 -a "$VAR" -lt 3
```

和以下写法是等价的

```bash
test "$VAR" -gt 1 && test "$VAR" -lt 3
```

### 5.3. case/esac

`case` 命令可类比 C 语言的 `switch`/`case` 语句，`esac` 表示 `case` 语句块的结束。C 语言的 `case` 只能匹配整型或字符型常量表达式，而 Shell 脚本的 `case` 可以匹配字符串和 Wildcard，每个匹配分支可以有若干条命令，末尾必须以 `;;` 结束，执行时找到第一个匹配的分支并执行相应的命令，然后直接跳到 `esac` 之后，不需要像 C 语言一样用 `break` 跳出。

```bash
#! /bin/sh

echo "Is it morning? Please answer yes or no."
read YES_OR_NO
case "$YES_OR_NO" in
yes|y|Yes|YES)
  echo "Good Morning!";;
[nN]*)
  echo "Good Afternoon!";;
*)
  echo "Sorry, $YES_OR_NO not recognized. Enter yes or no."
  exit 1;;
esac
exit 0
```

使用 `case` 语句的例子可以在系统服务的脚本目录 `/etc/init.d` 中找到。这个目录下的脚本大多具有这种形式（以 `/etc/apache2` 为例）：

```bash
case $1 in
	start)
		...
	;;
	stop)
		...
	;;
	reload | force-reload)
		...
	;;
	restart)
	...
	*)
		log_success_msg "Usage: /etc/init.d/apache2 {start|stop|restart|reload|force-reload|start-htcacheclean|stop-htcacheclean}"
		exit 1
	;;
esac
```

启动 `apache2` 服务的命令是

```bash
$ sudo /etc/init.d/apache2 start
```

`$1` 是一个特殊变量，在执行脚本时自动取值为第一个命令行参数，也就是 `start`，所以进入 `start)` 分支执行相关的命令。同理，命令行参数指定为 `stop`、`reload` 或 `restart` 可以进入其它分支执行停止服务、重新加载配置文件或重新启动服务的相关命令。

### 5.4. for/do/done

Shell 脚本的 `for` 循环结构和 C 语言很不一样，它类似于某些编程语言的 `foreach` 循环。例如：

```bash
#! /bin/sh

for FRUIT in apple banana pear; do
  echo "I like $FRUIT"
done
```

`FRUIT` 是一个循环变量，第一次循环 `$FRUIT` 的取值是 `apple`，第二次取值是 `banana`，第三次取值是 `pear`。再比如，要将当前目录下的 `chap0`、`chap1`、`chap2`等文件名改为 `chap0~`、`chap1~`、`chap2~` 等（按惯例，末尾有 `~` 字符的文件名表示临时文件），这个命令可以这样写：

```bash
$ for FILENAME in chap?; do mv $FILENAME $FILENAME~; done
```

也可以这样写：

```bash
$ for FILENAME in `ls chap?`; do mv $FILENAME $FILENAME~; done
```

### 5.5. while/do/done

`while` 的用法和 C 语言类似。比如一个验证密码的脚本：

```bash
#! /bin/sh

echo "Enter password:"
read TRY
while [ "$TRY" != "secret" ]; do
  echo "Sorry, try again"
  read TRY
done
```

下面的例子通过算术运算控制循环的次数：

```bash
#! /bin/sh

COUNTER=1
while [ "$COUNTER" -lt 10 ]; do
  echo "Here we go again"
  COUNTER=$(($COUNTER+1))
done
```

Shell 还有 `until` 循环，类似 C 语言的 `do...while` 循环。本章从略。

#### 习题

1. 把上面验证密码的程序修改一下，如果用户输错五次密码就报错退出。

### 5.6. 位置参数和特殊变量

有很多特殊变量是被Shell自动赋值的，我们已经遇到了 `$?` 和 `$1`，现在总结一下：

<p id="t31-4">表 31.4. 常用的位置参数和特殊变量</p>

| `$0`          | 相当于 C 语言 `main` 函数的 `argv[0]`                             |
| ------------- | ------------------------------------------------------------ |
| `$1`、`$2`... | 这些称为位置参数（Positional Parameter），相当于 C 语言 `main` 函数的 `argv[1]`、`argv[2]`... |
| `$#`          | 相当于 C 语言 `main` 函数的 `argc - 1`，注意这里的 `#` 后面不表示注释 |
| `$@`          | 表示参数列表 `"$1" "$2" ...`，例如可以用在 `for` 循环中的 `in` 后面。 |
| `$?`          | 上一条命令的 Exit Status                                      |
| `$$`          | 当前 Shell 的进程号                                            |

位置参数可以用 `shift` 命令左移。比如 `shift 3` 表示原来的 `$4` 现在变成 `$1`，原来的 `$5` 现在变成 `$2` 等等，原来的 `$1`、`$2`、`$3` 丢弃，`$0` 不移动。不带参数的 `shift` 命令相当于 `shift 1`。例如：

```bash
#! /bin/sh

echo "The program $0 is now running"
echo "The first parameter is $1"
echo "The second parameter is $2"
echo "The parameter list is $@"
shift
echo "The first parameter is $1"
echo "The second parameter is $2"
echo "The parameter list is $@"
```

### 5.7. 函数

和 C 语言类似，Shell 中也有函数的概念，但是函数定义中没有返回值也没有参数列表。例如：

```bash
#! /bin/sh

foo(){ echo "Function foo is called";}
echo "-=start=-"
foo
echo "-=end=-"
```

注意函数体的左花括号 `{` 和后面的命令之间必须有空格或换行，如果将最后一条命令和右花括号 `}` 写在同一行，命令末尾必须有 `;` 号。

在定义 `foo()` 函数时并不执行函数体中的命令，就像定义变量一样，只是给 `foo` 这个名字一个定义，到后面调用 `foo` 函数的时候（注意 Shell 中的函数调用不写括号）才执行函数体中的命令。Shell 脚本中的函数必须先定义后调用，一般把函数定义都写在脚本的前面，把函数调用和其它命令写在脚本的最后（类似 C 语言中的 `main` 函数，这才是整个脚本实际开始执行命令的地方）。

Shell 函数没有参数列表并不表示不能传参数，事实上，函数就像是迷你脚本，调用函数时可以传任意个参数，在函数内同样是用 `$0`、`$1`、`$2` 等变量来提取参数，函数中的位置参数相当于函数的局部变量，改变这些变量并不会影响函数外面的 `$0`、`$1`、`$2` 等变量。函数中可以用 `return` 命令返回，如果 `return` 后面跟一个数字则表示函数的 Exit Status。

下面这个脚本可以一次创建多个目录，各目录名通过命令行参数传入，脚本逐个测试各目录是否存在，如果目录不存在，首先打印信息然后试着创建该目录。

```bash
#! /bin/sh

is_directory()
{
  DIR_NAME=$1
  if [ ! -d $DIR_NAME ]; then
    return 1
  else
    return 0
  fi
}

for DIR in "$@"; do
  if is_directory "$DIR"
  then :
  else
    echo "$DIR doesn't exist. Creating it now..."
    mkdir $DIR > /dev/null 2>&1
    if [ $? -ne 0 ]; then
      echo "Cannot create directory $DIR"
      exit 1
    fi
  fi
done
```

注意 `is_directory()` 返回 0 表示真返回 1 表示假。

## 6. Shell 脚本的调试方法

Shell 提供了一些用于调试脚本的选项，如下所示：

- -n：读一遍脚本中的命令但不执行，用于检查脚本中的语法错误
- -v： 一边执行脚本，一边将执行过的脚本命令打印到标准错误输出
- -x：提供跟踪执行信息，将执行的每一条命令和结果依次打印出来

使用这些选项有三种方法，一是在命令行提供参数

```bash
$ sh -x ./script.sh
```

二是在脚本开头提供参数

```bash
#! /bin/sh -x
```

第三种方法是在脚本中用 set 命令启用或禁用参数

```bash
#! /bin/sh
if [ -z "$1" ]; then
  set -x
  echo "ERROR: Insufficient Args."
  exit 1
  set +x
fi
```

`set -x` 和 `set +x` 分别表示启用和禁用 `-x` 参数，这样可以只对脚本中的某一段进行跟踪调试。