# 前言

[gdb](https://www.sourceware.org/gdb/) 想必大家都听说过，作为最有名气的调试器之一，却由于其在命令行中运行的特点导致了周围用的人并不算多，加之现代编辑器中的调试功能已经成为标配，在很多时候用自带的调试器就能完成调试了。

但我依旧认为 gdb 十分重要，一是因为如果是多文件编程就不能用 VSCode 中自带的调试器（其实也是 gdb ，套了层 gui ，或许改下 json 能用，但是不打算改），二是因为市面上不同编辑器所带的调试功能用法虽然大同，但总是小异。特别是 SCNU 机房的电脑默认 VSCode 配置无法直接调试，但默认装有 gdb（神奇） ，就不用在期末考试时候换用不熟悉的 VS || DevC++ 了 ：）

# 正文

## 前期准备

- gdb 在启动时会首先弹一个版权信息，很烦，建议在 `~/.zshrc` || `~/.bashrc` 中添加 `alias`
    `alias gdb="gdb -q"` 或者启动时添加 `-q` 选项 `q for quiet`
    
- gdb 在退出时会弹一个确认信息，如果你觉得烦，也可以新建文件 `~/.gdbinit` 并添加
    `set confirm off`
    顺带一提，`~/.gdbinit` 文件如其名，是gdb在启动时会运行的命令，借此能够实现对 gdb 的扩展，如 [pwndbg](https://github.com/pwndbg/pwndbg)
    
- **编译时需要添加特殊参数 `-g` 代表这个程序编译时要留下调试信息**
    例如：`g++ main.cpp -o main -Wall -g`
    P.S. 似乎使用了 `-O3` 编译参数会对 gdb 产生影响？
    > The [gcc docs](http://gcc.gnu.org/onlinedocs/libstdc++/manual/debug.html) note that the default debug flags are `g` and `O2`, and using `g -O0 -fno-inline` disables any optimization and function inlining.
    这种情况使用 `-Og` 参数来避免影响（或者干脆 `-O0` ）

## 调试的理念

💡 这部分是我自己的理解，调试用的还少，学的很浅，希望有人补充

先稍微讲下我对调试的理解

一整个软件 | 代码是十分庞大的，人脑处理不过来，更别说各类循环嵌套个三四层（my shitty code would be like）之后基本上就很难判断中间过程实现的对错了。

而最基本的调试就是把大块的代码分开，在运行过程中暂停，能够看到中间变量值来判断实现的对错（breakpoint | print）

更进一步的调试则是底层方面的调试（栈，[栈帧](https://stackoverflow.com/questions/10057443/explain-the-concept-of-a-stack-frame-in-a-nutshell)，内存地址，call stack，汇编，寄存器 etc.）等不那么显而易见的调试

本文只涉及最基本调试

💡 在下面操作前，先 `gdb program_name` 进入 gdb 界面


## 断点

### 生成断点

从上面的理解来看，调试至少要有运行过程中暂停的功能，这个功能就是 `breakpoint` aka 断点

用法1： `break $line_number` || `break 9` 

这时候虽然你看不见，但是 gdb 已经打好断点了 `Breakpoint 1 at 0x5d94: file main.cpp, line 10.` 

不过有两点点需要注意，第一点如上文，输入的是第9行处打断点，实际上断点产生在第10行，这是因为 gdb 会自动找第一个有效执行语句而忽略其他预处理语句如 `include namespace define etc.` .第二点是如果在想在多文件编程时给其他文件打断点，需要使用 `break file_name.c:60` 

用法2：`break $function_nmae` || `break main`

直接找到函数打断点，适合不知道在第几行但是知道函数名的更多数情况，在使用函数模块化功能以及分文件编程时十分好用！

打断点什么都好，就有一点，break 这个单词太长了，使用频率太高了，虽然 **gdb 提供了 tab 自动补全**，但还是麻烦，懒是人类第一生产力，gdb 内置了部分使用频率高的功能的缩写。简单来说，对于 `break` 命令直接输 `b` 也是等价的 

用法3： `b 9` || `b main`

打完断点之后在 gdb 中直接输入 `run` 就行了，这就能在断点处暂停执行了，想要执行下一步需要使用 `next` || `step` || `continue`，将在稍后详细对比

### 删除断点

要是断点给错，用 `delete` 命令删除，缩写为 `d`

用法： `delete $breakpoint_index` || `delete 2`

注意这里的 2 不是行号，而是**断点序号**，如果忘记断点序号，使用 `info break` 查看。相似的，如果想看自己打了哪些断点，也可以使用 `info break` 来查看。

### 代码对照

上面提到打断点要知道行号或者函数名，那不得编辑器和 gdb 分屏看？太麻烦了！懒又是人类第一生产力，gdb 其实自带了代码和 gdb 命令行的对照

用法：`gdb -tui program_name` （在运行 gdb 时添加参数），也可以在 gdb 中 `ctrl-x` 进入， `ctrl-a` 退出。

![能够展示断点位置，当前执行情况，以及代码的对照](https://s2.loli.net/2021/12/30/kDN1KYVLzq3hXZP.png)

## 运行程序

在打完断点后就能运行了。在 gdb 中直接输入 `r` 就行（`run` 的缩写），这就能在断点处暂停执行了，想要执行下一步需要使用 `next` || `step` || `continue`，将在此详细对比

### Step

运行下一步代码，同时遇到自己定义的函数**将会**追踪进入函数，进一步debug。缩写为 `s`

### Next

运行下一步代码，但在遇到自己定义的函数**不会**追踪进入函数，而是直接获取返回值往下运行。简单来说，全程 `next` 只会执行完 `main` 函数，而 `step` 将会执行完每一行代码。缩写为 `n`

### Continue

运行到 breakpoint 或者运行完整个程序。缩写为 `c`

💡 实际上，上面的介绍顺序是按运行完程序所需操作数来排序的

### Until

until 运行至某行停止 (until reach the line) ，一般用来跳出循环，缩写为 `u`

用法：`u $line_number` || `u 60`

同时，在 gdb 下直接回车的作用是重复上一指令，对于单步调试（也就是`step` || `next` 这样一步步进行的调试）非常方便

## 变量值查看

打完断点，用 `step` || `next` || `continue` 执行下一步，不会给你任何有效信息，你还需要手动选择变量值进行查看。有两种方式，`print` || `watch`

### Print

单次打印一个值，但不仅是变量值，可以是任何 C 语言的有效表达式，包括数字，变量甚至是函数调用。缩写为 `p`

```
p a：将显示整数 a 的值
p ++a：将把 a 中的值加 1, 并显示出来
p name：将显示字符串 name 的值
p gdb_test(22)：将以整数 22 作为参数调用 gdb_test() 函数
p gdb_test(a)：将以变量 a 作为参数调用 gdb_test() 函数
```

### Watch

使用 `watch` 命令设置观察点，也就是当一个变量值发生变化时，程序会停下来。缩写为 `wa` 

用法：`watch a` 当 a 发生改变时就会停止

---

至此，基础的 gdb 调试就算介绍完了，退出使用 `quit` || `q` 即可

## 功能补充

### Info

用于查看各种信息

```
info function // 列出可执行文件的所有函数名称，可以用来打断点
info functions $regex // 支持正则表达式的搜索函数名
info breakpoints // 列出断点
info locals // 打印当前函数局部变量的值
info args // 打印传入函数的参数 arguments aka 实参
```

### Backtrace

一般程序崩溃时只会告诉你 Segmentation Fault ，通过 backtrace 可以知道时哪一行出现错误，缩写为 `bt`

![故意写错尝试下就知道了](https://s2.loli.net/2021/12/30/pfoIDcVnFd7Cwms.png)

## 其他补充

`start` vs `run`

> `run` will start (or restart) the program from the beginning and continue execution until a breakpoint is hit or the program exits. `start` will start (or restart) the program and stop execution at the beginning of the `main` function.
> 

`finish` 

> `continue` (shorthand: `c`) will resume execution until the next breakpoint is hit or the program exits. `finish` will run until the current function call completes and stop there. You can single-step through the C source using the `next` (shorthand: `n`) or `step` (shorthand: `s`) commands, both of which execute a line and stop. The difference between these two is that if the line to be executed is a function call, `next` executes the *entire function*, but `step` goes into the function implementation and stops at the first line.
> 

缩写一览，从缩写反学或许是个不错的主意

> b -> break
c -> continue
d -> delete
f -> frame
i -> info
j -> jump
l -> list
n -> next
p -> print
r -> run
s -> step
u -> until
aw -> awatch
bt -> backtrace
dir -> directory
disas -> disassemble
fin -> finish
ig -> ignore
ni -> nexti
rw -> rwatch
si -> stepi
tb -> tbreak
wa -> watch
win -> winheight
> 

当数值到 X 时设置断点，在断点设置后加 `if`

> `break iter.c:6 if i == 5` || `break main:looping if i == 5`

[CS107 Software Testing Strategies](https://web.stanford.edu/class/archive/cs/cs107/cs107.1194/testing.html)

# 参考资料

[1. gdb 调试利器 - Linux Tools Quick Tutorial](https://linuxtools-rst.readthedocs.io/zh_CN/latest/tool/gdb.html)

[100个gdb小技巧](https://wizardforcel.gitbooks.io/100-gdb-tips/content/)

[CS107 GDB and Debugging](https://web.stanford.edu/class/archive/cs/cs107/cs107.1194/resources/gdb)

[GDB: break if variable equal value](https://stackoverflow.com/questions/14390256/gdb-break-if-variable-equal-value)

最后预告下，寒假期间应该会写 Missing Semester 相关的系列文章，计划一天一篇，暂定内容为
- Command Line 
- git
- regex
- vim
- docker
- Misc

看看能不能成为国清师兄的寒假 Missing Semester 计划的补充-。-
<!--stackedit_data:
eyJoaXN0b3J5IjpbNDYwMDg3MzcyXX0=
-->