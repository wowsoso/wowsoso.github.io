## 简单聊聊分布式事务设计关键点与取舍

类似2PC,Paxos之类的强一致性方案在跨多个服务的场景下很吃力, 而且性能也不好, 所以应该考虑基于最终一致性的方案.

基于最终一致性的方案业界有很多方法实践, 比如本地事务表加异步确保,TCC补偿, Saga等, 这些方式可以从编排方式上分为两类: 去中心式和集中管理式. 下面简单聊聊。

### 去中心式
举个例子, 三个服务: Order, Payment, Stock:
##### 成功:
![Alt text](http://wowsoso-common.oss-cn-beijing.aliyuncs.com/a1.png)

##### 失败:
![Alt text](http://wowsoso-common.oss-cn-beijing.aliyuncs.com/a2%20%282%29.png)

每个服务有两种角色, 事件发布者和接收者

事件发布者需要考虑:
* 存储发送事件, 便于redo
* 要有全局事务id式事务可以被追溯

事件接收者需要考虑:
 *  存储接收的事件, 便于redo或者undo.
 * redo或者undo需要考虑幂等, 可以依靠全局事务id

#### 优点:
* 去中心化设计, 松耦合.
* 实现简单.

#### 缺点:
* 对业务有侵入性。
* 一个事务如果涉及太多服务, 那么会变得复杂, 当然, 这时也应该考虑下服务边界的设计是否有问题。
* 功能测试比较麻烦, 需要拉起所有服务。

### 集中管理式
还是上面那个例子:

![Alt text](http://wowsoso-common.oss-cn-beijing.aliyuncs.com/a3.png)

由一个事务管理器处理事务, 当Order向事务管理器发起一次请求, 事务管理器会分别向Payment和Stock调用, 若Payment和Stock都调用成功, 那么事务完成, 否则要执行undo操作:

![Alt text](http://wowsoso-common.oss-cn-beijing.aliyuncs.com/a4.png)

这两幅图里, 调用关系可以走消息队列, 不一定是request/response.

#### 优点:
* 对业务侵入性较小
* 当事务设计的服务增多, 复杂性线性增长
* 比较容易测试

#### 缺点:
* 实现比较麻烦
* 中心化的设计其实会造成耦合, 业务逻辑变更频繁的场景, 会很麻烦.
* 额外的维护成本

具体实现还是要根据场景做取舍, 当然本质就是BASE, 万变不离其宗。
Done.

## python下协程实现原理与greenlet源码解析

greenlet是一个高效的python协程扩展,与libtask不同,由于greenlet是跑在
python虚拟机上的,而python虚拟机在操作系统上模拟了进程,线程与栈帧结构,
所以greenlet协程在做切换时必须同时切换python虚拟机的栈帧与操作系统进程
的栈帧.而这也是greenlet实现比较tricky的地方,要搞清楚greenlet的具体实
现机制,必须先搞清楚python的三个重要结构体:

  PyFrameObject
  PyInterpreterState
  PyThreadState

先来看看PyFrameObject:

```c
    typedef struct _frame {
        // PyFrameObject就是python虚拟机的栈帧结构,
        // 也就是一段python的执行代码片,包括:
        // 指向前一个栈帧的指针, 执行代码片地址,builtins dict, globals dict, locals dict
        // 指向最后一个local地址的指针,指向栈顶地址的指针, 代码片所在线程的指针, 第一个local地址的指针

        PyObject_VAR_HEAD
        // f_back指向调用者的栈帧
        struct _frame *f_back;	/* previous frame, or NULL */
        // 执行代码片
        PyCodeObject *f_code;	/* code segment */
        PyObject *f_builtins;	/* builtin symbol table (PyDictObject) */
        PyObject *f_globals;	/* global symbol table (PyDictObject) */
        PyObject *f_locals;		/* local symbol table (any mapping) */
        // f_valuestack指向最后一个local地址
        PyObject **f_valuestack;	/* points after the last local */
        /* Next free slot in f_valuestack.  Frame creation sets to f_valuestack.
           Frame evaluation usually NULLs it, but a frame that yields sets it
           to the current stack top. */
        // f_stacktop指向栈顶, 当线程初始化时,f_stacktop和f_valuestack指向相同地址
        PyObject **f_stacktop;
        PyObject *f_trace;		/* Trace function */

        /* If an exception is raised in this frame, the next three are used to
         * record the exception info (if any) originally in the thread state.  See
         * comments before set_exc_info() -- it's not obvious.
         * Invariant:  if _type is NULL, then so are _value and _traceback.
         * Desired invariant:  all three are NULL, or all three are non-NULL.  That
         * one isn't currently true, but "should be".
         */
        PyObject *f_exc_type, *f_exc_value, *f_exc_traceback;

        PyThreadState *f_tstate;
        int f_lasti;		/* Last instruction if called */
        /* Call PyFrame_GetLineNumber() instead of reading this field
           directly.  As of 2.3 f_lineno is only valid when tracing is
           active (i.e. when f_trace is set).  At other times we use
           PyCode_Addr2Line to calculate the line from the current
           bytecode index. */
        int f_lineno;		/* Current line number */
        int f_iblock;		/* index in f_blockstack */
        PyTryBlock f_blockstack[CO_MAXBLOCKS]; /* for try and loop blocks */
        // f_localsplus指向第一个local
        PyObject *f_localsplus[1];	/* locals+stack, dynamically sized */
    } PyFrameObject;
```

下面是PyInterpreterState结构的源码:

```c
typedef struct _is {
    // _is是Python虚拟机下的进程模拟

    // next指向下一个Python虚拟机进程
    struct _is *next;
    // 指向这个进程的第一个线程
    struct _ts *tstate_head;

    // 存储当前进程下共享的modules,sysdict,builtins,
    PyObject *modules;
    PyObject *sysdict;
    PyObject *builtins;
    PyObject *modules_reloading;

    PyObject *codec_search_path;
    PyObject *codec_search_cache;
    PyObject *codec_error_registry;

#ifdef HAVE_DLOPEN
    int dlopenflags;
#endif
#ifdef WITH_TSC
    int tscdump;
#endif
} PyInterpreterState;
```

假如我们用编译型语言去实现一套协程机制,比如c, 其实是非常简单的,我们可
能只需要一个合理的上下文结构, 一套用于保存,恢复的上下文切换函数就可以
了, 但对于Python这种使用虚拟机模拟进程/线程结构的语言,做一次协程的上下
文切换,我们不仅要切换真实操作系统的上下文,同时也要切换虚拟机模拟的上下
文.因此, greenlet的实现有很多tricky的点, 在对源码的分析过程中遇到了不
少麻烦, 下面仅以我个人阅读greenlet源码的路径来分析一下这个在生产环境下
被使用无数的Python协程框架.

1. 基本执行单元

Task是一个协程的基本执行单元, 下面是greenlet源码里对Task的抽象:

```c
    typedef struct _greenlet {
        PyObject_HEAD
        // stack_start指向task的栈顶, 在main_task里, stack_start是-1,
        // 因为greenlet无法知道真实的栈顶地址,所以用-1表示最大的地址
        char* stack_start;
        // stack_stop指向栈底, main_task没有stack_stop,用1表示最小的地址.
        char* stack_stop;
        // stack_copy是指的保存在堆上的数据,当做上下文切换前,会把当前task的stack上
        // 的数据保存在堆上,切换之后再恢复入栈.
        char* stack_copy;
        // 标识当前是否已经把数据保存到堆上
        intptr_t stack_saved;
        // 比如在task1里切换到task2, 那么task2的stack_prev就指向task1
        struct _greenlet* stack_prev;
        // 指向父task
        struct _greenlet* parent;
        // 指向python线程的dict,通常是用于比较task是否属于当前线程(是否是在当前线程里创建的),
        // 在一个greenlet对象初始化时(非main greenlet task), 也用来暂存callback对象的指针
        PyObject* run_info;
        // 指向执行栈帧
        struct _frame* top_frame;
        // 递归深度,与所属线程保持一致
        int recursion_depth;
        PyObject* weakreflist;
        PyObject* exc_type;
        PyObject* exc_value;
        PyObject* exc_traceback;
        // 当前Python线程的dict字段
        PyObject* dict;
    } PyGreenlet;
```

下面是创建main task的源码:

```c
    static PyGreenlet* green_create_main(void)
    {
        PyGreenlet* gmain;
        PyObject* dict = PyThreadState_GetDict();
        if (dict == NULL) {
            if (!PyErr_Occurred())
                PyErr_NoMemory();
            return NULL;
        }

        /* create the main greenlet for this thread */
        gmain = (PyGreenlet*) PyType_GenericAlloc(&PyGreenlet_Type, 0);
        if (gmain == NULL)
            return NULL;
        gmain->stack_start = (char*) 1;
        gmain->stack_stop = (char*) -1;
        gmain->run_info = dict;
        Py_INCREF(dict);
        return gmain;
    }
```

对main task的初始化与task的初始化不同,初始化main task不用设置run属性
(grennlet的callback函数),稍后介绍普通task的初始化时会提到run属
性.maintask对greenlet框架的使用者是透明的,在import greenlet时会执行初
始化操作,每个线程都可以有自己的main task,它的stack_stop属性永远
为-1, stask_start属性永远为1


下面是创建并初始化task的源码:

```c
    static PyObject* green_new(PyTypeObject *type, PyObject *args, PyObject *kwds)
    {
        // 此函数构造一个新的greenlet task并返回, 设置新的task的parent为当前的task
        PyObject* o = PyBaseObject_Type.tp_new(type, ts_empty_tuple, ts_empty_dict);
        if (o != NULL) {
            if (!STATE_OK) {
                Py_DECREF(o);
                return NULL;
            }
            Py_INCREF(ts_current);
            ((PyGreenlet*) o)->parent = ts_current;
        }
        return o;
    }

    static int green_init(PyGreenlet *self, PyObject *args, PyObject *kwargs)
    {
        // 此函数初始化一个greenlet task, 设置run_info为参数run, 设置parent为参数parent
        PyObject *run = NULL;
        PyObject* nparent = NULL;
        static char *kwlist[] = {"run", "parent", 0};
        if (!PyArg_ParseTupleAndKeywords(args, kwargs, "|OO:green", kwlist,
                         &run, &nparent))
            return -1;

        if (run != NULL) {
            if (green_setrun(self, run, NULL))
                return -1;
        }
        if (nparent != NULL && nparent != Py_None)
            return green_setparent(self, nparent, NULL);
        return 0;
    }
```

在green_init函数里有调用green_setrun函数,green_setrun函数会把指定的回
调函数对象的指针暂时赋值给task的run_info字段, 当切换到此task时会调用它:

```c
    result = PyEval_CallObjectWithKeywords(
        run, args, kwargs);
```

上面的源码对应于:

```python
    import greenlet

    def foo(...):
        ....

    greenlet.greenlet(foo, ...)
```

python解释器执行imoirt greenlet时会初始化greenlet的 main_task, 执行
greenlet.greenlet(foo, ...)时会调用green_new创建一个greenlet task并调
用green_init初始化它.

关于切换, 之前提到greenlet的切换不仅要切换python解释器模拟的栈帧,同时
也要切换操作系统进程的栈帧：

```c
    static int g_switchstack(void)
    {
        // 这个函数分为三个部分, 保存Python虚拟机栈帧,保存并恢复操作系统进程栈帧,恢复Python虚拟机栈帧
        // 1. 在保存阶段, 会把当前线程的上下文数据赋给当前greenlet task,
        // 2. 然后进入操作系统进程栈帧切换阶段: 执行slp_switch函数, 这个函数非常关键,接下来会详细描述,
        // 3. 最后进入恢复阶段, 把切换目标的上下文数据写入当前线程
        int err;
        {   /* save state */
            PyGreenlet* current = ts_current;
            PyThreadState* tstate = PyThreadState_GET();
            current->recursion_depth = tstate->recursion_depth;
            current->top_frame = tstate->frame;
            current->exc_type = tstate->exc_type;
            current->exc_value = tstate->exc_value;
            current->exc_traceback = tstate->exc_traceback;
        }
        err = slp_switch();
        if (err < 0) {   /* error */
            PyGreenlet* current = ts_current;
            current->top_frame = NULL;
            current->exc_type = NULL;
            current->exc_value = NULL;
            current->exc_traceback = NULL;

            assert(ts_origin == NULL);
            ts_target = NULL;
        }
        else {
            PyGreenlet* target = ts_target;
            PyGreenlet* origin = ts_current;
            PyThreadState* tstate = PyThreadState_GET();
            tstate->recursion_depth = target->recursion_depth;
            tstate->frame = target->top_frame;
            target->top_frame = NULL;
            tstate->exc_type = target->exc_type;
            target->exc_type = NULL;
            tstate->exc_value = target->exc_value;
            target->exc_value = NULL;
            tstate->exc_traceback = target->exc_traceback;
            target->exc_traceback = NULL;

            assert(ts_origin == NULL);
            Py_INCREF(target);
            ts_current = target;
            ts_origin = origin;
            ts_target = NULL;
        }
        return err;
    }
```

slp_switch函数(以x86x64平台为例):

```c
    static int
    slp_switch(void)
    {
        // 这个函数将切换操作系统进程的栈帧
        // 按惯例rbp是栈帧, rbx需要被被调用者保存,
        // stackref存储的是栈顶地址

        int err;
        void* rbp;
        void* rbx;
        unsigned int csr;
        unsigned short cw;
        register long *stackref, stsizediff;
        __asm__ volatile ("" : : : REGS_TO_SAVE);
        __asm__ volatile ("fstcw %0" : "=m" (cw));
        __asm__ volatile ("stmxcsr %0" : "=m" (csr));
        // 保存栈帧至rbp变量
        __asm__ volatile ("movq %%rbp, %0" : "=m" (rbp));
        // 保存rbx内容至rbx变量
        __asm__ volatile ("movq %%rbx, %0" : "=m" (rbx));
        // 保存栈顶的地址至stackref
        __asm__ ("movq %%rsp, %0" : "=g" (stackref));
        {
            // SLP_SAVE_STATE宏做两件事情:
            //  1. 把当前greenlet task及它之前的task保存到堆, 界限为目标task的stack_stop
            //  2. check目标greenlet task是否是running状态,如果不是running状态,则不再执行后面的切换操作了,直接返回
            //  3. 算出目标greenlet task与当前greenlet task的地址差,用于切换
            //  当目标greenlet task被首次切换,说明其还不在running状态, 所以直接返回-1,上层函数
            // initialstub会通过把(char*) 1赋值给目标greenlet task的stack_start,
            // 从而使stack_start开始有值且值小于stack_stop(因为栈地址分配由下往上),
            // 然后会调用其run(callback)函数, 执行完毕后将继续进行一次切换,
            // 而这次切换因为stack_start有值所以会通过SLP_SAVE_STATE宏的步骤2(检测目标greenlet task是否为running),
            // 然后python解释器会接着往下执行.
            SLP_SAVE_STATE(stackref, stsizediff);
            // 到这一步目标greenlet task的run函数已经调用结束了或者执行到某一步待切换
            // 更改操作系统进程的栈帧,指向目标greenlet task
            // 由于增长了rsp和rbp的地址(stsizediff个字节), 从而回到了调用switch者(即ts_current)的栈帧
                __asm__ volatile (
                "addq %0, %%rsp\n"
                "addq %0, %%rbp\n"
                :=-=-
                : "r" (stsizediff)
                );
            // SLP_RESTORE_STATE宏把之前各相关greenlet task保存在堆上的数据写回stack,
            // 并释放内存空间,重置stack_saved为0
            SLP_RESTORE_STATE();
            __asm__ volatile ("xorq %%rax, %%rax" : "=a" (err));
        }
        __asm__ volatile ("movq %0, %%rbx" : : "m" (rbx));
        __asm__ volatile ("movq %0, %%rbp" : : "m" (rbp));
        __asm__ volatile ("ldmxcsr %0" : : "m" (csr));
        __asm__ volatile ("fldcw %0" : : "m" (cw));
        __asm__ volatile ("" : : : REGS_TO_SAVE);
        // 走到这一步err肯定为0, 操作系统进程栈帧切换完毕,
        // 之后会回到g_switchstack, 恢复python虚拟机的栈帧, 整个切换过程结束.
        return err;
    }
```

上面就是greenlet的切换过程, slp_switch函数就是整个切换过程的核心实现.在
python下,通过调用greenlet的switch方法触发调用PyGreenlet_Switch函
数,PyGreenlet_Switch会调用g_switchstack和g_initialstub函数完成一次切换,具
体细节可以参考roy的文章：http://rootk.com/post/python-greenlet.html

## 关于linux下协程的通用实现及libtask库源码解析

协程(coroutine)与subroutine同样作为程序执行单元的抽象被一些语言当作基
础实现,两者的抽象方式大致区别在于：

1. 多个执行单元之间的关系：

对于subroutine来说,存在一个调用与被调用的关系,比如在a-subroutine里调用
b-subroutine, 那么a-subroutine就是调用者,b-subroutine是被调用者,它们共
享一个线程的堆栈.

而多个coroutine之间的关系是对等的,即便在a-coroutine里创建了
b-coroutine,他们之间也不会有任何层级关系.

subroutine作为一种通用的抽象比较容易实现,而要实现coroutine至少需要两个
条件:

  * 要有一个全局的调度器并且每个coroutine得有一个堆栈空间.

  * 调度器用来在多个对等的coroutine之间做切换操作,每个coroutine的堆栈
    用于存储各自的上下文内容.

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
b-coroutine时,我们可以在任意点把当前的执行信息写回这块在堆上分配的空
间,然后把rsp指向a-coroutine的栈顶,rbp指向a-coroutine的栈frame,rip指向
a-coroutine的需要继续执行的指令的地址, 这也就是所谓的用户态的上下文切
换,当下一次需要继续执行b-coroutine的时候,保存当前coroutine的上下文,
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

```c
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

```c
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

当需要切换上下文时,需要调用swapcontext, swapcontext接受两个参数,
  1. 当前的上下文(u_context).

  2. 新的上下文(u_context).

x86-64下源码如下:

```c
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

从swapcontext的实现可以看出swapcontext所做的事很简单,保存,恢复.把当前
上下文按顺序保存到rdi的偏移,新的上下文(rsi指向的地址)覆盖老的上下文.

swapcontext实际上是getcontext/setcontext的结合体,比如参照ai64的实现:

```c
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

```c
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

    // 这个结构体里context就是ucontext_t:
    struct Context {
        ucontext_t uc;
    }
```

ucontext_t里存储了上下文内容, 也就是swapcontext函数里两个参数的类型stk
指向在堆上分配的空间,被做为栈赋值给ucontext_t的ss_sp字段, 这是
makecontext函数要求的,在调用makecontext函数前,必须为参数ucp分配一块地
址作为上下文的栈空间,libtask会执行初始化工作:

```c
    t->context.uc.uc_stack.ss_sp = t->stk+8;
    t->context.uc.uc_stack.ss_size = t->stksize-64;
    ...
```

在调用swapcontext前,可以通过比较当前context的地址是否大于stk的地址来判
断栈空间是否够用:

```c
    void needstack(int n)
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

![libtask1](https://wowsoso.github.io/images/libtask1.png)

所以当 (char*)&t <= (char*)t->task时,说明栈空间不够了.

如果栈空间足够,那么就可以调用swapcontext了:

```c
    static void
    contextswitch(Context *from, Context *to)
    {
        if(swapcontext(&from->uc, &to->uc) < 0){
            fprint(2, "swapcontext failed: %r\n");
            assert(0);
        }
    }
```

contextswitch函数是由taskscheduler函数驱动的,taskscheduler是一个task
全局调度器, 运行在进程的整个生命周期中,调度器结束,进程关闭:

```c
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

```c
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

github上有一个使用epoll的修改版本,另外如果想要利用多核, 还是需要使用线程,在每个线程里跑多个task,不过如果要实现这个工作量还比较大.