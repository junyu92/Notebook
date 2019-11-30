# Time management

## Timer

Linux提供**software tiemr**使得kernel functions能够在将来被调用。

Linux提供两种类型的**timer**:

* dynamic timers

* interval timers

### 初始化

在``start_kernel``初始化时，内核会调用``init_timers``函数。

```c
void __init init_timers(void)
{
        init_timer_cpus();
        open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
}
```

我们看一下每个函数的实现。

```c
static void __init init_timer_cpus(void)
{
        int cpu;

        for_each_possible_cpu(cpu)
                init_timer_cpu(cpu);
}
```

它为每个possible_cpu调用``init_timer_cpu``函数。

```c
static void __init init_timer_cpu(int cpu)
{
        struct timer_base *base;
        int i;

        for (i = 0; i < NR_BASES; i++) {
                base = per_cpu_ptr(&timer_bases[i], cpu);
                base->cpu = cpu;
                raw_spin_lock_init(&base->lock);
                base->clk = jiffies;
        }
}
```

``init_timer_cpu``函数会初始化每个cpu的``timer_base``。``timer_base``的结构如下：

```c
struct timer_base {
        raw_spinlock_t          lock;
        struct timer_list       *running_timer;
        unsigned long           clk;
        unsigned long           next_expiry;
        unsigned int            cpu;
        bool                    is_idle;
        bool                    must_forward_clk;
        DECLARE_BITMAP(pending_map, WHEEL_SIZE);
        struct hlist_head       vectors[WHEEL_SIZE];
} ____cacheline_aligned;

static DEFINE_PER_CPU(struct timer_base, timer_bases[NR_BASES]);
```

* ``lock``很显然
* ``running_tiemr``表示运行在该cpu上的timer
* ``cpu``表示timer属于哪个cpu
* ``clk`` the clk fields represents the earliest expiration time (it will be used by the Linux kernel to find already expired timers)

``init_timers``最后一个调用的函数是``open_softirq``.

```c
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
        softirq_vec[nr].action = action;
}
```

它给中断向量号``TIMER_SOFTIRQ``注册了deferred interrupt handler函数``run_timer_softirq``.

它会在do_IRQ中被调用，这里不展开讲了。

```c
/* arch/x86/kernel/irq.c */
/*
 * do_IRQ handles all normal device IRQ's (the special
 * SMP cross-CPU interrupts have their own specific
 * handlers).
 */
__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
        struct pt_regs *old_regs = set_irq_regs(regs);
        struct irq_desc * desc;
        /* high bit used in ret_from_ code  */
        unsigned vector = ~regs->orig_ax;

        entering_irq();

        /* entering_irq() tells RCU that we're not quiescent.  Check it. */
        RCU_LOCKDEP_WARN(!rcu_is_watching(), "IRQ failed to wake up RCU");

        desc = __this_cpu_read(vector_irq[vector]);

        if (!handle_irq(desc, regs)) {
                ack_APIC_irq();

                if (desc != VECTOR_RETRIGGERED && desc != VECTOR_SHUTDOWN) {
                        pr_emerg_ratelimited("%s: %d.%d No irq handler for vector\n",
                                             __func__, smp_processor_id(),
                                             vector);
                } else {
                        __this_cpu_write(vector_irq[vector], VECTOR_UNUSED);
                }
        }

        exiting_irq();

        set_irq_regs(old_regs);
        return 1;
}
```

``run_timer_softirq``是一个中断处理程序的handler

```c
/*
 * This function runs timers and the timer-tq in bottom half context.
 */
static __latent_entropy void run_timer_softirq(struct softirq_action *h)
{
        struct timer_base *base = this_cpu_ptr(&timer_bases[BASE_STD]);

        __run_timers(base);
        if (IS_ENABLED(CONFIG_NO_HZ_COMMON))
                __run_timers(this_cpu_ptr(&timer_bases[BASE_DEF]));
}
```

我们看一下``__run_timers``的实现

```c
        if (!time_after_eq(jiffies, base->clk))
                return;
}
```

首先它判断``jiffies``是否>=时钟的``clk``域，如果不大于等于，说明没有时钟到期，直接返回。
如果已经到期，则会进入一个循环。

```c
while (time_after_eq(jiffies, base->clk)) {

        levels = collect_expired_timers(base, heads);
        base->clk++;

        while (levels--)
                expire_timers(base, heads + levels);
}
```

### 使用timer

首先我们要定义一个timer_list结构体，然后使用```init_timer```或``TIMER_INITIALIZER``初始化它。

使用
```c
void add_timer(struct timer_list * timer);
```
和
```c
int del_timer(struct timer_list * timer);
```
来启用或删除一个timer.

## clocksource framework

Linux支持多种时钟源(clock sources)，例如``drivers/clocksource``下以及架构相关的时钟源，每个时钟源都有自己的频率(frequency)。
clocksource framework的目标是

1. 提供API来选择最佳的时钟源(即频率最高的时钟源)，
2. 将时钟源提供的atomic counter转为人类可读的时钟源(e.g. nanosecond)

### 数据结构

时钟源的结构体如下

```c
struct clocksource {
        u64 (*read)(struct clocksource *cs);
        u64 mask;
        u32 mult;
        u32 shift;
        u64 max_idle_ns;
        u32 maxadj;
#ifdef CONFIG_ARCH_CLOCKSOURCE_DATA
        struct arch_clocksource_data archdata;
#endif
        u64 max_cycles;
        const char *name;
        struct list_head list;
        int rating;
        int (*enable)(struct clocksource *cs);
        void (*disable)(struct clocksource *cs);
        unsigned long flags;
        void (*suspend)(struct clocksource *cs);
        void (*resume)(struct clocksource *cs);
        void (*mark_unstable)(struct clocksource *cs);
        void (*tick_stable)(struct clocksource *cs);

        /* private: */
#ifdef CONFIG_CLOCKSOURCE_WATCHDOG
        /* Watchdog related data, used by the framework */
        struct list_head wd_list;
        u64 cs_last;
        u64 wd_last;
#endif
        struct module *owner;
};
```

* ``list``记录了所有的时钟源
* ``mult``和``shift``用来将atomic counter转换为纳秒，通过

```c
ns ~= (clocksource * mult) >> shift
```

### API

注册时钟源使用下面的API

```c
static inline int clocksource_register_hz(struct clocksource *cs, u32 hz)
static inline int clocksource_register_khz(struct clocksource *cs, u32 khz)
```

注销时钟源使用

```c
int clocksource_unregister(struct clocksource *cs)
```

## clockevents framework

Main goal of the clockevents is to manage clock event devices or in other words - to manage devices that allow to register an event or in other words interrupt that is going to happen at a defined point of time in the future.

```c
struct timer_list {
        /*
         * All fields that change during normal runtime grouped to the
         * same cacheline
         */
        struct hlist_node       entry;
        unsigned long           expires;
        void                    (*function)(struct timer_list *);
        u32                     flags;

#ifdef CONFIG_LOCKDEP
        struct lockdep_map      lockdep_map;
#endif
};
```
