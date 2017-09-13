---
id: 259
title: 全局时钟事件设备的注册
date: 2012-03-13T23:09:39+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=259
permalink: /archives/259
views:
  - "6422"
categories:
  - kernel
  - timer
tags:
  - 2.6.32
  - kernel
  - timer
---
**<font size="2">全局时钟事件设备(Global Clock Event Device):HPET/PIT</font>**

* 主要负责提供周期时钟，更新 jiffies

* 全局时钟的角色由一个明选择的局部时钟承担，每个 cpu 都有 local apic,而 global clock 有一个特定的 cpu 承担，全局的时钟事件设备虽然附属于某一个特定的 CPU 上，但是完成的是系统相关的工作，例如完成系统的 tick 更新,说白了就是某个 cpu 一个人接俩活～

* 结构 struct clock\_event\_device

**<font size="2">局部时钟事件设备(Local Clock Event Device):lapic</font>**
     
* 每个 CPU 都有一个局部时钟，用作进程统计，最主要的实现了高分辨率定时器(只能工作在提供了lapic 的系统上)
     
* 主要完成统计运行在当前 CPU 上的进程的统计，以及设置 Local CPU 下一次中断
     
* 结构 struct clock\_event\_device

**<font size="2">时钟设备(tick device)</font>**
  
`</p>
<pre>struct tick_device {
        struct clock_event_device *evtdev;
        enum tick_device_mode mode;
};    

enum tick_device_mode {
        TICKDEV_MODE_PERIODIC,
        TICKDEV_MODE_ONESHOT,
};`</pre> 

tick device 只是 clock\_event\_device 的一包装器，增加了而外的字段用于指定设备的运行模式(周期或者单触发)。

全局 tick device
  
`</p>
<pre>static struct tick_device tick_broadcast_device;`</pre> 

tick\_broadcast\_device 很重要，后面会说到～
  


&nbsp;


  
查看当前系统的 Global Clock Event Device 以及 Local Clock Event Device
  
`</p>
<pre>cat /proc/timer_list`</pre> 

查看 tick_device
  
`</p>
<pre>cat /proc/timer_list | grep "Clock Event Device"
Clock Event Device: hpet
Clock Event Device: lapic
Clock Event Device: lapic
Clock Event Device: lapic
Clock Event Device: lapic`</pre> 

查看 event_handler
  
`</p>
<pre>cat /proc/timer_list | grep "event_handler"
 event_handler:  tick_handle_oneshot_broadcast
 event_handler:  hrtimer_interrupt
 event_handler:  hrtimer_interrupt
 event_handler:  hrtimer_interrupt
 event_handler:  hrtimer_interrupt`</pre> 

从以上信息可以得出 Global Clock Event Device 是 hpet,event\_hanler 为 tick\_handle\_oneshot\_broadcast,Local Clock Event Device 是 lapic,event\_hanler 为 hrtimer\_interrupt,当前系统使用的是高分辨率的 Timer.

event_handler 是什么呢就是中断到来所要执行的函数。(在中断处理程序中被调用)
  
_大概是这个这样的_
  
_Global_
  
`</p>
<pre>static irqreturn_t timer_interrupt(int irq, void *dev_id)
{
         ......

         global_clock_event->event_handler(global_clock_event);
         // tick_handle_oneshot_broadcast

         ......
}`</pre> 

_Local_
  
`</p>
<pre>
void __irq_entry smp_apic_timer_interrupt(struct pt_regs *regs)
{
         ......

         local_apic_timer_interrupt();

         ......
}

static void local_apic_timer_interrupt(void)
{
        int cpu = smp_processor_id();
        struct clock_event_device *evt = &per_cpu(lapic_events, cpu);

        ......

        evt->event_handler(evt);
        // hrtimer_interrupt
}`</pre> 

那 GLobal 与 Local 的 event_handler 是经过怎样的过程而最终得到的呢，下面就以此主线来分析～有关一些基本的概念大家可以参考 _《Professional Linux Kernel Architecture》Chapter 15_ 

