---
id: 284
title: 本地时钟事件设备的注册
date: 2012-03-18T20:02:54+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=284
permalink: /archives/284
views:
  - "4311"
categories:
  - kernel
  - timer
tags:
  - 2.6.32
  - kernel
  - timer
---
**<font size="2">话说上一回合已经完成了Global事件设备(hpet)的注册,现在来看下Local时钟事件的注册，就以 CPU0 为例.</font>**

在 kernel 初始化的末期，内核线程 1 中(还没有 up 其他 cpu),在 log 中出现了类似“CPU0: Intel(R) Xeon(TM) CPU 3.20GHz stepping 04”，发生了这样一个故事&#8230;&#8230;

<font size="3">局部时钟事件设备的注册</font>
  
`</p>
<pre>kernel_init()
  -> smp_prepare_cpus(setup_max_cpus)
     native_smp_prepare_cpus(64)
       -> x86_init.timers.setup_percpu_clockev()
          setup_boot_APIC_clock()
            -> setup_APIC_timer()
                 -> clockevents_register_device(levt)`</pre> 

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">static void __cpuinit setup_APIC_timer(void)
{
        struct clock_event_device *levt = &__get_cpu_var(lapic_events);

        if (cpu_has(&current_cpu_data, X86_FEATURE_ARAT)) {
                lapic_clockevent.features &= ~CLOCK_EVT_FEAT_C3STOP;
                /* Make LAPIC timer preferrable over percpu HPET */
                lapic_clockevent.rating = 150;
        }

        memcpy(levt, &lapic_clockevent, sizeof(*levt));
        levt-&gt;cpumask = cpumask_of(smp_processor_id());

        clockevents_register_device(levt);
        // 注册 CPU0 Local Clock Event Device
}
</pre>

&nbsp;


  
`</p>
<pre>
 +----------------------------------------------------------------------------------+
 |      /*                                                                          |
 |       * The local apic timer can be used for any function which is CPU local.    |
 |       */                                                                         |
 |      static struct clock_event_device lapic_clockevent = {                       |
 |              .name           = "lapic",                                          |
 |              .features       = CLOCK_EVT_FEAT_PERIODIC | CLOCK_EVT_FEAT_ONESHOT  |
 |                              | CLOCK_EVT_FEAT_C3STOP | CLOCK_EVT_FEAT_DUMMY,     |
 |              .shift          = 32,                                               |
 |              .set_mode       = lapic_timer_setup,                                |
 |              .set_next_event = lapic_next_event,                                 |
 |              .broadcast      = lapic_timer_broadcast,                            |
 |              .rating         = 100,                                              |
 |              .irq            = -1,                                               |
 |      };                                                                          |
 |                                                                                  |
 +----------------------------------------------------------------------------------+
` </pre> 

&nbsp;


  
_故事从这里就又开始了_

<pre class="brush: cpp; title: ; notranslate" title="">void clockevents_register_device(struct clock_event_device *dev)
{
        // dev=&lapic_clockevent
 
        unsigned long flags;
 
        BUG_ON(dev-&gt;mode != CLOCK_EVT_MODE_UNUSED);
        BUG_ON(!dev-&gt;cpumask);
        // 此时设备必须绑定到某个 CPU 上，此时还是 cpu0
 
        spin_lock_irqsave(&clockevents_lock, flags);
 
        list_add(&dev-&gt;list, &clockevent_devices);
        // 加入到 clockevent_devices 链表中,跟时钟源的注册很像，只要注意以下插入链表的方式就好
 
        clockevents_do_notify(CLOCK_EVT_NOTIFY_ADD, dev);
 
        clockevents_notify_released();
        // 当 Global Clock Event Device 注册完 hpet 和 cpu0 注册 lapic 之后会被调用.
 
        spin_unlock_irqrestore(&clockevents_lock, flags);
}

        ......

static int tick_check_new_device(struct clock_event_device *newdev)
{
        ......

        cpu = smp_processor_id();
        // 还是 CPU0
        if (!cpumask_test_cpu(cpu, newdev-&gt;cpumask))
                goto out_bc;
        // cpumask:            cpumask to indicate for which CPUs this device works

        td = &per_cpu(tick_cpu_device, cpu);
        // 还是 CPU0 的那个 struct tick_device 实例

        curdev = td-&gt;evtdev;
        // 注意此时 curdev 并不是 NULL 而是指向 hpet_clockevent

        ......

        if (curdev) {
                // curdev = &hpet_clockevent
                // newdev = &lapic_clockevent

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

                // 此处需要说明一下：一个 CPU 的一个 tick_device 只能对应一个 clock event device ,但是此时由于 CPU0 的 tick_device 上已经有了用作 global event device 的 hpet,所以此时就应该确定一下要选哪个根据 clcok event device 是否支持 one-shot mode 和 rating 值,此时顺序执行。
        }

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
        // 设置 lapic_clockevent 的 event_handler 为 tick_handle_periodic

        ......        

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

<pre class="brush: cpp; title: ; notranslate" title="">static void tick_setup_device(struct tick_device *td,
                              struct clock_event_device *newdev, int cpu,
                              const struct cpumask *cpumask)
{
        ......
        
        // 回到前文留下伏笔的 else
        else {
                handler = td-&gt;evtdev-&gt;event_handler;
                // handler 应该为 tick_handle_periodic

                next_event = td-&gt;evtdev-&gt;next_event;
                td-&gt;evtdev-&gt;event_handler = clockevents_handle_noop;
                // 设置 hpet_clockevent 的 event_handler 为 clockevents_handle_noop.
        }

        ......
}

</pre>

&nbsp;

<pre class="brush: cpp; title: ; notranslate" title="">void clockevents_exchange_device(struct clock_event_device *old,
                                 struct clock_event_device *new)
{
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
                // 设置为 UNUSED mode
                list_del(&old-&gt;list);
                // 从原来的 clockevent_devices 中删除
                list_add(&old-&gt;list, &clockevents_released);
                // 加到 clockevents_released 链表中
        }
 
        if (new) {
                BUG_ON(new-&gt;mode != CLOCK_EVT_MODE_UNUSED);
                clockevents_shutdown(new);
        }
        local_irq_restore(flags);
}
</pre>

