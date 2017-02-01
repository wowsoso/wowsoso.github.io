## 关于linux下协程的通用实现及libtask库源码解析

协程(coroutine)与subroutine同样作为程序执行单元的抽象被一些语言当作基
础实现,两者的抽象方式大致区别在于：

1. 多个执行单元之间的关系：

对于subroutine来说,存在一个调用与被调用的关系,比如在a-subroutine里调用
b-subroutine, 那么a-subroutine就是调用者,b-subroutine是被调用者,它们共
享一个线程的堆栈。

而多个coroutine之间的关系是对等的,即便在a-coroutine里创建了
b-coroutine,他们之间也不会有任何层级关系.

subroutine作为一种通用的抽象比较容易实现,而要实现coroutine至少需要两个
条件:

  * 要有一个全局的调度器并且每个coroutine得有一个堆栈空间.

  * 调度器用来在多个对等的coroutine之间做切换操作,每个coroutine的堆栈
    用于存储各自的上下文内容。

2. 执行单元的入口与出口:

在一个典型的subroutine实现里,执行单元的入口和出口只能有一个, 这是共享
调用栈带来的局限性, 比如(x86-64平台)我们在a-subroutine里调用
b-subroutine,那么会先把前6个参数依次写入寄存器:rdi,rsi,rdx,rcx,r8,r9,6
个以上的参数从右至左压栈,rsp不断上移指向栈顶,然后把b-subroutine调用之
后的那条指令地址压栈,rsp上移,然后rbp压栈,把rsp指向rbp的栈地址,最后把
b-subroutine里的局部变量依次压栈,rsp继续上移.这就是调用时的入口过程,
当执行是b-subroutine时,就得依次出栈,然后退出,执行a-subroutine里的下一
条指令,这是出口过程.很明显,这里b-subroutine只有一个入口和一个出口,因为
必须要等b-subroutine执行完成后(出栈)才能继续执行a-subroutine.

而一个典型的coroutine实现,执行单元可以有多个入口和出口, 因为栈不共享,
一个通用的实现模式是用堆来表示coroutine的调用栈, 当我们在a-coroutine里
创建b-coroutine, 在堆上分配一块空间用于表示b-coroutine的堆栈, 在执行
b-coroutine时，我们可以在任意点把当前的执行信息写回这块在堆上分配的空
间,然后把rsp指向a-coroutine的栈顶,rbp指向a-coroutine的栈frame，rip指向
a-coroutine的需要继续执行的指令的地址, 这也就是所谓的用户态的上下文切
换,当下一次需要继续执行b-coroutine的时候，保存当前coroutine的上下文，
恢复b-coroutine的上下文就行了.


上面描述的上下文切换是在用户态进行的,unix-like的环境下, glibc库通常都
会有一个描述上下文结构的定义ucontext_t在ucontext.h文件里,并有四个函
数:getcontext,setcontext,makecontext,swapcontext分别用于在用户态保存上
下文,恢复上下文,创建上下文和保存且恢复上下文,使用它们可以实现一个基本
的协程系统, 比如libtask就是一个运行在unix平台上的基础协程库, 能在用户
态实现多个执行流轮换执行,libtask对glibc的
getcontext,setcontext,makecontext,swapcontext做了简单的封装,因为这四个
函数是libtask实现的基础,所以要研究libtask前,先得了解这4个函数的作用与
实现机制.

这里有一个关键的数据结构,即user level context:
```C
    typedef struct ucontext
      {
        unsigned long int uc_flags;
        // 另一个执行流的上下文地址,在x86x64平台下也就是rbx寄存器里的内容
        struct ucontext *uc_link;
        //用于此上下文结构的堆栈,存储在堆区
        stack_t uc_stack;
        // mcontext_t结构体用于存储完整的进程状态信息
        mcontext_t uc_mcontext;
        // 需要block的信号掩码
        __sigset_t uc_sigmask;
        // fpu寄存器结构
        struct _libc_fpstate __fpregs_mem;
    } ucontext_t;
```

ucontext结构体里的uc\_stack是在堆上分配的,并做为堆栈用于此上下文,
makecontext函数将会设置uc\_stack与相应寄存器里的值uc\_stack的结构大致
是这样的:

        ---------------------------------------
        | 下一个上下文的地址                   |
        ---------------------------------------
        | 参数 7-n(假如回调函数的参数大于7个)   |
        ---------------------------------------
        |  返回地址                            |
%rsp -> ----------------------------------------


另外寄存器里的内容:

     %rdi,%rsi,%rdx,%rcx,%r8,%r9: 分别存储参数1-6

     %rbx   : 下一个上下文的地址

     %rsp   : 指向栈顶

