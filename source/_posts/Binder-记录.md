---
layout: pager
title: Binder 记录
date: 2020-09-17 11:13:45
tags: [Binder,Android]
description:  Binder 记录
---
### 简介
这俩天看了Android Binder的原理，这里主要记录下个人对于Binder的疑点,参考的内容是来自 老罗关于Binder的一系列的文章分析，下面是文章的地址 https://blog.csdn.net/luoshengyang/article/details/6618363 这篇文章只是在记录一下我个人的理解


### Binder整体认识
![结果显示](/uploads/Binder源码分析/1.png)
每个Android的进程，只能运行在自己进程所拥有的虚拟地址空间。对应一个4GB的虚拟地址空间，其中3GB是用户空间，1GB是内核空间，当然内核空间的大小是可以通过参数配置调整的。对于用户空间，不同进程之间彼此是不能共享的，而内核空间却是可共享的。Client进程向Server进程通信，恰恰是利用进程间可共享的内核内存空间来完成底层通信工作的，Client端与Server端进程往往采用ioctl等方法跟内核空间的驱动进行交互。


### 调用 fd = open_driver("/dev/binder") 的作用
使用open函数，打开/dev/binder/得到当前进程对应的文件描述符，然后将当前进程对应的 binder_proc 对象保存到 filp->private_data = proc;这样后续就能方便的取出，如果同一个进程里面多次调用,open_driver函数，也不会多次的创建，内不会根据当前的进程的pid在全局的 binder_procs中进行查找

### service_manager 特殊的service
service_manager 本身做为一个service的存在，内部用来管理当前系统的service的添加查找操作，在系统启动的时候，创建，service_manager Android进程间通信（IPC）机制Binder守护进程对象，也就是下面全局的对象保存的是 service_manager 这个进程的binder_node对象，而且获取引用对象的时候，service_manager 对应的引用为0，其他的则要得到对应的引用值static struct binder_node *binder_context_mgr_node ,static uid_t binder_context_mgr_uid = -1;
  
### 数据结构的传递和解析 
在调用 BBbinder或者BpBinder的transact函数的时候，会先将数据最先构成 flat_binder_object 对象，这个对象用来存储Binder对象，之后会再次封装成 binder_transaction_data 对象，最终再封装成 binder_write_read ，在Linux 内核会首先将数据拷贝到内核中，解析的过程为封装的逆过程

