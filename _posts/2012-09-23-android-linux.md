---
layout: post
title: "linux启动过程分析"
description: ""
category: "work linux" 
tags: []
sumarry: | 
  这篇文章的目的在于分析linux内核的启动过程,它到底做了些什么事情,这么多的事情,内核到底是怎样安排的,进程调度机制如何进行初始化的,内存方面又是如何操作的,文件系统又是如何的运行,什么时候开始挂载文件系统,什么时候开始操作设备.这些都是我们这篇文章所关注的.
---
{% include JB/setup %}
  这篇文章的目的在于分析linux内核的启动过程,它到底做了些什么事情,这么多的事情,内核到底是怎样安排的,进程调度机制如何进行初始化的,内存方面又是如何操作的,文件系统又是如何的运行,什么时候开始挂载文件系统,什么时候开始操作设备.这些都是我们这篇文章所关注的.



#从start_kernel函数开始

恩, 首先, 需要找到入口函数, 而入口函数是在init/main.c文件里, 我们首先来看看这个函数吧.

<pre class="brush: js;">   
asmlinkage void __init start_kernel(void)
{
	char * command_line;
	extern const struct kernel_param __start___param[], __stop___param[];
	smp_setup_processor_id();
	lockdep_init();
	debug_objects_early_init();
</pre>


1. 首先需要来看看smp_setup_processor_id函数,这个函数的作用在于获取当前正在执行初始化的处理器ID,如果仔细地阅读完初始化函数start_kernel，就会发现里面还有调用smp_processor_id()函数，这两个函数都是获取多处理器的ID，为什么会需要两个函数呢？其实这里有一个差别的，smp_setup_processor_id()函数可以不调用setup_arch()初始化函数就可以使用，而smp_processor_id()函数是一定要调用setup_arch()初始化函数后，才能使用。smp_setup_processor_id()函数是直接获取对称多处理器的ID，而smp_processor_id()函数是获取变量保存的处理器ID，因此一定要调用初始化函数。由于smp_setup_processor_id()函数不用调用初始化函数，可以放在内核初始化start_kernel函数的最前面使用，而函数smp_processor_id()只能放到setup_arch()函数调用的后面使用了。smp_setup_processor_id()函数每次都要中断CPU去获取ID，这样效率比较低。如果内核只是有单处理器系统，smp_setup_processor_id()函数是是空的，不必要做任保的处理。
2. lockdep_init(); 初始化互斥锁的dependency
3. debug_objects_early_init(); 初始化debug kernel相关

#初始化带防止栈溢出攻击保护的堆栈,cgroup,以及关中断

<pre class="brush: js;">  
	boot_init_stack_canary();//stack_canary的是带防止栈溢出攻击保护的堆栈。
	cgroup_init_early();//初始化cgroup
	local_irq_disable();//很明显,关中断,后面要做一些重要的事情.
	early_boot_irqs_disabled = true;
</pre>


#
<pre class="brush: js;">  
	tick_init();//初始化time ticket，时钟
	boot_cpu_init();//用以启动的CPU进行初始化。也就是初始化CPU0
	page_address_init();//初始化页面
	printk(KERN_NOTICE "%s", linux_banner);
	setup_arch(&amp;command_line);//CPU架构相关的初始化
	mm_init_owner(&amp;init_mm, &amp;init_task);//初始化内存管理
	mm_init_cpumask(&amp;init_mm);
	setup_command_line(command_line);//处理启动命令行
	setup_nr_cpu_ids();
	setup_per_cpu_areas();
	smp_prepare_boot_cpu();	//准备boot_cpu
</pre>

#解析启动参数
<pre class="brush: js;"> 
	build_all_zonelists(NULL);//zone初始化
	page_alloc_init();//初始化page allocation相关结构
	printk(KERN_NOTICE "Kernel command line: %s\n", boot_command_line);
	parse_early_param();
	parse_args("Booting kernel", static_command_line, __start___param,
		   __stop___param - __start___param,
		   &amp;unknown_bootoption);//解析启动参数
</pre>		   

# 

<pre class="brush: js;"> 
	setup_log_buf(0);
	pidhash_init();//初始化process ID hash表
	vfs_caches_init_early();//文件系统caches预初始化
	sort_main_extable();//初始化exception table
	trap_init();//初始化trap，用以处理错误执行代码
	mm_init();//初始化内存管理
	sched_init();//进程调度初始化
	/*	
</pre>	


# 
<pre class="brush: js;"> 
	preempt_disable();
	if (!irqs_disabled()) {
		printk(KERN_WARNING "start_kernel(): bug: interrupts were "
				"enabled *very* early, fixing it\n");
		local_irq_disable();
	}
	idr_init_cache();
	perf_event_init();
	rcu_init();//Read_Copy_Update机制初始
	radix_tree_init();
	/* init some links before init_ISA_irqs() */
	early_irq_init();
	init_IRQ();//初始化中断
	prio_tree_init();
	init_timers();
	hrtimers_init();
	softirq_init();
	timekeeping_init();
	time_init();//初始化时钟
	profile_init();//Profile初始化
	call_function_init();
	if (!irqs_disabled())
		printk(KERN_CRIT "start_kernel(): bug: interrupts were "
				 "enabled early\n");
	early_boot_irqs_disabled = false;
	local_irq_enable();
</pre>


# 

<pre class="brush: js;"> 
	gfp_allowed_mask = __GFP_BITS_MASK;
	kmem_cache_init_late();//初始化CPU Cache
	console_init();//初始化console
	if (panic_later)
		panic(panic_later, panic_param);
	lockdep_info();
	locking_selftest();//自测试锁
</pre>


# 

<pre class="brush: js;"> 
	page_cgroup_init();//页面初始
	enable_debug_pagealloc();//页面分配debug启用
	debug_objects_mem_init();//debug object是什么？
	kmemleak_init();//memory leak 侦测初始化
	setup_per_cpu_pageset();//设置每个CPU的页面集合
	numa_policy_init();// NUMA (Non Uniform Memory Access) policy 
	if (late_time_init)
		late_time_init();
	sched_clock_init();//初始化调度时钟
	calibrate_delay();//协同不同CPU的时钟
	pidmap_init();
	anon_vma_init();
</pre>


# 


<pre class="brush: js;"> 
	thread_info_cache_init();//初始化thread info
	cred_init();//credential
	fork_init(totalram_pages);//初始化fork
	proc_caches_init();//初始化/proc的cache?
	buffer_init();
	key_init();
	security_init();
	dbg_late_init();
	vfs_caches_init(totalram_pages);//文件系统cache初始化
	signals_init();
	/* rootfs populating might need page-writeback */
	page_writeback_init();
</pre>


# 

<pre class="brush: js;"> 
	cgroup_init();
	cpuset_init();
	taskstats_init_early();
	delayacct_init();
	check_bugs();
	acpi_early_init(); /* before LAPIC and SMP init */
	sfi_init_late();
	ftrace_init();
	/* Do the rest non-__init'ed, we're now alive */
	rest_init();//重点的动作就在这个函数里发生
</pre>	


# rest_init
<pre class="brush: js;"> 
static noinline void __init_refok rest_init(void)
{
	int pid;
	rcu_scheduler_starting();
	kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);//启动了另外的线程进行kernel_init操作
	numa_default_policy();
	pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
	rcu_read_lock();
	kthreadd_task = find_task_by_pid_ns(pid, &amp;init_pid_ns);
	rcu_read_unlock();
	complete(&amp;kthreadd_done);
	init_idle_bootup_task(current);
	preempt_enable_no_resched();
	schedule();
	preempt_disable();
	cpu_idle();
}
</pre>	


