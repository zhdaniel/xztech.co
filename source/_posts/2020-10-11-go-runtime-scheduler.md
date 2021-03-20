---
title: Go Runtime 调度器分析
tags:
  - go
  - runtime
  - goroutine
date: 2020-10-11 21:22:02
---

在 Go 语言中可以很方便的用协程来实现并发，但是协程是如何高效运行的呢？本文从设计的角度出发一探 Go Runtime 调度器的实现方式。所有了不起的工程都是源自需求驱动的，所以为了理解 **为什么需要协程** 和 **它是如何工作的** ，让我们回顾操作系统的历史以便我们找到问题，如果不能理解问题的根源那么可能无法很好的解决它。


### 操作系统历史


### 协程
协程是由 **Go runtime** 托管的轻量级的线程，开启一个新的协程只需要在函数调用的地方增加 **`go`** 关键字即可。下面看一个协程的例子：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(10)

	for i := 0; i < 10; i++ {
		go func(i int) {
			defer wg.Done()
			fmt.Printf("loop i is - %d\n", i)
		}(i)
	}

	wg.Wait()
	fmt.Println("Hello, Welcome to Go")
}
```

该例子可能的输出结果如下：

```
loop i is - 9
loop i is - 4
loop i is - 0
loop i is - 1
loop i is - 2
loop i is - 3
loop i is - 7
loop i is - 8
loop i is - 6
loop i is - 5
Hello, Welcome to Go
```

那么此时我们脑海便有了两个疑问：

1. **这 10 个协程是如何并行运行？**
2. **这 10 个协程以什么顺序运行？**

随之而来我们解决的两个问题就是：

1. **如何在运行于多核 CPU 的多个内核系统线程上分发多个协程？**（即：多协程 -> 多内核线程 -> 多CPU核 如何调度？）
2. **多个协程以什么顺序运行以保证公平？**


剩下的讨论将主要围绕从设计的角度解决 Go runtime 调度器特有的这些问题。和其他所有问题一样，我们需要定义好边界，否则问题陈述太含糊就无法得出结论。调度可能针对多个目标中的一个或多个问题，对于我们而言，我们将自己限定在以下要求之内：

1. 并行且可扩展，并保证调度公平性
2. 每个进程可以扩展出数百万个协程
3. 内存高效
4. 系统调用不应导致性能下降（最大化吞吐量，最小化等待时间）


### 调度器

因此，让我们为调度器建模，以逐步解决这些问题。

#### 1. 1:1 - 用户线程
即一个协程（用户线程）对应和一个内核线程，这个模型虽然实现了并行，但是扩展性却不够，一个进程根本无法扩展到百万级线程。

#### 2. M:N 线程模型

{% asset_img "m-n.png" "M:N模型" %}

实际执行代码和并行化需要一个内核线程来实现，但是创建内核线程代价是昂贵的。所以我们将N个协程映射到M个内核线程。Go 协程是 Go 代码，所以我们对它有全部控制权，而且运行在用户态所以创建代价也很低。

但是操作系统并不知道协程，所以**每个协程需要有一个状态以便调度器根据协程当前状态决策如何调度**。协程状态信息相比内核线程状态就简单很多，所以协程的上下文切换非常快。

- **Running** - 协程正由内核线程执行中
- **Runnable** - 协程等待内核线程执行
- **Blocked** - 协程正在等待某些条件（如：阻塞在某个 Channel、系统调用、互斥锁等）

{% asset_img "state.png" "协程状态" %}

因此，Go Runtime 调度区通过协程的状态来将 N 个协程复用到M内核线程来管理他们。

#### 2.1 简单 M:N 调度器

在我们的简单的 M:N 调度器中我们有个全局运行队列，一些操作会将一个新的协程放入运行队列中，M 个内核线程访问调度器从运行队列中取一个线程去执行，**因为多个线程会读写同一块内存区域，所以我们需要对这块内存加锁**。

{% asset_img "simple_m-n.png" "M:N 调度器" %}

此时的一个问题就是：**如果协程阻塞了怎么办？**，下面是会导致协程阻塞的操作：

1. 读/写 Channel
2. 网络 I/O
3. 阻塞的系统调用
4. 定时器
5. 互斥锁

所以我们应该把阻塞的协程放在哪里？阻塞的协程放在哪里的设计决策围绕的一个基本原则就是：

> ***阻塞的协程不应该阻塞底层的内核线程（避免协程切换的开销）***



+ ##### 阻塞在 Channel 上的协程

每个 Channel 有一个 `recvq(waitq)`  和一个 `sendq (waitq)` 分别用于存放读取和发送该 Channel 上阻塞了的协程。

{% asset_img "goroutine_blocked_channel.png" "阻塞在Channel上的协程" %}

待 Channel 操作结束后，Channel 自己会将为阻塞的协程放回运行队列中。

{% asset_img "goroutine_unblocked_channel.png" "Channel放回未阻塞的协程" %}



+ ##### 阻塞在系统调用的协程

首先，让我们回顾下系统调用，一个系统会阻塞底层的内核线程，所以我们不能再在该线程上调度其他协程，因为系统会降低并行级别。

{% asset_img "goroutine_blocked_syscall.png" "阻塞在系统调用上的协程" %}

因为 M2 线程阻塞在系统调用上不能调用其他协程，这就导致 CPU 浪费，我们还有其他任务要处理，然而却不能运行。恢复并行级别的方式就是在进入系统调用的时候唤醒另外一个线程，并从运行队列中取一个可以运行的协程运行。

{% asset_img "restore_parallelism_level.png" "恢复运行级别" %}

但是当系统调用完成后，我们的调度已经超额了。 为了避免这种情况，从系统调用返回后我们不会立即继续运行协程，而是将其放放入调度器的运行队列中。

{% asset_img "avoiding_oversubscribed_scheduling.png" "避免过度调度" %}

{% blockquote %}
因此，在程序运行时线程数大于内核数。 尽管没有明确说明线程数大于内核数，并且所有空闲线程也由运行时管理，以避免过多的线程。最大线程数初始值是 10000，如果超过此值程序会崩溃。[SetMaxThreads](https://golang.org/pkg/runtime/debug/#SetMaxThreads) 用于设置最大线程数
{% endblockquote %}




+ ##### 非阻塞系统调用

调用非阻塞系统调用的协程存储在 **runtime poller** 上，并且内核线程继续运行其他协程。

{% asset_img "poll_desc.png" "非阻塞系统调用协程" %}

例如，在诸如HTTP调用非阻塞I/O的时候，

例如，在非阻塞I/O（例如HTTP调用）的情况下。 由于资源尚未准备就绪，遵循先前工作流程的第一个syscall将不会成功，这将迫使 Go 使用网络轮询器（**network poller**）并将协程暂存。

下面是 `net.Read` 函数的部分实现：

```go
n, err := syscall.Read(fd.Sysfd, p)
        if err != nil {
            n = 0
            if err == syscall.EAGAIN && fd.pd.pollable() {
                if err = fd.pd.waitRead(fd.isFile); err == nil {
                    continue
                }
            }
```

先前的系统调用结束后，会显示告知资源还未准备就绪，该协程会暂存直到网络轮询器通知资源已就绪，在这种情况下线程 M 不会阻塞。轮询器会根据不同的操作系统使用 `select` / `kqueue` / `epoll` / `IOCP` 来知道已经就绪的文件描述符，一旦文件描述符读/写就绪后，会尽快将协程放回运行队列。

> There is also a Sysmon OS thread that will periodically poll network if it’s not polled for more than 10ms and will add the ready G to the queue.

基本上所有协程都被阻塞在：

1. Channel
2. 互斥量
3. 网络 I/O
4. 计时器

都有特定的队列，用于辅助协程的调度。



现在，runtime 有一个具有以下功能的调度器：

- 可以处理并行执行（多线程实现）
- 处理阻塞系统调用和网络 I/O
- 处理阻塞用户级（Channel）调用

然而，它的不具有扩展性

{% asset_img "global_run_queue.png "全局运行队列" %}

正如上图所展示，我们有一个带有互斥锁的全局运行队列，这样会遇到如下的问题：

1. 缓存一致性保证的开销
2. 在创建，销毁和调度协程时进行激烈的锁争用



{% asset_img "scheduler_per_thread.png" "分布式调度器" %}

为了解决扩展性和全局锁竞争的问题我们使用**分布式的调度器**，我们可以看到的直接好处是，每个线程本地运行队列现在都没有互斥锁。 虽然全局队列仍然带有互斥锁，但是在特定情况下才会使用，它不会影响可扩展性。现在我们有多个运行队列：

1. 本地运行队列
2. 全局运行队列
3. Network Poller

那么我们应该从哪里选择下一个运行的协程呢？在 Go 语言中，轮询的顺序被如下定义：

1. 本地运行队列
2. 全局运行队列
3. Network Poller
4. Work Stealing

也就是，首先检查本地运行队列，如果为空则检查全局运行队列，然后检查 Network Poller，最后再执行 Work Stealing。我们已经看了概览了前三部分，接下来我们看下 Work Stealing。



##### Work Stealing

{% asset_img "work_stealing.png" "Work Stealing" %}

如果本地运行队列为空，则尝试从其他队列中偷一个任务过来。Work Stealing 解决了如果一个线程有太多任务而另外一个线程却处于空闲状态。Work Stealing 遵循如下顺序：

+ 先从全局队列中取任务
+ 然后从 Network Poller 中取任务
+ 最后才从其他线程本地队列中取任务



截止目前，Go Runtime 调度器具有以下功能：

- 可以处理并行执行（多线程实现）
- 处理阻塞系统调用和网络 I/O
- 处理阻塞用户级（Channel）调用
- 可扩展性

但是这并不高效。还记得上文当阻塞在系统调用的时候恢复并行级别的手段的吗？

{% asset_img "restore_parallelism_level.png" "Work Stealing" %}

这意味着在一个系统调用中我们有多个内核线程（可以是10个或1000个），这可能会增加内核数。最终会带来固定开销：

+ Work Stealing 不得不扫描所有内核线程和本地运行队列，然而他们大多数都是空的。
+ 垃圾回收，内存回收也会遇到同样的问题

为了解决这个问题，所以我们引入 **M:P:N** 模型



#### M:P:N 调度模型

引入逻辑处理器 **P**，它可以看作运行在一个内核线程的本地调度器。逻辑处理器个数总是固定的，一般情况默认等于物理CPU核数。然后，将本地运行队列（LRQ）放入固定数量的逻辑处理器（P）中。Go runtime 首先会根据物理CPU核创建相应数量的逻辑处理器 **P**。每个协程（G）将在被分配在逻辑CPU（P）的内核线程（M）上运行。

{% asset_img "m-p-n.png" "M:P:N 调度模型" %}

所以，现在我们在以下期间没有固定的开销：

+ Work Stealing 只需扫描固定数量的逻辑处理器（P）的本地运行队列。
+ 垃圾回收，内存分配器也获得相同的好处。

那么引入逻辑处理器之后，系统调用如何处理呢？无论系统调用是否阻塞，Go 通过将它们包装在运行时中来优化系统调用。

{% codeblock lang:x86asm mark:2,14 highlight:true %}
TEXT ·Syscall(SB),NOSPLIT,$0-56
	CALL	runtime·entersyscall(SB)
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok
	MOVQ	$-1, r1+32(FP)
	MOVQ	$0, r2+40(FP)
	NEGQ	AX
	MOVQ	AX, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET
ok:
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	MOVQ	$0, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET
{% endcodeblock %}

阻塞调用 `SYSCALL` 被封装在 **`runtime.entersyscall(SB)`** 和  **`runtime.exitsyscall(SB)`**  之间，从调用名字上看，在正式进入和离开系统调用之前都执行一些逻辑。当发出一个阻塞的系统调用时，封装的系统调用会自动把处理器 **P** 和内核线程 **M** 分开以便另一个内核线程来运行它。这样就让 Go runtime 在不增加运行的队列的前提下高效处理阻塞系统调用。

那么在阻塞系统调用退出又会发生什么呢？

+ Runtime 会尝试获取先前取消关联的逻辑处理器 **P** 并继续执行
+ Runtime 会尝试从空闲列表中获取一个逻辑处理器 **P** 来执行
+ Runtime 会将协程放入全局队列，并将与之关联的内核线程 **M** 放回空闲列表

##### 自旋线程

当系统调用返回后 M2 线程应该做什么呢？理论上，如果一个线程完成了它需要做的事情，它应该被操作系统销毁，然后其他进程中的线程可能会被CPU调度执行。这就是我们通常所说的操作系统中线程的“抢占式调度”。

考虑上面系统调用的情况。如果我们销毁 M2 线程，M3 线程将进入系统调用。此时，在操作系统创建并计划新内核线程之前无法处理可运行的协程。频繁的线程前抢占操作不仅增加了操作系统上的负载，而且对于性能要求较高的程序也是不可接受的。

因此，为了合理利用系统资源和防止频繁的线程抢占导致操作系统负载，我们不会销毁内核线程 M2，而是进行一次自旋操作并保存自己以便后续使用。尽管这似乎浪费了一些系统资源，但是在线程间频繁的抢占和频繁地创建、销毁操作这样的理想线程相比较，前者付出的代价更小。

例如，在有一个内核线程 M 和一个逻辑处理器 P 的 Go 程序中，如果内核线程 M 因执行系统调用被阻塞，则需要与逻辑处理器 P 数量相同的自旋线程，以允许等待可运行的协程继续执行。因此，在此期间，内核线程 M 的数量大于 P （一个 自旋线程 + 一个阻塞线程），所以即使 **`runtime.GOMAXPROCS`** 的值设为 1，程序也是处于多线程状态。



##### 公平调度

和其他调度器一样，Go 的协程调度也有公平行约束，可运行的协程最终会被运行。Go 的调度器有下面四个典型的公平性约定。任何运行超过 **10ms** 的协程都会被标记为可抢占（软限制）。但是，抢占只能在函数的序言部分完成，Go 当前（Go 1.10之前）使用的是由编译器在函数的序言部分插入抢占点来实现抢占。Go 1.10 之后实现了[非交互式抢占](https://github.com/golang/proposal/blob/master/design/24543-non-cooperative-preemption.md)

> functions create the stack frame during the beginning of the function (called the **function prologue**) and tear it down at the end of a function (called the **function epilogue**). 
>
> **函数序言（function prologue）**是函数在启动的时候运行的一系列指令。
>
> ``` x86asm
> push ebp
> mov ebp,esp
> sub esp,X
> ```
>
> 这些指令的功能是：在栈里保存 `EBP` 寄存器的内容、将 `ESP` 的值复制到 `EBP` 寄存器，然后修改栈的调试，以便为本函数的局部变量申请存储空间。在函数执行期间，`EBP` 寄存器不受函数运行的影响它是函数访问局部变量和函数参的基准值。虽然我们也可使用 `ESP` 寄存器存储局部变量和运行参数，但是 `ESP` 寄存器的值总是变化的，使用起来不方便。
>
> **函数尾声（function epilogue）** 是在退出时，要做启动过程的反操作，释放栈中的申请内存，还原 `EBP` 寄存器的值，将代码控制权还原给调用者函数
>
> ``` x86asm
> mov esp,ebp
> pop ebp
> ret 0
> ```

+ 死循环
+ 本地运行队列
+ 网络



