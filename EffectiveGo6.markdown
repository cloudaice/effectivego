# Effective Go （六）
@(Go)[Go]

+ 初始化
  + 常量
  + 变量
  + init函数
+ 方法

## 初始化

Go中的初始化，虽然表面上看和C或者C++差别不大，但功能更加强大。在初始化过程中可以构建复杂的结构体，并且能够正确处理初始化对象之间，甚至不同程序包之间的顺序问题。

### 常量

Go中的常量仅仅就是—常量。它们是在编译时被创建，即使被定义为函数局部的也如此，并且只能是数字，字符（符文），字符串或者布尔类型。由于编译时的限制，定义它们的表达式必须为能被编译器求值的常量表达式。例如，1<<3是一个常量表达式，而math.Sin(math.Pi/4)不是，因为函数调用math.Sin需要在运行时才发生。

在Go中，枚举常量使用iota枚举器来创建。由于iota可以为表达式的一部分，并且表达式可以被隐式的重复，所以很容易创建复杂的值集。

    type ByteSize float64

    const (
        _           = iota // ignore first value by assigning to blank identifier
        KB ByteSize = 1 << (10 * iota)
        MB
        GB
        TB
        PB
        EB
        ZB
        YB
    )
    
可以将一个方法，比如String，附加到任何用户定义的类型上，这种能力使得任何值都可以自动格式化打印。虽然你会看到它经常用于结构体，但这种技术还可用于标量类型，比如ByteSize这样的浮点类型。

    func (b ByteSize) String() string {
        switch {
        case b >= YB:
            return fmt.Sprintf("%.2fYB", b/YB)
        case b >= ZB:
            return fmt.Sprintf("%.2fZB", b/ZB)
        case b >= EB:
            return fmt.Sprintf("%.2fEB", b/EB)
        case b >= PB:
            return fmt.Sprintf("%.2fPB", b/PB)
        case b >= TB:
            return fmt.Sprintf("%.2fTB", b/TB)
        case b >= GB:
            return fmt.Sprintf("%.2fGB", b/GB)
        case b >= MB:
            return fmt.Sprintf("%.2fMB", b/MB)
        case b >= KB:
            return fmt.Sprintf("%.2fKB", b/KB)
        }
        return fmt.Sprintf("%.2fB", b)
    }
    
表达式YB会打印出1.00YB，而ByteSize(1e13)会打印出9.09TB。

这里使用Sprintf来实现ByteSize的String方法是安全的（避免了无穷递归），这并不是因为做了转换，而是因为它是使用%f来调用Sprintf的，其不是一个字符串格式：Sprintf只有当想要一个字符串的时候，才调用String方法，而%f是想要一个浮点值。

### 变量

变量可以像常量那样进行初始化，不过初始值可以为运行时计算的通用表达式。

    var (
        home   = os.Getenv("HOME")
        user   = os.Getenv("USER")
        gopath = os.Getenv("GOPATH")
    )

### init函数

最后，每个源文件可以定义自己的不带参数的（niladic）init函数，来设置它所需的状态。（实际上每个文件可以有多个init函数。）init是在程序包中所有变量声明都被初始化，以及所有被导入的程序包中的变量初始化之后才被调用。

除了用于无法通过声明来表示的初始化以外，init函数的一个常用法是在真正执行之前进行验证或者修复程序状态的正确性。

    func init() {
        if user == "" {
            log.Fatal("$USER not set")
        }
        if home == "" {
            home = "/home/" + user
        }
        if gopath == "" {
            gopath = home + "/go"
        }
        // gopath may be overridden by --gopath flag on command line.
        flag.StringVar(&gopath, "gopath", gopath, "override default GOPATH")
    }

## 方法

### 指针 vs. 值

正如我们从ByteSize上看到的，任何命名类型（指针和接口除外）都可以定义方法（method）；接收者（receiver）不必为一个结构体。

在上面有关切片的讨论中，我们编写了一个Append函数。我们还可以将其定义成切片的方法。为此，我们首先声明一个用于绑定该方法的命名类型，然后将方法的接收者作为该类型的值。

    type ByteSlice []byte

    func (slice ByteSlice) Append(data []byte) []byte {
        // Body exactly the same as above
    }
    
这样还是需要方法返回更新后的切片。我们可以通过重新定义方法，接受一个ByteSlice的指针作为它的接收者，来消除这样笨拙的方式。这样，方法就可以改写调用者的切片。

    func (p *ByteSlice) Append(data []byte) {
        slice := *p
        // Body as above, without the return.
        *p = slice
    }
    
实际上，我们可以做的更好。如果我们将函数修改成标准Write方法的样子，像这样，

    func (p *ByteSlice) Write(data []byte) (n int, err error) {
        slice := *p
        // Again as above.
        *p = slice
        return len(data), nil
    }
    
那么类型*ByteSlice就会满足标准接口io.Writer，这样就很方便。例如，我们可以打印到该类型的变量中。

    var b ByteSlice
    fmt.Fprintf(&b, "This hour has %d days\n", 7)
    
我们传递ByteSlice的地址，是因为只有*ByteSlice才满足io.Writer。关于接收者对指针和值的规则是这样的，值方法可以在指针和值上进行调用，而指针方法只能在指针上调用。这是因为指针方法可以修改接收者；使用拷贝的值来调用它们，将会导致那些修改会被丢弃。

顺便说一下，在字节切片上使用Write的思想，是实现bytes.Buffer的核心。