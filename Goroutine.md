## Goroutine是什么？
### Goroutine定义
[Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike)

Goroutine 是一个与其他 goroutines 并行运行在同一地址空间的 Go 函数或方法。一个运行的程序由一个或更多个 goroutine 组成。它与线程、协程、进程等不同。它是一个 goroutine 
### Goroutine和Thread区别
- 内存占用
  - 创建一个 goroutine 的栈内存消耗为 2 KB，运行过程中，如果栈空间不够用，会自动进行扩容，最大1GB
  - 创建一个 thread 的栈内存消耗为1 - 8 MB (POSIX Thread)，而且还需要一个被称为 “guard page” 的区域用于和其他 thread 的栈空间进行隔离。栈内存空间一旦创建和初始化完成之后其大小就不能再有变化，这决定了在某些特殊场景下系统线程栈还是有溢出的风险
- 创建/销毁
  - goroutine是用户态线程，由go runtime管理，创建和销毁的消耗非常小
  - thread创建和销毁都会有巨大消耗，是内核级交互
- 调度切换
  - goroutine 的切换工作在用户态，切换成本小，约为 200 ns，一个纳秒平均可以执行 12-18 条指令，所以由于线程切换，执行指令的条数会减少 2400-3600
  - thread切换会消耗 1000-1500 纳秒，执行指令的条数会减少 12000-18000。
- 复杂性
  - thread创建和退出复杂，多个thread间通过share memory通讯，不能创建大量thread，编程复杂度高
  - goroutine使用CSP(communicating sequential processes)并发模型，goroutine间通过channel通讯，编程非常