### Binder内部的原理
每一个binder_proc 都有todo，wait队列，每次执行  res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);内部都会判断当前todo队列或者transaction队列是否为空，如果为空，则会处于挂起状态，等待唤醒,对于addService来说，由于handle传递的是0，所以操作的是service_manager对应的binder_proc
```java
先看内部阻塞的实现，当执行  res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);的时候会调用到 static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
内部会根据传递进来的cmd执行对应的操作，由于这里是 BINDER_WRITE_READ ，这个也是binder最重要的命令了,就会执行到
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
	int ret;
	struct binder_proc *proc = filp->private_data;
	struct binder_thread *thread;
	unsigned int size = _IOC_SIZE(cmd);
	...
	thread = binder_get_thread(proc);
	if (thread == NULL) {
		ret = -ENOMEM;
		goto err;
	}

	switch (cmd) {
	case BINDER_WRITE_READ:
		ret = binder_ioctl_write_read(filp, cmd, arg, thread);
		if (ret)
			goto err;
		break;
	}
	...
}
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;
	...
	if (bwr.write_size > 0) {
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
		trace_binder_write_done(ret);
		if (ret < 0) {
			bwr.read_consumed = 0;
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	if (bwr.read_size > 0) {
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		trace_binder_read_done(ret);
		binder_inner_proc_lock(proc);
		if (!binder_worklist_empty_ilocked(&proc->todo))
			binder_wakeup_proc_ilocked(proc);
		binder_inner_proc_unlock(proc);
		if (ret < 0) {
			if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
				ret = -EFAULT;
			goto out;
		}
	}
	...
	if (copy_to_user(ubuf, &bwr, sizeof(bwr))) {
		ret = -EFAULT;
		goto out;
	}
out:
	return ret;
}
内部会将传递过来的数据，拷贝到内核中，之后判断当前是否需要写操作，当写操作执行完毕之后，判断当前是否有必要执行读操作，也就是 binder_thread_read 函数
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	int ret = 0;
	int wait_for_proc_work;

	...

	//判断当前是否需要等待，也就是判断当前的binder_thread的 transaction_stack 队列释放为空，或者都todo队列为空，则wait_for_proc_work 就为false,意思为要等待操作
	trace_binder_wait_for_work(wait_for_proc_work,!!thread->transaction_stack, !binder_worklist_empty(proc, &thread->todo));
	if (wait_for_proc_work) {
		if (!(thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
					BINDER_LOOPER_STATE_ENTERED))) {
			binder_user_error("%d:%d ERROR: Thread waiting for process work before calling BC_REGISTER_LOOPER or BC_ENTER_LOOPER (state %x)\n",
				proc->pid, thread->pid, thread->looper);
			wait_event_interruptible(binder_user_error_wait,
						 binder_stop_on_user_error < 2);
		}
		binder_set_nice(proc->default_priority);
	}

	if (non_block) {
		if (!binder_has_work(thread, wait_for_proc_work))
			ret = -EAGAIN;
	} else {
		ret = binder_wait_for_work(thread, wait_for_proc_work);
	}

	thread->looper &= ~BINDER_LOOPER_STATE_WAITING;

	if (ret)
		return ret;
	...
}
如果当前需要等待则 wait_for_proc_work就为false，那么 执行binder_wait_for_work(thread, wait_for_proc_work);函数就会处于挂起状态
static int binder_wait_for_work(struct binder_thread *thread,
				bool do_proc_work)
{
	DEFINE_WAIT(wait);
	struct binder_proc *proc = thread->proc;
	int ret = 0;

	freezer_do_not_count();
	binder_inner_proc_lock(proc);
	for (;;) {
		prepare_to_wait(&thread->wait, &wait, TASK_INTERRUPTIBLE);
		if (binder_has_work_ilocked(thread, do_proc_work))
			break;
		if (do_proc_work)
			list_add(&thread->waiting_thread_node,
				 &proc->waiting_threads);
		binder_inner_proc_unlock(proc);
		schedule();
		binder_inner_proc_lock(proc);
		list_del_init(&thread->waiting_thread_node);
		if (signal_pending(current)) {
			ret = -ERESTARTSYS;
			break;
		}
	}
	finish_wait(&thread->wait, &wait);
	binder_inner_proc_unlock(proc);
	freezer_count();

	return ret;
}
```
接着看怎么样往目标的todo队列添加的实现
```java
target_node = binder_context_mgr_node;
target_proc = target_node->proc;
target_list = &target_proc->todo;
target_wait = &target_proc->wait; 
之后，创建一个binder_transaction事物，t，这里要注意这个事物t中的from是代表从哪个binder_thread发过来的，后续在寻找恢复的时候，根据这个from可以找到目标对象
t->from = thread;
t->sender_euid = proc->tsk->cred->euid;
t->to_proc = target_proc;
t->to_thread = target_thread;
添加到对应目标的todo列表中，之后唤醒目标对象,目标对象唤醒之后，会从todo队列中取出事物，然后将内核接收到内容传递到service_manager中
之后，就可以将这个todo事物从
表明这个事务虽然在驱动程序中已经处理完了，但是它仍然要等待Service Manager完成之后，给驱动程序一个确认，也就是需要等待回复，于是把当前事务t放在thread->transaction_stack队列的头部
t->to_parent = thread->transaction_stack;
t->to_thread = thread;
thread->transaction_stack = t;
注意这里并没有改变当前事物t的from值,当目标处理完之后，这里是service manager之后，会回复对方，就可以从这个transaction_stack 中找到这个事物，然后从from中找到target_thread
in_reply_to = thread->transaction_stack;
if (in_reply_to == NULL) {
    ......
    return_error = BR_FAILED_REPLY;
    goto err_empty_call_stack;
}
binder_set_nice(in_reply_to->saved_priority);
if (in_reply_to->to_thread != thread) {
    .......
    goto err_bad_call_stack;
}
thread->transaction_stack = in_reply_to->to_parent;
target_thread = in_reply_to->from;
```

所以也就可以找到当前线程对应的todo,wait队列，往对应的目标队列添加事物，之后唤醒对方，数据就传递过去了

