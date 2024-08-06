##
大家好，我是来自灵雀云的xx，负责在灵雀云维护我们基于openresty的k8s上的4层和7层网关alauda-alb。所有我们平台产品的系统组件都会用到这个我们的alb。当然也作为产品提供给了用户，作为他们自己的应用网关来使用。

@next

今天的分享主要分为两个部分。1是作为网关来讲，我们比较关心的性能问题，openresty的性能，特别是我们自己写的lua代码的性能。目前大家应该都是在用lua-stacks来画火焰图，这里分享下他的原理。
2是我们自己是怎么用openresty的，包括我们自己的开发过程中一些最佳实践和在k8s上用它做网关的一些分享
##
@next
我们可以直接用perf去抓nginx的火焰图，结果也就是上面这一张，但是这个显示的都是nginx中c函数的火焰图。对于我们的问题，我们自己的业务的lua代码到底慢在哪里，其实没什么帮助。我们需要的是下面这一张，lua代码级别的火焰图，比如这里可以很明显的看出来，为什么吐prometheus metrics的部分占了大头？
## 
@next

首先介绍下systemtap
systemtap是redhat出的一个定位类似于bpftrace的动态追踪工具，但是要比ebpf出现早很多。在09年就发布了。最小支持的版本是linux 3.10。
他能让我们注入自己的代码到内核函数或者用户函数上。
实际上他是一个多趟编译器，把stap脚本语言编译成c，在编译成内核模块，然后运行。

比如这里 nginx中ngx_http_module 这个全局变量是一个ngx_modules，他的index代表的是他在所有nginx模块中的idnex,有了这个index之后我们可以在ngx_modules这个数组里访问到他了。

对stap来讲，他会到内核中，在nginx的进程的task上，直接去读这个地址的8byte,后面的这个8l指的就是index这个字段相对于这个结构体头部的偏移值,我们可以看到这个偏移值是直接写死的，其实是stap在编译的时候自己去解析了dwarf，自己算出来的.
##
@next

stapxx 又在systemtap上包了一层，因为直接写stap需要指定具体的elf文件路径，运行时参数上也要用-d在指定elf文件。stapxx会自动做些变量替换 文件合并 符号查找 之类的事，用起来方便一些。基本上只要指定nginx pid就行了。
他实际上是一个不到600行的perl脚本。逻辑并不复杂。
举例来讲的话就是这里会把exe_path根据pid替换成真正的文件路径
## 
@next

这里额外插播一个systemtap的小tips，如果你直接去probe一个docker中的容器，大概率会报错说找不到文件。
这个因为容器中的进程是chnge root过的，跑在overlayfs上，而stap去找符号表是用的主机的root。
这个问题有挺多种解决方法的，比如在主机上对应的位置把容器里的文件拷贝一份。
但是一个比较简单粗暴的方式是直接让systemtap把容器的mergedir当root。然后让他不要真的去这下面找内核模块就行。
比如这里我们通过sysroot告诉systemtap把这个路径当作root，然后通过直接改掉systemtap的代码让他还用主机上的内核模块。

##
@next

好，终于到我们关心的部分了。当我们在抓火焰图时，我们实际上是用perf子系统在每次perf event时，到nginx进程上执行代码。

每个nginx的worker的根都是一个ngx_cycle,里面有所有nginx模块的指针，其中就有ngx_http_module，在其中就是openresty的ngx_http_lua.

在ngx_http_lua中的lua指针，指向的是lua_state,一个代表luavm所有状态的结构。
对于一个luavm来说，有一个所有协程共享的global_state,每个协程有一个自己的lua_state,在global_state上cur_L指向当前在执行的那个协程的lua_state

我们当前执行的函数可能是lua函数（纯lua或者被jit编译的lua），可能是c函数

在纯lua函数的情况下，cur_L的stack和base指向的指针之间的地址范围就是我们函数调用堆栈的地址范围

对于jit编译过的tace，global state的jit_base到cur_L的stack是他的函数调用栈的地址范围。
总的来讲，在任意的时刻，我们到一个nginx进程上，总是能通过ngx_cycle开始找到当前正在执行的lua函数的调用栈。
##
@next
对于每个调用栈的每一层都是一个frame，每个frame都相当于链表中的一环指向了下一个frame

对于每个frame我们可以通过他指向的gcobj指向的gcfunc,如果是lua函数的话，可以最终获取这个函数的lua文件名和这个函数在lua文件中的行号。
chunkname就是文件名。
每个gcfunc 如果是c函数的话，我们可以通过un-symbol-name 来获取c函数名。

@next

刚才这两张图所描述的其实就是luajit_debug_dumpstack的执行过程

实际上这里最终获取的是lua文件名和函数行号，在之后还需要根据这两个找到lua函数名。

