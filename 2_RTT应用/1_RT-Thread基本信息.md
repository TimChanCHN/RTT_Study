# RT_Thread应用(一)
## 1. RT-Thread启动流程
1. 方式1--全部部件(硬件、系统、各个线程)初始化完成，之后再启动RTOS
2. 方式2--全部部件初始化，再初始化初始线程，启动RTOS
   1. 初始线程用于创建其他线程，创建成功之后需要关闭初始线程
3. RT-Thread启动流程
   1. 初始化系统堆栈大小、中断向量表，并在`Reset_Handler`中进行系统初始化后开始跳去`main`函数
      1. 系统初始化`SystemInit`
         1. 设置系统时钟
         2. 中断向量表重定向至内部Flash
      2. 跳去`__main`函数
         1. `__main`为运行时库提供的函数
         2. 完成“段拷贝”，因为程序的加载地址和执行地址大概率不同(因为片内内存太小)
            1. `__scatterload()`:将非零（只读和读写）运行区域从其载入地址复制到运行地址(载入地址是RO，运行地址是RW), 清零ZI区域(未初始化变量区)
            2. `__rt_entry()`:负责初始化堆栈，完成库函数的初始化
   2. 完成`1.2.2`后程序会跳转到`component.c`的`$Sub$$main`(该函数是keil对main的扩展[对应宏`__CC_ARM`]，IAR和KEIL会有所区别)，在该函数中会进行启动rt-thread的工作
         1. 关中断
         2. rtthread_startup
   3. rtthread_startup：
      1. 关中断
      2. 硬件初始化:rt_hw_board_init
      3. 定时器初始化:rt_system_timer_init
      4. 调度器初始化:rt_system_scheduler_init
      5. 信号量初始化:rt_system_signal_init
      6. 创建初始线程(main在此处进去):rt_application_init
      7. 定时器线程初始化:rt_system_timer_thread_init
      8. 空闲线程初始化:rt_thread_idle_init
      9. 启动调度器:rt_system_scheduler_start
   4. rt_application_init
      1. 初始化初始线程，线程函数是:`main_thread_entry`
      2. 启动初始线程
   5. $Super$$main()
      1. 进入C程序的main函数中，其他线程就在main函数中启动

## 疑问：
   1. rtthread_startup的调用函数是entry，但是未见哪里调用了entry?
      1. 该函数在`__main`执行结束后，编译器会自动调用。在MDK的编译器中，并不会直接执行entry，而是执行`$Sub$$main`，而entry是在其他编译环境下使用。
   2. 哪种做法导致了原来正点原子的库函数工程接入了rt-thread系统？

## 备注:
1. STM32内存区域说明:`https://blog.csdn.net/qq_31073871/article/details/102569983`
2. RO,RW,ZI区别:`https://blog.csdn.net/jamestaosh/article/details/4348385`
3. 同上:`https://www.cnblogs.com/luckytimor/p/7182629.html`
