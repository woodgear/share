> [The collapse of complex software](https://nolanlawson.com/2022/06/09/the-collapse-of-complex-software/ "The collapse of complex software")


## 什么是lua级别的火焰图，为什么我们要关心它
![nginx火焰图](https://cong-storage.oss-cn-hangzhou.aliyuncs.com/sm/pj/share/ngshark/imgs/ngx.png)

![openresty lua 火焰图](https://cong-storage.oss-cn-hangzhou.aliyuncs.com/sm/pj/share/ngshark/imgs/lua.f.png)

作为一个在k8s上的网关，[alb](https://github.com/alauda/alb.git "alb 灵雀云开源k8s网关")关注性能，我们是基于openresty的框架在nginx上做开发，那么因为我自己的lua代码带来的性能损耗有多少？分别在那些地方？我们很关注。

如果直接从nginx火焰图来看，其实很难找到重点。使用perf，我们可以很容易的画出任意进程的火焰图，但是我们的lua代码在哪里？perf会解析调用栈的符号，对c函数很有用，但是对lua函数则不那么直观。

我们需要的是openresty的lua代码火焰图，从这里我们能很容易看出一些奇怪的地方，为什么log的时候处理prometheus metrics的部分花了这么长时间？

## 原理/用到的工具
openresty社区画火焰图的工具是[stapxx](https://github.com/Kong/stapxx?tab=readme-ov-file#lj-lua-stacks "stapxx")中的lj-lua-stacks。

它的原理可以概括为：借助systemtap在perf时注入代码到nginx进程上，遍历luajit中的lua函数调用堆栈，解析lua函数名，从而生成lua级别的火焰图。

下面会介绍下具体的技术细节。
### systemtap
首先要介绍的是systemtap，它是redhat于2009年推出的动态追踪工具，能够让我们在用户函数和内核函数上执行自己的代码。

从现在的角度来看，它和bpftrace的定位有些类似，但是不同于把代码运行在内核中的确保安全的ebpf虚拟机上。

systemtap实际上是一个多趟编译器，它会把stap脚本语言编译成c，在编译成内核模块，最后在内核中运行。
```stap
# the stap script lang.
index = @var("ngx_http_module", "/xxx/nginx")->index
```

```c
# struct of ngx_http_module
struct ngx_module_s {
    ngx_uint_t            ctx_index;
    ngx_uint_t            index;
    char                 *name;
```

```c
# the kernel module c code which stap compiled to.
l->__retvalue = uderef(8, ((((((int64_t) (/* pragma:vma */ (
{
 unsigned long addr = 0; 
 addr = _stp_umodule_relocate ("/usr/local/openresty/nginx/sbin/nginx", 0x2704e0, current);
 addr;
}
))))) + (((int64_t)8LL)))));
```
stap脚本语言中提供很多有用的语法糖，比如@var，指定一个可执行文件和其中的全局变量，在stap中就可以直接访问这个这个变量的任意属性。

stap编译器会帮我们找到这个变量在进程中的真正的地址，解析这个变量所代表的数据结构的内存布局，最终真正找到我们想去访问的字段的偏移值，使用uderef这种直接读取用户进程内存的系统调用，来获取我们想访问的，ngx_http_module的index。

### stapxx
```stapxx
index = @var("ngx_http_module", "$^exe_path")->index
 V
index = @var("ngx_http_module", "/xxx/nginx")->index
```
```bash
sudo stap \
  -k \ -x $NG_PID \
  -d $target/nginx/sbin/nginx \
  -d $target/luajit/lib/libluajit-5.1.so.2.1.0 \
  -d /usr/lib/ld-linux-x86-64.so.2 \
  ... 省略
  ./all-in-one.stap
```
在真正的场景中，我们更多的是通过stapxx中的[stap++](https://github.com/Kong/stapxx/blob/kong/stap%2B%2B "stap++")来写stap脚本。


直接写stap脚本需要指定具体的可执行文件路径，运行时 参数上也要用-d指定用到的所有符号文件。

stapxx会自动做些 变量替换/文件合并/符号查找 之类的事，用起来方便一些。基本上只要指定nginx pid就行了。

lj-lua-stacks就是这样的一个stap++脚本。

## lua级别的火焰图
### nginx/openresty/luajit 在进程内存中的数据结构
![](https://cong-storage.oss-cn-hangzhou.aliyuncs.com/sm/pj/share/ngshark/imgs/ngx.drawio.svg)


每个nginx的worker的根都是一个ngx_cycle,里面有所有nginx模块的指针，其中就有ngx_http_module，在其中就是openresty的ngx_http_lua.

在ngx_http_lua中的lua指针，指向的是lua_state,一个代表luavm所有状态的结构。

对于一个luavm来说，有一个所有协程共享的global_state,每个协程有一个自己的lua_state,在global_state上cur_L指向当前在执行的那个协程的lua_state。

我们当前执行的函数可能是lua函数（纯lua或者被jit编译的lua），可能是c函数。

在纯lua函数的情况下，cur_L的stack和base指向的指针之间的地址范围就是我们函数调用堆栈的地址范围

对于jit编译过的tace，global state的jit_base到cur_L的stack是他的函数调用栈的地址范围。

总的来讲，在任意的时刻，我们到一个nginx进程上，总是能通过ngx_cycle开始找到当前正在执行的lua函数的调用栈。

### lua vm callstacks
![height:600px](https://cong-storage.oss-cn-hangzhou.aliyuncs.com/sm/pj/share/ngshark/imgs/lua-frames.drawio.svg)
对于每个调用栈的每一层都是一个frame，每个frame都相当于链表中的一环指向了下一个frame。

对于每个frame我们可以通过他指向的gcobj指向的gcfunc,如果是lua函数的话，可以最终获取这个函数的lua文件名和这个函数在lua文件中的行号。chunkname就是文件名。

每个gcfunc,如果是c函数的话，我们可以通过usysname来获取c函数名。

实际上这里最终获取的是lua文件名和函数行号，在之后还需要根据这两个找到lua函数名。才能真正的生成火焰图。

### show me the code
这就是luajit_debug_dumpstack的具体过程。有兴趣的同学可以读下面的代码。

```stap
function luajit_debug_dumpstack(L, T, depth, base, simple)
    bot = $*L->stack->ptr64 + @sizeof_TValue //@LJ_FR2
    for (nextframe = frame = base - @sizeof_TValue; frame > bot; ) {
        if (@frame_gc(frame) == L) { tmp_level++ }
        if (tmp_level-- == 0) {
            size = (nextframe - frame) / @sizeof_TValue
            found_frame = 1
            break
        }
        nextframe = frame
        if (@frame_islua(frame)) {
            frame = @frame_prevl(frame)
        } else {
            if (@frame_isvarg(frame)) { tmp_level++;}
            frame = @frame_prevd(frame); } 
        }

    if (!found_frame) { frame = 0 size = tmp_level }
    if (frame) {
        nextframe = size ? frame + size * @sizeof_TValue : 0
        fn = luajit_frame_func(frame)
        if (@isluafunc(fn)) {
            pt = @funcproto(fn)
            line = luajit_debug_frameline(L, T, fn, pt, nextframe)
            name = luajit_proto_chunkname(pt)  /* GCstr *name */
            path = luajit_unbox_gcstr(name)
            bt .= sprintf("%s:%d\n", path, line)
        } 
    } else if (dir == 1) { break } else { level -= size }
```
### 谜题揭晓
发现了问题其实就很容易解决问题，经过排查我们最终发现lua火焰图中metrics部分处理耗时长的原因是因为在我们使用的prometheus库没有对多线程做优化。

在升级了依赖之后，metrics部分的损耗就到了一个可以接受的程度了。
## 畅想未来 ngshark?
> 随着软件架构的越来越复杂，完全的理解业务过程中，从用户使用到系统内核的每个关键路径的每个链条的每个细节越来越难。软件技术栈所搭建的巨塔越垒越高。
我们需要更多的可观测性，照亮层层抽象导致的深渊。

### systemtap的问题所在
systemtap有一个无法规避的问题就是他最终是用内核模块跑的，你甚至可以在他的脚本语言中写原生的c函数，这就导致了他如果爆炸的话，会把整个内核都炸掉。

虽然它自己在各种编译的过程中会做各种严格的检查，但拥有这种可能性，本身就是问题。

可以发现，实际上我们要做的就是
1. 在一个内存空间内，顺着指针读取各种结构中的字段，从而遍历调用栈
2. 想办法把每次调用栈的函数名拿到。

理论上完全可以用ebpf来做。所以都2024年了我们为什么不用ebpf重写呢？

### build block 
我们所需要是
1. 将ebpf attach到perf event上去，这一点ebpf可以做到了，我们可以把一个ebpf program attach到一个perf 上去。
2. 知道每个变量的位置，每个要访问的字段在内存中的具体位置。
#### pahole
关于每个字段在结构体中的offset，其实已经已dwarf格式存在了可执行文件的符号表中。手动写一个解析器，虽然有pyelftools这种库，但是还是很麻烦。

所幸，我们有[pahole](https://linux.die.net/man/1/pahole "pahole")。
给定可执行文件和想查询的结构体，它会生成这些结构体的头文件。  
所以实际上我们就可以这样直接把头文件include进来，然后开始读取字段。
```bash
pahole --compile -C GCobj,GG_State,lua_State,global_State /xx/luajit/lib/libluajit-5.1.so.2.1.0 >$out
sed -i '/.*typedef.*__uint64_t.*/d' $out
sed -i '/.*typedef.*__int64_t.*/d' $out
sed -i 's/Node/LJNode/g' $out
```
```c
struct global_State {
	lua_Alloc                  allocf;               /*     0     8 */
	void *                     allocd;               /*     8     8 */
	GCState                    gc;                   /*    16   104 */
	/* --- cacheline 1 boundary (64 bytes) was 56 bytes ago --- */
	GCstr                      strempty;             /*   120    24 */
	/* --- cacheline 2 boundary (128 bytes) was 16 bytes ago --- */
	uint8_t                    stremptyz;            /*   144     1 */
    // ... 省略
}
```
#### maybe rewrite via ebpf
作为一个小demo，理论上我们可以在ebpf中将stap++中的函数一一对应的翻译回c，用ebpf的方式做一个ngshark。
```c
#include <nginx.h>
#define READ_SRTUCT(ret, ret_t, p, type, access)            \
    do {                                                \
        type val;                                   \
        bpf_probe_read_user(&val, sizeof(type), p); \
        ret = (ret_t)((val)access);                 \
    } while (0)
void *GLP = (void *)0x7cc2e558c380; // TODO
void *luajit_G(){
    void *ret;
    READ_SRTUCT(ret, void *, GLP, lua_State, .glref.ptr64);
    return ret;
}
void *luajit_cur_thread(void *g){
    void *gco;
    size_t offset = offsetof(struct global_State, cur_L);
    READ_SRTUCT(gco, void *, g + offset, struct GCRef, .gcptr64);
    // gco is a union, th is lua_State and the point of th is gco itself
    // return &@cast(gco, "GCobj", "")->th
    return gco;
}
```