<font size="1">我觉得要彻底弄懂一个东西就得看源代码，不看是怎么实现的就是看再多书也没用，这是亲身感受～ ok,Let us Go.</font> 

&nbsp;

<font size="3">全局时钟事件设备的注册</font>
  
`</p>
<pre>
start_kernel()
  -> if (late_time_init)
           late_time_init()
           x86_late_time_init()
             -> hpet_time_init()
                  -> hpet_enable()
                       -> hpet_legacy_clockevent_register()`</pre> 

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static void hpet_legacy_clockevent_register(void)
{
        /* Start HPET legacy interrupts */
        hpet_enable_legacy_int();

        hpet_clockevent.mult = div_sc((unsigned long) FSEC_PER_NSEC,
                                      hpet_period, hpet_clockevent.shift);
        /* Calculate the min / max delta */
        hpet_clockevent.max_delta_ns = clockevent_delta2ns(0x7FFFFFFF,
                                                           &hpet_clockevent);
        /* 5 usec minimum reprogramming delta. */
        hpet_clockevent.min_delta_ns = 5000;

        /*
         * Start hpet with the boot cpu mask and make it
         * global after the IO_APIC has been initialized.
         */
        hpet_clockevent.cpumask = cpumask_of(smp_processor_id());
        // 当前 cpu 肯定是 cpu0
        clockevents_register_device(&hpet_clockevent);
        // 开始注册
        global_clock_event = &hpet_clockevent;
        // Global clock event
        printk(KERN_DEBUG "hpet clockevent registered\n");
}
</pre>

`</p>
<pre>
static struct clock_event_device hpet_clockevent = {
        .name           = "hpet",
        .features       = CLOCK_EVT_FEAT_PERIODIC | CLOCK_EVT_FEAT_ONESHOT,
        .set_mode       = hpet_legacy_set_mode,
        .set_next_event = hpet_legacy_next_event,
        .shift          = 32,
        .irq            = 0,
        .rating         = 50,
};`</pre> 

&nbsp;

_故事从这就开始了 ......_

<pre class="brush: cpp; title: ; notranslate" title="">void clockevents_register_device(struct clock_event_device *dev)
{
        // dev=&hpet_clockevent

        unsigned long flags;

        BUG_ON(dev-&gt;mode != CLOCK_EVT_MODE_UNUSED);
        BUG_ON(!dev-&gt;cpumask);
        // 此时设备必须绑定到某个 CPU 上，此时是 cpu0

        spin_lock_irqsave(&clockevents_lock, flags);

        list_add(&dev-&gt;list, &clockevent_devices);
        // 加入到 clockevent_devices 链表中,跟时钟源的注册很像，只要注意以下插入链表的方式就好

        clockevents_do_notify(CLOCK_EVT_NOTIFY_ADD, dev);

        clockevents_notify_released();
        // 当 Global Clock Event Device 注册完 hpet 和 cpu0 注册 lapic 之后会被调用.

        spin_unlock_irqrestore(&clockevents_lock, flags);
}

static void clockevents_do_notify(unsigned long reason, void *dev)
{
        // dev 为时钟事件设备 (hpet) 
        raw_notifier_call_chain(&clockevents_chain, reason, dev);
}

// clockevents_chain 已经在 tick_init 函数中初始化
// clockevents_chain-&gt;head = &tick_notifier

struct raw_notifier_head {
                struct notifier_block *head;
};

int raw_notifier_call_chain(struct raw_notifier_head *nh,
                                unsigned long val, void *v)
{
                return __raw_notifier_call_chain(nh, val, v, -1, NULL);
}

int __raw_notifier_call_chain(struct raw_notifier_head *nh,
                              unsigned long val, void *v,
                              int nr_to_call, int *nr_calls)
{
        return notifier_call_chain(&nh-&gt;head, val, v, nr_to_call, nr_calls);
}

