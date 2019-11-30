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
