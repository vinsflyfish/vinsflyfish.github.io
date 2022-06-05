---
title: "Libco 协程栈的切换理解"
date: 2018-04-26T18:58:35+08:00
draft: false
tags:
  - "协程"

---

# libco协程切换原理解读及简要使用
以前看过libco一点源码，最近组里面分享了一次协程的原理。花了点功夫，借助一点网上的资料，算是摸清楚了libco协程切换的来龙去脉。libco除了协程的切换还涉及系统hook以及相关工程的封装，篇幅及时间限制，这里不涉及。本篇主要把协程切换的来龙去脉以及原理从个人理解角度介绍下。明白和能说出来讲清楚是两种不同的理解程度，这也是本文的主要目的。

## 函数调用的原理
### linux 程序内存布局
传统linux程序（32bit)拥有4G的虚拟内存区域，高1G的区域供内核使用，剩余的3G内存供程序使用。按段划分，主要分程序段（text segement)、数据段、BSS段。BSS段用于未初始化的静态变量的初始化（0值初始化）。栈从高到低地址增长。堆从低到高增长。栈和堆的这两种不同的地址增长方向，需要关注下，后面协程切换中就涉及到该布局的不同。对内存布局有兴趣可以参看这篇文章[程序内存布局](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)。

![linux程序段内存布局](/img/linuxFlexibleAddressSpaceLayout.png)

### 栈帧定义
> In C and modern CPU design conventions, the stack frame is a chunk of memory, allocated from the stack, at run-time, each time a function is called, to store its automatic variables. Hence nested or recursive calls to the same function, each successively obtain their own separate frames.

> Physically, a function's stack frame is the area between the addresses contained in esp, the stack pointer, and ebp, the frame pointer (base pointer in Intel terminology). Thus, if a function pushes more values onto the stack, it is effectively growing its frame.

这里定义很清楚，栈帧是从栈上分配的一段内存，每次函数调用时，用于存储自动变量。从物理介质角度看，栈帧是位于esp（栈指针）及ebp（基指针）之间的一块区域。局部变量等分配均在栈帧上分配，函数结束自动释放。

> Traditionally, C function calls are made with the caller pushing some parameters onto the stack, calling the function, and then popping the stack to clean up those pushed arguments.

C函数调用，调用者将一些参数放在栈上，调用函数，然后弹出栈上存放的参数。