&nbsp;


  
好了，现在回到了 clockevents\_register\_device() 中的 clockevents\_notify\_released() 函数。

<pre class="brush: cpp; title: ; notranslate" title="">static void clockevents_notify_released(void)
{
        struct clock_event_device *dev;

        while (!list_empty(&clockevents_released)) {
                // 为空 list_empty 返回 1,如果 clockevents_released 链表不为空则进入循环
                // 现在链表中的设备为 hpet

                dev = list_entry(clockevents_released.next,
                                 struct clock_event_device, list);
                // dev 指向 hpet_clockevent  

                list_del(&dev-&gt;list);
                list_add(&dev-&gt;list, &clockevent_devices);
                // 再次加入到 clockevent_devices 链表
                clockevents_do_notify(CLOCK_EVT_NOTIFY_ADD, dev);
                // 再次设置 event_handler
        }
        // 在注册 global clock_event device 时因为 clockevents_released 链表为空，根本就没有进入循环体，但是现在不一样了。
}
</pre>

&nbsp;


  
_故事从这又 TMD 的开始了......_
  
**现在 CPU0 的 tick\_device 上的 clock\_event device 是 local 的 (lapic_clockevent),现在就是要处理全局的～**

<pre class="brush: cpp; title: ; notranslate" title="">static int tick_check_new_device(struct clock_event_device *newdev)
{
        ......

        cpu = smp_processor_id();
        // 还是 CPU0
        if (!cpumask_test_cpu(cpu, newdev-&gt;cpumask))
                goto out_bc;
        // cpumask:            cpumask to indicate for which CPUs this device works

        td = &per_cpu(tick_cpu_device, cpu);
        // 还是 CPU0 的那个 struct tick_device 实例

        curdev = td-&gt;evtdev;
        // 注意此时 curdev 并不是 NULL 而是指向 lapic_clockevent

        ......

        if (curdev) {
                // curdev = &lapic_clockevent
                // newdev = &hpet_clockevent

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
                        // 没办法只能去检查 hpet_clockevent 能否用作 broadcast device，因为它的 rating 值小于已经注册上的 lapic_clockevent

                // lapic rating = 100,hpet rating = 50 
                
        }

        ......

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

<pre class="brush: cpp; title: ; notranslate" title="">int tick_check_broadcast_device(struct clock_event_device *dev)
{
        if ((tick_broadcast_device.evtdev &&
             tick_broadcast_device.evtdev-&gt;rating &gt;= dev-&gt;rating) ||
             (dev-&gt;features & CLOCK_EVT_FEAT_C3STOP))
                return 0;

        clockevents_exchange_device(NULL, dev);

        tick_broadcast_device.evtdev = dev;
        // tick_broadcast_device 终于出现了，设置为 hpet

        if (!cpumask_empty(tick_get_broadcast_mask()))
                tick_broadcast_start_periodic(dev);
                // 设置周期模式，此函数在当前 context 中并没有被执行，也就是说 hpet_clockevent 的 event_handler 应该是 "clockevents_handle_noop"
        return 1;
}
</pre>

&nbsp;


  
好现现在说说现在所处的situation，Global 以及 CPU0 的 LAPIC 都已经注册完毕，假设系统 4-core,现在处于 kernel 初始化末期线程1中，马上要进行的是启动其他额外 CPU（1-3）,并且注册相应的 LAPIC clock_event device.

<pre class="brush: cpp; title: ; notranslate" title="">kernel_init()
  -&gt; smp_init()

static void __init smp_init(void)
{       
        unsigned int cpu;
        
        /* FIXME: This should be done in userspace --RR */
        for_each_present_cpu(cpu) {
                if (num_online_cpus() &gt;= setup_max_cpus)
                        break;
                if (!cpu_online(cpu))
                        cpu_up(cpu);
        }
        // 启动其他 CPU 调用 start_secondary()
                
        /* Any cleanup work */
        printk(KERN_INFO "Brought up %ld CPUs\n", (long)num_online_cpus()); 
        smp_cpus_done(setup_max_cpus);
                
        ......
}
</pre>

&nbsp;

<font size="3">局部时钟事件设备的注册</font>
  
`</p>
<pre>cpu_up()
  ...
  -> _cpu_up()
     smp_ops.cpu_up(cpu)
       -> native_cpu_up()
            -> do_boot_cpu()
                 -> wakeup_secondary_cpu_via_init()
                      -> startup_ipi_hook()
                         ......
                         maybe (这块没太看懂怎么 invoke 的 start_secondary)
                         start_secondary()
                           -> setup_secondary_APIC_clock()
                                -> setup_APIC_timer()
                                     -> clockevents_register_device(levt)`</pre> 

&nbsp;

其余的 CPU Lapic 注册过程和 CPU0 的一样也不复杂，最终都把相关 event\_handler 初始化为 "tick\_handle_periodic"
  
可是 ...... 但是 ...... 还是那句话，,故事远没有结束这才是一个开始 ……