### service的实例对象保存在当前进程对应的binder_proc,其他的都为引用
在binder_transaction中 当传输的为Binder实体的时候，type为 BINDER_TYPE_BINDER
```java
node = binder_get_node(proc, fp->binder);
if (!node) {
    node = binder_new_node(proc, fp);
    ...
}
查找过程，也就是通过binder的地址来索引
static struct binder_node *binder_get_node_ilocked(struct binder_proc *proc,binder_uintptr_t ptr)
{
    struct rb_node *n = proc->nodes.rb_node;
    struct binder_node *node;
    assert_spin_locked(&proc->inner_lock);
    while (n) {
        node = rb_entry(n, struct binder_node, rb_node);

        if (ptr < node->ptr)
            n = n->rb_left;
        else if (ptr > node->ptr)
           n = n->rb_right;
        else {
           binder_inc_node_tmpref_ilocked(node);
           return node;
        }
    }
    return NULL;
}
当前的proc为调用addService的进程也即是service的进程,首先从当前进程中的binder_proce中查找，如果查找不到,就会创建当前Binder对应的binder_node，
然后保存到当前进程中的binder_proc的node节点下，也就是保存到了当前服务进程中
```
> 现在，由于要把这个Binder实体MediaPlayerService交给target_proc，也就是Service Manager来管理，也就是说Service Manager要引用这个MediaPlayerService了，于是通过 binder_inc_ref_for_node 为MediaPlayerService创建一个引用，并且通过binder_inc_ref来增加这个引用计数，防止这个引用还在使用过程当中就被销毁。注意，到了这里的时候，t->buffer中的flat_binder_obj的type已经改为BINDER_TYPE_HANDLE，handle已经改为ref->desc，跟原来不一样了，因为这个flat_binder_obj是最终是要传给Service Manager的，而Service Manager只能够通过句柄值来引用这个Binder实体。