static int __kprobes notifier_call_chain(struct notifier_block **nl,
                                        unsigned long val, void *v,
                                        int nr_to_call, int *nr_calls)
{
        int ret = NOTIFY_DONE;
        struct notifier_block *nb, *next_nb;

        nb = rcu_dereference(*nl);
        // nb 指向 notifier_block

        while (nb && nr_to_call) {
                // nr_to_call = -1

                next_nb = rcu_dereference(nb-&gt;next);

#ifdef CONFIG_DEBUG_NOTIFIERS
                if (unlikely(!func_ptr_is_kernel_text(nb-&gt;notifier_call))) {
                        WARN(1, "Invalid notifier called!");
                        nb = next_nb;
                        continue;
                }
#endif
                ret = nb-&gt;notifier_call(nb, val, v);
                // 调用 tick_notify 函数 val = CLOCK_EVT_NOTIFY_ADD 
                // 返回 ret = NOTIFY_STOP

                if (nr_calls)
                        (*nr_calls)++;
                // nr_calls = NULL;

                if ((ret & NOTIFY_STOP_MASK) == NOTIFY_STOP_MASK)
                        break;
                nb = next_nb;
                nr_to_call--;
        }
        return ret;
}

#define NOTIFY_STOP_MASK        0x8000          /* Don't call further */
#define NOTIFY_DONE             0x0000          /* Don't care */
#define NOTIFY_OK               0x0001          /* Suits me */
#define NOTIFY_STOP             (NOTIFY_OK|NOTIFY_STOP_MASK) 

static int tick_notify(struct notifier_block *nb, unsigned long reason,
                               void *dev)
{
        switch (reason) {

        case CLOCK_EVT_NOTIFY_ADD:
                return tick_check_new_device(dev);
                // 这里

        case CLOCK_EVT_NOTIFY_BROADCAST_ON:
        case CLOCK_EVT_NOTIFY_BROADCAST_OFF:
        case CLOCK_EVT_NOTIFY_BROADCAST_FORCE:
                tick_broadcast_on_off(reason, dev);
                break;

        case CLOCK_EVT_NOTIFY_BROADCAST_ENTER:
        case CLOCK_EVT_NOTIFY_BROADCAST_EXIT:
                tick_broadcast_oneshot_control(reason);
                break;

        case CLOCK_EVT_NOTIFY_CPU_DYING:
                tick_handover_do_timer(dev);
                break;

        case CLOCK_EVT_NOTIFY_CPU_DEAD:
                tick_shutdown_broadcast_oneshot(dev);
                tick_shutdown_broadcast(dev);
                tick_shutdown(dev);
                break;

        case CLOCK_EVT_NOTIFY_SUSPEND:
                tick_suspend();
                tick_suspend_broadcast();
                break;

        case CLOCK_EVT_NOTIFY_RESUME:
                tick_resume();
                break;

        default:
                break;
        }

        return NOTIFY_OK;
}
</pre>

&nbsp;


  
接下来的两个函数很重要，当时看的时候丈二和尚摸不着头脑，但是经过各种 debug kernel,各种 brew,终于看出了点门道～