当我们需要创建一个用户态上下文的时候, 需要调用makecontext函数,此函数接
受一个ucontext_t类型的指针(ucp), 一个函数指针(切换到此上下文寄存器esp
所指向的地址, 多个函数参数的指针地址(都是int类型,所以在64位环境下需要
用两个参数描述一个待执行函数参数的指针地址))

x86x64环境下makecontext的源码：

```C
    __makecontext (ucontext_t *ucp, void (*func) (void), int argc, ...)
    {
      extern void __start_context (void);
      greg_t *sp;
      unsigned int idx_uc_link;
      va_list ap;
      int i;

      /* Generate room on stack for parameter if needed and uc_link.  */
      //把栈顶的地址赋给sp变量
      sp = (greg_t *) ((uintptr_t) ucp->uc_stack.ss_sp
               + ucp->uc_stack.ss_size);
      // 判断回调函数的参数是否大于6,如果大于6,那么需要把第7-n个参数地址压栈
      // rsp往上移
      sp -= (argc > 6 ? argc - 6 : 0) + 1;
      /* Align stack and make space for trampoline address.  */
      // 栈对齐并且rsp往上移8位
      sp = (greg_t *) ((((uintptr_t) sp) & -16L) - 8);

      // 用于定位下一个上下文的地址的索引
      idx_uc_link = (argc > 6 ? argc - 6 : 0) + 1;

      /* Setup context ucp.  */
      /* Address to jump to.  */
      // 下面几行代码把回调函数的地址写入RIP
      // 把下一个上下文的地址写入RBX
      // 把sp(栈顶)的地址写入RSP
      ucp->uc_mcontext.gregs[REG_RIP] = (uintptr_t) func;
      /* Setup rbx.*/
      ucp->uc_mcontext.gregs[REG_RBX] = (uintptr_t) &sp[idx_uc_link];
      ucp->uc_mcontext.gregs[REG_RSP] = (uintptr_t) sp;

      /* Setup stack.  */
      sp[0] = (uintptr_t) &__start_context;
      sp[idx_uc_link] = (uintptr_t) ucp->uc_link;

      // 下面把参数写入context, 与linux处理参数机制一致,
      // 当参数少于7时,对应寄存器:rdi, rsi, rdx, rcx, r8, r9
      // 当参数大于7,前6个参数仍然写入rdi, rsi, rdx, rcx, r8, r9, 之后的参数从后至前压栈
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

当需要切换上下文时，需要调用swapcontext, swapcontext接受两个参数,1:当
前的上下文(u_context), 2:新的上下文(u_context), x86-64下源码如下:

```C
    ENTRY(__swapcontext)
        /* Save the preserved registers, the registers used for passing args,
           and the return address.  */
        // 这里oRBX, oRBP...都是一些定义好的宏,
        // 扩展一下比如oRBX是:offsetof(ucontext_t, gregs[REG_##RBP])
        // 指的是RBP所在ucontext_t这个结构体里的偏移量, 所以
        //  movq %rbx, oRBX(%rid) => movq %rbx, <RBX的偏移量>(%rid)
        // 这里所做的工作是把当前寄存器中的内容写回堆栈(调用makecontext前在堆上分配的空间)
        // 并且把当前上下文的signalmask写回堆栈, 然后把新的上下文所在的栈地址写入各寄存器
        // 进栈顺序依次是rbx, rbp(栈基址), %r12, %r13, %r14, %15, %rdi(第2个参数), %rsi(第2个参数),
        // %rdx(第3个参数), %rcx(第4个参数),%8(第5个参数), %9(第6个参数), %rip(栈顶(原rsp寄存器)),
        // %rsp(栈顶+8(排除掉返回地址))
        movq	%rbx, oRBX(%rdi)
        movq	%rbp, oRBP(%rdi)
        movq	%r12, oR12(%rdi)
        movq	%r13, oR13(%rdi)
        movq	%r14, oR14(%rdi)
        movq	%r15, oR15(%rdi)

        movq	%rdi, oRDI(%rdi)
        movq	%rsi, oRSI(%rdi)
        movq	%rdx, oRDX(%rdi)
        movq	%rcx, oRCX(%rdi)
        movq	%r8, oR8(%rdi)
        movq	%r9, oR9(%rdi)

        movq	(%rsp), %rcx
        movq	%rcx, oRIP(%rdi)
        leaq	8(%rsp), %rcx		/* Exclude the return address.  */
        movq	%rcx, oRSP(%rdi)

        /* We have separate floating-point register content memory on the
           stack.  We use the __fpregs_mem block in the context.  Set the
           links up  correctly.  */
        leaq	oFPREGSMEM(%rdi), %rcx
        movq	%rcx, oFPREGS(%rdi)
        /* Save the floating-point environment.  */
        fnstenv	(%rcx)
        stmxcsr oMXCSR(%rdi)


        /* The syscall destroys some registers, save them.  */
        // 这里保存%rsi的内容进%r12寄存器,因为下面会执行系统调用
        movq	%rsi, %r12

        /* Save the current signal mask and install the new one with
           rt_sigprocmask (SIG_BLOCK, newset, oldset,_NSIG/8).  */
        leaq	oSIGMASK(%rdi), %rdx
        leaq	oSIGMASK(%rsi), %rsi
        movl	$SIG_SETMASK, %edi
        movl	$_NSIG8,%r10d
        movl	$__NR_rt_sigprocmask, %eax
        syscall
        cmpq	$-4095, %rax		/* Check %rax for error.  */

        jae	SYSCALL_ERROR_LABEL	/* Jump to error handler if error.  */

        /* Restore destroyed registers.  */
        // 恢复rsi寄存器, rsi里目前存储的是新的上下文结构体所在的地址
        movq	%r12, %rsi

        /* Restore the floating-point context.  Not the registers, only the
           rest.  */
        movq	oFPREGS(%rsi), %rcx
        fldenv	(%rcx)
        ldmxcsr oMXCSR(%rsi)

        // 下面依次把%rsi(新的上下文)内容写入寄存器
        /* Load the new stack pointer and the preserved registers.  */
        movq	oRSP(%rsi), %rsp
        movq	oRBX(%rsi), %rbx
        movq	oRBP(%rsi), %rbp
        movq	oR12(%rsi), %r12
        movq	oR13(%rsi), %r13
        movq	oR14(%rsi), %r14
        movq	oR15(%rsi), %r15

        /* The following ret should return to the address set with
        getcontext.  Therefore push the address on the stack.  */
        movq	oRIP(%rsi), %rcx
        pushq	%rcx

        /* Setup registers used for passing args.  */
        // 按顺序(rsi还需要使用故除外)依次写入参数: rdi, rdx, rcx, r8,r9
        movq	oRDI(%rsi), %rdi
        movq	oRDX(%rsi), %rdx
        movq	oRCX(%rsi), %rcx
        movq	oR8(%rsi), %r8
        movq	oR9(%rsi), %r9

        /* Setup finally  %rsi.  */
        // 把第二个参数写入rsi
        movq	oRSI(%rsi), %rsi

        /* Clear rax to indicate success.  */
        xorl	%eax, %eax
        ret
    PSEUDO_END(__swapcontext)