```java
ret = binder_inc_ref_for_node(target_proc, node,fp->hdr.type == BINDER_TYPE_BINDER,&thread->todo, &rdata);

static int binder_inc_ref_for_node(struct binder_proc *proc,
			struct binder_node *node,
			bool strong,
			struct list_head *target_list,
			struct binder_ref_data *rdata)
{
	struct binder_ref *ref;
	struct binder_ref *new_ref = NULL;
	int ret = 0;

	binder_proc_lock(proc);
	ref = binder_get_ref_for_node_olocked(proc, node, NULL);
	if (!ref) {
		binder_proc_unlock(proc);
		new_ref = kzalloc(sizeof(*ref), GFP_KERNEL);
		if (!new_ref)
			return -ENOMEM;
		binder_proc_lock(proc);
		ref = binder_get_ref_for_node_olocked(proc, node, new_ref);
	}
	ret = binder_inc_ref_olocked(ref, strong, target_list);
	*rdata = ref->data;
	binder_proc_unlock(proc);
	if (new_ref && ref != new_ref)
		/*
		 * Another thread created the ref first so
		 * free the one we allocated
		 */
		kfree(new_ref);
	return ret;
}
static struct binder_ref *binder_get_ref_for_node_olocked(
					struct binder_proc *proc,
					struct binder_node *node,
					struct binder_ref *new_ref)
{
	struct binder_context *context = proc->context;
	struct rb_node **p = &proc->refs_by_node.rb_node;
	struct rb_node *parent = NULL;
	struct binder_ref *ref;
	struct rb_node *n;

	while (*p) {
		parent = *p;
		ref = rb_entry(parent, struct binder_ref, rb_node_node);

		if (node < ref->node)
			p = &(*p)->rb_left;
		else if (node > ref->node)
			p = &(*p)->rb_right;
		else
			return ref;
	}
	if (!new_ref)
		return NULL;

	binder_stats_created(BINDER_STAT_REF);
	new_ref->data.debug_id = atomic_inc_return(&binder_last_id);
	new_ref->proc = proc;
	new_ref->node = node;
	rb_link_node(&new_ref->rb_node_node, parent, p);
	rb_insert_color(&new_ref->rb_node_node, &proc->refs_by_node);
	...
}
```
上面的过程做的就是创建一个 binder_ref 对象，这个代表的是当前引用的binder_node节点,然后保存到 target_proc中的 binder_ref 节点下面，本身是一颗红黑树结构，代表当前进程所引用的binder实体,下面是 binder_ref 的结构体定义,注意 binder_ref 中的 proc 赋值为当前所拥有它的binder_proce，在这里也就是service_manager对象,而node为binder的实体对象
```java
struct binder_ref {
	/* Lookups needed: */
	/*   node + proc => ref (transaction) */
	/*   desc + proc => ref (transaction, inc/dec ref) */
	/*   node => refs + procs (proc exit) */
	struct binder_ref_data data;
	struct rb_node rb_node_desc;
	struct rb_node rb_node_node;
	struct hlist_node node_entry;
	struct binder_proc *proc;
	struct binder_node *node;
	struct binder_ref_death *death;
};

由于binder_node 中又有 标识当前这个实体属于哪个binder_proce的字段,所以后续就算拿到的是当前新产生的binder引用，也可以找到实体对象
struct binder_node {
	int debug_id;
	spinlock_t lock;
	struct binder_work work;
	union {
		struct rb_node rb_node;
		struct hlist_node dead_node;
	};
	struct binder_proc *proc;
	...
}
下面看看是怎么样找的，现在假设是客户端要获取到服务端的binder引用 对应的类型就为 BINDER_TYPE_HANDLE，对应的处理函数就为  binder_translate_handle，这是由service_manager来触发的

static int binder_translate_handle(struct flat_binder_object *fp, struct binder_transaction *t, struct binder_thread *thread)
{
	struct binder_proc *proc = thread->proc;
	struct binder_proc *target_proc = t->to_proc;
	struct binder_node *node;
	struct binder_ref_data src_rdata;
	int ret = 0;

	//首先获取到引用对象
	node = binder_get_node_from_ref(proc, fp->handle,fp->hdr.type == BINDER_TYPE_HANDLE, &src_rdata);
	if (!node) {
		binder_user_error("%d:%d got transaction with invalid handle, %d\n", proc->pid, thread->pid, fp->handle);
		return -EINVAL;
	}
	if (security_binder_transfer_binder(proc->tsk, target_proc->tsk)) {
		ret = -EPERM;
		goto done;
	}

	binder_node_lock(node);
	if (node->proc == target_proc) {
		if (fp->hdr.type == BINDER_TYPE_HANDLE)
			fp->hdr.type = BINDER_TYPE_BINDER;
		else
			fp->hdr.type = BINDER_TYPE_WEAK_BINDER;
		fp->binder = node->ptr;
		fp->cookie = node->cookie;
		if (node->proc)
			binder_inner_proc_lock(node->proc);
		else
			__acquire(&node->proc->inner_lock);
		binder_inc_node_nilocked(node,
					 fp->hdr.type == BINDER_TYPE_BINDER,
					 0, NULL);
		if (node->proc)
			binder_inner_proc_unlock(node->proc);
		else
			__release(&node->proc->inner_lock);
		trace_binder_transaction_ref_to_node(t, node, &src_rdata);
		binder_debug(BINDER_DEBUG_TRANSACTION,
			     "        ref %d desc %d -> node %d u%016llx\n",
			     src_rdata.debug_id, src_rdata.desc, node->debug_id,
			     (u64)node->ptr);
		binder_node_unlock(node);
	} else {
		struct binder_ref_data dest_rdata;
		//由于不是实体进程，所以这里再创建一个引用
		binder_node_unlock(node);
		ret = binder_inc_ref_for_node(target_proc, node,fp->hdr.type == BINDER_TYPE_HANDLE,NULL, &dest_rdata);
		if (ret)
			goto done;

		fp->binder = 0;
		fp->handle = dest_rdata.desc;
		fp->cookie = 0;
		trace_binder_transaction_ref_to_ref(t, node, &src_rdata,
						    &dest_rdata);
		binder_debug(BINDER_DEBUG_TRANSACTION,
			     "        ref %d desc %d -> ref %d desc %d (node %d)\n",
			     src_rdata.debug_id, src_rdata.desc,
			     dest_rdata.debug_id, dest_rdata.desc,
			     node->debug_id);
	}
done:
	binder_put_node(node);
	return ret;
}

//从binder_proce中的 binder_ref 节点中查找对应的引用对象
static struct binder_ref *binder_get_ref_olocked(struct binder_proc *proc, u32 desc, bool need_strong_ref)
{
	struct rb_node *n = proc->refs_by_desc.rb_node;
	struct binder_ref *ref;

	while (n) {
		ref = rb_entry(n, struct binder_ref, rb_node_desc);

		if (desc < ref->data.desc) {
			n = n->rb_left;
		} else if (desc > ref->data.desc) {
			n = n->rb_right;
		} else if (need_strong_ref && !ref->data.strong) {
			binder_user_error("tried to use weak ref as strong ref\n");
			return NULL;
		} else {
			return ref;
		}
	}
	return NULL;
}

static struct binder_node *binder_get_node_from_ref(struct binder_proc *proc,u32 desc, bool need_strong_ref,struct binder_ref_data *rdata)
{
	struct binder_node *node;
	struct binder_ref *ref;

	binder_proc_lock(proc);
	ref = binder_get_ref_olocked(proc, desc, need_strong_ref);
	if (!ref)
		goto err_no_ref;
	node = ref->node;
	/*
	 * Take an implicit reference on the node to ensure
	 * it stays alive until the call to binder_put_node()
	 */
	binder_inc_node_tmpref(node);
	if (rdata)
		*rdata = ref->data;
	binder_proc_unlock(proc);

	return node;

err_no_ref:
	binder_proc_unlock(proc);
	return NULL;
}

```
首先从当前的binder_proc中获取到 fp->handle 对应的引用节点，由于我们在addService的时候，有将节点保存到目标 binder_proce中的binder_ref节点，所以这里一定能找到，因为当前的proce,也就是service_manager对应的binder_proce,查找的函数上面可见 node = binder_get_node_from_ref(proc, fp->handle,fp->hdr.type == BINDER_TYPE_HANDLE, &src_rdata);

