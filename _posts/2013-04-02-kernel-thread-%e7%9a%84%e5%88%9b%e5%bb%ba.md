---
id: 447
title: Kernel Thread 的创建
date: 2013-04-02T23:28:03+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=447
permalink: /archives/447
views:
  - "9286"
dsq_thread_id:
  - "6101177293"
categories:
  - general
  - kernel
  - process
tags:
  - general
  - kernel
  - process
---
在 Linux 中有很多的内核线程，可以通过 `ps` command 查看到，比如: _kthreadd_ _ksoftirqd_ _watchdog_ 等等等 &#8230; 它们都是由内核从无到有创建的，通过它们的 pid 以及 ppid 可以得出以下几点:

  * 在内核初始化 `rest_init` 函数中，由进程 0 (swapper 进程)创建了两个 process <ol style="list-style-type: decimal;">
      <li>
        <em>init</em> 进程 <em>(pid = 1, ppid = 0)</em>
      </li>
      <li>
        <strong>kthreadd</strong> <em>(pid = 2, ppid = 0)</em>
      </li>
    </ol>

  * 所有其它的内核线程的 ppid 都是 2，也就是说它们都是由 **kthreadd** thread 创建的
  * 所有的内核线程在大部分时间里都处于阻塞状态(TASK_INTERRUPTIBLE)只有在系统满足进程需要的某种资源的情况下才会运行

创建一个内核 thread 的接口函数是:

    kthread_create()
    kthread_run()

_这两个函数的区别就是 `kthread_run()` 函数中封装了前者，由 `kthread_create()` 创建的 thread 不会立即运行，而后者创建的 thread 会立刻运行，原因是在 `kthread_run()` 中调用了 `wake_up_process()`.用什么函数看你自己的需求，如果你要让你创建的 thread 运行在指定的 cpu 上那必须用前者(因为它创建的 thread 不会运行)，然后再用 `kthread_bind()` 完成绑定，最后 wake up._

* * *

下面就大概说一下内核 thread 创建的过程.

## kthreadd {#kthreadd}

    rest_init()
    {
        pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
        kthreadd_task = find_task_by_pid(pid);
    }

## kernel thread {#kernel-thread}

`kthread_create(usb_stor_control_thread, us, "usb-storage")`

    struct task_struct *kthread_create(int (*threadfn)(void *data),
                                   void *data,
                                   const char namefmt[],
                                   ...)
    {
    
        struct kthread_create_info create;
    
        create.threadfn = threadfn;
        新创建的线程运行的函数
        create.data = data;
        init_completion(&create.done);
        初始化一个 completion 用于一个线程告诉其它线程某个工作已经完成
    
        spin_lock(&kthread_create_lock);
        list_add_tail(&create.list, &kthread_create_list);
        加到待创建 thread list 中去
        spin_unlock(&kthread_create_lock);
    
        wake_up_process(kthreadd_task);
        唤醒 kthreadd 线程来创建新的内核线程 
        wait_for_completion(&create.done);
        等待新的线程创建完毕(睡眠在等待队列中)
    
        ......
    
    }

kthreadd 是一个死循环，大部分时间在睡眠，只有创建内核线程时才被唤醒.

    int kthreadd(void *unused)
    {
        关注循环体
    
            for (;;) {
                set_current_state(TASK_INTERRUPTIBLE);
                if (list_empty(&kthread_create_list))
                        schedule();
                首先将线程状态设置为 TASK_INTERRUPTIBLE, 如果当前
                没有要创建的线程则主动放弃 CPU 完成调度.此进程变为阻塞态
    
                __set_current_state(TASK_RUNNING);
                运行到此表示 kthreadd 线程被唤醒(就是我们当前)
                设置进程运行状态为 TASK_RUNNING
    
                spin_lock(&kthread_create_lock);
    
                while (!list_empty(&kthread_create_list)) {
                        struct kthread_create_info *create;
    
                        create = list_entry(kthread_create_list.next,
                                            struct kthread_create_info, list);
                        从链表中取得 kthread_create_info 结构的地址，在上文中已经完成插入操作(将
                        kthread_create_info 结构中的 list 成员加到链表中，此时根据成员 list 的偏移
                        获得 create)  
    
                        list_del_init(&create->list);
                        取出后从列表删除
                        spin_unlock(&kthread_create_lock);
    
                        create_kthread(create);
                        完成真正线程的创建
    
                        spin_lock(&kthread_create_lock);
                }
    
                spin_unlock(&kthread_create_lock);
           }
    
    }

create_kthread 函数完成真正线程的创建

    static void create_kthread(struct kthread_create_info *create)
    {       
        int pid;
    
        pid = kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
        其实就是调用首先构造一个假的上下文执行环境，最后调用 do_fork()
        返回进程 id, 创建后的线程执行 kthread 函数
        if (pid < 0) {
                create->result = ERR_PTR(pid);
                complete(&create->done);
        }
    }