<pre class="brush: cpp; title: ; notranslate" title="">static int tick_check_new_device(struct clock_event_device *newdev)
{
        // newdev = &hpet_clockevent;

        struct clock_event_device *curdev;
        struct tick_device *td;
        int cpu, ret = NOTIFY_OK;
        unsigned long flags;

        spin_lock_irqsave(&tick_device_lock, flags);

        cpu = smp_processor_id();
        if (!cpumask_test_cpu(cpu, newdev-&gt;cpumask))
                goto out_bc;
        // cpumask:cpumask to indicate for which CPUs this device works

        td = &per_cpu(tick_cpu_device, cpu);
        // td 时钟设备，tick_cpu_device 是一个每CPU链表，包含了系统中每个CPU对应的struct tick_device 实例.关于每 CPU 变量这里就不说了，因为也不是一两句话能说明白的。

        curdev = td-&gt;evtdev;
        // 此时由于刚注册时钟设备上没有时钟事件设备，所以 curdev 为 NULL,而之后发生的可完全不一样那时 curdev 不为空，到时候再说

        /* cpu local device ? */
        if (!cpumask_equal(newdev-&gt;cpumask, cpumask_of(cpu))) {

                /*
                 * If the cpu affinity of the device interrupt can not
                 * be set, ignore it.
                 */
                if (!irq_can_set_affinity(newdev-&gt;irq))
                        goto out_bc;

                /*
                 * If we have a cpu local device already, do not replace it
                 * by a non cpu local device
                 */
                if (curdev && cpumask_equal(curdev-&gt;cpumask, cpumask_of(cpu)))
                        goto out_bc;
        } 
        // 这几行代码没太看懂貌似根本就没有被执行？

        /*
         * If we have an active device, then check the rating and the oneshot
         * feature.
         */
        if (curdev) {
                // 2.curdev = hpet
                // 2.newdev = lapic

                // 3.curdev = lapic
                // 3.newdev = hpet

                /*
                 * Prefer one shot capable devices !
                 */
                if ((curdev-&gt;features & CLOCK_EVT_FEAT_ONESHOT) &&
                    !(newdev-&gt;features & CLOCK_EVT_FEAT_ONESHOT))
                        goto out_bc;
                /*
                 * Check the rating
                 */
                if (curdev-&gt;rating &gt;= newdev-&gt;rating)
                        goto out_bc;

                // lapic rating = 100,hpet rating = 50 
        }
        // curdev = NULL ,ignore this "if",这块是个关键点

        /*
         * Replace the eventually existing device by the new
         * device. If the current device is the broadcast device, do
         * not give it back to the clockevents layer !
         */
        if (tick_is_broadcast_device(curdev)) {
                clockevents_shutdown(curdev);
                curdev = NULL;
        }

        clockevents_exchange_device(curdev, newdev);

        tick_setup_device(td, newdev, cpu, cpumask_of(cpu));
        // 建立 clock event device

        if (newdev-&gt;features & CLOCK_EVT_FEAT_ONESHOT)
                tick_oneshot_notify();
                // here

        spin_unlock_irqrestore(&tick_device_lock, flags);
        return NOTIFY_STOP;

out_bc:
        /*
         * Can the new device be used as a broadcast device ?
         */
        if (tick_check_broadcast_device(newdev))
                ret = NOTIFY_STOP;

        spin_unlock_irqrestore(&tick_device_lock, flags);

        return ret;
}
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">int tick_is_broadcast_device(struct clock_event_device *dev)
{
        return (dev && tick_broadcast_device.evtdev == dev);
}

void clockevents_exchange_device(struct clock_event_device *old,
                                 struct clock_event_device *new)
{
        // &lt;condition 1&gt;
        //old = NULL
        //new = &hpet_clockevent 

        // &lt;condition 2&gt;
        // old = hpet 
        // new = lapic

        unsigned long flags;

        local_irq_save(flags);
        /*
         * Caller releases a clock event device. We queue it into the
         * released list and do a notify add later.
         */
        if (old) {
                clockevents_set_mode(old, CLOCK_EVT_MODE_UNUSED);
                list_del(&old-&gt;list);
                list_add(&old-&gt;list, &clockevents_released);
                // 加到 clockevents_released 链表中
        }