```

从swapcontext的实现可以看出swapcontext所做的事很简单,保存,恢复。把当前
上下文按顺序保存到rdi的偏移,新的上下文(rsi指向的地址)覆盖老的上下文。

swapcontext实际上是getcontext/setcontext的结合体,比如参照ai64的实现:

```C
    int
    __swapcontext (ucontext_t *oucp, const ucontext_t *ucp)
    {
      struct rv rv = __getcontext (oucp);
      if (rv.first_return)
        __setcontext (ucp);
      return 0;
    }
```

之前说到glibc的这4个函数是libtask的基础, 其实更准确的说libtask是其较浅
的封装,我们先来看看libtask的核心结构:task的实现:

```C
    struct Task
    {
        char name[256];// offset known to acid
        char state[256];
        Task *next;
        Task *prev;
        Task *allnext;
        Task *allprev;
        Context context;
        uvlong alarmtime;
        uint id;
        uchar *stk; /*stack start location*/
        uint stksize;
        int exiting;
        int alltaskslot;
        int system;
        int ready;
        void (*startfn)(void*);
        void *startarg;
        void *udata;
    };
    #+END_SRC
    这个结构体里context就是ucontext_t:
    #+BEGIN_SRC c
        struct Context {
            ucontext_t uc;
        }
```

ucontext_t里存储了上下文内容, 也就是swapcontext函数里两个参数的类型stk
指向在堆上分配的空间,被做为栈赋值给ucontext_t的ss_sp字段, 这是
makecontext函数要求的,在调用makecontext函数前,必须为参数ucp分配一块地
址作为上下文的栈空间,libtask会执行初始化工作:

```C
    t->context.uc.uc_stack.ss_sp = t->stk+8;
    t->context.uc.uc_stack.ss_size = t->stksize-64;
    ...