这里值得一提的是，调用约定，__cdecl 、 __stdcall、 __fastcall，调用约定涉及参数的入栈顺序（从左到右还是从右到左）、参数入栈和清理的是调用者(caller)还是被调用者(callee)，函数名的处理。详细的文档可以参看[Win32 calling conventions](http://www.unixwiz.net/techtips/win32-callconv.html)、[The history of calling conventions](https://blogs.msdn.microsoft.com/oldnewthing/20040108-00/?p=41163)

> __cdecl is the default calling convention for C and C++ programs. [参考地址](https://msdn.microsoft.com/en-us/library/zkwh89ks.aspx)

这里我们看一下一个栈帧在内存中的布局：

![栈帧内存布局](/img/frame_memory_layout.png)

红色区域是进入新的函数中时，栈帧的内存布局。这里需要关注红色区域之上的参数2，参数1，和返回地址。即每次进行函数调用时，采用__cdecl调用约定的调用者会将参数从右到左的入栈，最后将返回地址入栈。这个返回地址是指，函数调用结束后的下一行执行的代码地址。获取参数及返回地址通过EBP相对偏移即可获得，图示为32位系统对应的偏移值。

## libco协程切换汇编代码解析
基本概念介绍的有些仓促，详细的可以参看推荐的链接。这里借助协程切换的汇编，继续回顾函数调用的相关知识。这里协程切换出于简化的目的，仅介绍32位系统下的汇编，64位下寄存器多一些寄存器，参数传递约定也有所区别，原理类似。

这里介绍下，协程切换的核心函数coctx_swap，这个函数底层由汇编实现，是进行协程切换的核心，也是本文关注的核心。大部分自定义的协程库，在这个精简的函数基础上可以进行封装。

```
void coctx_swap( coctx_t *,coctx_t* )//coctx_swap对应的函数原型

//用于分配coctx_swap两个参数内存区域的结构体，仅32位下使用，64位下两个参数直接由寄存器传递
struct coctx_param_t
{
	const void *s1;
	const void *s2;
};

//32bit下的coctx_t结构体
struct coctx_t
{
	void *regs[ 8 ];//用于保存或者设定特定寄存器值
	size_t ss_size;//栈帧区域的size
	char *ss_sp;//协程栈帧内存区域，这个区域一般在堆上分配
};

```
coctx_t 这个结构定义了整个协程的核心，设计协程的执行地址，ESP栈帧的地址设置以及其他重要寄存器值的保存。


在函数进入协程切换函数coctx_swap函数前的栈帧的位置如下：

![进入coctx_swap函数前的栈帧布局](/img/coctx_swap_enter_before.png)

ESP往下，是常规的进入函数调用后存放的寄存器、变量等信息，这里并不是coctx_swap执行后的场景，coctx_swap没有按照标准的函数进入的约定来,做了一些保存上下文的事情，所以进入coctx_swap后的栈帧内存布局有所区别。

coctx_swap 汇编函数的代码：
```
leal 4(%esp), %eax //这里ESP获取到的是对应图中old %EIP的地址，加4对应第一个参数的地址，这里不用第一个参数的地址这样表述，因为可能没有参数，保守起见还是称返回地址+4的位置
movl 4(%esp), %esp //这里将ESP往上移动四个字节，即第一个参数所在的位置

对应的ESP地址,此时ESP已经指向了第一个参数，coctx_t* 类型的变量

| *ss_sp  |
| ss_size |
| regs[7] |
| regs[6] |
| regs[5] |
| regs[4] |
| regs[3] |
| regs[2] |
| regs[1] |
| regs[0] |
--------------   <---ESP


leal 32(%esp), %esp //这里取第一个参数的地址，加32。第一个参数的类型coctx_t*，偏移32，即4bytes * 8，刚好位于regs末尾

对应的ESP当前位置

| *ss_sp  |
| ss_size |
-------------   <-- ESP (加了32到了这里)
| regs[7] |
| regs[6] |
| regs[5] |
| regs[4] |
| regs[3] |
| regs[2] |
| regs[1] |
| regs[0] |
----------- (old ESP)

pushl %eax //将EAX的值压入栈，这里压入的是返回地址+4的值,对应压入了regs[7],栈是往下增长的，push会将ESP往下调

pushl %ebp //push into regs[6]
pushl %esi //push into regs[5]
pushl %edi //push into regs[4]
pushl %edx //push into regs[3]
pushl %ecx //push into regs[2]
pushl %ebx //push into regs[1]
pushl -4(%eax) //push into regs[0] 这里将eax（返回地址+4） 又做了减4操作，得到的刚好是返回地址，放入regs[0]中

七个寄存器压入完毕后ESP对应的位置

| *ss_sp  |
| ss_size |
| regs[7] |
| regs[6] |
| regs[5] |
| regs[4] |
| regs[3] |
| regs[2] |
| regs[1] |
| regs[0] |
-----------  <-- ESP


movl 4(%eax), %esp //第一个参数在往上偏移4个字节，及第二个参数的地址,ESP指向coctx_t regs[0]处

| *ss_sp  |
| ss_size |
| regs[7] |
| regs[6] |
| regs[5] |
| regs[4] |
| regs[3] |
| regs[2] |
| regs[1] |
| regs[0] |
--------------   <---ESP

popl %eax  //将第二个coctx_t的regs[0]的值放入EAX中，pop对应的将ESP往上移动
popl %ebx  //push into regs[1]
popl %ecx  //push into regs[2]
popl %edx  //push into regs[3]
popl %edi  //push into regs[4]
popl %esi  //push into regs[5]
popl %ebp  //push into regs[6]

| *ss_sp  |
| ss_size |
| regs[7] |
--------------    <---ESP
| regs[6] |
| regs[5] |
| regs[4] |
| regs[3] |
| regs[2] |
| regs[1] |
| regs[0] |
-----------

popl %esp  //regs[7] -> ESP  //ESP调整为 regs[7]存储的地址


pushl %eax //set ret func addr 这里再push 放入的就是新的堆栈的返回地址的位置，栈的地址是向下增长的，上一步将ESP调到了返回地址+4的位置，这一步ESP刚好减4,即指向应该放入返回地址的位置，压入regs[0]中保存的值

xorl %eax, %eax
ret //这里包含两步，pop当前栈顶即压入的返回地址，然后跳转到返回地址执行，这里执行流程实际上就走到了由regs[0]执行的地址。
```

第一个参数的寄存器及返回地址保存顺序由coctx_swap 设置完成，而当前需要转向的地址需要程序设置，这里libco封装了一个接口，这里为了阅读方便，常量直接替换为值：
```
int coctx_make( coctx_t *ctx,coctx_pfn_t pfn,const void *s,const void *s1 )
{
	//make room for coctx_param
	//这里ctx->ss_sp 对应的地址是新的协程执行的栈空间，即进入到pfn中能够使用的栈空间。
	char *sp = ctx->ss_sp + ctx->ss_size - sizeof(coctx_param_t);//这里回顾下，函数调用时，会将参数从右到左入栈，然后将返回地址入栈，这里的调整是从栈空间上挪出位置给参数存放。
	
	//ctx->ss_sp 对应的空间是在堆上分配的，地址是从低到高的增长，而堆栈是往低地址方向增长的，所以要使用这一块人为改变的栈帧区域，首先地址要调到最高位，即ss_sp + ss_size的位置
	//------- ss_sp + ss_size
	//|     |
	//|     |
	//------- ss_sp
	
	sp = (char*)((unsigned long)sp & -16L);//这里字节对齐，这里为什么是16，下文单独列出一点介绍

	
	//这里给从栈区域空下来的参数区设置值，32位系统下，参数通过堆栈传递
	coctx_param_t* param = (coctx_param_t*)sp ;
	param->s1 = s;
	param->s2 = s1;
	
	//------- ss_sp + ss_size
	//|pading| 这里是对齐区域
	//|s2    |
	//|s1    |
	//-------  <- sp
	//|      |
	//------- ss_sp

	memset(ctx->regs, 0, sizeof(ctx->regs));

	ctx->regs[ 7 ] = (char*)(sp) - sizeof(void*); //函数调用，压入参数之后，还有一个返回地址要压入，所以还需要将sp往下移动4个字节，32位汇编获取参数是通过EBP+4, EBP+8来分别获取第一个参数，第二个参数的，这里减去4个字节是为了对齐这种约定,这里可以看到对齐以及参数还有4个字节的虚拟返回地址已经占用了一定的栈空间，所以实际上供协程使用的栈空间是小于分配的空间。另外协程且走调用co_swap参数入栈也会占用空间，
	ctx->regs[ 0 ] = (char*)pfn;//regs[0]填入函数地址，即新协程要执行的指令地址

	return 0;
}
```
coctx_make填充的regs[7]和 coctx_swap 保存的regs[7]有所区别：

1. coctx_make 填充的regs[7]只是在自定义的内存区域上的一个地址，该值会被存入ESP中，作为ESP的开始。按照参数传递的规则，该值+4的位置为第一个参数，该值+8的位置为第二个参数。 coctx_swap 执行完，控制流程进入pfn，同时ESP被设定。进入pfn函数时的堆栈指针如下：
```
------------------ ss_sp + ss_size
|   ...          |
|   ...          |
|     s2 		 |
|     s1		 |
|   void *		 |
------------------ <---ESP
| pfn 栈帧空间   |
|                |
|                |
------------------ ss_sp
```
这里可以看到```ctx->regs[ 7 ] = (char*)(sp) - sizeof(void*)```这行代码的用意了，这里void*作为返回地址的占位符。这样再pfn函数中才能按照约定的读取参数1(ESP+4)和参数2(ESP+8)。

2. coctx_swap main所在的regs[7]保存的是返回地址值+4，这样regs[0]的值放入eax后再压栈放入的刚好是应该存放返回地址的位置。

### 栈空间16直接对齐的哲学
为什么栈的地址 sp需要 16字节对齐：
```
sp = (char*)((unsigned long)sp & -16L);
```
16这个数看起来一点不特殊，32位下4字节对齐，64位系统下8字节对齐，感觉更好理解，那么16是什么。查了一堆资料，原来GCC默认的堆对齐设置的就是16字节，Stack Overflow上找到了一个靠谱的解释。 小小的魔术值，真的藏龙卧虎。
[Why does System V / AMD64 ABI mandate a 16 byte stack alignment?
](https://stackoverflow.com/questions/49391001/why-does-system-v-amd64-abi-mandate-a-16-byte-stack-alignment)


## 协程切换思考
这里再梳理下一个简单例子的协程切换

main 中对应的协程相关代码
```
coctx_make(&pending_ctx, (coctx_pfn_t)RoutineFunc, &pending,&mainRo);
coctx_swap(mainRo.ctx, pending.ctx);
```

RoutineFunc 实现：
```
static int RoutineFunc(stRo_t *cur, stRo_t *pre){
    if(cur->fn){
        (*cur->fn)(&cur->counter);
    }
    coctx_swap(cur->ctx, pre->ctx);
    return 0;
}
```
mainRo 指代main本身
pending 指代 即将切换到的协程

回顾下汇编代码，两个最关键的存储：

regs[0] 存放下一个指令执行地址，也即返回地址。

regs[7] 存放切换到新协程后，ESP指针调整的新地址。

程序的指令和数据均会重新设置，这也是程序能够切换的技术保证。

### main -> pending
pending 通过coctx_make构建，main 通过第一次切换时函数 coctx_swap 填充。

pengding 主要设置 regs[7] 位新协程ESP的开始地址，regs[0] 为新协程函数执行地址。

main 主要regs[0] 保存了 coctx_swap 执行往后的下一个指令的地址，这里主要是利用了 __cdecl 约定在函数调用发起的时候，由caller将参数及返回地址入栈，这样刚进入coctx_swap时 ESP指向的就是 coctx_swap 执行完毕后的下一个指令的地址，libco这里有ESP +4及 -4这样调整，实际最终放入的还是ESP的值。regs[7] 用于调整ESP的栈帧，coctx_swap保存的main 的regs[7]为返回地址+4的位置，也即第一个参数所在的栈帧位置。当函数调用完成后ret指令返回后，返回地址已经被pop出来了，执行完函数回到caller中时ESP此时应该就是指向返回地址+4的位置。返回主协程后，按照调用约定，有caller清空调用时入栈的参数，这样ESP就恢复到了正常的位置。

这里回顾下 call及ret两个汇编指令，call将返回地址入栈，然后跳转到目标地址。ret 将返回地址出栈，将控制流程恢复到返回地址所在指令。所以函数调用完成返回后ESP的标准位置ret弹出返回地址之后的位置，也就是返回地址+4的位置。

libco的汇编代码里面关于返回地址 及返回地址+4的这种操作，初看很奇怪。仔细思考下__cdecl调用约定及函数调用过程，就能明白其设计的精妙。通过在返回地址+4后面执行一个push操作覆盖返回地址为需要执行的指令地址，达到欺骗程序的目的实现自定义跳转。这个欺骗是建立在ESP的基础上的，push操作严格依赖ESP位置，ESP指向了返回地址+4的位置，这样再push就没有问题了。这就是为何coctx_swap在main的 ctx regs[7]保存的是返回地址+4的位置。

这里和工程实践最关键的问题出来了，ESP怎么调，调到哪里，影响是什么。ESP既然是栈帧，就不可能无限增长，能用多大，取决于我们预先分配。ctx->ss_sp就是程序制定的新协程执行的栈帧空间，这个空间使用其实并不简单，在协程里面进行函数调用变量使用，均会消耗栈帧空间。稍不注意超过了ctx->ss_sp能访问的空间程序就会core。libco提供了两种方式分配改栈帧空间，私有的和共享的。利弊很多文章均有表述，主要还是取决于实际应用场景。

### pending -> main

main coctx 并非用户主动创建，而是由第一次coctx_swap 自动保存。为何不能主动创建？没有函数调用保存ESP、返回地址和对应的汇编修改函数的默认行为，在main中是无法自动创建这样的上下文的。这个自动保存需要非常谨慎，一旦恢复不回来整个程序就core了，毕竟main的栈帧由操作系统严格分配，执行完后，涉及main函数的EBP等的恢复，如果回到了错误的内存区域，其结果完全是未定义的。main并不是创世者，也是一个函数调用，改变其栈帧不及时恢复，完全破坏了其原本行为，会导致core。可以试验下 协程切走了，不切回来，看看会不会core。从pending切换回来， 需要恢复main的执行现场，这里先不管是调用的什么函数切走的。按照正常的函数调用，执行结束后，控制流程回到main中时，此时ESP应该指向的是该函数返回地址+4的位置，然后由main执行coctx_swap 函数调用时入栈参数的清理工作（如果有）。所以保存现场需要保存的ESP的地址为 返回地址+4。返回地址很容易获得，进入任何一个callee总时，ESP还没有发生变化，此时ESP指向就是返回地址 4(ESP)即返回地址+4的位置。返回地址本身则保存在regs[0]中，用于指令切换回来。

从pending切到main，pending的上下文也发生了改变，这里的上下文通过coctx_swap保存重新设置，切出去那个点的返回地址压入到regs[0], 返回地址+4的值压入到regs[7]，和从main切过来过程就完全一样了。

### 贡献一个helloworld
libco例子封装的有点复杂，其实理解切换流程，留一个helloworld对于理解二者的切换再好不过。如果要实现自己的协程库，只需要包含coctx.h coctx.cpp coctx_swap.S 三个文件，就可以展开手脚干活了，可以不用沿用libco过渡封装的这一套。

```
#include "coctx.h"
#include <cstdlib>
#include <cstdio>
#include <cstring>
#include <string>

using std::string;

const static int STACK_SIZE = 128 * 1024;
const static int BUFF_SIZE = 2 * STACK_SIZE;
extern "C"
{
	extern void coctx_swap( coctx_t *,coctx_t* ) asm("coctx_swap");
};

struct stRo_t{
	coctx_t *ctx;
	string content;	
	void (*fn) (stRo_t *);
};
static int RoutineFunc(stRo_t *cur, stRo_t *pre){
	if(cur->fn){
		(*cur->fn)(cur);
	}
	coctx_swap(cur->ctx, pre->ctx);//第一次切回去
	
	if(cur->fn){
		(*cur->fn)(cur);
	}
	coctx_swap(cur->ctx, pre->ctx);//第二次切回去
	printf("invalid here!\n");//走到这里来，表示错误的切换过来了，后续没有切回去的代码了,走到这里会core
	return 0;
}

static void UserFunc(stRo_t* co){
	printf("%s", co->content.c_str()) ;
	return;
}

int main(){
	char* buffer = new char[2 * STACK_SIZE];
	memset(buffer,0,sizeof(char) * BUFF_SIZE);
	coctx_t main_ctx;
	main_ctx.ss_size = STACK_SIZE;
	main_ctx.ss_sp = buffer;

	coctx_t pending_ctx;
	pending_ctx.ss_size = STACK_SIZE;
	pending_ctx.ss_sp = buffer + STACK_SIZE;
	
	stRo_t pending, mainRo;
	pending.ctx = &pending_ctx;
	pending.fn = UserFunc;
	pending.content = "hello ";
	mainRo.ctx = &main_ctx;
	mainRo.fn = NULL;

	//printf("main 0, before switch to hello\n");
	//构建pending协程coctx_t结构
	coctx_make(&pending_ctx, (coctx_pfn_t)RoutineFunc, &pending,&mainRo);
	
	//第一次切到协程
	coctx_swap(mainRo.ctx, pending.ctx);

	//printf("main 1, switch back to main\n");
	pending.content = "world!\n";

    //第二次切到协程
	coctx_swap(mainRo.ctx, pending.ctx);

	//printf("main 2, switch back to main\n");

	delete []buffer;
	return 0;
}
```
执行结果：
```
hello world!
```
## 总结

协程切换看起来有点高深，确实比较精妙。
缺乏汇编的经验，对这些寄存器相关的理解并不深刻，有种讳莫如深的感觉。
协程切换利用了函数调用caller会将参数及返回地址入栈这一特性，
使得coctx_swap有机会保存现场指令及栈帧位置，
实现代码在切来切去还能保证指令的延续以及数据的延续。
切的本质也是巧妙的利用了ret指令约定从当前栈顶弹出返回地址放入IP寄存器，让出控制流。

(因为懒，先发在内网上了，今天才挪出来，不涉及内部任何信息，纯粹个人学习心得)
