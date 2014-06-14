# Effective Go （七）
@(Go)[Go]

## 接口和其它类型

### 接口

Go中的接口为指定对象的行为提供了一种方式：如果事情可以这样做，那么它就可以在这里使用。我们已经看到一些简单的例子；自定义的打印可以通过String方法来实现，而Fprintf可以通过Write方法输出到任意的地方。只有一个或两个方法的接口在Go代码中很常见，并且它的名字通常来自这个方法，例如实现Write的io.Writer。

类型可以实现多个接口。例如，如果一个集合实现了sort.Interface，其包含Len()，Less(i, j int) bool和Swap(i, j int)，那么它就可以通过程序包sort中的程序来进行排序，同时它还可以有一个自定义的格式器。在这个人造的例子中，Sequence同时符合这些条件。

    type Sequence []int

    // Methods required by sort.Interface.
    func (s Sequence) Len() int {
        return len(s)
    }
    func (s Sequence) Less(i, j int) bool {
        return s[i] < s[j]
    }
    func (s Sequence) Swap(i, j int) {
        s[i], s[j] = s[j], s[i]
    }

    // Method for printing - sorts the elements before printing.
    func (s Sequence) String() string {
        sort.Sort(s)
        str := "["
        for i, elem := range s {
            if i > 0 {
                str += " "
            }
            str += fmt.Sprint(elem)
        }
        return str + "]"
    }

### 转换

Sequence的String方法重复了Sprint对切片所做的工作。如果我们在调用Sprint之前，将Sequence转换为普通的[]int，则可以共享所做的工作。

    func (s Sequence) String() string {
        sort.Sort(s)
        return fmt.Sprint([]int(s))
    }
    
这个对象方法算是转换技术的另一个例子，从String方法中安全地调用Sprintf。因为如果我们忽略类型名字，这两个类型（Sequence和[]int）是相同的，在它们之间进行转换是合法的。该转换并不创建新的值，只不过是暂时使现有的值具有一个新的类型。（有其它的合法转换，像整数到浮点，是会创建新值的。）

将表达式的类型进行转换，来访问不同的方法集合，这在Go程序中是一种常见用法。例如，我们可以使用已有类型sort.IntSlice来将整个例子简化成这样：

    type Sequence []int

    // Method for printing - sorts the elements before printing
    func (s Sequence) String() string {
        sort.IntSlice(s).Sort()
        return fmt.Sprint([]int(s))
    }
    
现在，Sequence没有实现多个接口（排序和打印），相反的，我们利用了能够将数据项转换为多个类型（Sequence，sort.IntSlice和[]int）的能力，每个类型完成工作的一部分。这在实际中不常见，但是却可以很有效。

### 接口转换和类型断言

类型switch为一种转换形式：它们接受一个接口，在switch的每个case中，从某种意义上将其转换为那种case的类型。这里有一个简化版本，展示了fmt.Printf中的代码如何使用类型switch将一个值转换为字符串。如果其已经是字符串，那么我们想要接口持有的实际字符串值，如果其有一个String方法，则我们想要调用该方法的结果。

    type Stringer interface {
        String() string
    }

    var value interface{} // Value provided by caller.
    switch str := value.(type) {
    case string:
        return str
    case Stringer:
        return str.String()
    }
    
第一种情况找到一个具体的值；第二种将接口转换为另一个。使用这种方式进行混合类型完全没有问题。

如果我们只关心一种类型该如何做？如果我们知道值为一个string，只是想将它抽取出来该如何做？只有一个case的类型switch是可以的，不过也可以用类型断言。类型断言接受一个接口值，从中抽取出显式指定类型的值。其语法借鉴了类型switch子句，不过是使用了显式的类型，而不是type关键字：

value.(typeName)
结果是一个为静态类型typeName的新值。该类型或者是一个接口所持有的具体类型，或者是可以被转换的另一个接口类型。要抽取我们已知值中的字符串，可以写成：

    str := value.(string)
    
不过，如果该值不包含一个字符串，则程序会产生一个运行时错误。为了避免这样，可以使用“comma, ok”的习惯用法来安全地测试值是否为一个字符串：

    str, ok := value.(string)
    if ok {
        fmt.Printf("string value is: %q\n", str)
    } else {
        fmt.Printf("value is not a string\n")
    }
    
如果类型断言失败，则str将依然存在，并且类型为字符串，不过其为零值，一个空字符串。

这里有一个if-else语句的实例，其效果等价于这章开始的类型switch例子。

    if str, ok := value.(string); ok {
        return str
    } else if str, ok := value.(Stringer); ok {
        return str.String()
    }

### 概述

如果一个类型只是用来实现接口，并且除了该接口以外没有其它被导出的方法，那就不需要导出这个类型。只导出接口，清楚地表明了其重要的是行为，而不是实现，并且其它具有不同属性的实现可以反映原始类型的行为。这也避免了对每个公共方法实例进行重复的文档介绍。

这种情况下，构造器应该返回一个接口值，而不是所实现的类型。作为例子，在hash库里，crc32.NewIEEE和adler32.New都是返回了接口类型hash.Hash32。在Go程序中，用CRC-32算法来替换Adler-32，只需要修改构造器调用；其余代码都不受影响。

