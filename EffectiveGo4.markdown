# Effective Go（四）
@(Go)[Go]

# 函数

### 多个返回值

Go的一个不同寻常的特点是，函数和方法可以返回多个值。这种形式可以用来改进C程序中几个笨拙的语言风格：返回一个错误，例如`-1`对应于`EOF`，同时修改一个由地址传递的参数。

在C中，一个写错误是由一个负的计数和一个隐藏在易变位置（a volatile location）的错误代码来表示的。在Go中，Write可以返回一个计数和一个错误：*是的，你写了一些字节，但并没有全部写完，由于设备已经被填满了*。在标准库os包中，`Write`方法的定义是：

    func (file *File) Write(b []byte) (n int, err error)
    
正如文档所言，其返回写入的字节数和一个非零的error，当`n!= len(b)`的时候。这是一种常见的风格；更多的例子可以参见错误处理章节。

类似的方法使得不再需要传递一个返回值指针来模拟一个引用参数。这里有一个非常简单的函数，用来从字节切片中的一个位置抓取一个数，返回该数和下一个位置。

    func nextInt(b []byte, i int) (int, int) {
        for ; i < len(b) && !isDigit(b[i]); i++ {
        }
        x := 0
        for ; i < len(b) && isDigit(b[i]); i++ {
            x = x*10 + int(b[i]) - '0'
        }
        return x, i
    }
    
你可以使用它来扫描输入切片b中的数字，如：

    for i := 0; i < len(b); {
        x, i = nextInt(b, i)
        fmt.Println(x)
    }

### 命名的结果参数

Go函数的返回或者结果*参数*可以给定一个名字，并作为一个普通变量来使用，就像是输入参数一样。当被命名时，它们在函数起始处被初始化为对应类型的**零值**；如果函数执行了没有参数的return语句，则结果参数的当前值便被作为要返回的值。

名字并不是强制的，但是可以使代码更加简短清晰：它们也是文档。如果我们将nextInt的结果进行命名，则其要返回的int是对应的哪一个就很显然了。

    func nextInt(b []byte, pos int) (value, nextPos int) {
    
因为命名结果是被初始化的，并且与没有参数的return绑定在一起，所以它们即简单又清晰。这里是一个io.ReadFull的版本，很好地使用了这些特性：

    func ReadFull(r Reader, buf []byte) (n int, err error) {
        for len(buf) > 0 && err == nil {
            var nr int
            nr, err = r.Read(buf)
            n += nr
            buf = buf[nr:]
        }
        return
    }

### 延期执行

Go的defer语句用来调度一个函数调用（被延期的函数），使其在执行defer的函数即将返回之前才被运行。这是一种不寻常但又很有效的方法，用于处理类似于不管函数通过哪个执行路径返回，资源都必须要被释放的情况。典型的例子是对一个互斥解锁，或者关闭一个文件。

    // Contents returns the file's contents as a string.
    func Contents(filename string) (string, error) {
        f, err := os.Open(filename)
        if err != nil {
            return "", err
        }
        defer f.Close()  // f.Close will run when we're finished.
        var result []byte
        buf := make([]byte, 100)
        for {
            n, err := f.Read(buf[0:])
            result = append(result, buf[0:n]...) // append is discussed later.
            if err != nil {
                if err == io.EOF {
                    break
                }
                return "", err  // f will be closed if we return here.
            }
        }
        return string(result), nil // f will be closed if we return here.
    }
    
对像`Close`这样的函数调用进行延期，有两个好处。首先，其确保了你不会忘记关闭文件，如果你之后修改了函数增加一个新的返回路径，会很容易犯这样的错。其次，这意味着关闭操作紧挨着打开操作，这比将其放在函数结尾更加清晰。

被延期执行的函数，它的参数（包括接收者，如果函数是一个方法）是在defer执行的时候被求值的，而不是在调用执行的时候。这样除了不用担心变量随着函数的执行值会改变，这还意味着单个被延期执行的调用点可以延期多个函数执行。这里有一个简单的例子。

    for i := 0; i < 5; i++ {
        defer fmt.Printf("%d ", i)
    }
    
被延期的函数按照LIFO的顺序执行，所以这段代码会导致在函数返回时打印出4 3 2 1 0。一个更加真实的例子，这是一个跟踪程序中函数执行的简单方法。我们可以编写几个类似这样的，简单的跟踪程序：

    func trace(s string)   { fmt.Println("entering:", s) }
    func untrace(s string) { fmt.Println("leaving:", s) }

    // Use them like this:
    func a() {
        trace("a")
        defer untrace("a")
        // do something....
    }
    
利用被延期的函数的参数是在defer执行的时候被求值这个事实，我们可以做的更好些。trace程序可以为untrace程序建立参数。这个例子：

    func trace(s string) string {
        fmt.Println("entering:", s)
        return s
    }

    func un(s string) {
        fmt.Println("leaving:", s)
    }

    func a() {
        defer un(trace("a"))
        fmt.Println("in a")
    }

    func b() {
        defer un(trace("b"))
        fmt.Println("in b")
        a()
    }

    func main() {
        b()
    }
    
会打印出

    entering: b
    in b
    entering: a
    in a
    leaving: a
    leaving: b
    
对于习惯于其它语言中的块级别资源管理的程序员，defer可能看起来很奇怪，但是它最有趣和强大的应用正是来自于这样的事实，这是基于函数的而不是基于块的。我们将会在panic和recover章节中看到它另一个可能的例子。
