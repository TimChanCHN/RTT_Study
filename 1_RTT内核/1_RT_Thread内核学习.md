# RT_Thread内核学习
## 1. 裸机系统与多线程系统
### 1. 裸机系统
1. 轮询系统
   1. 在裸机编程时，先初始化相关硬件，后在主程序的一个死循环中循环处理各种事件
   2. 缺点：
      1. 不能实时处理事件，可能会发生事件的丢失；
   
2. 前后台系统
   1. 在轮询系统的基础上加上中断。因为中断处理的事情要尽量少，否则会影响系统的实时性。如按键中断，设定标志来确认按键事件发生，之后在主函数中处理该事件。
   2. 和轮询系统相比，前后台系统有较好的实时性。好的前后台系统，可以达到RTOS的效果。

### 2. 多线程系统
1. 多线程系统的事件响应也是在中断中完成，事件处理是在线程中完成，即事件响应、处理都是实时进行的
2. 加入操作系统，可以使得编程变得简单，并且各个功能模块之间的干扰可以得到避免，付出的代价就是操作系统会占据FLASH和RAM，但是占据的并不大

## 2. RTT的线程
### 1. 线程
1. 线程：
   1. 独立且无法返回的函数，对外而言，线程等同于裸机系统的main函数；
   2. 每个线程下相互独立，互不干扰；
   3. 每个线程都需要被分配独立的栈空间，栈空间是一个预先定义好的全局数组，也可以是动态分配的一段内存空间
2. 创建线程流程
   1. 定义线程栈:本质上是一个uint8型的数组
   ```c
        rt_uint8_t rt_flag1_thread_stack[512];
   ```
   2. 定义线程函数
   3. 定义线程控制块
    ```c
        // 线程控制块相当于线程的身份证，里面存储了线程的所有信息
        // 目前只是一个简单的线程控制块内容，后续可以不断增加
        // 相当于给线程控制块增加属性内容
        struct rt_thread
        {
            void *sp;                       /* 线程栈指针 */
            void *entry;                    /* 线程入口地址 */
            void *parameter;                /* 线程形参 */
            void *stack_addr;               /* 线程栈起始地址 */
            rt_uint32_t stack_size;         /* 线程栈大小，单位为字节 */
            ...
        };
        typedef struct rt_thread *rt_thread_t;
    ```
   4. 实现线程创建函数
   ```c
        // args1:线程控制块指针
        // args2:线程函数入口地址
        // args3:线程参数
        // args4:线程栈起始地址
        // args5:线程栈大小
        rt_err_t rt_thread_init(struct rt_thread *thread,
            void (*entry)(void *parameter), 
            void *parameter, 
            void *stack_start, 
            rt_uint32_t stack_size){}
        
        // 创建函数的过程
        1. 初始化链表节点：rt_list_init(&(thread->tlist))
        2. 把函数参数赋值控制块中
        3. 初始化线程栈，并返回线程栈指针:
        (void *)rt_hw_stack_init( thread->entry,
                                    thread->parameter,
                                    (void *)((char *)thread->stack_addr + thread->stack_size - 4) );
        // 栈入口(栈顶指针)地址：线程栈数组的起始地址+栈大小
        // -4：在函数内部会+4来抵消
   ```
    5. 线程栈初始化函数
       1. 获取栈顶地址，并令其8字节对其(兼容64位系统)
       2. 栈指针下移`sizeof(struct stack_frame)`,目的是存储寄存器，该寄存器是在异常发生时，自动加载到对应的CPU寄存器里面，具体寄存器类型可以查看`struct stack_frame`的内容
       3. 对sp指针的类型强转成`struct stack_frame`并对其初始化：在该初始化中，完成CPU的r0~r14等寄存器的初始化
       4. 返回堆栈指针

3. 实现就绪列表
   1. 线程创建好之后，需要把线程添加到就绪列表中，表示线程已经就绪，供系统随时调度
   ```c
        // 就绪列表
        // 数组的下标对应着线程的优先级
        rt_list_t rt_thread_priority_table[RT_THREAD_PRIORITY_MAX];
   ```
   2. 将线程插入就绪列表
   ```c
        // args1:就绪列表的下表决定了线程的优先级
        // args2:线程控制块的tlist成员，通过线程控制块的tlist把线程插入到就绪列表中实现把线程和就绪列表之间的关联
        rt_list_insert_before( &(rt_thread_priority_table[0]),&(rt_flag1_thread.tlist) );
   ```

4. 实现调度器
   1. 调度器：操作系统的核心，实现线程的切换，先从就绪列表中找到优先级最高的线程，之后会执行该线程；优先级最高的线程执行结束后，会令其优先级变为最低，如此轮回，从而实现线程的调度；
   2. 调度器的初始化
      1. `void rt_system_scheduler_init(void)`
      2. 主要工作：
         1. 初始化线程就绪列表，所有的就绪列表都为空
         2. 初始化当前线程控制块指针
      3. 调度器的初始化要在硬件初始化之后，线程创建之前
   3. 启动调度器
      1. `void rt_system_scheduler_start(void)`
      2. 


