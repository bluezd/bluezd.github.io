---
id: 336
title: Posix timers clock_gettime 分析
date: 2012-08-24T22:39:50+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=336
permalink: /archives/336
views:
  - "6950"
categories:
  - kernel
  - timer
tags:
  - 2.6.32
  - kernel
  - timer
---
    int clock_getres(clockid_t clk_id, struct timespec *res)

这个函数就是根据 clk_id 返回相应的 time:

    CLOCK_REALTIME     real_time clock 系统绝对时间
    CLOCK_MONOTONIC    单调时间

关于这个函数更详细的介绍请参考 man 手册

&nbsp;

首先假设在用户态 

<pre class="brush: cpp; title: ; notranslate" title="">struct timespec now;

clock_gettime(CLOCK_MONOTONIC, &now)

</pre>

&nbsp;

走起：

_首先要从初始化开始：_
  
_kernel/posix-timers.c_

<pre class="brush: cpp; title: ; notranslate" title="">__initcall(init_posix_timers);

#define __initcall(fn) device_initcall(fn)

#define device_initcall(fn)             __define_initcall("6",fn,6)
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static __init int init_posix_timers(void)
{      
        struct k_clock clock_realtime = {
                .clock_getres = hrtimer_get_res,
        };
        struct k_clock clock_monotonic = {
                .clock_getres = hrtimer_get_res,
                .clock_get = posix_ktime_get_ts,
                .clock_set = do_posix_clock_nosettime,
        };
        struct k_clock clock_monotonic_raw = {
                .clock_getres = hrtimer_get_res,
                .clock_get = posix_get_monotonic_raw,
                .clock_set = do_posix_clock_nosettime,
                .timer_create = no_timer_create,
                .nsleep = no_nsleep,
        };
        struct k_clock clock_realtime_coarse = {
                .clock_getres = posix_get_coarse_res,
                .clock_get = posix_get_realtime_coarse,
                .clock_set = do_posix_clock_nosettime,
                .timer_create = no_timer_create,
                .nsleep = no_nsleep,
        };
        struct k_clock clock_monotonic_coarse = {
                .clock_getres = posix_get_coarse_res,
                .clock_get = posix_get_monotonic_coarse,
                .clock_set = do_posix_clock_nosettime,
                .timer_create = no_timer_create,
                .nsleep = no_nsleep,
        };
       
        register_posix_clock(CLOCK_REALTIME, &clock_realtime);
        register_posix_clock(CLOCK_MONOTONIC, &clock_monotonic);
        // 注册
        register_posix_clock(CLOCK_MONOTONIC_RAW, &clock_monotonic_raw);
        register_posix_clock(CLOCK_REALTIME_COARSE, &clock_realtime_coarse);
        register_posix_clock(CLOCK_MONOTONIC_COARSE, &clock_monotonic_coarse);

        posix_timers_cache = kmem_cache_create("posix_timers_cache",
                                        sizeof (struct k_itimer), 0, SLAB_PANIC,
                                        NULL);
        idr_init(&posix_timers_id);
        return 0;
}
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">void register_posix_clock(const clockid_t clock_id, struct k_clock *new_clock)
{
        if ((unsigned) clock_id &gt;= MAX_CLOCKS) {
                printk("POSIX clock register failed for clock_id %d\n",
                       clock_id);
                return;
        }

        posix_clocks[clock_id] = *new_clock;
}

// 这段代码真的很简单,不需要过多的解释

static struct k_clock posix_clocks[MAX_CLOCKS];

#define CLOCK_REALTIME                  0
#define CLOCK_MONOTONIC                 1

#define CLOCK_SGI_CYCLE                 10
#define MAX_CLOCKS                      16
#define CLOCKS_MASK                     (CLOCK_REALTIME | CLOCK_MONOTONIC)
#define CLOCKS_MONO                     CLOCK_MONOTONIC
</pre>

&nbsp;

_来了_

<pre class="brush: cpp; title: ; notranslate" title="">SYSCALL_DEFINE2(clock_gettime, const clockid_t, which_clock,
                struct timespec __user *,tp)
{
        struct timespec kernel_tp;
        int error;

        if (invalid_clockid(which_clock))
                return -EINVAL;
        // 首先检查 clock_id 是否 valid,此时是 CLOCK_MONOTONIC

        error = CLOCK_DISPATCH(which_clock, clock_get,
                               (which_clock, &kernel_tp));
        // posix_ktime_get_ts(CLOCK_MONOTONIC, &kernel_tp);

        if (!error && copy_to_user(tp, &kernel_tp, sizeof (kernel_tp)))
                error = -EFAULT;

