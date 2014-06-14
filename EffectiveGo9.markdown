# Effective Go （九）
@(Go)[Go]

## 并发

### 以通信实现共享

并发程序设计是一个比较大的主题，这里我们只讨论一些Go语言特有的亮点。

由于需要考虑很多繁琐的细节以保证对共享变量访问的正确型，使得并发编程在很多情况下都会变得异常复杂。Go语言鼓励开发者采用一种不同的方法，即将共享 变量通过Channel相互传递 —— 事实上并没有真正在不同的执行线程间共享数据 —— 的方式解决上述问题。在任意时刻，仅有一个Goroutine可以访问某个变量。数据竞争问题在设计上就被规避了。为了鼓励采用这种思维模式，我们将其总 结为一句口号：

勿以共享方式通信，以通信实现共享。
这种方法还可以走得更远。举例而言，“引用计数”最好的实现途径可能就是通过在一个共享的整数周围加一个锁进行保护。但是在更高的层次，通过使用Channel控制共享整数访问可以梗容易的写出整洁、正确的程序。

试着用下面的方法来分析上述模型：想象我们只是在处理传统的单线程程序，该程序仅运行在一个物理CPU上。基于这个前提进行开发，是无需提供任何同步原语 的。现在，启动另一个类似的实例；它同样也不需要任何同步原语。然后让这两个实例进行通信；如果将通信本身算作一种同步原语，那么它是系统中仅有的同步原 语。Unix操作系统的管道（Pipeline）就是上述模型的一个很好实例。尽管Go语言的并发模型源自Hoare的CSP模型 （Communicating Sequential Processes, 国内译为“通信顺序进程”，台湾译为“交谈循序程序”），但它也可以被看成是一种类型安全的、一般化的Unix管道。

### Goroutines

之所以称之为Goroutine，主要是由于现有的一些概念—“线程”、“协程” 以及 “进程” 等—都不足以准确描述其内涵。每个Goroutine都对应一个非常简单的模型：它是一个并发的函数执行线索，并且在多个并发的Goroutine间，资 源是共享的。Goroutine非常轻量，创建的开销不会比栈空间分配的开销大多少。并且其初始栈空间很小 —— 这也就是它轻量的原因 —— 在后续执行中，会根据需要在堆空间分配（或释放）额外的栈空间。

Goroutine与操作系统线程间采用“多到多”的映射方式，因此假设一个Goroutine因某种原因阻塞 —— 比如等待一个尚未到达的IO —— 其他Goroutine可以继续执行。我们在实现中屏蔽了许多底层关于线程创建、管理的复杂细节。

在一个函数或是方法前加上go关键字就可以创建一个Goroutine并调用该函数或方法。当该函数执行结束，Goroutine也随之隐式退出。（这种效果很像在Unix Shell里用&符号在后台启动一个命令。）

    go list.Sort()  // run list.Sort concurrently; don't wait for it.
    
还可以将“函数文本”（function literals）嵌入到一个Goroutine创建之际，方法如下：

    func Announce(message string, delay time.Duration) {
        go func() {
            time.Sleep(delay)
            fmt.Println(message)
        }()  // Note the parentheses - must call the function.
    }
    
在Go中，这种“函数文本”的形式就是闭包（closure）：实现保证了在这类函数中被引用的变量在函数结束之前不会被释放。

以上的例子并不是很实用，因为执行函数无法发布其完成的信号。因此，我们还需要channel这一结构。

### Channels

与map结构类似，channel也是通过make进行分配的，其返回值实际上是一个指向底层相关数据结构的引用。如果在创建channel时提供一个可选的整型参数，会设置该channel的缓冲区大小。该值缺省为0，用来构建默认的“无缓冲channel”，也称为“同步channel”。

    ci := make(chan int)            // unbuffered channel of integers
    cj := make(chan int, 0)         // unbuffered channel of integers
    cs := make(chan *os.File, 100)  // buffered channel of pointers to Files
    
无缓冲的channel使得通信—值的交换—和同步机制组合—共同保证了两个执行线索（Goroutines）运行于可控的状态。

对于channel，有很多巧妙的用法。我们通过以下示例开始介绍。上一节中，我们曾在后台发起过一个排序操作。通过使用channel，可以让发起操作的Gorouine等待排序操作的完成。

    c := make(chan int)  // Allocate a channel.
    // Start the sort in a goroutine; when it completes, signal on the channel.
    go func() {
        list.Sort()
        c <- 1  // Send a signal; value does not matter.
    }()
    doSomethingForAWhile()
    <-c   // Wait for sort to finish; discard sent value.
    