这里要注意 node = ref->node; 也即是得到的是引用节点里面的实体binder节点,然后判断 if (node->proc == target_proc)也即是判断当前的target_proc是否就为当前binder实体所拥有的对象,由于当前是客户端进程，所以不满足 会执行 ret = binder_inc_ref_for_node(target_proc, node,fp->hdr.type == BINDER_TYPE_HANDLE,NULL, &dest_rdata);前面已经分析过了，这里会再创建一个引用，同时将 fp->handle = dest_rdata.desc; 所以客户端得到的是另外一个引用

### 客户端拿到引用之后的调用过程
```java
if (tr->target.handle) {
	struct binder_ref *ref;
	binder_proc_lock(proc);
	//首先获取到引用
	ref = binder_get_ref_olocked(proc, tr->target.handle, true);
	if (ref) {
		target_node = binder_get_node_refs_for_txn(ref->node, &target_proc,&return_error);
	} else {
		binder_user_error("%d:%d got transaction to invalid handle\n", proc->pid, thread->pid);
		return_error = BR_FAILED_REPLY;
	}
		binder_proc_unlock(proc);
} else {
	mutex_lock(&context->context_mgr_node_lock);
	target_node = context->binder_context_mgr_node;
	if (target_node)
		target_node = binder_get_node_refs_for_txn(target_node, &target_proc,&return_error);
	else
		return_error = BR_DEAD_REPLY;
		mutex_unlock(&context->context_mgr_node_lock);
		if (target_node && target_proc->pid == proc->pid) {
				binder_user_error("%d:%d got transaction to context manager from process owning it\n",
						  proc->pid, thread->pid);
				return_error = BR_FAILED_REPLY;
				return_error_param = -EINVAL;
				return_error_line = __LINE__;
				goto err_invalid_target_handle;
			}
		}
...
}
由于客户端传递过来的handle引用时不为空，这个引用就是服务端为客户端创建的另一个引用，而且这个引用时保存在了客户端的binder_proce中的，所以首先调用binder_get_ref_olocked获取到引用

static struct binder_ref *binder_get_ref_olocked(struct binder_proc *proc, u32 desc, bool need_strong_ref)
{
	struct rb_node *n = proc->refs_by_desc.rb_node;
	struct binder_ref *ref;

	while (n) {
		ref = rb_entry(n, struct binder_ref, rb_node_desc);
		if (desc < ref->data.desc) {
			n = n->rb_left;
		} else if (desc > ref->data.desc) {
			n = n->rb_right;
		} else if (need_strong_ref && !ref->data.strong) {
			binder_user_error("tried to use weak ref as strong ref\n");
			return NULL;
		} else {
			return ref;
		}
	}
	return NULL;
}
获取到之后，获取到这个引用包含的真实的Bidner引用 target_node = binder_get_node_refs_for_txn(ref->node, &target_proc,&return_error);

//这里ref->node 就为真实的服务端Binder_node实体对象了
static struct binder_node *binder_get_node_refs_for_txn(struct binder_node *node,struct binder_proc **procp,uint32_t *error)
{
	struct binder_node *target_node = NULL;
	binder_node_inner_lock(node);
	if (node->proc) {
		target_node = node;
		binder_inc_node_nilocked(node, 1, 0, NULL);
		binder_inc_node_tmpref_ilocked(node);
		node->proc->tmp_ref++;
		*procp = node->proc;
	} else
		*error = BR_DEAD_REPLY;
	binder_node_inner_unlock(node);
	
	return target_node;
}

```
这样就为procp 赋值了，也就是为target_proc 赋值了，也就是当前实体所拥有的真正的binder_proc对象，后续的操作就是类似的添加事物到todo队列，激活真实的service，为了得到服务端一个回应内核也会将事物添加到目标线程的 transaction_stack的头部，这样服务端回应的时候，就可以从transaction_stack 中取出这个事物，根据这个事物的from字段的值，得到对应的target_proc，也就可以创建事物t添加到target_proc中的todo队列，同时唤醒客户端线程，这里也为了能得到一个客户端响应，在唤醒的线程中，内核会将todo队列的事物移除，同时将这个事物添加到 transaction_stack的头部客户端得到结果之后，也会根据transaction_stack中的from给对方一个回应的确定