```

在调用swapcontext前,可以通过比较当前context的地址是否大于stk的地址来判
断栈空间是否够用:

```C
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
```

这个实现有点tricky, 画副图说明:

![libtask1](https://wowsososo.github.io/images/libtask1.png)

所以当 (char*)&t <= (char*)t->task时,说明栈空间不够了.

如果栈空间足够,那么就可以调用swapcontext了:

```C
    static void
    contextswitch(Context *from, Context *to)
    {
        if(swapcontext(&from->uc, &to->uc) < 0){
            fprint(2, "swapcontext failed: %r\n");
            assert(0);
        }
    }
```

contextswitch函数是由taskscheduler函数驱动的，taskscheduler是一个task
全局调度器, 运行在进程的整个生命周期中,调度器结束,进程关闭:

```C
    static void
    taskscheduler(void)
    {
        int i;
        Task *t;

        taskdebug("scheduler enter");
        // 进入主循环
        for(;;){
            if(taskcount == 0)
                //非system task数量为0, 退出进程
                exit(taskexitval);
            t = taskrunqueue.head;
            if(t == nil){
                //没有可执行的task, 退出进程
                fprint(2, "no runnable tasks! %d tasks stalled\n", taskcount);
                exit(1);
            }
            // 从全局taskrunqueue里删除当前准备执行的task,
            // 通过把t赋值给taskrunning,把t设置为准备执行的task
            // 然后调用contextswitch切换上下文
            deltask(&taskrunqueue, t);
            t->ready = 0;
            taskrunning = t;
            tasknswitch++;
            taskdebug("run %d (%s)", t->id, t->name);
            contextswitch(&taskschedcontext, &t->context);

            // taskrunning复位,
            // 判断task的exiting字段是否为1,如果为1并且task不是system task,
            //  那么么全局task计数器taskcount减1
            taskrunning = nil;
            if(t->exiting){
                if(!t->system)
                    taskcount--;
                i = t->alltaskslot;
                alltask[i] = alltask[--nalltask];
                alltask[i]->alltaskslot = i;
                free(t);
            }
        }
    }
```

另外contextswitch也可以手工调用,使用taskyield函数:

```C
    int
    taskyield(void)
    {
        int n;

        n = tasknswitch;
        taskready(taskrunning);
        taskstate("yield");
        taskswitch();
        return tasknswitch - n - 1;
    }
```

这样更具灵活性,因为有时候task需要主动让出CPU,这是一个通用的模式:在即将
执行堵塞系统调用前主动让出CPU.

libtask会独占main函数, 在main函数里会执行调度器:taskscheduler, 所以在
自己的程序里不能有main函数了,但实际上可以绕过这个机制,稍微hack下
task.c,由自己的程序去调用taskscheduler.另外libtask只是一个基础的封装，
直接拿来使用除了能直接在c语言中体验协作式编程的机制并不能对程序性能带
来提升,github上有一个使用epoll的修改版本可以让异步编程更加轻松一点,另
外如果想要利用多核, 那么还是得加上线程机制,在每个线程里跑多个task，不
过如果要实现这个要做的工作可就多了.


## 关于postgresql的多版本并发控制,表锁与行锁的使用与原理

一个支持并发的系统,难以避免需要与共享内存打交道，不同的执行流访问同一
块内存区域,如果不解决数据争用的问题,那么就会引发问题:

```C
   #include <pthread.h>
   #include <stdio.h>

   int count = 0;

   void* foo(void* arg) {
       count = count + 2;
   }

   int main() {
       pthread_t t1, t2;
       void* retval;
       int parg1 = 1;
       int parg2 = 2;

       pthread_create(&t1, 0, foo, 0);
       pthread_create(&t2, 0, foo, 0);
       pthread_join(t1, &retval);
       pthread_join(t2, &retval);
       printf ("%d\n", count);
   }
```

上面那段程序启动了两个线程, 每个线程都会更新count的值, count是全局变量，
会被所有线程共享。这段代码存在数据争用的问题,因为函数foo实际上是两条指
令(在64位环境下用gcc -s可以查看汇编码):

```ASM
    foo:
      ...
      movl count(%rip), %eax
      addl$2, %eax
      ...
```

比较常用的解决办法是使用锁把相关的内存区域保护起来:

```C
   #include <pthread.h>
   #include <stdio.h>

   int count = 0;
   pthread_mutex_t mutex;

   void* foo(void* arg) {
       pthread_mutex_lock(&mutex);
       count = count + 2;
       pthread_mutex_unlock(&mutex);
   }

   int main() {
       pthread_t t1, t2;
       void* retval;
       int parg1 = 1;
       int parg2 = 2;

       pthread_create(&t1, 0, foo, 0);
       pthread_create(&t2, 0, foo, 0);
       pthread_join(t1, &retval);
       pthread_join(t2, &retval);
       printf ("%d\n", count);
   }