接收方会一直阻塞直到有数据到来。如果channel是无缓冲的，发送方会一直阻塞直到接收方将数据取出。如果channel带有缓冲区，发送方会一直阻塞直到数据被拷贝到缓冲区；如果缓冲区已满，则发送方只能在接收方取走数据后才能从阻塞状态恢复。

带缓冲区的channel可以像信号量一样使用，用来完成诸如吞吐率限制等功能。在以下示例中，到来的请求以参数形式传入handle函数，该函数从channel中读出一个值，然后处理请求，最后再向channel写入以使“信号量”可用，以便响应下一次处理。该channel的缓冲区容量决定了并发调用process函数的上限，因此在channel初始化时，需要传入相应的容量参数。

    var sem = make(chan int, MaxOutstanding)

    func handle(r *Request) {
        <-sem          // Wait for active queue to drain.
        process(r)     // May take a long time.
        sem <- 1       // Done; enable next request to run.
    }

    func init() {
        for i := 0; i < MaxOutstanding; i++ {
            sem <- 1
        }
    }

    func Serve(queue chan *Request) {
        for {
            req := <-queue
            go handle(req)  // Don't wait for handle to finish.
        }
    }
    
由于在Go中，数据同步发生在从channel接收数据阶段（也就是说，发送操作发生在接收操作之前，参见Go内存模型），因此获取信号量的操作必须实现在channel的接收阶段，而不是发送阶段。

这样的设计会引入一个问题： Serve会为每个请求创建一个新的Goroutine，尽管在任意时刻只有最多MaxOutstanding个可以执行。如果请求到来的速度过快，将迅速导致系统资源完全消耗。我们可以通过修改Serve的实现来对Goroutine的创建进行限制。以下给出一个简单的实现，请注意其中包含一个BUG，我们会在后续进行修正：

    func Serve(queue chan *Request) {
        for req := range queue {
            <-sem
            go func() {
                process(req) // Buggy; see explanation below.
                sem <- 1
            }()
        }
    }
    
刚才说的BUG源自Go中for循环的实现，循环的迭代变量会在循环中被重用，因此req变量会在所有Goroutine间共享。这不是我们所乐见的，我们需要保证req变量是每个Goroutine私有的。这里提供一个方法，将req的值以参数形式提供给goroutine对应的闭包：

    func Serve(queue chan *Request) {
        for req := range queue {
            <-sem
            go func(req *Request) {
                process(req)
                sem <- 1
            }(req)
        }
    }
    
请与之前有BUG的实现进行对比，看看闭包在声明和运行上的不同之处。另一个解决方案是，干脆创建一个新的同名变量，示例如下：

    func Serve(queue chan *Request) {
        for req := range queue {
            <-sem
            req := req // Create new instance of req for the goroutine.
            go func() {
                process(req)
                sem <- 1
            }()
        }
    }
    
这样写可能看起来怪怪的

    req := req
    
但它确实是合法的并且在Go中是一种惯用的方法。你可以如法泡制一个新的同名变量，用来为每个Goroutine创建循环变量的私有拷贝。

回到实现通用服务器的问题上来，另一个有效的资源管理途径是启动固定数量的handle Goroutine，每个Goroutine都直接从channel中读取请求。这个固定的数值就是同时执行process的最大并发数。Serve函数还需要一个额外的channel参数，用来等待退出通知；当创建完所有的Goroutine之后， Server 自身阻塞在该channel上等待结束信号。

    func handle(queue chan *Request) {
        for r := range queue {
            process(r)
        }
    }

    func Serve(clientRequests chan *Request, quit chan bool) {
        // Start handlers
        for i := 0; i < MaxOutstanding; i++ {
            go handle(clientRequests)
        }
        <-quit  // Wait to be told to exit.
    }

### Channel类型的Channel

Channel在Go语言中是一个 first-class 类型，这意味着channel可以像其他 first-class 类型变量一样进行分配、传递。该属性的一个常用方法是用来实现安全、并行的解复用（demultiplexing）处理。

在上节的那个例子中，handle是一个理想化的请求处理，但我们并没有定义处理的类型。如果处理的类型中包括一个用来响应的channel，则每个客户端可以提供其独特的响应方式。这里提供一个简单的Request类型定义：

    type Request struct {
        args        []int
        f           func([]int) int
        resultChan  chan int
    }
    
客户端提供了一个函数及其参数，以及一个内部的channel变量用来接收回答消息。

    func sum(a []int) (s int) {
        for _, v := range a {
            s += v
        }
        return
    }

    request := &Request{[]int{3, 4, 5}, sum, make(chan int)}
    // Send request
    clientRequests <- request
    // Wait for response.
    fmt.Printf("answer: %d\n", <-request.resultChan)
    
在服务器端，只有处理函数handle需要改变：

    func handle(queue chan *Request) {
        for req := range queue {
            req.resultChan <- req.f(req.args)
        }
    }
    
