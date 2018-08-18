---
title: libtask分析
date: 2017-12-29 23:36:33
tags:
	- libtask
---



`libtask` 是一个开源的 `C` 语言协程库。

`C` 语言协程可以通过更改寄存器，切换协程上下文实现协程调度。

更改寄存器可以通过 `C` 语言内联汇编实现，通过汇编代码直接更改寄存器的内容。

也可以使用 `ucontext` 配合 `getContext` 、 `setContext` 、`makeContext` 、`swapContext` 函数来实现。

`ucontext` 结构封装了寄存器信息和栈信息，是协程执行的上下文，而其他四个函数分别用于获取当前执行的上下文，设置当前上下文，创建上下文和交换上下文，这些函数已经封装了对寄存器内容的交换工作。

##### Task

一个Task可以看成是一个需要异步执行的任务，coroutine的抽象描述。

```c
typedef struct Context Context;	
struct Context
{
	ucontext_t	uc;	//ucontext封装了协程执行的上下文信息
};
struct Task
{
	char	name[256];	// offset known to acid
	char	state[256];
	Task	*next;
	Task	*prev;
	Task	*allnext;
	Task	*allprev;
	Context	context;	//协程上下文
	uvlong	alarmtime;
	uint	id;
	uchar	*stk;	//协程栈指针
	uint	stksize;	//栈大小
	int	exiting;
	int	alltaskslot;	//在全局task数组内的index
	int	system;
	int	ready;
	void	(*startfn)(void*);	//Task需要执行的函数
	void	*startarg;	//startfn 的参数
	void	*udata;
};
```

##### Task创建

```c
int taskcreate(void (*fn)(void*), void *arg, uint stack){
	int id;
	Task *t;

	t = taskalloc(fn, arg, stack);	//分配task和stack的空间
	taskcount++;	
	id = t->id;
  //判断数组是否还有足够空间
	if(nalltask%64 == 0){
      //扩展数组
		alltask = realloc(alltask, (nalltask+64)*sizeof(alltask[0]));
		if(alltask == nil){
			fprint(2, "out of memory\n");
			abort();
		}
	}
  	//保存Task在alltask数组内的index
	t->alltaskslot = nalltask;
	alltask[nalltask++] = t;	//保存task到alltask数组
	taskready(t);	//设置为ready，可以被调度执行
	return id;
}

//taskalloc分配task和stack的空间
static Task* taskalloc(void (*fn)(void*), void *arg, uint stack){
	Task *t;
	sigset_t zero;
	uint x, y;
	ulong z;

	/* allocate the task and stack together */
	t = malloc(sizeof *t+stack);	//分配内存，stack紧跟在task之后
	if(t == nil){
		fprint(2, "taskalloc malloc: %r\n");
		abort();
	}
  	//清除task的内存
	memset(t, 0, sizeof *t);
	t->stk = (uchar*)(t+1);	//设置stack指针，stack紧跟task之后，t+1指针偏移一个Task大小
	t->stksize = stack;	//设置stack大小
	t->id = ++taskidgen;	//设置id
	t->startfn = fn;	//设置任务需要执行的函数
	t->startarg = arg;	//startfn的参数

	/* do a reasonable initialization */
	memset(&t->context.uc, 0, sizeof t->context.uc);
	sigemptyset(&zero);
	sigprocmask(SIG_BLOCK, &zero, &t->context.uc.uc_sigmask);

	/* must initialize with current context */
	if(getcontext(&t->context.uc) < 0){	//获取当前ucontext，并保存到t->context.uc
		fprint(2, "getcontext: %r\n");
		abort();
	}

	/* call makecontext to do the real work. */
	/* leave a few words open on both ends */
  	//设置栈顶指针和栈大小，两端都保留一点空间
	t->context.uc.uc_stack.ss_sp = t->stk+8;	
	t->context.uc.uc_stack.ss_size = t->stksize-64;
#if defined(__sun__) && !defined(__MAKECONTEXT_V2_SOURCE)		/* sigh */
#warning "doing sun thing"
	/* can avoid this with __MAKECONTEXT_V2_SOURCE but only on SunOS 5.9 */
	t->context.uc.uc_stack.ss_sp = 
		(char*)t->context.uc.uc_stack.ss_sp
		+t->context.uc.uc_stack.ss_size;
#endif
	/*
	 * All this magic is because you have to pass makecontext a
	 * function that takes some number of word-sized variables,
	 * and on 64-bit machines pointers are bigger than words.
	 */
//print("make %p\n", t);
  //计算startfn的参数:y,x
  /**
  taskstart的参数是uint，即32位，而指针如果是64位，则需要将指针的高32位和低32位分离，分别传递
  将指针分离为高32位(x)和低32位(y)，在taskstart内再通过两个参数合成task指针
  该方法可以同时适用于32位和64位的编译器
  **/
	z = (ulong)t;
	y = z;
	z >>= 16;	/* hide undefined 32-bit shift from 32-bit compilers */
	x = z>>16;
  //这里传入的是taskstart函数，在该函数内执调用t->startfn，并传入t->startarg
	makecontext(&t->context.uc, (void(*)())taskstart, 2, y, x);

	return t;
}

/**
初始化uc_context，set the context of coroutine
**/
#ifdef NEEDAMD64MAKECONTEXT
void
makecontext(ucontext_t *ucp, void (*func)(void), int argc, ...)
{
	long *sp;
	va_list va;	//用于遍历可变长参数的指针

	memset(&ucp->uc_mcontext, 0, sizeof ucp->uc_mcontext);
	if(argc != 2)
		*(int*)0 = 0;	//报错
	va_start(va, argc);	//遍历可变参数
	//前6个参数可以使用寄存器（%rdi，%rsi，%rdx，%rcx，%r8，%r9）保存，后面参数入栈
	//用于传递函数参数，rdi：第一个参数，rsi：第二个参数；调用func时传入rdi和rsi
	ucp->uc_mcontext.mc_rdi = va_arg(va, int);
	ucp->uc_mcontext.mc_rsi = va_arg(va, int);
	va_end(va);
	/**设置栈指针**/
	sp = (long*)ucp->uc_stack.ss_sp+ucp->uc_stack.ss_size/sizeof(long);	//移动sp指针
	sp -= argc;	
	sp = (void*)((uintptr_t)sp - (uintptr_t)sp%16);	/* 16-align for OS X */ //地址对齐
	*--sp = 0;	/* return address */
	ucp->uc_mcontext.mc_rip = (long)func;	//ip，rip存放下一条指令地址
	ucp->uc_mcontext.mc_rsp = (long)sp;	//栈顶指针
}
#endif

static void
taskstart(uint y, uint x)
{
	Task *t;
	ulong z;
	// t = (x<<32)|y
	z = x<<16;	/* hide undefined 32-bit shift from 32-bit compilers */
	z <<= 16;
	z |= y;
	t = (Task*)z;	//获取task地址

//print("taskstart %p\n", t);
	t->startfn(t->startarg);	//调用startfn
//print("taskexits %p\n", t);
	taskexit(0);	//startfn结束，设置退出标志位
//print("not reacehd\n");
}
```

