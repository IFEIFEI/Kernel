# Optee 阅读问题记录

---
 ### 2018/03/16 15:23
> read_daif()函数未定义
> sn_thread_alloc_and_run 函数

1. 初始化寄存器spsr的状态
2. 加锁，分配procs数据内容
3. 设置寄存器 pc spsr sp和优先级 p_prio
4. 设置通用寄存器 X0=入口地址 X1=endpoint X29=4
5. 调用函数call_resume

> 加锁过程 lock_global
``` c
1. #1 in lock_global() 
2. #2 in cpu_spin_lock(&thread_global_lock)
            // thread_global_lock = SPINLOCK_UNLOCK = 0x0000
3. #2       __cpu_spin_lock(lock)
4. #2       spinlock_count_incr()
5. #3 in __cpu_spin_lock
``` 
> proc结构
```c
struct proc {
    struct pcb_regs regs;
    struct pcb_regs *uregs;
    uint64_t map;
    struct pgt_cache pgt_cache;
    uint32_t p_num;
    int p_endpoint;
    uint32_t time_res;
    uint32_t p_prio;
    uint32_t p_priv;
	uint32_t p_rts_flags;
	uint32_t p_misc_flags;
	uint32_t p_pending;
	struct message p_sendmsg;
	int p_sendto;	
	struct message p_recvmsg;	
	int p_getfrom;
	void* p_recvaddr;	
    struct run_info run_info;
    uint64_t k_stack;
	struct proc* p_caller_q;
	struct proc* p_q_link;
	struct list_head link;
} __aligned(16);

struct pcb_regs {
    uint64_t sp;
    uint64_t pc;
    uint64_t spsr;
    uint64_t x[31];
} __aligned(16);

struct run_info {
    uint64_t entry;
    uint64_t load_addr;
    struct mobj *mobj_code;                     // defined in file <mobj.h>
    struct mobj *mobj_stack;
    struct tee_mmu_info *mmu;                   // defined in file <mobj.h>
};
// short for definition struct pgt_cache
```
> 函数 int call_resume(struct pcb_regs *regs, uint32_t spsr)
```c
1. 
```

> 说明：  
> X1 表示通用寄存器1，pc,spsr,sp为对应的arm芯片的寄存器  
> proc为内核设置的进程表  
> #1 表示栈桢第一层  