## Goroutine调度模型
### GM调度器
![1 1](https://user-images.githubusercontent.com/7836739/112926984-d6a7ee00-9146-11eb-873b-b6d6fcd6d479.png)
- Go1.2前
- [存在的问题](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit)
  - Single global mutex and centralized state
    - 所有goroutine的创建，结束，重新调度等都要上全局锁
  - Goroutine hand-off
    - M之间频繁切换可运行的Goroutine，导致调度延迟和额外开销
  - Per-M memory cache
    - 每个M持有mcache(2MB)，然而只有M运行Go代码时才需要这个cache，M处于syscall时不需要，造成过多无用的资源浪费
  - Aggressive thread blocking/unblocking
    - 系统调用时，线程M经常被阻塞或解除阻塞，增加了很多的开销。
### GMP调度器
![1 2](https://user-images.githubusercontent.com/7836739/112927169-25ee1e80-9147-11eb-8fbf-a27ae0df1624.png)
- P：执行Go代码的逻辑处理器，通过 runtime.GOMAXPROCS设置，默认为可用核数
  - G：Goroutine
  - M：OS线程，没P找P，有P找G
<img width="465" alt="1 3" src="https://user-images.githubusercontent.com/7836739/112927185-2e465980-9147-11eb-8f95-29739c0ffd49.png">

- local queue：每个M在当前关联P的local queue，global queue，其他P的local queue中找G，减少全局锁影响
## Goroutine调度算法
### work-stealing
![1 4](https://user-images.githubusercontent.com/7836739/112927372-75cce580-9147-11eb-9a32-8cc73486c706.png)
- 当一个 P 执行完一个G，先从本地队列中再获取一个G，如果本地队列中没有G了，会随机挑选另一个 P，从它的本地队列中窃取一半的 G。
- 解决全局队列饥饿问题：新建 G时P的本地队列放不下达到256个的时候会放半数 G到全局队列去，阻塞的系统调用返回时找不到空闲P也会放到全局队列
![1 5](https://user-images.githubusercontent.com/7836739/112927380-782f3f80-9147-11eb-9c4b-dedaaf4ceddf.png)
### syscall
<img width="926" alt="1 6" src="https://user-images.githubusercontent.com/7836739/112927389-7d8c8a00-9147-11eb-8dcb-2c83c4f1be7a.png">

- 调用 syscall 后会解绑 P，然后 M 和 G 进入阻塞，而 P 此时的状态就是 syscall，表明这个 P 的 G 正在 syscall 中，这时的 P 是不能被调度给别的 M 的。如果在短时间内阻塞的 M 就唤醒了，那么 M 会优先来重新获取这个 P，能获取到就继续绑回去，这样有利于数据的局部性。
- 系统监视器 (system monitor)，称为 sysmon，会定时扫描。在执行 syscall 时, 如果某个 P 的 G 执行超过一个 sysmon tick(10ms)，就会把他设为 idle，重新调度给需要的 M，强制解绑。
- syscall 结束后 M 先尝试获取同一个P，恢复执行G，如果获取不到，则尝试获取idle list中其他空闲P，恢复执行G，如果还找不到，把G放回global queue，把M放回idle list
- 使用了syscall，是无法限制M的数量的，有可能会带来pthread exhaust问题。GOMAXPROCS限制的是在同时执行Go代码的M的数量，无法限制阻塞在系统调用的M的数量。
### spining
- 相对线程阻塞，线程自旋就是循环执行一个逻辑(不停找G)。费CPU，但降低M上下文切换成本，减少调度延迟。没P找P，有P找G，都是M自旋方式。
- 改善GM的Aggressive thread blocking/unblocking问题
### network poller
<img width="1009" alt="1 7" src="https://user-images.githubusercontent.com/7836739/112927400-80877a80-9147-11eb-9245-b0d537e126e8.png">

- G 发起网络 I/O 操作不会导致 M 被阻塞(仅阻塞G)，从而不会导致大量 M 被创建出来。将异步 I/O 转换为阻塞 I/O 的部分称为 netpoller。打开或接受连接都被设置为非阻塞模式。如果你试图对其进行 I/O 操作，并且文件描述符数据还没有准备好，G 会进入 gopark 函数(G置为waiting状态，等待goready唤醒，在锁和channel中也会使用)，将当前正在执行的 G 状态保存起来，然后切换到新的堆栈上执行新的 G。
- M不停找G的schedule函数中，处于就绪状态的 fd 对应的 G 就会被调度回来。
### sysmon
监控M，无需绑定P，是一个死循环，每20us~10ms循环一次，循环完一次就 sleep 一会，为避免空转，每次循环没什么事需要做，sleep时间就加长一点
- 释放闲置超过5分钟的 span 物理内存；
- 如果超过2分钟没有垃圾回收，强制执行；
- 收回因 syscall 长时间阻塞的 P
- 将长时间未处理的 netpoll 添加到全局队列；
- 向长时间运行的 G 任务发出抢占调度
<img width="931" alt="1 8" src="https://user-images.githubusercontent.com/7836739/112927405-841b0180-9147-11eb-8ab3-51677a9f4f51.png">

### scheduler affinity
<img width="982" alt="1 9" src="https://user-images.githubusercontent.com/7836739/112927487-a6ad1a80-9147-11eb-96b1-3971e9ca5ed3.png">

- 在 chan 来回通信的 goroutine 会导致频繁的 blocks，即频繁地在本地队列中重新排队。然而，由于本地队列是 FIFO 实现，如果另一个 goroutine 占用线程，unblock goroutine 不能保证尽快运行。如图，goroutine #9 在 chan 被阻塞后恢复。但是，它必须等待#2、#5和#4之后才能运行。goroutine #5将阻塞其线程，从而延迟goroutine #9，并使其面临被另一个 P 窃取的风险。
<img width="983" alt="1 10" src="https://user-images.githubusercontent.com/7836739/112927494-a90f7480-9147-11eb-8e1f-8e8f662df76b.png">
- Go 1.5 在 P 中引入了 runnext 特殊的一个字段，可以高优先级执行 unblock G。如图，goroutine #9现在被标记为下一个可运行的。这种新的优先级排序允许 goroutine 在再次被阻塞之前快速运行。这一变化对运行中的标准库产生了总体上的积极影响，提高了一些包的性能。
## Goroutine生命周期
### Go runtime启动
<img width="1062" alt="1 11" src="https://user-images.githubusercontent.com/7836739/112928395-238cc400-9149-11eb-8b62-b6bfa6b98aa3.png">

整个程序始于一段汇编，而在随后的 runtime·rt0_go（也是汇编程序）中，会执行很多初始化工作。
- 绑定 m0 和 g0，m0就是程序的主线程，程序启动必然会拥有一个主线程，这个就是 m0。g0 负责调度，即 shedule() 函数。
- 创建 P，绑定 m0 和 p0，首先会创建 GOMAXPROCS 个 P ，存储在 sched 的 空闲链表(pidle)。
- 新建任务 g 到 p0 本地队列，m0 的 g0 会创建一个 指向 runtime.main() 的 g ，并放到 p0 的本地队列。
  - runtime.main(): 启动 sysmon 线程；启动 GC 协程；执行 init，即代码中的各种 init 函数；执行 main.main 函数	
### M创建时机
<img width="272" alt="1 12" src="https://user-images.githubusercontent.com/7836739/112928402-27204b00-9149-11eb-892c-eec425ce5bbf.png">

P0-M0-G1
- main函数执行print "hello"
- go func()创建新的goroutine
  - 检查到有空闲的P(P1->P2->P3)
  - 且当前所有M都没在spining状态(M0当前没有处于没P找P有P找G的状态，而是在执行go代码)，则唤醒或创建一个M(这里是创建M1)。
P1-M1-G2
- 不管G2是被放到P0还是P1的队列，G2的代码这时就有可能在M1执行print "world"
### Goroutine调度
- 每个M创建时，会执行一个特殊的G，即g0,负责管理和调度G
- G的切换十分轻便，只需要保存两个状态，所以g到g0或者g0到g的切换十分迅速，约8~10us
  - PC：记录goroutine停止运行前执行的指令
  - G的堆栈：保存局部变量
- g0调度确定下一个要运行的g，需要检查很多资源更耗时，约20~100us
### Goroutine回收
<img width="598" alt="1 13" src="https://user-images.githubusercontent.com/7836739/112928407-2982a500-9149-11eb-979f-d5f858a01474.png">

- G 很容易创建，栈很小以及快速的上下文切换。基于这些原因，开发人员非常喜欢并使用它们。然而，一个产生许多 shortlive 的 G 的程序将花费相当长的时间来创建和销毁它们。
- 每个 P 维护一个 freelist G，保持这个列表是本地的，这样做的好处是不使用任何锁来 push/get 一个空闲的 G。当 G 退出当前工作时，它将被 push 到这个空闲列表中。