```

或者使用原子指令:

```C
   #include <pthread.h>
   #include <stdio.h>

   int count = 0;

   void* foo(void* arg) {
       __sync_fetch_and_add(&count, 2);
   }

   int main() {
       pthread_t t1, t2;
       void* retval;
       int parg1 = 1;
       int parg2 = 2;

       pthread_create(&t1, 0, foo, 0);
       pthread_create(&t2, 0, foo, 0);
       pthread_join(t1, &retval);
       pthread_join(t2, &retval);
       printf ("%d\n", count);
   }
```

再看编译后的汇编代码:

```ASM
   foo:
       lock addl       $2, count(%rip)
```

上面谈的是关于并发的基础知识, 对一个支持并发事务的数据库来说,也会有数据争用的问题，所以必须提供一套锁机制.
在上面的例子里使用的是互斥锁，这种锁对于一个数据库事务来说粒度太粗,有时候事务仅仅是对某张表做读操作(比如select),并不想堵塞其他事务读这张表,postgresql定义了SHARE锁,这种锁可以被多个事务加在同一张表上,但只要有一个事务没有释放锁,那么另外的事务就不能写这张表了.因为对一张表做写操作需要给这张表上EXCLUSIVE锁, 一旦对一张表加了这种锁,其他事务对这张表既不能写也不能读.

对于现今的应用程序,有很多场景需要对一张表频繁的写,如果每次写都加
EXCLUSIVE锁，那么性能是非常低的，因为有时业务在一个事务写表的时候并不
介意另外的事务去读表,所以,对于这种场景,可以使用ACCESS EXCLUSION锁,顾名
思义,这种锁的意思是一旦有事务对一张表加锁,那么其他的事务不能对这张表做
写操作,但可以读.相对应的锁是ACCESS SHARE锁,这是粒度最小的锁,当还有事务
在读一张表的时候,也允许其他事务写。postgresql在上面4种锁模式上还做了写
扩充,具体可以看源码:
```C
   /* NoLock is not a lock mode, but a flag value meaning "don't get a lock" */
   #define NoLock               0

   #define AccessShareLock      1/* SELECT */
   #define RowShareLock         2/* SELECT FOR UPDATE/FOR SHARE */
   #define RowExclusiveLock3/* INSERT, UPDATE, DELETE */
   #define ShareUpdateExclusiveLock 4/* VACUUM (non-FULL),ANALYZE, CREATE
   * INDEX CONCURRENTLY */
   #define ShareLock5/* CREATE INDEX (WITHOUT CONCURRENTLY) */
   #define ShareRowExclusiveLock6/*
   * SHARE */
   #define ExclusiveLock7/* blocks ROW SHARE/SELECT...FOR
   * UPDATE */
   #define AccessExclusiveLock 8/* ALTER TABLE, DROP TABLE, VACUUM
   * FULL, and unqualified LOCK TABLE */
```

当执行select操作时，会自动给相关的表加上一把AccessShareLock锁,上面提到
这是粒度最小的锁,只与AccessExclusiveLock锁模式冲突.

当执行select ... for update或者 select ... for share的时候,会给相关的
表加上一把RowShareLock锁，同时还有一把ExclusiveLock锁，锁的是相关的事
务。RowShareLock是配合行锁使用的,表示前这个事务今后可能会对这张表的一
些行做update操作, 一旦拿到了这把锁,那么其他的表就不能对这些行做dml操作
了.可以看看这种锁:

```SHELL
    >> BEGIN;
    >> select * from t where id=1 for update;

    id | detail
    ----+--------
    1 | test2
    1 | test1
    1 | test1
    1 | test1
    1 | test2
    (5 rows)

    >> select txid_current();

    txid_current
    --------------
    733

    >> select locktype, mode, transactionid from  pg_locks;

    locktype    |      mode       | transactionid
    ---------------+-----------------+---------------
    relation      | AccessShareLock |
    virtualxid    | ExclusiveLock   |
    relation      | RowShareLock    |
    virtualxid    | ExclusiveLock   |
    transactionid | ExclusiveLock   |           733
    (5 rows)