#kernel_init
<pre class="brush: js;"> 
static int __init kernel_init(void * unused)
{
	/*
	 * Wait until kthreadd is all set-up.
	 */
	wait_for_completion(&amp;kthreadd_done);
	/*
	 * init can allocate pages on any node
	 */
	set_mems_allowed(node_states[N_HIGH_MEMORY]);
	/*
	 * init can run on any cpu.
	 */
	set_cpus_allowed_ptr(current, cpu_all_mask);
	cad_pid = task_pid(current);
	smp_prepare_cpus(setup_max_cpus);
	do_pre_smp_initcalls();
	lockup_detector_init();
	smp_init();
	sched_init_smp();
	do_basic_setup();//基本的初始化
</pre>	



#do_basic_setup

<pre class="brush: js;"> 
static void __init do_basic_setup(void)
{
	cpuset_init_smp();
	usermodehelper_init();
	init_tmpfs();
	driver_init();//驱动初始化
	init_irq_proc();
	do_ctors();
	do_initcalls();//开始调用所有的初始化函数.
}
</pre>	

#do_initcalls

<pre class="brush: js;"> 
static void __init do_initcalls(void)
{
	initcall_t *fn;
	for (fn = __early_initcall_end; fn &lt; __initcall_end; fn++)
		do_one_initcall(*fn);
}
</pre>	


在这里,我们需要来理解一个东西, 就是在__early_initcall_end和__initcall_end之间的初始化的动作,到底是如何操作的,是在哪里赋值的呢?下面我们重点来看看.

<pre class="brush: js;"> 
/* include/linux/init.h */
/* initcalls are now grouped by functionality into separate 
* subsections. Ordering inside the subsections is determined
* by link order. 
* For backwards compatibility, initcall() puts the call in 
* the device init subsection.
*/

#define __define_initcall(level,fn) \
       static initcall_t __initcall_##fn __attribute_used__ \
       __attribute__((__section__(".initcall" level ".init"))) = fn

#define core_initcall(fn)        __define_initcall("1",fn)
#define postcore_initcall(fn)        __define_initcall("2",fn)
#define arch_initcall(fn)        __define_initcall("3",fn)
#define subsys_initcall(fn)            __define_initcall("4",fn)
#define fs_initcall(fn)                     __define_initcall("5",fn)
#define device_initcall(fn)           __define_initcall("6",fn)
#define late_initcall(fn)         __define_initcall("7",fn)

#define __initcall(fn) device_initcall(fn)
#define __exitcall(fn) \
       static exitcall_t __exitcall_##fn __exit_call = fn

#define console_initcall(fn) \
       static initcall_t __initcall_##fn \
       __attribute_used__ __attribute__((__section__(".con_initcall.init")))=fn

#define security_initcall(fn) \
       static initcall_t __initcall_##fn \
       __attribute_used__ __attribute__((__section__(".security_initcall.init"))) = fn
#define module_init(x)   __initcall(x);//从这里知道module_init的等级为6，相对靠后
#define module_exit(x)  __exitcall(x);
</pre>

可以发现这些initcall(fn)最终都是通过\__define_initcall(level,fn)宏定义生成的。
\__define_initcall宏定义如下：

<pre class="brush: js;"> 
#define __define_initcall(level,fn) \
       static initcall_t __initcall_##fn __attribute_used__ \
       __attribute__((__section__(".initcall" level ".init"))) = fn
</pre>       
这句话的意思为定义一个initcall_t型的初始化函数，函数存放在.initcall”level”.init section内。.initcall”level”.init section定义在vmlinux.lds内。


<pre class="brush: js;"> 
__initcall_start = .; 
*(.initcallearly.init) __early_initcall_end = .; 
*(.initcall0.init) 
*(.initcall0s.init) 
*(.initcall1.init) 
*(.initcall1s.init) 
*(.initcall2.init) 
*(.initcall2s.init) 
*(.initcall3.init) 
*(.initcall3s.init) 
*(.initcall4.init) 
*(.initcall4s.init) 
*(.initcall5.init) 
*(.initcall5s.init) 
*(.initcallrootfs.init) 
*(.initcall6.init) 
*(.initcall6s.init) 
*(.initcall7.init) 
*(.initcall7s.init) 
__initcall_end = .;
</pre>

正好包括了上面init.h里定义的从core_initcall到late_initcall等7个level等级的.initcall”level”.init section. 因此通过不同的\*_initcall声明的函数指针最终都会存放不同level等级的.initcall”level”.init section内。这些不同level的section按level等级高低依次存放。

#结语
由于对于内核启动的具体细节很不清楚,本文只是一个初稿,以后笔者会不断的来完善这篇文章的.