显然，上述例子还有很大的优化空间以提高其可用性，但是这套代码已经可以作为一类对速度要求不高、并行、非阻塞式RPC系统的实现框架了，而且实现中没有使用任何显式的互斥语法。

### 并行

上述这些想法的另一个应用场景是将计算在不同的CPU核心之间并行化，如果计算可以被划分为不同的可独立执行的部分，那么它就是可并行化的，任务可以通过一个channel发送结束信号。

假设我们需要在数组上进行一个比较耗时的操作，并且操作的值在每个数据上是独立的，正如下面这个理想化的例子一样：

    type Vector []float64

    // Apply the operation to v[i], v[i+1] ... up to v[n-1].
    func (v Vector) DoSome(i, n int, u Vector, c chan int) {
        for ; i < n; i++ {
            v[i] += u.Op(v[i])
        }
        c <- 1    // signal that this piece is done
    }
    
我们在每个CPU上加载一个循环无关的迭代计算。这些计算可能以任意次序完成，但这是无关紧要的；我们仅需要在创建完所有Goroutine后，从channel中读取结束信号进行计数即可。

    const NCPU = 4  // number of CPU cores

    func (v Vector) DoAll(u Vector) {
        c := make(chan int, NCPU)  // Buffering optional but sensible.
        for i := 0; i < NCPU; i++ {
            go v.DoSome(i*len(v)/NCPU, (i+1)*len(v)/NCPU, u, c)
        }
        // Drain the channel.
        for i := 0; i < NCPU; i++ {
            <-c    // wait for one task to complete
        }
        // All done.
    }

在目前的Go runtime 实现中，这段代码在默认情况下是不会被并行化的。对于用户态任务，我们默认仅提供一个物理CPU进行处理。任意数目的Goroutine可以阻塞在系统调 用上，但默认情况下，在任意时刻，只有一个Goroutine可以被调度执行。我们未来可能会将其设计的更加智能，但是目前，你必须通过设置GOMAXPROCS环境变量或者导入runtime包并调用runtime.GOMAXPROCS(NCPU), 来告诉Go的运行时系统最大并行执行的Goroutine数目。你可以通过runtime.NumCPU()获得当前运行系统的逻辑核数，作为一个有用的参考。需要重申：上述方法可能会随我们对实现的完善而最终被淘汰。

注意不要把“并发”和“并行”这两个概念搞混：“并发”是指用一些彼此独立的执行模块构建程序；而“并行”则是指通过将计算任务在多个处理器上同时执行以 提高效率。尽管对于一些问题，我们可以利用“并发”特性方便的构建一些并行的程序部件，但是Go终究是一门“并发”语言而非“并行”语言，并非所有的并行 编程模式都适用于Go语言模型。要进一步区分两者的概念，请参考这篇博客的相关讨论。

### 一个“Leaky Buffer”的示例

并发编程的工具甚至可以更简单的表达一些非并发的想法。下面提供一个示例，它是从RPC的一个包里抽象而来的。客户端从某些源 —— 比如网络 —— 循环接收数据。为了避免频繁的分配、释放内存缓冲，程序在内部实现了一个空闲链表，并用一个Buffer指针型channel将其封装。当该 channel为空时，程序为其分配一个新的Buffer对象。一旦消息缓冲就绪，它就会被经由serverChan发送到服务器端。

    var freeList = make(chan *Buffer, 100)
    var serverChan = make(chan *Buffer)

    func client() {
        for {
            var b *Buffer
            // Grab a buffer if available; allocate if not.
            select {
            case b = <-freeList:
                // Got one; nothing more to do.
            default:
                // None free, so allocate a new one.
                b = new(Buffer)
            }
            load(b)              // Read next message from the net.
            serverChan <- b      // Send to server.
        }
    }
    
服务器端循环从客户端接收并处理每个消息，然后将Buffer对象返回到空闲链表中。

    func server() {
        for {
            b := <-serverChan    // Wait for work.
            process(b)
            // Reuse buffer if there's room.
            select {
            case freeList <- b:
                // Buffer on free list; nothing more to do.
            default:
                // Free list full, just carry on.
            }
        }
    }
    
客户端会尝试从空闲链表freeList中获取Buffer对象；如果没有可用对象，则分配一个新的。服务器端会将用完的Buffer对象 b 加入到空闲链表freeList中，如果链表已满，则将b丢弃，垃圾收集器会在未来某个时刻自动回收对应的内存单元。（select语句中的default分支会在没有其他case分支满足条件时执行，这意味着select语句块不会阻塞。）以上就是一个 Leaky Bucket Free List 的简单实现，借助Go语言中带缓冲的channel以及“垃圾收集”机制，我们仅用几行代码就将其搞定了。