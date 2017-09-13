---
id: 396
title: Kernel Debug Tips
date: 2012-12-27T21:17:04+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=396
permalink: /archives/396
views:
  - "7349"
categories:
  - kernel
tags:
  - kernel
  - tips
---
printk 是 kernel debug 中经常用到的方法，可以在某个代码片段中加入 printk 来 view 某些关键的变量，但是如果要调试的代码是中断处理程序，比如 timer\_interrupt, smp\_apic\_timer\_interrupt 那用 printk 的话结果就悲剧了，但是我们可以通过某种方法来设置何时 printk .此时就用到了这个 panic\_on\_oops 变量

  * 使用方法很简单，不过要重新 build 个 kernel <ol style="list-style-type: decimal;">
      <li>
        修改 /kernel/panic.c 中 panic_on_oops 变量为 0
      </li>
      <li>
        在需要调试的代码段中加入 if (panic_on_oops) { instructions }
      </li>
    </ol>

_Example :_ 

<pre class="brush: cpp; title: ; notranslate" title="">diff --git a/arch/x86_64/kernel/time.c b/arch/x86_64/kernel/time.c
index 187e252..ac816b7 100644
--- a/arch/x86_64/kernel/time.c
+++ b/arch/x86_64/kernel/time.c
@@ -624,6 +624,11 @@ static inline void set_cyc2ns_scale(unsigned long cpu_khz)

 static inline unsigned long long cycles_2_ns(unsigned long long cyc)
 {
+        if (panic_on_oops){
+                printk(KERN_INFO "=&gt; XXXXXX");
+                ......
+        }
+
        return (cyc * cyc2ns_scale) &gt;&gt; NS_SCALE;
 }

diff --git a/kernel/panic.c b/kernel/panic.c
index edb779e..37cb952 100644
--- a/kernel/panic.c
+++ b/kernel/panic.c
@@ -20,7 +20,7 @@
 #include &lt;linux/kexec.h&gt;
 #include &lt;linux/debug_locks.h&gt;

-int panic_on_oops = 1;
+int panic_on_oops = 0;
 int tainted;
 static int pause_on_oops;
 static int pause_on_oops_flag;
</pre>

## Checking and Setting {#checking-and-setting}

待新 kernel build 好之后，查看以及设置当前 kernel panic\_on\_oops vaule

  * Checking
  
    cat /proc/sys/kernel/panic\_on\_oops
  
    
  * Setting 
      * enable
  
        <span style="color: #00ff00;">echo &#8220;1&#8221; > /proc/sys/kernel/panic_on_oops</span>
      * disable
  
        echo &#8220;0&#8221; > /proc/sys/kernel/panic\_on\_oops