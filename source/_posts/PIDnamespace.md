---
title: PID namespace
date: 2016-10-10 21:00:00
tags:
- linux内核
- PID namespace  

---
### PID Namespace
PID namespace能够允许创建任务的集合，使得这样的集合像一个独立的机器一样运行。在不同的namespace中，任务能够拥有相同的ID。  
PID Namespace主要能够解决不同host之间，containers的移植问题，由于使用了一个独立的namespace，从一个host移植到另一个host时，是能够保持PID值不变的。如果没有这个特性，那么移植过程很可能失败，因为具有相同ID的进程可能在目标node上是存在的，这样就会造成冲突。  
PID namespaces也是层次结构的，在一个新的PID namespace被创建了，相同PID namespace中的所有task都能够互相可见，但是新的namespace不能看到之前namespace中的task，也就是说任务可能会有多个pid，每个namespace对应一个。  
### Userspace API
为了创建一个新的namespace，进程需要调用clone系统调用，并且使用CLONE_NEWPID标识位。  
在一个新的namespace当中，第一个task的PID是1，它也就是这个namespace的init，以及child_reaper。但这个init是可以死亡的，此时这个namespace都会终止。  
在把tasks分割出来之后，还必须对proc进行处理，让它只显示当前task可见的PID。为了实现这个目的，procfs应该在每个namespace被使用一次。  
### Internal API
一个task所拥有的所有PID都在struct pid中被描述了。这个数据结构如下：  

	struct upid {
	int nr;					/* moved from struct pid */
	struct pid_namespace *ns;		/* the namespace this value
						 * is visible in
						 */
	struct hlist_node pid_chain;		/* moved from struct pid */
    };

    struct pid {
	atomic_t count;
	struct hlist_head tasks[PIDTYPE_MAX];
	struct rcu_head rcu;
	int level;				/* the number of upids */
	struct upid numbers[0];
    };
   
这里，struct upid表示PID值，它储存在hash当中，并且拥有PID值。为了转换得到这个pid值，可以使用task_pid_nr,pid_nr_ns(),find_task_by_vpid等函数。  
这些函数的后缀有一些规律：  
\_\_nr()：对“全局”的PID进行操作，这里全局指的是在整个系统中也是独一无二的。pid_nr会告诉你struct pid的global PID，这只在PID值不会离开kernel时使用。  
\_\_vnr()：对“virtual”PID进行操作，也就是进程可见的ID，例如task_pid_vnr会告诉你一个task的PID。  
\_nr\_ns()：对指定namespace中的PID进行处理，如果希望得到某个task的PID，可以通过task_pid_nr_ns来获得pid number，在用find_task_by_pid_ns来找到这个task。这个方法在系统调用中很常见，特别是当PID来自用户空间时。在这种情况下，task可能是在另一个namespace中的。