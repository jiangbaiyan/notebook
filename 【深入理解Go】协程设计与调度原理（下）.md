## 回顾
在上一篇文章中，我们讲述了基本的调度流程。但是我们没有解决如果协程内部如果存在阻塞的情况下该如何处理。比如某个G中存在对channel的收发等操作会发生阻塞，那么这个协程就不能一直占用M的资源，如果一直占用可能就会导致所有M都被阻塞住了。所以我们需要把当前G暂时挂起，待阻塞返回之后重新调度这个G来运行。

接导语，我们需要一种调度机制，及时释放阻塞G占用的资源，重新触发一次调度器的调度逻辑，把当前G先挂起，让其他未执行过的G来执行，从而实现资源利用率的最大化。

## runtime可以拦截的阻塞
什么是runtime可以拦截的？一般是我们在代码中的阻塞，大概有这几种：
 - channel生产/消费阻塞 
 - select
 - lock
 - time.sleep
 - 网络读写

在runtime可以拦截的情况下，会先让G进某种数据结构，待ready后重新调度G来继续执行，阻塞期间不会继续持有线程M。接下来我们以第一种情况channel为例，看看这个流程具体是如何执行的。
### 以channel阻塞为例
如刚才channel的那个例子所述，由于阻塞了，所以这个G需要被动的让出所持有的P和M。我们以channel这个例子过一遍这个流程。假设有这么一行代码：
```go
ch <- 1
```
这是一个无缓冲的通道，此时往通道里写入了一个值1。假设此时消费端还没有去消费，这个时候这个通道写入的操作就会阻塞。channel的数据结构叫做hchan：
![](https://baiyan-1300428464.cos.ap-beijing.myqcloud.com/article/2021/9/21/1632193176178.png)

```go
type hchan struct {
    // 通道里元素的数量
    qcount   uint
    // 循环队列的长度
    dataqsiz uint
    // 指针，指向存储缓冲通道数据的循环队列
    buf      unsafe.Pointer
    // 通道中元素的大小
    elemsize uint16
    // 通道是否关闭的标志
    closed   uint32
    // 通道中元素的类型
    elemtype *_type
    // 已接收元素在循环队列的索引
    sendx    uint  
    // 已发送元素在循环队列的索引
    recvx    uint
    // 等待接收的协程队列
    recvq    waitq
    // 等待发送的协程队列
    sendq    waitq
    // 互斥锁，保护hchan的并发读写，下文会讲
    lock mutex
}
```
这里我们重点关注recvq和sendq这两个字段。他们是一个链表，存储阻塞在这个channel的发送端和接收端的G。以上ch <- 1其实底层实现是一个chansend函数，实现如下：
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
   ...
	// 尝试从recvq，也就是接收方队列中出队一个元素，如果非空，则说明找到了一个正在等待的receiver，可以发送数据过去了。发送完数据直接return
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 代码走到这里，说明没有接收方，需要阻塞住等待接收方（如果是无缓冲channel的话）
	if !block {
		unlock(&c.lock)
		return false
	}
	
	// 把当前channel和G，打包生成一个sudog结构，后面会讲为什么这样做
	gp := getg()
	mysg := acquireSudog()
	mysg.g = gp
	mysg.c = c	
	... 
	// 将sudog放到sendq中
	c.sendq.enqueue(mysg)
	
	// 调用gopark，这里内部实现会讲M与G解绑，并触发一次调度循环
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
    
	return true
}
```
我们总结一下这个chansend的流程（以无缓冲通道为例）：
 - 尝试从recvq中获取消费者
 - 若recvq不空，发送数据；若为空，则需要阻塞
 - 获取一个sudog结构，给g字段赋值为当前G
 - 把sudog挂到sendq上等待唤醒
 - 调用gopark将M与G解绑，重新触发一次调度，M去执行其他的G

### 为什么是sudog而非G
那么这里为什么要用sudog而非原始的G结构呢。答案在于，一个G可以在多个等待链表上。recvq和sendq都是一个waitq结构。是一个双向链表。假如第一个G已经挂到了链表上，那么他必然要存储下一个G的地址，才能成功的完成双向链表的逻辑，如：
```go
type g struct {
	next *g
	prev *g
}
```
而g又可能挂在多个等待链表上（如select操作，一个G可能会阻塞在多个channel上），所以g里的next和prev必然会有多个值的情况。即next和prev的地址在多个等待链表上的值可能是不一样的。G和等待链表的关系是多对多的关系，所以这个prev和next必然不能在G上直接维护，所以我们就会将G和channel一起打包成sudog结构。它和我们MySQL中多对多的中间表设计有异曲同工之妙，相当于维护了一个g_id和channel_id：
```go
type sudog struct {

	// 原始G结构。相当于g_id
	g *g

	// 等待链表上的指针
	next *sudog
	prev *sudog

	// 所属的channel，相当于channel_id
	c    *hchan
}
```

最终的效果如下：
![](https://baiyan-1300428464.cos.ap-beijing.myqcloud.com/article/2021/9/21/1632196329587.png)

### gopark
我们知道，在将sudog打包好放到sendq之后，会调用go_park执行阻塞逻辑。go_park内部又会调用park_m方法，切换到g0栈，解除M与当前G的绑定，重新触发一次调度，让M去绑定其他G执行：
```go
// park continuation on g0.
func park_m(gp *g) {
	_g_ := getg()

	// 将G的状态设置为waiting
	casgstatus(gp, _Grunning, _Gwaiting)
	
	// 解除M与G的绑定
	dropg()

	// 重新执行一次调度循环
	schedule()
}
```

### 什么时候唤醒
那么问题来了，当前G已经阻塞在sendq上了，那么谁来唤醒这个G让他继续执行呢？显然是channel的接收端，在源码中和chansend相对的操作即chanrecv：
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {

	...
	// 尝试从sendq中拿一个等待协程出来
	if sg := c.sendq.dequeue(); sg != nil {
		// 如果拿到了，那么接收数据，刚才我们的channel就属于这种情况
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 同上，打包成一个sudog结构，挂到recvq等待链表上
	gp := getg()
	mysg := acquireSudog()
	mysg.g = gp
	mysg.c = c
	c.recvq.enqueue(mysg)

	// 同理，拿不到sendq，调用gopark阻塞
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
	...
	
	return true, success
}
```

