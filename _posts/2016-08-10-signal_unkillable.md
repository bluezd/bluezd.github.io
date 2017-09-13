---
id: 762
title: SIGNAL_UNKILLABLE
date: 2016-08-10T13:30:07+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=762
permalink: /archives/762
views:
  - "594"
categories:
  - kernel
  - signal
tags:
  - kernel
  - signal
---
众所周知 `SIGKILL` 信号不能被捕获，可以通过如下方法给进程发送此信号来结束它:

```{.bash}
kill -s SIGKILL `pgrep XXX`
```

那么问题来了，可以给 init 进程发送此信号结束它吗? 显然是不能，如果 init 进程挂了，系统不就挂了吗?

验证一下:

```{.bash}
kill -s SIGKILL 1
```

init 进程也就是 systemd 还在那里，没有任何改变。

* * *

原因就是内核在创建 init 进程的时候设置了 `SIGNAL_UNKILLABLE` 这个 flag:

```{.c}
kernel/fork.c
--

static struct task_struct *copy_process()
{
        ...
        if ((clone_flags & CLONE_SIGHAND) && !(clone_flags & CLONE_VM))
                 return ERR_PTR(-EINVAL);
        ...
        if (likely(p->pid)) {
                ptrace_init_task(p, (clone_flags & CLONE_PTRACE) || trace);

                init_task_pid(p, PIDTYPE_PID, pid);
                if (thread_group_leader(p)) {
                        init_task_pid(p, PIDTYPE_PGID, task_pgrp(current));
                        init_task_pid(p, PIDTYPE_SID, task_session(current));

                        if (is_child_reaper(pid)) {
                                ns_of_pid(pid)->child_reaper = p;
                                p->signal->flags |= SIGNAL_UNKILLABLE;
                        }
        ...
}
```

内核 signal 相应处理:

```{.c}
kernel/signal.c
--

static void complete_signal(int sig, struct task_struct *p, int group)
{
        ...
        if (sig_fatal(p, sig) &&
            !(signal->flags & (SIGNAL_UNKILLABLE | SIGNAL_GROUP_EXIT)) &&
            !sigismember(&t->real_blocked, sig) &&
            (sig == SIGKILL || !t->ptrace)) {
                ...
        }
        ...
}
```

* * *

写了个内核模块用于测试，主要是检索所有进程，查看是否设置了 `SIGNAL_UNKILLABLE` flag, 如果设置了，打引出进程名字，并且如果 pid == 1 的话则 clear 这个 flag.

生成了模块后通过如下方法对模块进行签名(不签名也可以，取决于内核选项，一般也可以载入成功):


```{.c}
CONFIG_MODULE_SIG_HASH="sha256"

./sign-file sha256 ./signing_key.priv ./signing_key.x509 /path/to/XXX.ko

```

signing_key.priv 和 signing_key.x509是在编译内核的时候通过 x509.genkey 配置文件生成的，通过如下命令:
```{.c}
kernel/Makefile
--

openssl req -new -nodes -utf8 -$(CONFIG_MODULE_SIG_HASH) -days 36500 \
        -batch -x509 -config x509.genkey \
        -outform DER -out signing_key.x509 \
        -keyout signing_key.priv 2>&1
```

然后内核把 signing\_key.x509(公钥) 写到 x509\_certificate_list最后加载到内核中去，当手工载入模块的时候 内核会通过这个公钥对这个模块进行验证。所以在生成模块后需要用那两个公钥和私钥对模块进行签名，以便内核 可以验证并且知道这个模块是安全的。

* * *

Summary:

  1. 在 Bare Metal系统中只有 init 进程设置了 `SIGNAL_UNKILLABLE` flag.
  2. clear 这个 flag 后如果发送 SIGKILL 信号 init 进程, 进程被 kill 掉了，系统也 panic 挂掉了。
