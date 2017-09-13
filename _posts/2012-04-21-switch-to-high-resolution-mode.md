---
id: 304
title: Switch to High Resolution Mode
date: 2012-04-21T21:13:25+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=304
permalink: /archives/304
views:
  - "5322"
categories:
  - kernel
  - timer
tags:
  - 2.6.32
  - kernel
  - timer
---
继续:
  
此时 global and local clock event device 已经注册完毕，系统现在就开始响应中断了，当内核从RTC 取出时间后，内核就应该自己更新时间了，所以中断处理程序的很重要的一部分工作就是计时。而在 timer_interrupt 函数中:

    global_clock_event->event_handler(global_clock_event);

实际上执行的就是 tick\_handle\_periodic（）

Note:
  
当系统启动时是默认工作在 low resolution mode 中，时钟源为 PIT,当系统中存在高精度的时钟源时，时钟源就应该切换到高精度的上(比如 tsc,hpet),在系统初始化末期 clocksource\_done\_booting() ,kernel 会选择 the best clocksource 然后完成切换操作，在大多数情况下都是从 PIT 切换到 HPET，details see  [时钟源注册](http://www.bluezd.info/archives/%E6%97%B6%E9%92%9F%E6%BA%90%E7%9A%84%E6%B3%A8%E5%86%8C) .此时高精度的时钟源已经存在,在正常情况下系统就应该将 clock event device 切换到高分辨率模式(在 boot parameter &#8220;highres=off&#8221; 没有设置的情况下).那么究竟在何时内核回检查并且将时钟事件设备切换到高分辨率模式呢？那就是在每次的时钟中断处理程序中。
  


&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">void tick_handle_periodic(struct clock_event_device *dev)
{     
        int cpu = smp_processor_id();
        ktime_t next;
     
        tick_periodic(cpu);

        if (dev-&gt;mode != CLOCK_EVT_MODE_ONESHOT)
                return;

        // 此时直接返回 时钟事件设备的 mode 应该为  CLOCK_EVT_MODE_UNUSED
        // 若为 one-shot 模式 ，则需要编程设置下一个时钟事件

        /*   
         * Setup the next period for devices, which do not have
         * periodic mode:
         */     
        next = ktime_add(dev-&gt;next_event, tick_period);
        for (;;) {
                if (!clockevents_program_event(dev, next, ktime_get()))
                        return;
                /*   
                 * Have to be careful here. If we're in oneshot mode,
                 * before we call tick_periodic() in a loop, we need
                 * to be sure we're using a real hardware clocksource.
                 * Otherwise we could get trapped in an infinite
                 * loop, as the tick_periodic() increments jiffies,
                 * when then will increment time, posibly causing
                 * the loop to trigger again and again.
                 */
                if (timekeeping_valid_for_hres())
                        tick_periodic(cpu);
                next = ktime_add(next, tick_period);
        }    
}
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static void tick_periodic(int cpu)
{
        if (tick_do_timer_cpu == cpu) {
                // 如果当前 cpu 是 在 tick_setup_device () 中设置的 cpu,一般情况下为 CPU0,值允许一个 CPU 负责更新时间.
                write_seqlock(&xtime_lock);

                /* Keep track of the next tick event */
                tick_next_period = ktime_add(tick_next_period, tick_period);

                do_timer(1);
                // 更新 wall time 和 jiffies

                write_sequnlock(&xtime_lock);
        }

        update_process_times(user_mode(get_irq_regs()));
        profile_tick(CPU_PROFILING);
}
</pre>

&nbsp;


  
此时我们关注 update\_process\_times()
  
update\_process\_times()

     -> run_local_timers()
         -> raise_softirq(TIMER_SOFTIRQ) // 激活软中断下半部

软中断下半部处理函数在 init_timers () 中注册

    run_timer_softirq()                                              
     -> hrtimer_run_pending()                                                                  
          -> hrtimer_switch_to_hres()                                                           
               -> tick_init_highres()                                                          
                    -> tick_switch_to_oneshot()                                               
                         -> tick_broadcast_switch_to_oneshot()                               
                             -> tick_broadcast_setup_oneshot()

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">void hrtimer_run_pending(void)
{
        if (hrtimer_hres_active())
                return;
        // 如果此时已经是高精度模式则无需转换
        
        /*
         * This _is_ ugly: We have to check in the softirq context,
         * whether we can switch to highres and / or nohz mode. The
         * clocksource switch happens in the timer interrupt with
         * xtime_lock held. Notification from there only sets the
         * check bit in the tick_oneshot code, otherwise we might
         * deadlock vs. xtime_lock.
         */
        if (tick_check_oneshot_change(!hrtimer_is_hres_enabled()))
                hrtimer_switch_to_hres();
        // 检查系统中是否存在适用于高分辨率定时器的时钟事件设备
}

static inline int hrtimer_is_hres_enabled(void)
{               
        return hrtimer_hres_enabled;
        // 一般情况下 为 1
}       
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">int tick_check_oneshot_change(int allow_nohz)
{
        // allow_nohz=0

        struct tick_sched *ts = &__get_cpu_var(tick_cpu_sched);

        if (!test_and_clear_bit(0, &ts-&gt;check_clocks))
                return 0;
        
        if (ts-&gt;nohz_mode != NOHZ_MODE_INACTIVE)
                return 0;
        
        if (!timekeeping_valid_for_hres() || !tick_is_oneshot_available())
                return 0;
                
        if (!allow_nohz)
                return 1;
        // 一般情况下执行到此，返回 1
                
        tick_nohz_switch_to_nohz();
        return 0;
}                       
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">/**
 * timekeeping_valid_for_hres - Check if timekeeping is suitable for hres
 */
int timekeeping_valid_for_hres(void)
{
        unsigned long seq;
        int ret;
        
        do {
                seq = read_seqbegin(&xtime_lock);
                
                ret = timekeeper.clock-&gt;flags & CLOCK_SOURCE_VALID_FOR_HRES;
                
                // 当时钟源为高精度已经由 pit 切换到 hpet,此时 timekeeper 中使用的 clock source 是 hpet,而 clocksource_hpet 中的 flag 已经在注册时钟源时在 clocksource_enqueue_watchdog 被设置，所以此时 ret = 1,表示可以切换
        
        } while (read_seqretry(&xtime_lock, seq));
        
        return ret;
}               
</pre>

**<font size="2"><br /> Note:<br /> 在系统没有切换到 high resolution clock source 时，在每次时钟中断处理程序中也都会检查时钟时间设备是否能够切换到 high resolution mode.但是每当执行到 tick_check_oneshot_change（）-> timekeeping_valid_for_hres() 时，由于 timekeeper 当前使用的是 clocksource_jiffies,PIT 并没有 CLOCK_SOURCE_VALID_FOR_HRES flag 所以每次 tick_check_oneshot_change() 函数都会返回 0,所以不会执行切换操作。但是一旦系统切换时钟源到某个高分辨率时钟源(tsc,hpet)上,就会执行真正切换操作！！<br /> </font>**

好继续 hrtimer\_switch\_to_hres() 执行时钟事件设备的注册。

<pre class="brush: cpp; title: ; notranslate" title="">hrtimer_switch_to_hres（）
  -&gt; tick_init_highres ()

int tick_init_highres(void)
{       
        return tick_switch_to_oneshot(hrtimer_interrupt);
}
</pre>

&nbsp;


  
Note:
  
此函数应该被执行的次数就是 CPU 的个数

<pre class="brush: cpp; title: ; notranslate" title="">int tick_switch_to_oneshot(void (*handler)(struct clock_event_device *))
{      
        struct tick_device *td = &__get_cpu_var(tick_cpu_device);
        // 还是获得那个 tick_device

        struct clock_event_device *dev = td-&gt;evtdev;
        // tick device 上的时钟时间设备 此时假设是 CVPU0,那 cpu0 的时钟事件设备此时就是 Lapic

        if (!dev || !(dev-&gt;features & CLOCK_EVT_FEAT_ONESHOT) ||
                    !tick_device_is_functional(dev)) {
               
                printk(KERN_INFO "Clockevents: "
                       "could not switch to one-shot mode:");
                if (!dev) {
                        printk(" no tick device\n");
                } else {
                        if (!tick_device_is_functional(dev))
                                printk(" %s is not functional.\n", dev-&gt;name);
                        else   
                                printk(" %s does not support one-shot mode.\n",
                                       dev-&gt;name);
                }
                return -EINVAL;
        }
       
        td-&gt;mode = TICKDEV_MODE_ONESHOT;
        dev-&gt;event_handler = handler;
        // 设置 event_handler 为 hrtimer_interrupt 就是这里啦

        clockevents_set_mode(dev, CLOCK_EVT_MODE_ONESHOT);
        // 设置为 oneshot 模式

        tick_broadcast_switch_to_oneshot();
        // Select oneshot operating mode for the broadcast device

        return 0;
}
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">void tick_broadcast_switch_to_oneshot(void)
{       
        struct clock_event_device *bc;
        unsigned long flags;
        
        spin_lock_irqsave(&tick_broadcast_lock, flags);
        
        tick_broadcast_device.mode = TICKDEV_MODE_ONESHOT;
        // 设置广播的时钟时间设备的模式
        // tick_broadcast_device 不陌生吧，在 tick_check_broadcast_device() 中被设置，此时就是 HPET

        bc = tick_broadcast_device.evtdev;
        // bc 一般情况下为 &hpet_clockevent

        if (bc)
                tick_broadcast_setup_oneshot(bc);
        // 设置广播设备

        spin_unlock_irqrestore(&tick_broadcast_lock, flags);
}

</pre>

&nbsp;


  
Note:
  
此函数只会被执行一次

<pre class="brush: cpp; title: ; notranslate" title="">void tick_broadcast_setup_oneshot(struct clock_event_device *bc)
{
        /* Set it up only once ! */
        if (bc-&gt;event_handler != tick_handle_oneshot_broadcast) {
                // 此时 bc-&gt;event_handler 应该为 clockevents_handle_noop

                int was_periodic = bc-&gt;mode == CLOCK_EVT_MODE_PERIODIC;
                int cpu = smp_processor_id();
        
                bc-&gt;event_handler = tick_handle_oneshot_broadcast;
                // 设置 hpet_clockevent 的 event_handler 为 tick_handle_oneshot_broadcast

                clockevents_set_mode(bc, CLOCK_EVT_MODE_ONESHOT);

                /* Take the do_timer update */
                tick_do_timer_cpu = cpu;
                // 设置负责 do_timer 的 CPU.

                /*
                 * We must be careful here. There might be other CPUs
                 * waiting for periodic broadcast. We need to set the
                 * oneshot_mask bits for those and program the
                 * broadcast device to fire.
                 */
                cpumask_copy(to_cpumask(tmpmask), tick_get_broadcast_mask());
                cpumask_clear_cpu(cpu, to_cpumask(tmpmask));
                cpumask_or(tick_get_broadcast_oneshot_mask(),
                           tick_get_broadcast_oneshot_mask(),
                           to_cpumask(tmpmask));

                if (was_periodic && !cpumask_empty(to_cpumask(tmpmask))) {
                        tick_broadcast_init_next_event(to_cpumask(tmpmask),
                                                       tick_next_period);
                        tick_broadcast_set_event(tick_next_period, 1);
                } else
                        bc-&gt;next_event.tv64 = KTIME_MAX;
        }
}
</pre>

&nbsp;


  
好的,总结一下:
  
在完成从低分辨率到高分辨率时钟事件设备的切换过程中，hrtimer\_switch\_to\_hres()被调用的次数正好为 CPU 的个数，将每个CPU 的 local apic clock event device 的 event\_handler 设置为 hrtimer\_interrupt;而在 tick\_broadcast\_setup\_oneshot() 函数中将 global clock event device 的 event\_handler 设置为tick\_handle\_oneshot\_broadcast，tick\_broadcast\_setup\_oneshot() 只会被某个 CPU 执行，不一定是 CPU0,但是只会执行一次，至此完成切换操作。之后在每个 CPU 相应时钟中断时还会去检查是否需要切换到高分辨率模式(连写这块代码的人都说这块有点 ugly)，但是在 hrtimer\_run_pending（）中会首先检查如果已经切换到高分辨率则直接返回。

`</p>
<pre>cat /proc/timer_list | grep "event_handler"
 event_handler:  tick_handle_oneshot_broadcast
 event_handler:  hrtimer_interrupt
 event_handler:  hrtimer_interrupt
 event_handler:  hrtimer_interrupt
 event_handler:  hrtimer_interrupt`</pre> 

&nbsp;


  
说明一下，不同的 boot parameter 相应的 event_handler 也会不一样，比如：
  
`</p>
<pre>boot parameter "highres=off"
   event_handler:  tick_handle_oneshot_broadcast
   event_handler:  tick_nohz_handler
   event_handler:  tick_nohz_handler
   event_handler:  tick_nohz_handler
   event_handler:  tick_nohz_handler

boot parameter "nohz=off highres=off"
   (1)
   event_handler:  clockevents_handle_noop
   event_handler:  tick_handle_periodic
   event_handler:  tick_handle_periodic
   event_handler:  tick_handle_periodic
   event_handler:  tick_handle_periodic
   (2)
   event_handler:  tick_handle_periodic_broadcast
   event_handler:  tick_handle_periodic
   event_handler:  tick_handle_periodic
   event_handler:  tick_handle_periodic
   event_handler:  tick_handle_periodic`</pre>