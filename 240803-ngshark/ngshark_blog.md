## 什么是lua级别的火焰图，为什么我们关心它
我们可以直接用perf去抓nginx的火焰图，结果也就是上面这一张，但是这个显示的都是nginx中c函数的火焰图。对于我们的问题，我们自己的业务的lua代码到底慢在哪里，其实没什么帮助。我们需要的是下面这一张，lua代码级别的火焰图，比如这里可以很明显的看出来，为什么吐prometheus metrics的部分占了大头？
## 原理/用到的工具
### systemtap
systemtap是redhat出的一个定位类似于bpftrace的动态追踪工具，但是要比ebpf出现早很多。在09年就发布了。最小支持的版本是linux 3.10。
他能让我们注入自己的代码到内核函数或者用户函数上。
实际上他是一个多趟编译器，把stap脚本语言编译成c，在编译成内核模块，然后运行。
比如这里 nginx中ngx_http_module 这个全局变量是一个ngx_modules，他的index代表的是他在所有nginx模块中的idnex,有了这个index之后我们可以在ngx_modules这个数组里访问到他了。
对stap来讲，他会到内核中，在nginx的进程的task上，直接去读这个地址的8byte,后面的这个8l指的就是index这个字段相对于这个结构体头部的偏移值,我们可以看到这个偏移值是直接写死的，其实是stap在编译的时候自己去解析了dwarf，自己算出来的.
### stapxx
stapxx 又在systemtap上包了一层，因为直接写stap需要指定具体的elf文件路径，运行时参数上也要用-d在指定elf文件。stapxx会自动做些变量替换 文件合并 符号查找 之类的事，用起来方便一些。基本上只要指定nginx pid就行了。
他实际上是一个不到600行的perl脚本。逻辑并不复杂。
举例来讲的话就是这里会把exe_path根据pid替换成真正的文件路径
### systemtap的小tips
这里额外插播一个systemtap的小tips，如果你直接去probe一个docker中的容器，大概率会报错说找不到文件。
这个因为容器中的进程是chnge root过的，跑在overlayfs上，而stap去找符号表是用的主机的root。
这个问题有挺多种解决方法的，比如在主机上对应的位置把容器里的文件拷贝一份。
但是一个比较简单粗暴的方式是直接让systemtap把容器的mergedir当root。然后让他不要真的去这下面找内核模块就行。
比如这里我们通过sysroot告诉systemtap把这个路径当作root，然后通过直接改掉systemtap的代码让他还用主机上的内核模块。

## lua级别的火焰图
### nginx/luajit 在进程内存中的数据结构
好，终于到我们关心的部分了。当我们在抓火焰图时，我们实际上是用perf子系统在每次perf event时，到nginx进程上执行代码。

每个nginx的worker的根都是一个ngx_cycle,里面有所有nginx模块的指针，其中就有ngx_http_module，在其中就是openresty的ngx_http_lua.

在ngx_http_lua中的lua指针，指向的是lua_state,一个代表luavm所有状态的结构。
对于一个luavm来说，有一个所有协程共享的global_state,每个协程有一个自己的lua_state,在global_state上cur_L指向当前在执行的那个协程的lua_state

我们当前执行的函数可能是lua函数（纯lua或者被jit编译的lua），可能是c函数

在纯lua函数的情况下，cur_L的stack和base指向的指针之间的地址范围就是我们函数调用堆栈的地址范围

对于jit编译过的tace，global state的jit_base到cur_L的stack是他的函数调用栈的地址范围。
总的来讲，在任意的时刻，我们到一个nginx进程上，总是能通过ngx_cycle开始找到当前正在执行的lua函数的调用栈。

对于每个调用栈的每一层都是一个frame，每个frame都相当于链表中的一环指向了下一个frame
对于每个frame我们可以通过他指向的gcobj指向的gcfunc,如果是lua函数的话，可以最终获取这个函数的lua文件名和这个函数在lua文件中的行号。
chunkname就是文件名。
每个gcfunc 如果是c函数的话，我们可以通过un-symbol-name 来获取c函数名。
刚才这两张图所描述的其实就是luajit_debug_dumpstack的执行过程
实际上这里最终获取的是lua文件名和函数行号，在之后还需要根据这两个找到lua函数名。
## 畅想未来 ngshark?
### systemtap的问题所在
systemtap有一个无法规避的问题就是他最终是用内核模块跑的，你可以在他的脚本语言中写原生的c函数，这就导致了他如果爆炸的话，会把整个内核都炸掉。
虽然他自己在各种编译的过程中会做检查，但是只要不小心，还是能炸掉内核的。
可以发现，实际上我们要做的就是在一个内存空间内，顺着指针读取各种结构中的字段，从而遍历调用栈，想办法把每次调用栈的函数名拿到。
理论上完全可以用ebpf来做。所以都2024年了我们为什么不用ebpf重写呢？
### build block 
我们所需要是
1. 将ebpf attach到perf event上去，这一点ebpf可以做到了，我们可以把一个ebpf program attach到一个perf 上去。
2. 知道每个变量的位置，每个要访问的字段在内存中的具体位置。
#### pahole
关于每个字段在结构体中的offset，我们需要解析elf的dwarf信息。
pahole 帮我们做了这件事情。
给定elf文件和想查询的结构体，他会生成这些结构体的头文件。
所以实际上我们就可以这样直接把头文件include进来，然后开始读取字段。
#### dwarf parser
#### 一个最基本的示例