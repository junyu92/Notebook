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

## 使用timer

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
