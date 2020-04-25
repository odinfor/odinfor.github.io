---
layout: post
title: "golang 调度模型"
subtitle: 'golang 调度模型分析'
author: "odin"
header-style: text
tags:
  - golang
---
### 进程、线程、协程
在聊调度器一定绕不开进程、线程、协程这些内容
##### 1.关于进程和线程：
多个线程可以属于同一个进程并共享内存空间。因为多线程不需要创建新的虚拟内存空间，所以它们也不需要内存管理单元处理上下文的切换，线程之间的通信也正是基于共享的内存进行的，与重量级的进程相比，线程显得比较轻量。但是线程在调度时也有比较大的开销，每个线程大致占用2M左右内存空间，在对线程进行切换时不光会消耗较多的内存恢复寄存器中的内容还需要向操作系统申请或者销毁对应的资源，每一次线程上下文的切换都需要消耗 ~1us 左右的时间。
##### 2.关于协程：
go的协程是基于用户态线程，创建协程的内存开销远远小于内核线程的开销，并且go的调度器对协程的上下文切换比内核线程的上下文切换减少了80%的额外开销。
Go 语言的调度器通过使用与 CPU 数量相等的线程减少线程频繁切换的内存开销，同时在每一个线程上执行额外开销更低的 Goroutine 来降低操作系统和硬件的负载。

### golang M:N模型
根据线程的调度实现，将线程分为内核线程和用户线程。其中，内核线程是由操作系统调度，而用户线程是由用户自己实现的调度器进行调度。根据用户线程和内核线程的关系分为下面几种线程模型：
* 1：1模型：将一个用户线程映射或绑定到一个内核线程上。
缺点：上下文切换慢，消耗性能。
* N：1模型：将多个用户线程映射或绑定到一个内核线程上。
优点：上下文切换快
缺点：无法使用多核的优势
* M：N模型：结合上面两个模型，每个核上绑定多个用户线程。在单个核内上下文切换快，也能利用多核优势执行。

### GM模型
![]({{site.baseurl}}/img/in-post/post-golang/model-gm.png)
在go早期采用的是GM模型，即Gorotuine与machine，线程从全局Goroutine队列中取Goroutine执行。
该模型，当遇到Goroutine阻塞时，G从新回到Global队列，下一次并不能保证G分配到同一个M，需要频繁的切换上下文。非常影响性能。

只有一个全局队列还带来了另外一个问题，因为从队列中获取 goroutine 必须要加锁，导致锁的争用非常频繁。尤其是在大量 goroutine 被调度的情况下，对性能的影响也会非常明显。
当然除了以上提到的之外还具有其他的缺陷，比如激进的线程阻塞/解阻塞。因为系统调用导致工作线程经常被阻塞和解阻塞，这增加了很多的负担等，不再过多阐述。

### GMP模型
![]({{site.baseurl}}/img/in-post/post-golang/model-pgm.png)
在go 1.1之后调度器使用了上面的实现方案。
* **G(goroutine)**
调度系统的最基本单位goroutine，存储了goroutine的执行strack信息、goroutine状态以及goroutine任务函数等信息。轻量级协程，栈初始2kb，调度不涉及系统调用。
go使用 `go`关键字声明协程，go的每个实例都有任务函数。
    ```golang
    // 以下go关键字创建了一个匿名函数goroutine
    go func() {
        fmt.Println("this is a go func")
    }
    ```
    
* **M(machine)**
M才是真正的执行计算机资源，可以认为就是系统线程。M在绑定有效的P后，进入调度循环，而且M并不保留G状态，这是G可以跨M调度的基础。
    
* **P(processor)**
逻辑processor，只是代表逻辑processor，并不执行任何代码，仅代表线程M的执行上下文。P拥有的各种G对象队列、链表、以及cache和G的状态是最重要的作用。可以这么说P的数量代表了golang的执行并发度。
对G来说，P相当于CPU核，G只有绑定到P才能被调度执行，对M来说，P提供了相关的执行环境(Context)，如内存分配状态(mcache)，任务队列(G)等。

该方案的好处在于：
* 新增的P角色代替了M的一部分功能，M 要执行 G 的时候，就被调度绑定一个 P，然后去执行 G，这样对内存的占用就大大减少。
* 每个P都有自己的goroutine队列，新的G放到P的G队列中，队列满了才会放到global queue。这样设计大大减少了锁的竞争，并且保证尽量少的在多个核心上传递G。
* 协程同步阻塞或者其他阻塞，仅仅是切换协程而不阻塞线程。当 M 因为执行系统调用阻塞或 cgo 运行一段时间后，sysmon 协程会将 P 和 M分离，由其他的 M 来结合 P 进行调度。这样 M 就不用因为阻塞而占用不必要的资源。
* 当 G 执行网络操作和锁切换时，G 和 M 分离，M 通过调度执行新的 G。这样就可以保证用户在 G 中执行网络操作时不用考虑阻塞线程的问题。

简单用图举例一下几个调度场景
调度P：M0执行过程中G0阻塞，那么会将M0的P调度给M1执行，M0等待阻塞G0执行
![]({{site.baseurl}}/img/in-post/post-golang/diaodu-p.png)
抢占goroutine：多个M中其中一个M执行完成，可以从另外一个M的P队列中拿取等待执行的G到自己的P的G队列中
![]({{site.baseurl}}/img/in-post/post-golang/diaodu-p-1.png)
这是一个详细的GPM模型流程图
![]({{site.baseurl}}/img/in-post/post-golang/all-gpm.png)
goroutine的状态流转
![]({{site.baseurl}}/img/in-post/post-golang/goroutine-change.png)

Gidle：刚刚被分配，还没有初始化
Grunnable：表示在runqueue上，还没有被运行
Grunning：go协程可能在执行go代码，不再runqueue上，与M，P已绑定
Gsyscall：go协程在执行系统调用，没有执行go代码，没有在runqueue上只是与M绑定
Gwaiting：go协程被阻塞，不再runqueue上但是一定在某个地方，比如channel，锁，排队中
Gdead：协程现在没有在使用，也许是执行完成
Gcopystack：栈正在复制，此时没有go代码也没有在runqueue上
Gscan：与runcnle，running，syscall。waiting等状态结合，表示GC 正在扫描这个G的栈

总结：
* goroutine是轻量级的协程，初始栈2KB左右，调度部涉及系统调用。
* Go的调度时为M（线程）找到P（内存，执行票据）和可运行的G。
* 用户代码中的协程的阻塞仅仅是切换协程，而不是阻塞线程，m和p仍旧结合区寻找新的可执行G。
* 每个P均有local runqueue，大多数时间只是与local queue交互，新生G优先放入local queue中，满了才会放到global queue中。
* 调度时会从全局runqueue取G，然后local queue，global runqueue，都没有可执行的G时会从其他P的队列中取G。
* sysmon，对于运行过久的G设置抢占标识，对于过久的syscall的P，进行M和P的分离，将分离的P给到其他的M防止P被占用过久影响调度。

### 参考资料
https://wudaijun.com/2018/01/go-scheduler/
https://zboya.github.io/post/go_scheduler/
https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/
https://www.myway5.com/index.php/2020/02/01/go-%e8%b0%83%e5%ba%a6%e5%99%a8%e7%9a%84%e5%ae%9e%e7%8e%b0/