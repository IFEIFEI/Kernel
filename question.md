# Optee 阅读问题记录

---
 ### 2018/03/16 15:23
> read_daif()函数未定义
> sn_thread_alloc_and_run 函数

1. 初始化寄存器spsr的状态
2. 加锁，分配procs数据内容
3. 设置寄存器 pc spsr sp和优先级 p_prio
    1). pc设置为函数sn_thread_std_smc_entry的入口
    2). 
4. 设置通用寄存器 X0=入口地址 X1=endpoint X29=0
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



---
### 2018/04/08
> sn_tee_ta_exec函数
> parameter : *ta_addr 程序运行的地址
> parameter : pn       给程序分配的proc槽号
> return : res         见分析

1. 调用 res = sn_ta_load(ta_addr, proc).
    proc = &procs[pn]
2. enqueue(proc)

> 函数 int sn_ta_load(struct shdr *signed_ta, struct proc *proc)
> parameter : signed_ta     程序运行的地址
> parameter : proc          分配给这个进程的proc的槽的引用

1. 调用res = sn_load_elf(proc,signed_ta)
2. 调用sn_tee_mmu_set_ctx(proc)
3. 设置uregs
    1). spsr = read_daif() & (SPSR_64_DAIF_MASK << SPSR_64_DAIF_SHIFT)
    2). usr_stack = mmu->regions[0].va) + mobj_stack->size    // proc里面信息
    3). x29 = 0
    4). sp = usr_stack
    5). pc = entry
4. 调用vfp_enable()
5. return TEE_SUCCESS

> 函数sn_load_elf(struct proc *proc, struct shdr *shdr)
> parameter : proc      proc槽号
> parameter : shdr      进程运行地址
> return    : TEE_Result    加载elf操作的结果

> 函数sn_tee_mmu_set_ctx(proc)
> parameter : proc      进程槽号
> return    : void
> remark    : 这部分主要是设置进程的页表（not focused now)

1. sn_core_mmu_create_user_map(proc)
2. core_mmu_set_user_map(&user_map)         // user_map 用proc-> map进行初始化

```c
/*
 * struct core_mmu_table_info - Properties for a translation table
 * @table:	Pointer to translation table
 * @va_base:	VA base address of the transaltion table
 * @level:	Translation table level
 * @shift:	The shift of each entry in the table
 * @num_entries: Number of entries in this table.
 */
struct core_mmu_table_info {
	void *table;
	vaddr_t va_base;
	unsigned level;
	unsigned shift;
	unsigned num_entries;
};
```
```c 
 #0 sn_core_mmu_create_user_map
 #1 sn_core_mmu_get_user_pddir
 #2 core_mmu_get_user_va_range
 #2 core_mmu_set_info_table
        完成对core_mmu_table_info信息的初始化
 #1 sn_core_mmu_populate_user_map
        ？？ struct pgt_cache 信息unfound
        
 #1 virt_to_phys
```
```c
reference data structure
struct pgt {
	void *tbl;
#if defined(CFG_PAGED_USER_TA)
	vaddr_t vabase;
	struct tee_ta_ctx *ctx;
	size_t num_used_entries;
#endif
#if defined(CFG_WITH_PAGER)
#if !defined(CFG_WITH_LPAE)
	struct pgt_parent *parent;
#endif
#endif
#ifdef CFG_SMALL_PAGE_USER_TA
	SLIST_ENTRY(pgt) link;
#endif
}; // used in sn_core_mmu_populate_user_map

/**
 * TA加载映像的时候用于检测验证的数据
 */
struct shdr {
	uint32_t magic;
	uint32_t img_type;
	uint32_t img_size;
	uint32_t algo;
	uint16_t hash_size;
	uint16_t sig_size;
};
```
---

### 2018/04/20
```c
内核数据对象：以数据结构为载体

//进程控制块
struct mproc
{
    unint32_t mp_num;       // 进程号
    int mp_endpoint;        // 进程 endpoint
    int mp_father;          // 父进程号
}
// op on mproc

//消息结构
struct message {
    int from;               // 发送者
    int to;                 // 接收者
    int type;               // 类型
    union {
        char msg[64];       // 消息体
        uint64_t ts;        // 时间戳
		int mp_pid;         // 发送的消息进程的pid
    } u;
};
```
```coq
Inductive mess_u :=
| MESSUUndef
| MESSU (msg: List Char) (ts: nat) (mp_pid: Z)
Inductive message :=
| MESSUndef
| MESS (from: Z) (to: Z) (type: Z)
```