此时回到 kthreadd thread,它在完成了进程的创建后继续循环，检查 kthread\_create\_list 链表，如果为空，则 kthreadd 内核线程昏睡过去 &#8230;&#8230;

当前有三个线程:

<ol style="list-style-type: decimal;">
  <li>
    kthreadd thread 已经光荣完成使命，睡眠
  </li>
  <li>
    唤醒 kthreadd 的线程由于新创建的线程还没有创建完毕而继续睡眠 <em>(在 kthread_create 函数中)</em>
  </li>
  <li>
    新创建的线程已经正在运行 <code>kthread</code>，但是由于还有其它工作没有做所以还没有最终创建完成.
  </li>
</ol>

新创建的线程 kthread 函数

<pre class="brush: cpp; title: ; notranslate" title="">static int kthread(void *_create)
{
    struct kthread_create_info *create = _create;
    // create 指向 kthread_create_info 中的 kthread_create_info
    int (*threadfn)(void *data) = create-&gt;threadfn;
    // 新的线程创建完毕后执行的函数
    void *data = create-&gt;data;
    struct kthread self;
    int ret;

    self.should_stop = 0;
    // 在 kthread_should_stop() 函数中会检查这个值,表示当前线程是否
    // 运行结束. kthread_should_stop() 常被用于线程函数中.
    init_completion(&self.exited);
    current-&gt;vfork_done = &self.exited;

    /* OK, tell user we're spawned, wait for stop or wakeup */
    __set_current_state(TASK_UNINTERRUPTIBLE);
    // 设置运行状态为 TASK_UNINTERRUPTIBLE
    create-&gt;result = current;
    // current 表示当前新创建的 thread 的 task_struct 结构
    complete(&create-&gt;done);
    // new thread 创建完毕

    schedule();
    // 执行任务切换，让出 CPU

    ......

}
</pre>

线程创建完毕:

  * 创建新 thread 的进程恢复运行 `kthread_create()` 并且返回新创建线程的任务描述符
  * 新创建的线程由于执行了 `schedule()` 调度，此时并没有执行.

最后唤醒新创建的线程 :

<pre><span style="color: #0000ff;"><code>wake_up_process(p);</code></span></pre>

当线程被唤醒后，继续 `kthread()`

        ret = -EINTR;
        if (!self.should_stop)
                ret = <span style="color: #ff0000;">threadfn</span>(data);
        执行相应的线程函数
    
        do_exit(ret);
        最后退出

## 总结 {#总结}

  * 任何一个内核线程入口都是 `kthread()`
  * 通过 `kthread_create()` 创建的内核线程不会立刻运行．需要手工 wake up.
  * 通过 `kthread_create()` 创建的内核线程有可能不会执行相应线程函数`threadfn`而直接退出

最后写了一个创建内核线程的模块仅供参考:

<pre class="brush: cpp; title: ; notranslate" title="">#include &lt;linux/init.h&gt;
#include &lt;linux/time.h&gt;
#include &lt;linux/errno.h&gt;
#include &lt;linux/module.h&gt;
#include &lt;linux/sched.h&gt;
#include &lt;linux/kthread.h&gt;
#include &lt;linux/cpumask.h&gt;

#define sleep_millisecs 1000*60

static int thread(void *arg)
{
	long ns1, ns2, delta;
	unsigned int cpu;
	struct timespec ts1, ts2;

	cpu = *((unsigned int *)arg);

	printk(KERN_INFO "### [thread/%d] test start \n", cpu);

	while (!kthread_should_stop()) {
                /*
                 * Do What you want
                 */
		schedule_timeout_interruptible(
				msecs_to_jiffies(1));
	}

	printk(KERN_INFO "### [thread/%d] test end \n", cpu);

	return 0;
}

static int __init XXXX(void)
{
	int cpu;
	unsigned int cpu_count = num_online_cpus();
	unsigned int parameter[cpu_count];
	struct task_struct *t_thread[cpu_count];

	for_each_present_cpu(cpu){
		parameter[cpu] = cpu;

		t_thread[cpu] = kthread_create(thread, (void *) (parameter+cpu), "thread/%d", cpu);

		if (IS_ERR(t_thread[cpu])) {
			printk(KERN_ERR "[thread/%d]: creating kthread failed\n", cpu);

			goto out;
		}

		kthread_bind(t_thread[cpu], cpu);
		wake_up_process(t_thread[cpu]);
	}

	schedule_timeout_interruptible(
			msecs_to_jiffies(sleep_millisecs));

	for (cpu = 0; cpu &lt; cpu_count; cpu++) {
		kthread_stop(t_thread[cpu]);
	}

out:
	return 0;
}

static void __exit XXXX_exit(void)
{
}

module_init(XXXX);
module_exit(XXXX_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("bluezd");
MODULE_DESCRIPTION("Kernel study");
</pre>