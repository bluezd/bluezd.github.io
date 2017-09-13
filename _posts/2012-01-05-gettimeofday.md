---
id: 204
title: 系统调用 gettimeofday 分析
date: 2012-01-05T21:37:59+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=204
permalink: /archives/204
views:
  - "10794"
dsq_thread_id:
  - "6101101435"
categories:
  - kernel
  - timer
tags:
  - 2.6.32
  - kernel
  - timer
---
**背景:**
  
因为需要所以最近总会写一些 timer 相关的程序，既然是和 timer 相关的那少不了调用 gettimeofday system call。这个函数很有名，在某个版本的内核中创建多个线程 invoke gettimeofday，此时 kernel timer 会 backwards。带着各种疑问我决定阅读 timer 相关内核代码。个人认为 timer 部分不算太好理解，涉及到东西很多(中断，进程调度，CFS, 还有那可恶的 SMP 等等等等)，毕竟它是系统的心跳，虽然以前接触点 timer 相关的东西但是我也不太确定能把这部分代码理解透彻。总之 do my best !很喜欢一句歌词：出发了不要问那路在哪 &#8230;&#8230;

gettimeofday 就是获取 wall time。

用户态调用 gettimefoday，int 0X80 —> 软中断 -> IDT 表中断描述符 -> 段选择子 -> GDT 表段描述符 &#8230; 执行中断处理程序 -> 最终执行 sys_gettimeofday.(中间还有什么特权级切换啊，等等不再赘述)

sys_gettimeofday 是内核的入口函数可是我费了很大力气才找到，大概是这个样子：

<pre class="brush: cpp; title: ; notranslate" title="">SYSCALL_DEFINE2(gettimeofday, struct timeval __user *, tv,
                struct timezone __user *, tz)
{
        // tv 和 tz 都是来自用户空间，tv 指向的结构保存获得的当前秒数和微妙数，tz 通常那为 NULL 
        if (likely(tv != NULL)) {
                struct timeval ktv;
                do_gettimeofday(&ktv);
                if (copy_to_user(tv, &ktv, sizeof(ktv)))
                        return -EFAULT;
        }
        if (unlikely(tz != NULL)) {
                if (copy_to_user(tz, &sys_tz, sizeof(sys_tz)))
                        return -EFAULT;
        }
        return 0;
}
</pre>

为什么非得弄个宏 SYSCALL\_DEFINE2，直接 sys\_gettimeofday 不行吗？ 搞得人家很不爽！
  


&nbsp;


  
核心就是下面的函数：

<pre class="brush: cpp; title: ; notranslate" title="">void do_gettimeofday(struct timeval *tv)
{
        struct timespec now;

        getnstimeofday(&now);
        // now 中返回当前时间 秒数以及纳秒数

        tv-&gt;tv_sec = now.tv_sec;
        // 秒数
        tv-&gt;tv_usec = now.tv_nsec/1000;
        // now.tv_nsec 不足一秒的纳秒数 转换为微秒数.
}
</pre>

&nbsp;

> CONFIG\_GENERIC\_TIME=y
  
> // rhel6 中被定义，本文基于 2.6.32 以上版本内核 

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">struct timeval {
        __kernel_time_t         tv_sec;         /* seconds */
        __kernel_suseconds_t    tv_usec;        /* microseconds */
};
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">void getnstimeofday(struct timespec *ts)
{
        unsigned long seq;
        s64 nsecs;

        WARN_ON(timekeeping_suspended);

        do {
                seq = read_seqbegin(&xtime_lock);
                // 加读锁，如果此时已经加上的写锁，则自旋在这里

                *ts = xtime;
                // 此时很有可能更新 xtime.(时钟中断处理程序)

                nsecs = timekeeping_get_ns();

                /* If arch requires, add in gettimeoffset() */
                nsecs += arch_gettimeoffset(); 
                // NULL function ?

        } while (read_seqretry(&xtime_lock, seq));
        // 顺序锁，判断是否重读. 因为此时可能发生时钟中断更新 xtime .这样读出来的值就不准确需要重读.

        timespec_add_ns(ts, nsecs);
        // add nsecs 到 struct timespec 结构.
}
</pre>

&nbsp;

> 这里有三种情况:
     
> 1 循环只执行一遍，准确获得当前时间
     
> 2 顺利加上读锁，但是与此同时时钟中断到来需要更新 xtime 加上写锁(有更高的优先级)，然后 read_seqretry 返回 1 表示时间已经不准却需要重新读取，再次循环读取。以此类推～
     
> 3 读锁阻塞，因为此时已经加上了写锁(更新 xtime),等到写锁释放后便可读取. 

