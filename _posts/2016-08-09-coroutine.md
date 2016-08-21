---
layout: post
title: Linux协程-1
category: default
---

很早就想把协程这块的知识总结一下，这样那样的事情错过了。。

**两个比较出名国内协程库**

> [云风大神的协程库](https://github.com/cloudwu/coroutine/)  
> [腾讯开源的协程库](https://code.csdn.net/Tencent/libco/tree/master)

###1. ucontext定义

```cpp

/* Linux下的实现， Userlevel context.  */
typedef struct ucontext
  {
    unsigned long int uc_flags;
    struct ucontext *uc_link;     // 保存当前context结束后继续执行的context记录
    stack_t uc_stack;             // 该context运行的栈信息
    mcontext_t uc_mcontext;       // 保存具体的程序执行上下文——如PC值、堆栈指针、寄存器值等信息——其实现方式依赖于底层运行的系统架构，是平台、硬件相关的。
    __sigset_t uc_sigmask;        // context运行阶段需要屏蔽的信号
    struct _libc_fpstate __fpregs_mem;
  } ucontext_t;

/* MAC 下的实现 */
_STRUCT_UCONTEXT
{
	int                     uc_onstack;
	__darwin_sigset_t       uc_sigmask;     /* signal mask used by this context */
	_STRUCT_SIGALTSTACK     uc_stack;       /* stack used by this context */
	_STRUCT_UCONTEXT        *uc_link;       /* pointer to resuming context */
	__darwin_size_t	        uc_mcsize;      /* size of the machine context passed in */
	_STRUCT_MCONTEXT        *uc_mcontext;   /* pointer to machine specific context */
#ifdef _XOPEN_SOURCE
	_STRUCT_MCONTEXT        __mcontext_data;
#endif /* _XOPEN_SOURCE */
};

```

###2. Linux下实现上下文切换的函数

顾名思义，是讲当前执行点的执行状态上下文保存到ucp中;
> int getcontext(ucontext_t *ucp);  


该函数用来将当前程序执行线索切换到参数ucp所指向的上下文状态，
在执行正确的情况下，该函数直接切入到新的执行状态，不再会返回。
如果一个函数中直接切换的setcontext中，原先的堆栈去哪了？
> int setcontext(const ucontext_t *ucp); 

**注意：**
如果单独使用setcontext和getcontext强制进行上下文是很危险的，实际上，即便封装成协程库的方式，如果没有超时机制进行清理，也容易出现问题。这个地方在项目中踩过很多坑；
看下面简单的测试代码：

```cpp
static ucontext_t context;

void fun1(){
	printf("into fun1\n");
	char *data = new char[1000];
	/* 这个地方，强制进行上下文切换，而fún1上下文没有保存下来。
	   切换后fún的上下文丢失永远无法执行到setcontext后面的代码.
	   如果此时setcontext有内存分配的操作，那么这块内存将得不到释放； */
	setcontext(&context);     
	printf("exit fun2 \n"); 
	delete []data;
}

void fun2() {
	getcontext(&context);
	printf("into fun2\n");
	printf("exit fun2\n");
}

int main()
{
	fun2();
	fun1();

	return 0;
}
```

> void makecontext(ucontext_t *ucp, void (*func)(), int argc, ...); 



```cpp

void
__makecontext (ucontext_t *ucp, void (*func) (void), int argc, ...)
{
  extern void __start_context (void);
  greg_t *sp;
  unsigned int idx_uc_link;
  va_list ap;
  int i;

  /* Generate room on stack for parameter if needed and uc_link.  */
  sp = (greg_t *) ((uintptr_t) ucp->uc_stack.ss_sp
		   + ucp->uc_stack.ss_size);
  sp -= (argc > 6 ? argc - 6 : 0) + 1;
  /* Align stack and make space for trampoline address.  */
  sp = (greg_t *) ((((uintptr_t) sp) & -16L) - 8);

  idx_uc_link = (argc > 6 ? argc - 6 : 0) + 1;

  /* Setup context ucp.  */
  /* Address to jump to.  */
  ucp->uc_mcontext.gregs[REG_RIP] = (uintptr_t) func;
  /* Setup rbx.*/
  ucp->uc_mcontext.gregs[REG_RBX] = (uintptr_t) &sp[idx_uc_link];
  ucp->uc_mcontext.gregs[REG_RSP] = (uintptr_t) sp;

  /* Setup stack.  */
  sp[0] = (uintptr_t) &__start_context;
  sp[idx_uc_link] = (uintptr_t) ucp->uc_link;

  va_start (ap, argc);
  /* Handle arguments.

     The standard says the parameters must all be int values.  This is
     an historic accident and would be done differently today.  For
     x86-64 all integer values are passed as 64-bit values and
     therefore extending the API to copy 64-bit values instead of
     32-bit ints makes sense.  It does not break existing
     functionality and it does not violate the standard which says
     that passing non-int values means undefined behavior.  */
  for (i = 0; i < argc; ++i)
    switch (i)
      {
      case 0:
	ucp->uc_mcontext.gregs[REG_RDI] = va_arg (ap, greg_t);
	break;
      case 1:
	ucp->uc_mcontext.gregs[REG_RSI] = va_arg (ap, greg_t);
	break;
      case 2:
	ucp->uc_mcontext.gregs[REG_RDX] = va_arg (ap, greg_t);
	break;
      case 3:
	ucp->uc_mcontext.gregs[REG_RCX] = va_arg (ap, greg_t);
	break;
      case 4:
	ucp->uc_mcontext.gregs[REG_R8] = va_arg (ap, greg_t);
	break;
      case 5:
	ucp->uc_mcontext.gregs[REG_R9] = va_arg (ap, greg_t);
	break;
      default:
	/* Put value on stack.  */
	sp[i - 5] = va_arg (ap, greg_t);
	break;
      }
  va_end (ap);

}

```

> int swapcontext(ucontext_t *oucp, ucontext_t *ucp);





