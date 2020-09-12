# RT_Thread内核学习(1)
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
      2. 主要工作:
         1. 通过宏`rt_list_entry`找到当前优先级最高的线程的线程控制块
         2. 标记此线程控制块为全局变量`rt_current_thread`
         3. 切换到该线程`rt_hw_context_switch_to((rt_uint32_t)&to_thread->sp)`
   4. 第一次线程切换
      1. `rt_hw_context_switch_to() `
         1. 把即将要调度的线程通过r0赋值到`rt_interrupt_to_thread`中
         2. 令`rt_interrupt_from_thread`的值为0，表示启动第一次线程切换
         3. 令`rt_thread_switch_interrupt_flag`为1，当执行了`PendSVC Handler`时，该flag清零
         4. 设置PendSV异常的优先级为最低，并触发PendSV异常(产生上下文切换)
         5. 需要中断打开后才会执行PendSV中断
      2. `PendSV_Handler()`
         1. 关中断，保证上下文切换不被中断
         2. 判断`rt_thread_switch_interrupt_flag`是否为0，是则退出`PendSVC Handler`，否则令该flag清零
         3. 判断`rt_interrupt_from_thread`是否为0，是则直接跳去下文切换，否则跳去上文保存
         4. 上文保存:
            1. `xPSR,PC（线程入口地址）,R14,R12,R3,R2,R1,R0（线程的形参）`会自动保存
            2. 手动保存r4-r11
         5. 下文切换:
            1. 获取`rt_interrupt_to_thread`的值，这是线程栈指针SP的指针，之后获取SP的值
            2. 把栈里面的内容加载到CPU寄存器r4~r11
            3. 并把线程栈指针r1更新到psp
         6. 中断恢复
            1. 恢复中断
            2. 新的线程栈中的`xPSR,PC（线程入口地址）,R14,R12,R3,R2,R1,R0（线程的形参）`会自动保存
   5. 系统调度
      1. `rt_schedule()`：只是一个简单的if..else..函数，用于判断当前最高优先级的线程控制块，之后就执行下个优先级的线程控制块
      2. `rt_hw_contex_switch()`:产生上下文切换
         1. 把`rt_thread_switch_interrupt_flag`加载到r2
         2. 把r2的值暂存到r3并和1比较，如果相等则启动切换，否则对`rt_thread_switch_interrupt_flag`进行置1
         3. 把上一个线程栈指针存储到`rt_interrupt_from_thread`
         4. 把下一个线程栈指针存储到`rt_interrupt_to_thread`
         5. 启动上下文切换 
   
5. main函数结构
   1. 定义线程栈
   2. 定义/声明线程函数
   3. 调度器初始化
   4. 线程初始化、将线程插入就绪列表
   5. 启动系统调度器