&nbsp;


  
`</p>
<pre>
struct timespec {
        time_t  tv_sec;         /* seconds */
        long    tv_nsec;        /* nanoseconds */
};`</pre> 

&nbsp;

> struct timespec xtime \_\_attribute\_\_ ((aligned (16))); 

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">__cacheline_aligned_in_smp DEFINE_SEQLOCK(xtime_lock);

#define DEFINE_SEQLOCK(x) \
                seqlock_t x = __SEQLOCK_UNLOCKED(x)                                     

typedef struct {
        unsigned sequence;
        spinlock_t lock;
} seqlock_t;
//记录了写着进程访问临界资源的过程 初始化为 0 写时 +1,解除写锁时再 +1 ,该直为奇数时处于写&gt;锁定态
//读直接访问不需要加锁

#define __SEQLOCK_UNLOCKED(lockname) \
                 { 0, __SPIN_LOCK_UNLOCKED(lockname) }

# define __SPIN_LOCK_UNLOCKED(lockname) \
        (spinlock_t)    {       .raw_lock = __RAW_SPIN_LOCK_UNLOCKED,   \
                                SPIN_DEP_MAP_INIT(lockname) }
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static __always_inline unsigned read_seqbegin(const seqlock_t *sl)
{
        unsigned ret;

repeat:
        ret = sl-&gt;sequence;
        smp_rmb();
        if (unlikely(ret & 1)) {
                // 如果当前值为奇数，表示已经加上写锁则等待直到写锁释放 ......
                cpu_relax();
                goto repeat;
        }

        return ret;
}

static __always_inline int read_seqretry(const seqlock_t *sl, unsigned start)
{
        smp_rmb();

        return (sl-&gt;sequence != start);
        // unlikely -&gt; 如果此时读取的值与最开始读取的值不同则表示在这期间加了写锁更新了 xtime,返回 1
        // likely   -&gt; 如果相等返回 0
}
</pre>

&nbsp;


  
下个比较重要的函数:

<pre class="brush: cpp; title: ; notranslate" title="">static inline s64 timekeeping_get_ns(void)
{
        cycle_t cycle_now, cycle_delta;
        struct clocksource *clock;

        clock = timekeeper.clock;
        // 时钟源 PIT/HPET/TSC
        // timekeeper 是什么 ？ 在哪里初始化？先卖个关子，后面的文章会继续介绍。简单说下，timekeeper 是 gettimeofday 等等系统调用的接口，timekeeper 下面是 clocksource 

        cycle_now = clock-&gt;read(clock);
        // 读取时钟周期的当前计数值 假设为 TSC --&gt; read_tsc()

        /* calculate the delta since the last update_wall_time: */
        cycle_delta = (cycle_now - clock-&gt;cycle_last) & clock-&gt;mask;
        // clock-&gt;cycle_last 应该由时钟中断处理程序负责更新.

        /* return delta convert to nanoseconds using ntp adjusted mult. */
        return clocksource_cyc2ns(cycle_delta, timekeeper.mult,
                                  timekeeper.shift);
        //转换成纳秒
}
</pre>

timekeeper.mult timekeeper.shift 细节之后会介绍
  


&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static inline s64 clocksource_cyc2ns(cycle_t cycles, u32 mult, u32 shift)
{
                return ((u64) cycles * mult) &gt;&gt; shift;
}
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static __always_inline void timespec_add_ns(struct timespec *a, u64 ns)
{
                a-&gt;tv_sec += __iter_div_u64_rem(a-&gt;tv_nsec + ns, NSEC_PER_SEC, &ns);
                // 返回 秒数 一般情况下返回 0
                a-&gt;tv_nsec = ns;
                // 不足 1s 的纳秒数 (此值会一直叠加)
}
</pre>

&nbsp;


  
最后一个函数：

<pre class="brush: cpp; title: ; notranslate" title="">static __always_inline u32
__iter_div_u64_rem(u64 dividend, u32 divisor, u64 *remainder)
{
        // dividend = a-&gt;tv_nsec + ns
        u32 ret = 0;

        while (dividend &gt;= divisor) {
                // 如果 纳秒大于一秒 NSEC_PER_SEC 

                /* The following asm() prevents the compiler from
                   optimising this loop into a modulo operation.  */
                asm("" : "+rm"(dividend));

                dividend -= divisor;
                // 减去一秒

                ret++;
                // 秒数加1
        }

        *remainder = dividend;

        return ret;
}
</pre>

&nbsp;


  
**总结:**
  
不知不觉中，gettimeofday 已经介绍完了，但是这里面还有几个细节没有讲述，比如 时钟源，时钟源的注册 timekeeping 等等.后面的文章中会陆续介绍，说实话 gettimeofday 此函数实现不算很难，我觉得难理解的在后面，就当它是个开胃小菜吧！