        return error;

}
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static inline int invalid_clockid(const clockid_t which_clock)
{
        if (which_clock &lt; 0)    /* CPU clock, posix_cpu_* will check it */
                return 0;
        if ((unsigned) which_clock &gt;= MAX_CLOCKS)
                return 1;
        if (posix_clocks[which_clock].clock_getres != NULL)
                return 0;
        if (posix_clocks[which_clock].res != 0)
                return 0;
        return 1;
}
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">#define CLOCK_DISPATCH(clock, call, arglist) \
        ((clock) &lt; 0 ? posix_cpu_##call arglist : \
         (posix_clocks[clock].call != NULL \
          ? (*posix_clocks[clock].call) arglist : common_##call arglist))

CLOCK_DISPATCH(CLOCK_MONOTONIC,clock_get,(CLOCK_MONOTONIC, &kernel_tp)) \
((clock) &lt;  0 ? posix_cpu_##call arglist : (posix_clocks[clock].clock_get != NULL ? (*posix_clocks[clock].call) arglist : common_##call arglist))

// 其实最终就是这个样子
posix_ktime_get_ts(CLOCK_MONOTONIC, &kernel_tp);

</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static int posix_ktime_get_ts(clockid_t which_clock, struct timespec *tp)
{
        ktime_get_ts(tp);
        return 0;
}


void ktime_get_ts(struct timespec *ts)
{
        struct timespec tomono; unsigned int seq; s64 nsecs;

        WARN_ON(timekeeping_suspended);

        do {
                seq = read_seqbegin(&xtime_lock);
                // 加读锁

                *ts = xtime;
                // 当前时间,内核时间(UTC 时间)

                tomono = wall_to_monotonic;
                nsecs = timekeeping_get_ns();
                // 得到距离上一次得到时间中间走过的时间(纳秒)

        } while (read_seqretry(&xtime_lock, seq));

        set_normalized_timespec(ts, ts-&gt;tv_sec + tomono.tv_sec,
                                ts-&gt;tv_nsee + tomono.tv_nsec + nsecs);
}
</pre>

这个函数和 getnstimeofday 很像，细节可以参考 [gettimeofday](http://www.bluezd.info/archives/gettimeofday) 

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">void set_normalized_timespec(struct timespec *ts, time_t sec, s64 nsec)
{      
        while (nsec &gt;= NSEC_PER_SEC) {
                /*
                 * The following asm() prevents the compiler from
                 * optimising this loop into a modulo operation. See
                 * also __iter_div_u64_rem() in include/linux/time.h
                 */
                asm("" : "+rm"(nsec));
                nsec -= NSEC_PER_SEC;
                ++sec;
        }
        while (nsec &lt; 0) {
                asm("" : "+rm"(nsec));
                nsec += NSEC_PER_SEC;
                --sec;
        }
        ts-&gt;tv_sec = sec;
        ts-&gt;tv_nsec = nsec;
}
</pre>

其实原理就是记录一下开始的时间在 timekeeping_init() 中，然后用当前时间减去开始的时间就是经过的时间也就是单调时间～

&nbsp;

但是 wall\_to\_monotonic 这个东东就不得不说一下了，他到底是神马玩意儿呢？ 关于它还得追溯到 timekeeping_init() 中

<pre class="brush: cpp; title: ; notranslate" title="">struct timespec wall_to_monotonic __attribute__ ((aligned (16)));

void __init timekeeping_init(void)
{      
        struct clocksource *clock;
        unsigned long flags;
        struct timespec now, boot;

        read_persistent_clock(&now);
        // 读取 RTC chip,get the UTC time
 
        read_boot_clock(&boot);

        ......
 
        xtime.tv_sec = now.tv_sec;
        xtime.tv_nsec = now.tv_nsec;
        raw_time.tv_sec = 0;
        raw_time.tv_nsec = 0;
        if (boot.tv_sec == 0 && boot.tv_nsec == 0) {
                boot.tv_sec = xtime.tv_sec;
                boot.tv_nsec = xtime.tv_nsec;
        }

        set_normalized_timespec(&wall_to_monotonic,
                                -boot.tv_sec, -boot.tv_nsec);

        ......
}
</pre>

首先读取 RTC 获得 UTC 时间，保存到 xtime 中，最后记录到 wall\_to\_monotonic,只不过是个负值。其实就是记录一下系统启动时候的 real time.

&nbsp;


  
clock_gettime() 函数 基本上也就说完了，这个函数的实现比较简单，但是仍然有几点需要说明:

> **1. 它和 gettimeofday() 的区别是什么 ？** 
> 
> 当 clock\_gettime() clock\_id 指定为 CLOCK_REALTIME 时，它与 gettimeofday 完全一样，只不过它返回的是纳秒，而 gettimeofday 返回的是微秒。
> 
> struct timespec ts;
  
> struct timeval tv;
> 
> clock\_gettime(CLOCK\_REALTIME,&ts);
  
> gettimeofday(&tv,NULL); 

<pre class="brush: cpp; title: ; notranslate" title="">SYSCALL_DEFINE2(clock_gettime, const clockid_t, which_clock,
                struct timespec __user *,tp)
{
        ......

        error = CLOCK_DISPATCH(which_clock, clock_get,
                               (which_clock, &kernel_tp));
        // 其实就是调用 common_clock_get(CLOCK_REALTIME, &kernel_tp);
        

        ......

}

static int common_clock_get(clockid_t which_clock, struct timespec *tp)
{      
        ktime_get_real_ts(tp);
        return 0;
}
        
#define ktime_get_real_ts(ts)   getnstimeofday(ts)

// 就是调用 getnstimeofday(),同理再 gettimeofday() 中也是调用 getnstimeofday() ,关于这个函数细节参考下:&lt;a href="http://www.bluezd.info/archives/gettimeofday"&gt;gettimeofday&lt;/a&gt;
</pre>

&nbsp;

> **2.CLOCK_MONOTONIC 单调时间，那究竟什么是单调时间呢？**
> 
> CLOCK_MONOTONIC 表示单调时间，此时间会一直增加，不会受系统时间改变的影响。其实就是表示系统启动了多长时间，可通过 uptime 查看.
> 
> 比如我们用 settimeofday 往回设置时间，假设 current 20:00:00 往回设置 10 s,这样当用 gettimeofday 获得当前时间时是获得的绝对的时间，为 19:50:00,也就是说 gettimeofday 获得的&#8221;值&#8221;会比没设置前小。但是当用 clock\_getime 并指定 clock\_id 为 CLOCK\_MONOTONIC 时，在时间设置前后其“值”都是递增的。因为在 do\_settimeofday 中 wall\_to\_monotonic 也会被设置，会被往回设置 10 s.
> 
> 举个例子：
  
> 比如系统启动时的 UTC 时间是 20:00:00 ,此时 wall\_to\_monotonic 记录着这个时间，假设当前时间为 21:00:00,那么系统启动了 1h,通过 clock\_gettime(CLOCK\_MONOTONIC, &#8230;. ) 即可得到这个 1h.好现在往回设置时间，设置到 20:30:00,那么 相应 wall\_to\_monotonic 的值也会往回设置，为 19：30：00，xtime 回在这个基础上增加，这就是为什么当我们用 clock_gettime 获得单调时间时它的值一直增加了: 

下面的代码说明了这一点：

<pre class="brush: cpp; title: ; notranslate" title="">int do_settimeofday(struct timespec *tv)
{
        ......

        // tv 是要设置的时间
        ts_delta.tv_sec = tv-&gt;tv_sec - xtime.tv_sec;
        ts_delta.tv_nsec = tv-&gt;tv_nsec - xtime.tv_nsec;
        wall_to_monotonic = timespec_sub(wall_to_monotonic, ts_delta);

        ......
}

static inline struct timespec timespec_sub(struct timespec lhs,
                                                struct timespec rhs)
{
        struct timespec ts_delta;
        set_normalized_timespec(&ts_delta, lhs.tv_sec - rhs.tv_sec,
                                lhs.tv_nsec - rhs.tv_nsec);
        return ts_delta;
}

</pre>

&nbsp;


  
写个测试程序来验证这一点：

<pre class="brush: cpp; title: ; notranslate" title="">#include &lt;stdio.h&gt;
#include &lt;time.h&gt;
#include &lt;sys/time.h&gt;

int main(int argc, const char *argv[])
{      
        struct timeval tv1,tv2;
        struct timespec ts1,ts2;
        struct timeval temp;


        gettimeofday(&tv1,NULL);
        clock_gettime(CLOCK_MONOTONIC,&ts1);

        temp = tv1;
        temp.tv_sec -= 10;

        settimeofday(&temp,NULL);

        gettimeofday(&tv2,NULL);
        clock_gettime(CLOCK_MONOTONIC,&ts2);

        printf("gettimeofday start = %ld.%6ld,end = %ld.%6ld, \n\t =&gt; diff = %f \n",tv1.tv_sec, tv1.tv_usec, tv2.tv_sec, tv2.tv_usec, ((tv2.tv_sec * 1000000 + tv2.tv_usec) - (tv1.tv_sec * 1000000 + tv1.tv_usec))/1000000.0);

        printf("clock_gettime start = %ld.%9ld,end = %ld.%9ld, \n\t =&gt; diff = %f \n",ts1.tv_sec, ts1.tv_nsec, ts2.tv_sec, ts2.tv_nsec, ((ts2.tv_sec * 1000000000 + ts2.tv_nsec) - (ts1.tv_sec * 1000000000 + ts1.tv_nsec))/1000000000.0);

        return 0;
}
</pre>

&nbsp;

    # ./clock_gettime_gettimeofday
    gettimeofday start = 1345788337.265370,end = 1345788327.265383,  << end 值明显变小
             => diff = -9.999987
    clock_gettime start = 2595.332710380,end = 2595.332726631,      << end 值增大
             => diff = 0.000016

最后 clock\_gettime 就大致 BB 完了，还有几个与时间有关的 posix 函数(clock\_getres,clock\_settime),这几个函数的实现与 clock\_gettime 大致相同