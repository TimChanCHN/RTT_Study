# RT_Thread内核学习(2)
## 1. 临界段保护
### 1.基本概念
1. 临界段：在执行过程中不能被中断的代码段；
2. 对临界段的操作是常常是对全局变量的操作
3. 临界段被打断主要是中断和系统调度；
4. RTT对临界段的保护处理时直接关掉全部中断，除了NMI FAULT和HARDWARE FAULT外

### 2.中断相关操作
1. CPS指令：对中断快速开关的指令
   ```bash
        CPSID I ;PRIMASK=1      ;关中断
        CPSIE I ;PRIMASK=0      ;开中断
        CPSID F ;FAULTMASK=1    ;关异常
        CPSIE F ;FAULTMASK=0    ;开异常
   ```
2. Cortex-M内核中断屏蔽寄存器
   1. PRIMASK:单一bit的寄存器，作用是开关所有可屏蔽的中断
   2. FAULTMASK:单一bit的寄存器，作用是开关所有中断，即使是硬件FAULT
   3. BASEPRI:定义被屏蔽中断的优先级

### 3. 开关中断
1. 在RTT中，封装了函数`rt_hw_interrupt_disable`/`rt_hw_interrupt_enable`对中断进行开关设置
2. 对中断的开关，当然也可以直接如2.1一样操作，但是这种后果会导致临界段嵌套时，内层临界段结束退出时的关闭中断会使得外层的临界段也中断，因此在有临界段嵌套的情况下，使用3.1的中断开关函数

## 2. 对象容器的实现
### 1.基本概念
1. 对象:RTT中，所有的数据结构都称之为对象
2. 对象类型枚举：通过枚举来表示各种对象类型
3. 对象数据类型定义
   ```c
        struct rt_object
        {
            char name[RT_NAME_MAX];     /* 内核对象的名字 */
            rt_uint8_t type;            /* 内核对象的类型 */
            rt_uint8_t flag;            /* 内核对象的状态 */
            rt_list_t list;             /* 内核对象的列表节点 */
        };
        typedef struct rt_object *rt_object_t;
   ```
4. 容器：在RTT中，利用容器来管理系统当前的对象，数据类型是`struct rt_object_information`
5. 容器的定义:
   ```c
    rt_object_container[RT_Object_Info_Unknown] = { 
    {
        RT_Object_Class_Thread,                                     // 定义对象类型            
        _OBJ_CONTAINER_LIST_INIT(RT_Object_Info_Thread),            // 初始化对象列表节点头，当有新对象加入时，则往该链表中加入即可
        sizeof(struct rt_thread)                                    // 对象大小
    },
    ...
   ```
6. 因为对象`RT_Object_Info_Thread`是RTT最基本的对象，因此在对象定义枚举中，只有该对象不需要添加宏定义，其他对象都需要添加宏定义

### 2. 容器接口
1. 获取指定类型的对象信息
   1. `struct rt_object_information *rt_object_get_information(enum rt_object_class_type type)`
   2. 输入对象类型的type，返回容器成员地址
2. 对象初始化
   1. `void rt_object_init(struct rt_object *object, enum rt_object_class_type type, const char *name)`
   2. 工作过程
      1. 从容器中获取对应的对象列表头指针
      2. 设置对象类型为静态
      3. 拷贝对象名字给对象
      4. 关闭中断，避免把对象插入到容器列表时被打断
      5. 把对象插到容器的列表中
      6. 开中断
   3. 在线程的初始化函数中，会先初始化对象

## 3. 空闲进程与延时阻塞
### 1. 空闲进程
1. 空闲线程:优先级最低的一个线程，当系统的其他线程都在阻塞等待的时候，系统就转去运行空闲线程，空闲线程进而转去进行一些系统内存清除工作
2. 空闲线程初始化：
   1. 初始化线程
   2. 将线程插入就绪列表尾部

### 2. 阻塞延时
1. 线程调用延时函数后，线程会被剥离CPU控制权，进而进入阻塞状态，需要等到延时结束且线程重新获取CPU控制权才可以继续运行
2. 线程延时函数: 
   1. 先获取系统当前的线程控制块
   2. 把要延时的节拍数设置到当前的线程控制块的成员`remaining_tick`中
   3. 启动系统调度

### 3.SysTick_Handler中断服务函数
1. 该函数类就是软件定时器中断服务函数，每倒计时一次，就会触发该服务函数，进而进行一些加减数操作
2. 通过函数`SysTick_Config`设置系统的节拍时间，并初始化系统SysTick,同时还配置SysTick的中断优先级(本质就是一个定时器)
3. 系统时基更新函数：
   1. 对时基计数器进行统计处理
   2. 遍历就绪列表所有的线程控制块，对其remaining_tick进行减一处理，待remaining_tick减为0时说明延时结束