### goready
我们看到，chanrecv和chansend逻辑大体一致，这里就不详细展开。由于刚才我们的sendq上有数据，那么这里一定会进入recv()方法接收数据。在这里会调用goready()方法：
```go
// Mark gp ready to run.
func ready(gp *g, traceskip int, next bool) {

	...
	status := readgstatus(gp)

	// 标记G为grunnable状态
	_g_ := getg()
	casgstatus(gp, _Gwaiting, _Grunnable)
	
	// 放入runq中等待调度循环消费
	runqput(_g_.m.p.ptr(), gp, next)
	
	// 唤醒一个闲置的P来执行G
	wakep()
	releasem(mp)
}
```
goready和gopark是一对操作，gopark是阻塞，goready是唤醒。它会将sudog中绑定的G拿出来，传入ready()方法，把它从gwaiting置为grunnable的状态。并再次执行runqput。将G放到P的本地队列/全局队列上等待调度循环来消费。这样整个流程就能跑起来了。

总结一下，
 - sender调用gopark挂起，一定是由receiver（或close）通过goready唤醒
 - receiver调用gopark挂起，一定是由sender（或close）通过goready唤醒

## runtime不能拦截的阻塞
什么是runtime不能拦截的？即CGO代码和系统调用。CGO这里先不讲，由于系统调用这里也有可能发生阻塞，且不属于runtime层面的阻塞，runtime也不能让G进某一个相关数据结构，runtime无法捕获到。

那么这个时候就需要一个后台监控的特殊线程sysmon来监控这种情况。它会定期循环不断的执行。它会申请一个单独的M，且不需要绑定P就可以执行，优先级最高。

sysmon的核心是sysmon()方法。监控会在循环中调用retake()方法抢占处于长时间阻塞中的P，该函数会遍历运行时的所有P。retake()的实现如下：
```go
func retake(now int64) uint32 {
	n := 0
	for i := 0; i < len(allp); i++ {
		_p_ := allp[i]
		pd := &_p_.sysmontick
		s := _p_.status
		//当处理器处于_Prunning或者_Psyscall状态时，如果上一次触发调度的时间已经过去了10ms，我们会调用preemptone()抢占当前P
		if s == _Prunning || s == _Psyscall {
			t := int64(_p_.schedtick)
			if pd.schedwhen+forcePreemptNS <= now {
				preemptone(_p_)
			}
		}
		// 当处理器处系统调用阻塞状态时，当处理器的运行队列不为空或者不存在空闲P时，或者当系统调用时间超过了10ms，会调用handoffp将P从M上剥离
		if s == _Psyscall {
			if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}
			if atomic.Cas(&_p_.status, s, _Pidle) {
				n++
				_p_.syscalltick++
				handoffp(_p_)
			}
		}
	}
	return uint32(n)
}
```

sysmon通过在后台监控循环中抢占P，来避免同一个G占用M太长时间造成长时间阻塞及饥饿问题。