        if (new) {
                BUG_ON(new-&gt;mode != CLOCK_EVT_MODE_UNUSED);
                clockevents_shutdown(new);
        }
        local_irq_restore(flags);
        // 此时我们考虑 condition 1
}
</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static void tick_setup_device(struct tick_device *td, 
                              struct clock_event_device *newdev, int cpu, 
                              const struct cpumask *cpumask)
{
        ktime_t next_event;
        void (*handler)(struct clock_vent_device *) = NULL;

        /*   
         * First device setup ?
         */
        if (!td-&gt;evtdev) {
        // 此时钟设备没有相关的时钟事件设备
     
                /*   
                 * If no cpu took the do_timer update, assign it to
                 * this cpu:
                 */
                if (tick_do_timer_cpu == TICK_DO_TIMER_BOOT) {
                        // 如果没有选定时钟设备来承担全局时钟设备的角色，那么将选择当前设备来承担此职责

                        tick_do_timer_cpu = cpu; 
                        // 设置为当前设备所属处理器编号
                        tick_next_period = ktime_get();

                        tick_period = ktime_set(0, NSEC_PER_SEC / HZ); 
                        // 时钟周期，纳秒
                        //HZ = 1000 
                }    

                /*   
                 * Startup in periodic mode first.
                 */
                td-&gt;mode = TICKDEV_MODE_PERIODIC;
                // 设备运行模式 --&gt; 周期模式

        } else {
                // 关于这个 else ,我们某天会来到这里，现在 ignore it.
                handler = td-&gt;evtdev-&gt;event_handler;
                next_event = td-&gt;evtdev-&gt;next_event;
                td-&gt;evtdev-&gt;event_handler = clockevents_handle_noop;
        }    

        td-&gt;evtdev = newdev;
        //为时钟设备指定事件设备

        /*
         * When the device is not per cpu, pin the interrupt to the
         * current cpu:
         */
        if (!cpumask_equal(newdev-&gt;cpumask, cpumask))
                irq_set_affinity(newdev-&gt;irq, cpumask);

        /*
         * When global broadcasting is active, check if the current
         * device is registered as a placeholder for broadcast mode.
         * This allows us to handle this x86 misfeature in a generic
         * way.
         */
        // check whether enable the broadcast mode,如果系统处于省电模式，而局部时钟停止工作，则会使用广播机制
        if (tick_device_uses_broadcast(newdev, cpu))
                return;

        if (td-&gt;mode == TICKDEV_MODE_PERIODIC)
                tick_setup_periodic(newdev, 0);
                // 周期模式 invoke this ......
        else
                tick_setup_oneshot(newdev, handler, next_event);
                // 单触发模式
}

</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">void tick_setup_periodic(struct clock_event_device *dev, int broadcast)
{
        tick_set_periodic_handler(dev, broadcast);
        // broadcast = 0

        /* Broadcast setup ? */
        if (!tick_device_is_functional(dev))
                return;

        if ((dev-&gt;features & CLOCK_EVT_FEAT_PERIODIC) &&
            !tick_broadcast_oneshot_active()) {
                clockevents_set_mode(dev, CLOCK_EVT_MODE_PERIODIC);
                // here
                // 设置成周期模式
        } else {
                unsigned long seq;
                ktime_t next;

                do {
                        seq = read_seqbegin(&xtime_lock);
                        next = tick_next_period;
                } while (read_seqretry(&xtime_lock, seq));

                clockevents_set_mode(dev, CLOCK_EVT_MODE_ONESHOT);

                for (;;) {
                        if (!clockevents_program_event(dev, next, ktime_get()))
                                return;
                        next = ktime_add(next, tick_period);
                }
        }
}

void tick_set_periodic_handler(struct clock_event_device *dev, int broadcast)
{
        if (!broadcast)
                dev-&gt;event_handler = tick_handle_periodic;
                // here
        else
                dev-&gt;event_handler = tick_handle_periodic_broadcast;
}

static inline int tick_device_is_functional(struct clock_event_device *dev)
{
        return !(dev-&gt;features & CLOCK_EVT_FEAT_DUMMY);
}
</pre>

此时 Global event\_handler 的注册就接近尾声了，event\_handler = tick\_handle\_periodic,不对啊应该是 "tick\_handle\_oneshot_broadcast"啊，是啊，莫急,故事远没有结束这才是一个开始 ......