### 客户端调用是阻塞调用
客户端的一般调用流程
```java
public IBinder getService(String name) throws RemoteException {
	Parcel data = Parcel.obtain();
	Parcel reply = Parcel.obtain();
	data.writeInterfaceToken(IServiceManager.descriptor);
	data.writeString(name);
	mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
	IBinder binder = reply.readStrongBinder();
	reply.recycle();
	data.recycle();
	return binder;
}
最终都会调用到 mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0); 也即是 BpBinder中的transact 函数，BpBinder中的transact函数会有调用到 IPCThreadState::transact函数
status_t IPCThreadState::transact(int32_t handle,uint32_t code, const Parcel& data,Parcel* reply, uint32_t flags){
	...
	err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
	...
	err = waitForResponse(reply);
}
writeTransactionData 是准备数据，将数据转成 binder_transaction_data类型，之后调用 waitForResponse 执行数据的发送，以及响应结果

status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        IF_LOG_COMMANDS() {
            alog << "Processing waitForResponse Command: "
                << getReturnString(cmd) << endl;
        }

        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;

        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;

        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;

        case BR_ACQUIRE_RESULT:
            {
                ALOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");
                const int32_t result = mIn.readInt32();
                if (!acquireResult) continue;
                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
            }
            goto finish;

        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(NULL,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t), this);
                    }
                } else {
                    freeBuffer(NULL,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t), this);
                    continue;
                }
            }
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}
```
而内部的 talkWithDriver() 函数 会有执行  ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)根据前面的分析可以知道，这个函数是会阻塞线程的，所以在没得到对方的回应之前，就会挂起线程

### 服务端的调用是在Binder线程中的
```java
ProcessState::self()->startThreadPool();//看名字，启动Process的线程池？
IPCThreadState::self()->joinThreadPool();//将自己加入到刚才的线程池？

//线程启动的时候，最终会调用到这个方法，处理结果
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());

    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
            abort();
        }

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%d\n",
        (void*)pthread_self(), getpid(), result);

    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}

status_t IPCThreadState::getAndExecuteCommand()
{
   status_t result;
   int32_t cmd;

   result = talkWithDriver();
   ...
   //处理结果
   result = executeCommand(cmd);
   ...
   return result;
}
```
可以看到内部又是通过调用 talkWithDriver 得到结果的，结果的处理是executeCommand来执行分发的，内部就会调用服务端的方法了，这里就不看了

### Android的服务是在哪里调用这俩句的
```java
也就是Android进程启动的时候，也即是 ActivityManagerService中的startProcessLocked方法，内部会完成进程的启动
class AppRuntime : public AndroidRuntime
{
	......
 
	virtual void onZygoteInit()
	{
		sp<ProcessState> proc = ProcessState::self();
		if (proc->supportsProcesses()) {
			LOGV("App process: starting thread pool.\n");
			proc->startThreadPool();
		}
	}
	......
}
```
可以看到Android进程在启动的时候，就已经添加了 proc->startThreadPool(); 所以开启了线程来处理当前进程的service请求

### 总结
总体而言Binder机制还是很复杂的，关于Binder在Android上层的时候，看老罗的文章就已经非常详细了

### 参考链接
1. [Android进程间通信（IPC）机制Binder简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/6618363)
2. [Android应用程序进程启动过程的源代码分析](https://blog.csdn.net/luoshengyang/article/details/6747696)