---
layout: post
title: "goroutine泄漏分析"
subtitle: '总结一下可能造成goroutine泄漏的原因和场景'
author: "odin"
header-style: text
tags:
  - golang
---

语言级别的并发支持是go的一大优势，golang中提供goroutine使得我们可以很方便的解决并发问题，看起来十分简单。但是在使用的时候也存在很多需要我们注意的地方，稍不谨慎往往就会造成goroutine泄漏，而这些造成goroutine泄漏的原因大部分是容易让人忽略的代码细节。也就意味着我们不能滥用goroutine。
这里总结一下常见的一些引起goroutine泄漏的使用场景。

### goroutine泄漏
什么是goroutine泄漏？
goroutine是一种常见的内存泄漏，涉及到内存管理go在编译时使用逃逸分析来决定值在内存中的位置，运行时通过垃圾回收器跟踪和管理堆分配，使得创建内存泄漏的可能性大大降低(但还是会有)。goroutine泄漏简单来说就是：开启了一个你认为会终止的goroutine，但是由于某些原因导致他一直无法终止，任何分配给goroutine的内存都不能释放，便造成了泄漏。

### 基本泄漏
```golang
func main() {
	defer func() {
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	go func() {
		for true {
			fmt.Println("Hello gorotine")
			time.Sleep(time.Second)
		}
	}()
	fmt.Println("Hello main")
}
```
创建的goroutine中执行了一个是循环，很明显goroutine得不到释放，这个示例很简单，相信大部分人能一眼看出问题。该场景不做过多分析。

### channel泄漏
往往channel引起的泄漏，大部分可以归为：
* 发送不接收：发送者一般都会配有相应的接收者。理想情况下，我们希望接收者总能接收完所有发送的数据，这样就不会有任何问题。但现实是，一旦接收者发生异常退出，停止继续接收上游数据，发送者就会被阻塞。
* 接收不发送：同上导致阻塞
* 无缓冲的channel：向无缓冲channel发送和接收数据都想导致阻塞，这种情况一般在我们定义channel的时候
#### 示例一
针对发送不接收和接收不发送的场景简单示例：
```golang
func leak() {
     ch := make(chan int)
     go func() {
        val := <-ch
        fmt.Println("We received a value:", val)
    }()
 }
```
上面这个`leak`函数，启动了一个`goroutine`，该`goroutine`阻塞等待接收`channel`发送的数据，`leak执`行结束`val`被清除，`goroutine`将没有接收对象，`channel`永远不会被关闭，`goroutine`被锁死。造成泄漏。
通过这个基本的示例，能大致对`goroutine`泄漏有一个基本的概念。我们**永远不要在不知道如何停止的情况下去开启一个goroutine**

#### 示例二
针对无缓冲的channel的场景示例：
```golang
// search 模拟成一个查找记录的函数
// 在查找记录时。执行此工作需要 200 ms。
func search(term string) (string, error) {
     time.Sleep(200 * time.Millisecond)
     return "some value", nil
}

// serach 函数得到的返回值用 result 结构体来保存, 通过单个 channel 来传递这两个值
type result struct {
    record string
    err    error
}

// process 函数是一个用来寻找记录的函数, 然后打印，如果超过 100 ms 就会失败 .
func process(term string) error {
     // 创建一个在 100 ms 内取消上下文的 context
     ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
     defer cancel()
     // 为 Goroutine 创建一个传递结果的 channel
     ch := make(chan result)
     // 启动一个 goroutine 来寻找记录，然后得到结果
     // 将返回值从 channel 中返回
     go func() {
         record, err := search(term)
         ch <- result{record, err}
     }()

     // 阻塞等待从 goroutine 接收值
     // 通过 channel 和 context 来取消上下文操作
     select {
     case <-ctx.Done():
         return errors.New("search canceled")
     case result := <-ch:
         if result.err != nil {
            return result.err
         }
         fmt.Println("Received:", result.record)
         return nil
    }
}
```

`process`函数创建`Context`在100ms内取消上下文，然后在第 `17` 行，程序创建一个无缓冲的 `channel`，允许 `Goroutines` 传递 `result` 类型的数据。在第 `23` 到 `26` 行，定义了匿名函数，此处称为 `Goroutine`. `Goroutine` 调用 `search` 函数并尝试通过第 `24` 行的 `channel` 发送其返回值。  
当 `Goroutine` 正在执行其工作时，`process` 函数执行第 `30` 行上的 `select` 模块。该模块有两种情况，它们都是 `channel` 接收操作。在第 `31` 行，有一个从` ctx.Done() channel `接收的 `case`。如果上下文被取消（100 ms 持续时间到达），将执行此 `case`。如果执行此 `case`，则 `process` 函数将返回错误，代表着取消了等待第 `32` 行的 `search`。或者，第 `33` 行上的 `case` 从 `ch channel` 接收并将值分配给名为 `result` 的变量。与前面在顺序实现中一样，程序在第 `34` 行和第 `35` 行检查和处理错误。如果没有错误，程序将在第 `37` 行打印记录，并返回 `nil` 以指示成功。  
这个函数看起来没有什么问题，但是存在隐藏的 `goroutine`风险。关键在于`17`行创建的`channel`是一个`无缓冲的channel`。在`25`行，`goroutine`通过`channel`发送，在此`channel`上发送将阻塞执行，直到接收到内容，在超时的情况下，接收方停止等待`goroutine`的接收工作并继续执行，将导致goroutine永远阻塞等待一个永远不会发生的接收器出现。这就是隐藏的`goroutine`泄漏风险。  
规避这个隐藏风险只需要把`channel`改为`有缓冲的channel`即可：
```golang
// 为goroutine创建一个传递结果的缓冲值为1的channel，以至于发送接收不会阻塞
ch := make(chan result, 1)
```

### 锁竞争泄漏
```golang
func main() {
	total := 0

	defer func() {
		time.Sleep(time.Second)
		fmt.Println("total: ", total)
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	var mutex sync.Mutex
	for i := 0; i < 2; i++ {
		go func() {
			mutex.Lock()
			total += 1
		}()
	}
}
```
上述示例中创建了两个goroutine，使用互斥锁但并没有释放。导致i = 1的goroutine将一直阻塞等待锁的释放，造成泄漏。解决方式也很简单，记得将锁释放即可。

### waitgroup泄漏
```golang
func handle() {
	var wg sync.WaitGroup

	wg.Add(2)

	go func() {
		fmt.Println("访问表3")
		wg.Done()
	}()

	wg.Wait()
}

func main() {
	defer func() {
		time.Sleep(time.Second)
		fmt.Println("the number of goroutines: ", runtime.NumGoroutine())
	}()

	go handle()
	time.Sleep(time.Second)
}
```
上述示例中，`handle`中向`waitgroup`中添加了2个任务，但是只有1个并发任务，最后`wg.Wait()`等待退出条件将无法完成，造成`handle`一直阻塞。本示例中任务数较少，所以比较容易一眼看出来，往往任务数较多时很容易忽略`add`的任务数量和并发任务数量不等。

### 总结
大致可能造成goroutine泄漏的情况，其实无论是死循环、channel 阻塞、锁等待，只要是会造成阻塞的写法都可能产生泄露。因而，如何防止 goroutine 泄露就变成了如何防止发生阻塞。为进一步防止泄露，有些实现中会加入超时处理，主动释放处理时间太长的 goroutine。