```

当执行insert/update/delete操作时, 会给相关的表加上RowExclusiveLock锁，
同时加一把ExclusiveLock锁,锁的是相关的事务. ExclusiveLock锁是粒度最粗
的锁， 与所有锁模式冲突,只有在对表做ddl操作或者vacuum full时会加上这把
锁.

余下的锁模式可以看文档或者源代码,里面有非常详细的介绍.

对于一个长时间运行的事务，如果锁的粒度太大，肯定是非常影响性能的,可以
根据业务场景从应用层做些調优,比如优化程序设计，或者调整表结构, 再者,尽
量使用使用粒度小的锁.


对于一个长时间运行的事务，如果锁的粒度太大，肯定是非常影响性能的,可以
根据业务场景从应用层做些調优,比如优化程序设计，或者调整表结构, 再者,尽
量使用使用粒度小的锁.

postgresql的锁类型有很多,可以给很多对象加锁，并不限于表和行, 下面列出
了postgresql所有的锁类型:

```C
   typedef enum LockTagType
   {
        LOCKTAG_RELATION,/* whole relation */
        /* ID info for a relation is DB OID + REL OID; DB OID = 0 if shared */
        LOCKTAG_RELATION_EXTEND,/* the right to extend a relation */
        /* same ID info as RELATION */
        LOCKTAG_PAGE,/* one page of a relation */
        /* ID info for a page is RELATION info + BlockNumber */
        LOCKTAG_TUPLE,/* one physical tuple */
        /* ID info for a tuple is PAGE info + OffsetNumber */
        LOCKTAG_TRANSACTION,/* transaction (for waiting for xact done) */
        /* ID info for a transaction is its TransactionId */
        LOCKTAG_VIRTUALTRANSACTION, /* virtual transaction (ditto) */
        /* ID info for a virtual transaction is its VirtualTransactionId */
        LOCKTAG_OBJECT,/* non-relation database object */
       /* ID info for an object is DB OID + CLASS OID + OBJECT OID + SUBID */

       /*
        * Note: object ID has same representation as in pg_depend and
        * pg_description, but notice that we are constraining SUBID to 16 bits.
        * Also, we use DB OID = 0 for shared objects such as tablespaces.
        */
       LOCKTAG_USERLOCK,/* reserved for old contrib/userlock code */
       LOCKTAG_ADVISORY/* advisory user locks */
   } LockTagType;
```

LOCKTAG_RELATION锁的整个relation,比如说一张表.

LOCKTAG_RELATION_EXTEND只适用与一种场景,就是当某个relation空间不够了，
在扩展空间前,需要加上这种锁.

LOCKTAG_PAGE锁的是relation的一页,比如一张表的某一页，粒度较
LOCKTAG_RELATION要小.

LOCKTAG_TUPLE锁的是一个tuple，比如一张表里的一行记录,粒度较
LOCKTAG_PAGE要小.

LOCKTAG_TRANSACTION锁的是一个事务.

LOCKTAG_VIRTUALTRANSACTION锁的是一个虚拟事务.

LOCKTAG_OBJECT锁的是一个非数据库对象.

LOCKTAG_ADVISORY是应用程序自定义锁,行为上类似linxu下的advisory lock文件锁.

advisory lock是一种应用级的锁,在postgresql9.1之前只有session级别的,在
不需要锁的时候需要主动释放.9.1引入了事务级别的,当事务结束自动释放. 这
种锁并不是强制行的,不同的事务要事先做个约定,必须保证在自己的事务执行
dml或者dll指令前获取到advisory锁,如果没有获得锁就执行dml或dll指令,那么
指令也不会被堵塞,但这样明显有问题. 这种特性其实就是linux的文件
锁:advisory lock的实现,可以看看这篇文章:
http://www.thegeekstuff.com/2012/04/linux-file-locking-types/

另外所谓的行锁,在postgresql其实也是对表的锁定,可以看官方文档并发控制那
章对锁模式的分类,凡是前面带ROW的锁都跟行锁定有关.当在一个事务里执行
"select ... from ... where ... for update/share;",那么postgresql会加一
把ExclusiveLock锁住当前事务,加一把RowShareLock锁住相关的表,这时其他事
务对相关的行可读但不可写.


## 关于postgresql的多版本并发控制,事务隔离性细节

对于一个事务来说,有几种必要的状态,来标识事务目前所处的阶段.事务的任何
一个操作结束后,都必须映射到一种状态:

  活动状态:
    表示事务开始执行.

  待提交状态:
    当执行完最后一条指令,还未执行END或者COMMIT之前，事务处于待提交状态,这
    时事务已经正确完成了所有操作,但还未被数据库所应用.

  失败待状态:
    任何一个失败的操作将导致事务进入此状态,这时需要回滚数据库.

  失败已回滚状态:
    执行ROLLBACK后到达此状态，事务结束。

  成功提交状态:
    执行END或COMMIT操作成功,到达此状态，事物结束.

在postgresql里,为COMMIT的事务所做的操作结果其他事务是看不到的,因为
postgresql没有read uncommitted级别,在事务没有commited前,事务所做的操作
结果仅仅是保存在当前的快照，对其他事务是不可见的。postgresql使用MVCC来
实现事务的隔离性，只要不使用ACCESS EXCLUSIVE锁，那么写不会堵塞读。使用
MVCC的数据库有很多,但实现机制各不相同,主要区别在于对多个版本数据的处理，
在对一张表做DML操作时，postgresql给相关的行附着了一些额外的信息,用于标
识相关的行最近被那条已经commit了的事务更新了(xmin)，被哪条正在执行的事
务锁住了(xmax)。这些信息就是postgresql实现事务的隔离性的关键,比如事务A
使用read committed级别只能读到xmin比自己事务id(xid)大或者等于的行记录,
使用repeatable read和serializable级别只能读到xmin比自己事务id小或者等
于的行记录.当xmax大于自己的事务id时，此条记录不可写。上面有几个重要的
列名: xid指的是事务的id,xmin指的是插入或更新该条记录的xid,xmax指的是删
除或锁定该条记录的xid:

```C
  typedef struct HeapTupleFields
  {
      TransactionId t_xmin;/* inserting xact ID */
      TransactionId t_xmax;/* deleting or locking xact ID */
      union
      {
          CommandIdt_cid;/* inserting or deleting command ID, or both */
          TransactionId t_xvac;/* old-style VACUUM FULL xact ID */
      }t_field3;
  } HeapTupleFields;