类似的方式可以使得在不同crypto程序包中的流密码算法，可以与链在一起的块密码分离开。crypto/cipher程序包中的Block接口，指定了块密码的行为，即提供对单个数据块的加密。然后，根据bufio程序包类推，实现该接口的加密包可以用于构建由Stream接口表示的流密码，而无需知道块加密的细节。

crypto/cipher接口看起来是这样的：

    type Block interface {
        BlockSize() int
        Encrypt(src, dst []byte)
        Decrypt(src, dst []byte)
    }

    type Stream interface {
        XORKeyStream(dst, src []byte)
    }
    
这里有一个计数器模式（CTR）流的定义，其将块密码转换为流密码；注意块密码的细节被抽象掉了：

    // NewCTR returns a Stream that encrypts/decrypts using the given     Block in
    // counter mode. The length of iv must be the same as the Block's block size.
    func NewCTR(block Block, iv []byte) Stream
    
NewCTR并不只是用于一个特定的加密算法和数据源，而是用于任何对Block接口的实现和任何Stream。因为它们返回接口值，所以将CTR加密替换为其它加密模式只是一个局部的改变。构造器调用必须被修改，不过因为上下文代码必须将结果只作为Stream来处理，所以其不会注意到差别。

### 接口和方法

由于几乎任何事物都可以附加上方法，所以几乎任何事物都能够满足接口的要求。一个示例是在http程序包中，其定义了Handler接口。任何实现了Handler的对象都可以为HTTP请求提供服务。

    type Handler interface {
        ServeHTTP(ResponseWriter, *Request)
    }
    
ResponseWriter本身是一个接口，提供了对用于向客户端返回响应的方法的访问。这些方法包括了标准的Write方法，所以任何可以使用io.Writer的地方，都可以使用http.ResponseWriter。

简单起见，让我们忽略POST，假设HTTP请求总是GET；这种简化不影响建立处理的方式。这里有一个简单而完整的handler实现，用于计算页面的访问次数。

    // Simple counter server.
    type Counter struct {
        n int
    }

    func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req     *http.Request) {
        ctr.n++
        fmt.Fprintf(w, "counter = %d\n", ctr.n)
    }
    
(题外话，注意Fprintf是如何能够打印到http.ResponseWriter的。）作为参考，下面给出了如何将该服务附加到URL树上的节点。

    import "net/http"
    ...
    ctr := new(Counter)
    http.Handle("/counter", ctr)
    
但是为什么Counter为一个结构体？只需要一个整数就可以了。（接收者需要为一个指针，这样增量才能对调用者可见。）

    // Simpler counter server.
    type Counter int

    func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req     *http.Request) {
        *ctr++
        fmt.Fprintf(w, "counter = %d\n", *ctr)
    }
    
如果你的程序具有某个内部状态，当页面被访问时需要被告知，那么该如何？可以将一个channel绑定到网页上。

    // A channel that sends a notification on each visit.
    // (Probably want the channel to be buffered.)
    type Chan chan *http.Request

    func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
        ch <- req
        fmt.Fprint(w, "notification sent")
    }
    
最后，比方说我们想在/args上展现我们唤起服务二进制时所使用的参数。这很容易编写一个函数来打印参数。

    func ArgServer() {
        fmt.Println(os.Args)
    }
    
我们怎么将它转换成HTTP服务？我们可以将ArgServer创建为某个类型的方法，忽略该类型的值，不过有一种更干净的方式。既然我们可以为除了指针和接口以外的任何类型来定义方法，那么我们可以为函数编写一个方法。http程序包包含了这样的代码：

    // The HandlerFunc type is an adapter to allow the use of
    // ordinary functions as HTTP handlers.  If f is a function
    // with the appropriate signature, HandlerFunc(f) is a
    // Handler object that calls f.
    type HandlerFunc func(ResponseWriter, *Request)

    // ServeHTTP calls f(c, req).
    func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
        f(w, req)
    }
    
HandlerFunc为一个类型，其具有一个方法，ServeHTTP，所以该类型值可以为HTTP请求提供服务。看下该方法的实现：接收者为一个函数，f，并且该方法调用了f。这看起来可能有些怪异，但是这与接收者为channel，方法在channel上进行发送数据并无差别。

要将ArgServer放到HTTP服务中，我们首先将其签名修改正确。

    // Argument server.
    func ArgServer(w http.ResponseWriter, req *http.Request) {
        fmt.Fprintln(w, os.Args)
    }
    
ArgServer现在具有和HandlerFunc相同的签名，所以其可以被转换为那个类型，然后访问它的方法，就像我们将Sequence转换为IntSlice，来访问IntSlice.Sort一样。代码实现很简洁：

    http.Handle("/args", http.HandlerFunc(ArgServer))
    
当有人访问页面/args时，在该页上安装的处理者就具有值ArgServer和类型HandlerFunc。HTTP服务将会调用该类型的方法ServeHTTP，将ArgServer作为接收者，其将转而调用ArgServer（通过在HandlerFunc.ServeHTTP内部调用f(c, req)）。然后，参数就被显示出来了。

在这章节，我们分别通过结构体，整数，channel，以及函数创建了HTTP服务，这都是因为接口就是一个方法的集合，其可以针对（几乎）任何类型来定义。