### Task调度

```c
typedef struct Tasklist Tasklist;
struct Tasklist	/* used internally */
{
	Task	*head;
	Task	*tail;
};
```

```c
Context	taskschedcontext;	
Tasklist	taskrunqueue;
Task	*taskrunning;

void
needstack(int n)
{
	Task *t;

	t = taskrunning;

	if((char*)&t <= (char*)t->stk
	|| (char*)&t - (char*)t->stk < 256+n){
		fprint(2, "task stack overflow: &t=%p tstk=%p n=%d\n", &t, t->stk, 256+n);
		abort();
	}
}

void
taskswitch(void)
{
	needstack(0);
	contextswitch(&taskrunning->context, &taskschedcontext);
}

int
swapcontext(ucontext_t *oucp, const ucontext_t *ucp)
{
	if(getcontext(oucp) == 0)	//get the context into *oucp
		setcontext(ucp);	//set the context as *ucp
	return 0;
}

void
taskready(Task *t)
{
	t->ready = 1;
	addtask(&taskrunqueue, t);	//加入调度队列
}

int
taskyield(void)
{
	int n;
	
	n = tasknswitch;
	taskready(taskrunning);	//将当前task加入等待队列
	taskstate("yield");
	taskswitch();	//切换
	return tasknswitch - n - 1;
}

int
anyready(void)
{
	return taskrunqueue.head != nil;	//判断等待队列队首是否为空
}

void
taskexit(int val)
{
	taskexitval = val;
	taskrunning->exiting = 1;
	taskswitch();	//切换上下文，执行调度程序
}


static void
taskscheduler(void)
{
	int i;
	Task *t;

	taskdebug("scheduler enter");
	for(;;){
		if(taskcount == 0)
			exit(taskexitval);
		t = taskrunqueue.head;
		if(t == nil){
			fprint(2, "no runnable tasks! %d tasks stalled\n", taskcount);
			exit(1);
		}
		deltask(&taskrunqueue, t);
		t->ready = 0;
		taskrunning = t;	//设置taskrunning
		tasknswitch++;
		taskdebug("run %d (%s)", t->id, t->name);
      //当前context保存到taskschedcontext，即taskschedcontext为调度上下文
		contextswitch(&taskschedcontext, &t->context);	//交换 task context
//print("back in scheduler\n");
		taskrunning = nil;
		if(t->exiting){	//如果退出
			if(!t->system)
				taskcount--;
			i = t->alltaskslot;
			alltask[i] = alltask[--nalltask];	//替换为全局数组的最后一个task
			alltask[i]->alltaskslot = i;
			free(t);	//释放task
		}
	}
}

/*
 * startup
 */

static int taskargc;
static char **taskargv;
int mainstacksize;

static void
taskmainstart(void *v)
{
	taskname("taskmain");
	taskmain(taskargc, taskargv);
}

int
main(int argc, char **argv)
{
	struct sigaction sa, osa;

	memset(&sa, 0, sizeof sa);
	sa.sa_handler = taskinfo;
	sa.sa_flags = SA_RESTART;
	sigaction(SIGQUIT, &sa, &osa);

#ifdef SIGINFO
	sigaction(SIGINFO, &sa, &osa);
#endif

	argv0 = argv[0];
	taskargc = argc;
	taskargv = argv;

	if(mainstacksize == 0)
		mainstacksize = 256*1024;
	taskcreate(taskmainstart, nil, mainstacksize);
	taskscheduler();
	fprint(2, "taskscheduler returned in main!\n");
	abort();
	return 0;
}
```
