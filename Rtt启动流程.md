# 1.在stm32上启动

## 1.初始状态

使用rt-thread 5.02版本中bsp中stm32f407-rt-spark进行调试

该例程仿真启动是从内部flash启动，即在复位状态之后，从0x00000000映射到0x08000000处取MSP初始值，之后会自动获取下一个32位地址取出复位中断入口向量地址即0x08000004从而获取PC初始值，继而跳转执行复位中断服务程序。

打开map文件后可查到0x80000000处是中断向量表的初始地址：![image-20240614101156803](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614101156803.png)

MSP在__Vectors获取到 _initial_sp，并设置SP = _initial_sp  。_
在Map文件中查询 _initial_sp得知为其值为0x20000fd8，也就是栈顶指针;
![image-20240614140443908](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614140443908.png)

在0x08000004地址获取得到的PC是中断向量表的的Reset_Handler ：
![image-20240614102858198](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614102858198.png)

Reset_Handler的地址在map文件中查询得知为0x080003B5， 也就是进行debug后，执行startup_stm32f407xx.s文件的Reset_Handler函数并开始执行第一条语句，此时的寄存器状态是：

![image-20240614140724835](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614140724835.png)

SP设置为_initial_sp即0x20000fd8，PC设置为0x0800003B4(**map显示Reset_Handler地址是0x080003B5而PC设置为0x080003B4可能进行debug后还并未真正执行Reset_Handler，而进行函数调用的时候PC指针会自动+1，所以执行到Reset_Handler时候PC会被设置为0x080003B5**)

## 2.**Reset_Handler**

Reset_Handler函数内主要有SystemInit和__main进行执行；

![image-20240614104329827](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614104329827.png)

### 1.进入SystemInit函数：

FPU以及系统时钟初始化配置

### 2.进入_main():

在$Sub$$main函数打断点并全速执行，会发现在执行完SystemInit()函数后会执行$Sub$$main()函数
![image-20240614142931126](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614142931126.png)

先执行$Sub$$main函数中的 rt_hw_interrupt_disable()函数：

![image-20240614143031270](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614143031270.png)

在这个函数内，会关闭系统中断，使用 MRS 指令将 PRIMASK 寄存器的值保存到 r0 寄存器里，然后使用 “CPSID I” 指令关闭全局中断，最后使用 BX 指令返回。上图中可以看到在执行MRS语句后R0寄存器的值为0；关闭中断之后执行rtthread_startup()：
在这个rtthread_startup()函数中会进行一系列的初始化：
![image-20240614143142882](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614143142882.png)

在rt_hw_board_init函数内会使能I_CACHE，D_CACHE,Hal库，堆栈等初始化
执行rt_show_version函数会打印系统信息；

之后的函数分别进行初始化任务定时器，初始化任务调度器，初始化系统信号，在rt_application_init()创建主线程，之后创建定时器线程，和空闲线程。在开启调度器之前系统并未开始执行线程，各创建好的线程会处于就绪状态，会在开启rt_system_scheduler_start()即系统调度之后开始调度就绪状态的线程。
RTT在MDK中使用了扩展功能 `$Sub$$` 和 `$Super$$`，在rt_application_init函数中创建了一个main主线程;
![image-20240614143730736](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614143730736.png)

该主线程被创建之后即被开始调度(调度器未开始，就绪状态)，在主线程的入口函数中可以看到，如果使用的是ARM汇编器的话就会执行$Super$$main()。
执行rt_system_scheduler_start();在开启调度时，系统中处于就绪状态的线程有main_thread_entry，rt_thread_timer_entry，rt_thread_idle_entry。

在完成rtthread_startup()后会跳转到main()函数进行执行：
![image-20240614144042866](C:\Users\RTT\Desktop\学习资料\rtt启动流程学习记录\image\image-20240614144042866.png)