##
@next
systemtap有一个无法规避的问题就是他最终是用内核模块跑的，你可以在他的脚本语言中写原生的c函数，这就导致了他如果爆炸的话，会把整个内核都炸掉。
虽然他自己在各种编译的过程中会做检查，但是只要不小心，还是能炸掉内核的。
可以发现，实际上我们要做的就是在一个内存空间内，顺着指针读取各种结构中的字段，从而遍历调用栈，想办法把每次调用栈的函数名拿到。
理论上完全可以用ebpf来做。所以都2024年了我们为什么不用ebpf重写呢？

我们所需要是
1. 将ebpf attach到perf event上去，这一点ebpf可以做到了，我们可以把一个ebpf program attach到一个perf 上去。
2. 知道每个变量的位置，每个要访问的字段在内存中的具体位置。

@next

关于每个字段在结构体中的offset，我们需要解析elf的dwarf信息。
pahole 帮我们做了这件事情。
给定elf文件和想查询的结构体，他会生成这些结构体的头文件。

@next

所以实际上我们就可以这样直接把头文件include进来，然后开始读取字段。

## 
@next

火焰图的部分就是这样了。

接下来的部分就是我们是怎么样使用openresty的。
alb是一个k8s上的网关，其实就是用户的流量入口，负责把流量转到对应的pod上。
@next

我们没有用纯lua来写所有的逻辑，主要是用go掉k8s api很方便，库很多。
因此天然的将程序分成了两个部分，go 的部分负责从k8s 收集信息 生成配置文件给openresty，在发现用户添加一个新的端口，或者用户自己注入了一些配置，导致nginx配置变化时，reload一下。相当于一个控制面。
我们是直接把流量转到pod ip上去的，这样就让我门自己做一些iphash之类的轮询策略。

@next

lua部分则负责将配置同步到各个worker，在请求来时候，每个worker上的luavm就能根据配置做各种处理。修改请求，设置upstream为期望pod ip之类的。
cache flow我们的实现比较简单，借助mlcache将配置从全局词典中读到各worker的词典上去来避免每次都去抢锁带来额外的性能损耗。

顺便一提最开始的部分metrics的部分性能有很大的损耗主要就是我们用的库比较老，实际上每次写日志都去更新一个全局的词典了，导致每次都抢锁了。
##
@next
这样两个世界的边界和责任划分的就很明显了。问题是如何把他们在粘合起来。
现在来讲的话，lua的开发确实是比之前好很多了。比如我们有luslsp这种lanuange-server这种，能在每个ide上直接安装使用，也有cli的工具可以调用。
他有一些type annotation的能力，能够给我们一些类型上的保证，最起码能够检测到访问了不存在的属性这种问题。
我们自己写了一个go的代码生成器tylua,类似生成crd一样，能从go结构直接生成lualsp的类型注释。
比如说这里的otelconfig，左边的go代码和右边的注释就是一一对应的。这样后面我们做重构的时候就更有底气了。
实际上因为luajit停在了lua5.1，所以理论上各种类似typescript一样给lua加类型校验的新的方言，我们都可以加上，多多益善嘛。
比如这个luau就是游戏界他们搞出来的，像是想给lua加类型，看起来就很有趣。
@next
真正写lua的体验，很有go的既视感。他们都是最朴素的通过多返回值的方式来做显示的错误检查。
用了lualsp之后，像这种checknill的检查，和基本的类型检查就都有了。
这样就是很好的消除不同语言的交界的问题
##
@next

关于如何测试lua代码，社区有test-nginx的测试框架可以直接启动nginx，他推荐的方式应该是在perl中通过test-nginx定义好的几个原语来声明测试，但是我觉的使用起来还是有点不方便.
比如我之前有个需求是我想用curl去测https，但是我们的证书是自签名的，然后发现他不支持，最后还是给test-nginx提了个pr才行。
所以觉得写起来比较爽的方式是test-nginx作为一个nginx booter来使用，尽量的把测试的逻辑用纯lua来写。这样的话ide的支持也比较好。

具体来讲就是，在xx.t 也就是在test-nginx的测试的perl文件中声明这个测试case的lua文件

@next

在这个lua文件中，去设置规则，用resty-http去发起请求，检查返回值之类的

@next

实际上，我们就是单纯的定义了一个测试专用的端口，当/t请求来时，调用lua代码。

## 
@next
除来网关的业务逻辑代码之外，对我们来讲还有一个问题是如何把openresty部署起来。
用chart可以，但给用户用，过于细节了。而且chart的values的类型是什么？
所以我们是有一个operator专门负责部署的。
他会根据alb cr，去创建deployment，如果用户需要使用容器网络，还会自动创建lbsvc，并把分配到的地址更新到alb自己的status上。
对用户来讲，我们希望他操作是我们提供的alb的抽象。

对于gateway api也是类似，operator在看到用户部署了指向我们的gateway之后，会自动创建alb。

具体的gateway-api的配置是由启动起来的那个alb自己处理。

## 
@next

在这种架构下，我们就可以比较灵活的支持一些用户的需求。
比如多租户，用户可能多个项目共用一个alb，通过端口来进行隔离。
只有某些ns下的ingress才可以给某些alb使用等。
当前端发生变化时，比如支持gatewayapi,ingress，基本只有改动go的部分。openresty的部分都是通用的。
