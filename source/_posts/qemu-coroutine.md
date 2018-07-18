title: QEMU中的协程---qemu-coroutine
date: 2016-06-24 20:38:53
tags: QEMU
categories: Programme
---
之前只使用过Golang的协程，听说过Lua的协程，这里第一次在qemu中看到C语言也可以实现自己的协程，qemu在对块数据的读写过程中大量使用了协程，借此对C版的协程相关做一个分析和记录。

<!--more--> 

## 协程
协程是一种用户态的轻量级线程，相对于系统独立，有自己的上下文，协程的切换也由自己控制，所以相对于进程和线程来说其运行的开销要小得多。

## 为什么qemu要使用协程
根据qemu的commit log中给出的解释：
[http://qemu.11.n7.nabble.com/PATCH-v3-0-4-Coroutines-for-better-asynchronous-programming-td9809.html](http://qemu.11.n7.nabble.com/PATCH-v3-0-4-Coroutines-for-better-asynchronous-programming-td9809.html)
QEMU作为虚拟化软件其主体架构采用的是事件驱动模式，在main-loop中监控各种、大量的文件，事件，消息和状态的变化并进行各种操作，当大量的阻塞操作发生时，为不影响VM环境的执行效率，一般都采用异步的方式。而异步方式则需要设定callback函数的调用时机，同时保存大量的执行状态，导致逻辑代码支离破碎，复杂并难以理解。所以最好的解决方式是采用协程的方式将同步方式的代码异步化。

## qemu-coroutine API
源码：
qemu-coroutine.c    
coroutine-ucontext.c   
coroutine-ucontext.h   

```c
/*
 * 创建新的协程
 * 使用qemu_coroutine_enter() 进入协程执行环境.
 */
Coroutine *qemu_coroutine_create(CoroutineEntry *entry)

/**
 * 进入协程执行环境
 *
 * opaque为传递给协程执行入口参数
 */
void qemu_coroutine_enter(Coroutine *coroutine, void *opaque);

/**
 * 转移执行控制权到协程的调用者处
 *
 * 这个函数不会返回除非使用qemu_coroutine_enter()重新进入协程
 */
void coroutine_fn qemu_coroutine_yield(void);
```
qemu协程使用非常简单，创建并启动协程：
```c
coroutine = qemu_coroutine_create(my_coroutine); 
qemu_coroutine_enter(coroutine, my_data); 
```
coroutine协程则会执行直到退出或者yields
```c
void coroutine_fn my_coroutine(void *opaque) { 
      MyData *my_data = opaque; 

      /* do some work */ 

      qemu_coroutine_yield(); 

      /* do some more work */ 
  } 
```
yielding 会切换回qemu_coroutine_enter的调用者，qemu中则在发起一个异步的I/O请求后切回主线程的event loop。

## qemu协程基础
qemu-coroutine的实现有gthread,ucontext,sigalstack等几种模式，这里主要看ucontext模式,而coroutine的基础是setjmp/longjmp.

### setjmp/longjmp
setjmp/longjmp是C语言的一组库函数，主要作用是“非本地跳转”，区别于goto，它们能够完成函数外部的跳转。
```c
int setjmp( jmp_buf env );   //用于保存程序的运行时的堆栈环境
void longjmp( jmp_buf env, int value ); //用于恢复先前程序中调用的setjmp函数时所保存的堆栈环境,参数value为setjmp返回值
```
从setjmp/longjmp的源码实现看，其主要作用就是save、restore当前程序的运行上下文，包括寄存器、堆栈信息等到jmp_buf中。
源码位置glibc：
sysdeps/i386/setjmp.S       
sysdeps/i386/__longjmp.S

下面是个简单的例子：
```c
#include <setjmp.h>

main()
{
  jmp_buf env;
  int i;

  i = setjmp(env);  //保存此处的程序运行上下文到env中
  printf("i = %d\n", i);

  if (i != 0) exit(0);

  longjmp(env, 2);    //跳转到env保存的上下文中，即setjmp处，是setjmp返回值为2
  printf("Does this line get printed?\n");

}
```
运行结果为：
```bash
i = 0
i = 2
```
### ucontext
ucontext函数组为setjmp/longjmp的升级版：
```c
int  getcontext(ucontext_t *); //初始化ucontext_t结构体，将当前的上下文保存到ucontext_t中
int  setcontext(const ucontext_t *);  //设置当前的上下文为ucontext_t,并跳转至其中
void makecontext(ucontext_t *, (void *)(), int, ...); //制造一个上下文，并设置入口函数
int  swapcontext(ucontext_t *, const ucontext_t *); //保存当前上下文到第一个参数中，然后切换到第二个参数代表的上下文。 
```
例程：
```c
#include <stdio.h>
#include <ucontext.h>
#include <unistd.h>

int main(int argc, const char *argv[]){
	ucontext_t context;
	
	getcontext(&context);
	puts("Hello world");
	sleep(1);
	setcontext(&context);
	return 0;
}
```
运行结果：
```bash
~$ ./example   
Hello world  
Hello world  
Hello world  
Hello world
^C  
~$  
```
## qemu-coroutine的实现
qemu-coroutine主要基于setjmp/longjmp实现，更为轻量级。
```c
Coroutine *qemu_coroutine_create(CoroutineEntry *entry)
{
    Coroutine *co = qemu_coroutine_new();  // 创建一个新的coroutine
    co->entry = entry;   //设置coroutine的入口函数为entry
    return co;
}

static Coroutine *coroutine_new(void)
{
    const size_t stack_size = 1 << 20;
    CoroutineUContext *co;
    ucontext_t old_uc, uc;
    jmp_buf old_env;
    union cc_arg arg = {0};

    if (getcontext(&uc) == -1) {     //获得当前上下文
        abort();
    }

    co = g_malloc0(sizeof(*co));
    co->stack = g_malloc(stack_size);
    co->base.entry_arg = &old_env; /* stash away our jmp_buf */

    uc.uc_link = &old_uc;
    uc.uc_stack.ss_sp = co->stack;
    uc.uc_stack.ss_size = stack_size;
    uc.uc_stack.ss_flags = 0;

    arg.p = co;

    makecontext(&uc, (void (*)(void))coroutine_trampoline,
                2, arg.i[0], arg.i[1]);     //制造一个上下文，设置该上下文的栈空间及相关信息

    /* swapcontext() in, longjmp() back out */
    if (!setjmp(old_env)) {         //保存当前上下文到old_env中，此时old_env的地址作为co->base.entry_arg
        swapcontext(&old_uc, &uc);   //切换至uc代表的上下文中，入口函数为coroutine_trampoline，返回点为old_uc中
    }
    return &co->base;
}

static void coroutine_trampoline(int i0, int i1)
{
    union cc_arg arg;
    CoroutineUContext *self;
    Coroutine *co;

    arg.i[0] = i0;
    arg.i[1] = i1;
    self = arg.p;
    co = &self->base;   //获取了通过coroutine_new创建的coroutine结构

    /* Initialize longjmp environment and switch back the caller */
    if (!setjmp(self->env)) {   //保存当前上下文到co（新协程）的env buffer中，由于第一次setjmp返回的是0，则调用下面的longjmp
        longjmp(*(jmp_buf *)co->entry_arg, 1); //此时co->entry_arg为coroutine_new中的old_env保存点，而value给的是1，则swapcontext不会执行，直接return，qemu_coroutine_create就直接返回了
    }

    while (true) {
        co->entry(co->entry_arg);
        qemu_coroutine_switch(co, co->caller, COROUTINE_TERMINATE);
    }
}
```
经过create的过程，新创建的co的env保存了coroutine_trampoline中setjmp(self->env)的上下文。

```c
void qemu_coroutine_enter(Coroutine *co, void *opaque)
{
    Coroutine *self = qemu_coroutine_self();   //获取当前协程的co结构，第一次可认为是主协程的控制信息

    trace_qemu_coroutine_enter(self, co, opaque);

    if (co->caller) {
        fprintf(stderr, "Co-routine re-entered recursively\n");
        abort();
    }

    co->caller = self;   //新协程的caller为主线程co
    co->entry_arg = opaque;
    coroutine_swap(self, co);   //通过swap操作从主协程切换至新创建的co
}

//qemu_coroutine_switch 为协程切换的关键函数
CoroutineAction qemu_coroutine_switch(Coroutine *from_, Coroutine *to_,
                                      CoroutineAction action)
{
    CoroutineUContext *from = DO_UPCAST(CoroutineUContext, base, from_);
    CoroutineUContext *to = DO_UPCAST(CoroutineUContext, base, to_);
    CoroutineThreadState *s = coroutine_get_thread_state();
    int ret;

    s->current = to_;

    ret = setjmp(from->env);   //保存当前的上下文到主协程的env中，相当于主协程的上下文在qemu_coroutine_enter中
    if (ret == 0) {
        longjmp(to->env, action);  //跳转至新协程的上下文，新协程的上下文保存点为coroutine_trampoline中的setjmp处，此处action给的是非0，则直接进入co->entry(co->entry_arg);执行create时给的entry.
    }
    return ret;
}
```
若co->entry(co->entry_arg)中使用qemu_coroutine_yield
```c
void coroutine_fn qemu_coroutine_yield(void)
{
    Coroutine *self = qemu_coroutine_self();
    Coroutine *to = self->caller;

    trace_qemu_coroutine_yield(self, to);

    if (!to) {
        fprintf(stderr, "Co-routine is yielding to no one\n");
        abort();
    }

    self->caller = NULL;
    coroutine_swap(self, to);   //此处self为新协程，to为主协程
}
再次通过coroutine_swap操作来进行切换：
CoroutineAction qemu_coroutine_switch(Coroutine *from_, Coroutine *to_,
                                      CoroutineAction action)
{
    CoroutineUContext *from = DO_UPCAST(CoroutineUContext, base, from_);
    CoroutineUContext *to = DO_UPCAST(CoroutineUContext, base, to_);
    CoroutineThreadState *s = coroutine_get_thread_state();
    int ret;

    s->current = to_;

    ret = setjmp(from->env);  //保存当前上下文到from的evn即新协程的env，此处的调用栈为qemu_coroutine_yield的内部
    if (ret == 0) {
        longjmp(to->env, action); //切换至主协程的上下文，在上面enter的分析中可以得到，此时主协程的上下文在qemu_coroutine_enter中。
    }
    return ret;
}
```
这样yield处的上下文被保存在新协程的env中，而程序逻辑调回了qemu_coroutine_enter中继续执行，即从qemu_coroutine_enter退出。

在适当的时机，再次调用qemu_coroutine_enter则会恢复yield处的上下文继续执行。

![](http://7j1zal.com1.z0.glb.clouddn.com/qemu-coroutine.PNG)

## 基于ucontext的实现（云风）
[https://github.com/cloudwu/coroutine/](https://github.com/cloudwu/coroutine/)