```

在每个事务所读到的快照里,xmin可以认为存的是最近更新过此条记录并且已经
commited的事务的id,xmax可以认为存的是当前正在对此条行记录做DML操作的事
务的id。

这里可以实验一下 在两个session里分别开启一个事务, t1和t2:

```SHELL
    >> create table t(id integer, detail text);
    >> insert into t (id, detail) values(1,'test1');
    >> select xmin, xmax, * from t;

    xmin | xmax | id | detail
    ------+------+----+--------
    715 |    0 |  1 | test1


    >> begin;
    >> update t set detail='test2' where id=1;
    >> select xmin, xmax, * from t where id=1;

    xmin | xmax | id | detail
    ------+------+----+--------
    716 |    0 |  1 | test2

    >> select txid_current();

    txid_current
    --------------
    716

    # 可以看到当t1执行了update操作后xmin变成了自己的xid,而xmax为0,那么这时切换到t2

    >> begin;
    >> select txid_current();

    txid_current
    --------------
    717

    >> select xmin, xmax, * from t where id=1;

    xmin | xmax | id | detail
    ------+------+----+--------
    715 |  716 |  1 | test1

    # 在t2的快照里,xmin为715,xmax为716,detail为test1,说明目前t1对这一行的更新对于t2
    # 是不可见的, t2只能读到比xmax较小的版本,下面我们切到t1做一次commit:

    >> commit;

    # 再切到t2:

    >> select xmin, xmax, * from t where id=1;
    xmin | xmax | id | detail
    ------+------+----+--------
    716 |  0 |  1 | test2

    # t1 commit后, t2可以读到t1做的更新了,同时xmax变为了0,说明当前没有其
    # 他未committed的事务对此行做过更新。这里之所以t2可以读t1的更新是因为
    # postgresql默认使用的是read committed事务隔离级别,若使用
    # repeatable read或者serializable隔离级别或者t1对此行加了锁(比如
    # select for update), 则是读不到的,指定隔离级别的语法:

    >> begin;
    >> SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

在一个事务内,对于一条记录,每做一次DML操作，postgresql都会存储相关操作
的结果。只是因为隔离级别的限制,其他事务可能无法读到而已:

```C
     typedef struct HeapTupleHeaderData
     {
         ...
         ItemPointerData t_ctid;/* current TID of this or newer tuple */
         ...
    } HeapTupleHeaderData;
```

对于每个事务,当执行一次DML操作时,实际上postgresql会为此次操作保存一份
新的副本,并不会替换之前已经存在的数据, 比如update操作:

```SHELL
    >> select ctid, * from t where id=1;

    ctid | id | detail
    ------+----+--------
    (0, 3) | 1 | test2 |

    >> update t set detail="test3" where id=1;
    >> select ctid, * from t where id=1;

    ctid | id | detail
    ------+----+--------
    (0, 4) | 1 | test3 |

    # 这里的ctid指的是一个表示(存储的数据块，偏移量）的tuple，在此事务
    # update之前,ctid的值是(0,3),update后,ctid的值变成了(0,4),说明
    #  postgresql并没有在update后覆盖之前的数据,而是新产生了一份数据.这
    #  么一来,当系统执行update操作非常多的情况下,必定会有大量的dead
    # tuples存在.postgresql默认会启动一个专门用于清除dead tuples的进程:

    >> ps ax | grep postgres

       824 ?        Ss     0:06 postgres: autovacuum launcher process

    # autovacuum会判断当前dead tuples的数量是否达到了一个阈值(可配置),如
    # 果达到了阈值范围,则会执行清理操作.这个阈值的参数设置非常重要,需要根
    # 据具体业务做取舍,如果阈值设的非常大,那么执行清理操作的过程会非常耗
    # 时,且数据库会有大量冗余数据,如果设的非常小，那么也会给系统带来不少
    # 压力。具体的配置参数可以参考文档:
    # http://www.postgresql.org/docs/9.1/static/runtime-config-autovacuum.html
```

