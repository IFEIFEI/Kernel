# Optee 阅读问题记录

---
 ### 2018/03/16 15:23
> read_daif()函数未定义
> sn_thread_alloc_and_run 函数

- 初始化寄存器spsr的状态
- 加锁，分配procs数据内容
- 设置寄存器 pc spsr sp和优先级 p_prio
- 设置通用寄存器 X0=入口地址 X1=endpoint X29=4

> 说明：
> X1 表示通用寄存器1，pc,spsr,sp为对应的arm芯片的寄存器
> proc为内核设置的进程表