## 关于postgresql的并发控制,ACID实现细节

一个可靠的关系数据库事务必须实现ACID特性，即原子性，一致性，隔离性，持久性。

事务的原子性跟CPU提供的原子指令有一定区别，原子指令是一条指令，本质上
就是串行的，而事务的原子性只保证事务中的指令，要么都执行，要么都不执行，
至于是否需要串行化，取决于事务的隔离级别。

事务的一致性指的是一个事务执行前后数据库都必须处于正确状态。如果一个事
务成功commit了,那么数据库必须正确地应用此事务的所有操作，如果事物
rollback了,那么数据库必须回滚此次事务中的所有操作。

事务的隔离性指的是多个事务间的操作可见性.sql标准定义了4个隔离级别:

1. Read Uncommitted(读未提交)

  即一个事务在执行过程中可以读其他事务已更新或插入但尚未commit的行。

2. Read Commited(读已提交)

  即一个事务在执行过程中可以读其他事务已经更新或插入并且commit了的行.

  3. Repeatable Read(可重复读)

  即一个事务在执行过程中可以读其他事务已经插入并且commit了的行，不能读
  已经更新了的行。

4. Serializable(串行)

  同一时间只允许一个事务执行，不允许并发，所以不存在可见性的问题串行是
  最安全的隔离级别，但失去了并发性，而其他隔离级别在有事务并发执行时都
  会有一些问题，可以归纳为以下4种：

  1. 丢失更新

  事务t1和t2并发访问表a：

  step | t1 | t2
    ------------ | ---------------------- | ------------
    1            | begin;                 |
    2            |                        | begin;
    3            | select id from a;      |
    4            | newid=id + 2;          |
    5            |                        | select id from a;
    6            | update a set id=newid; |
    7            |                        | update a set id=newid;
    8            | commit;                |
    9            |                        | commit;

  这里很明显t1对表a的相关行更新被丢失了,假如两个事务尚未执行前id的值是
  5,两个事务执行完毕后id的值会是8,而不是10Read Uncommitted隔离级别会造
  成此问题。

  2. 脏读:

  事务t1和t2并发访问表a:

  | step | t1                    | t2                    |
     |------|-----------------------|-----------------------|
     |    1 | begin;                |                       |
     |    2 |                       | begin;                |
     |    3 | select id from a;     |                       |
     |    4 | update a set id=id+2; |                       |
     |    5 |                       | update a set id=id+2; |
     |    6 | rollback;             |                       |
     |    7 |                       | commit;               |

  这里t1最终rollback了,但它之前的对表a的一些行的更新结果被t2读到了,所
  以t2实际上读到的是脏数据。Read Uncommitted隔离级别会造成此问题。


  3. 不可重复读

  事务t1和t2并发访问表a:

  | step | t1                    | t2                    |
    |------|-----------------------|-----------------------|
    |    1 | begin;                |                       |
    |    2 |                       | begin;                |
    |    3 |                       | update a set id=id+2; |
    |    4 |                       | commit;               |
    |    5 | update a set id=id+2; |                       |
    |    6 | commit;               |                       |

  这里t1在update的时候id的值已经被t2更新过了，而与t1 begin时id的值不一
  致.read uncommitted 和read committed隔离级别会造成此问题

  4. 幻象读

  事务t1和t2并发访问a表:

  | step | t1                          | t2                            |
    |------|-----------------------------|-------------------------------|
    |    1 | begin;                      |                               |
    |    2 |                             | begin;                        |
    |    3 |                             | insert into a (id) values (5) or delete table a where id=5|
    |    4 |                             | commit;                       |
    |    5 | select id from a where id=5 |                               |
    |    6 | commit;                     |                               |

  在这里t1会读到t2新插入的一行或本来应该读到的行被t2删除了,这就是幻象
  读。read uncommitted，read committed，repeatable read隔离级别会造成
  此问题。

事务的持久性指的是一旦事务成功提交，那么此事务的执行结果就不应该被丢失。
其实数据丢失问题是不可能避免的，所以关系数据库大多会提供数据恢复机制。


Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/wowsoso/wowsoso.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
