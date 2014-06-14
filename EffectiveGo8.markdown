# Effective Go （八）
@(Go)[Go]

## 空白标识符

截至目前，我们已经两次提及“空白标识符”这个概念了，一次是在讲for range loops形式的循环时，另一次是在讲maps结构时。空白标识符可以赋值给任意变量或者声明为任意类型，只要忽略这些值不会带来问题就可以。这有点像在Unix系统中向/dev/null文件写入数据：它为那些需要出现但值其实可以忽略的变量提供了一个“只写”的占位符。但正如我们之前看到的那样，它实际的用途其实不止于此。

### 空白标识符在多赋值语句中的使用

空白标识符在for range循环中使用的其实是其应用在多语句赋值情况下的一个特例。

一个多赋值语句需要多个左值，但假如其中某个左值在程序中并没有被使用到，那么就可以用空白标识符来占位，以避免引入一个新的无用变量。例如，当调用的函 数同时返回一个值和一个error，但我们只关心error时,那么就可以用空白标识符来对另一个返回值进行占位，从而将其忽略。

    if _, err := os.Stat(path); os.IsNotExist(err) {
	    fmt.Printf("%s does not exist\n", path)
    }
    
有时，你也会发现一些代码用空白标识符对error占位，以忽略错误信息；这不是一种好的做法。好的实现应该总是检查返回的error值，因为它会告诉我们错误发生的原因。

    // Bad! This code will crash if path does not exist.
    fi, _ := os.Stat(path)
    if fi.IsDir() {
        fmt.Printf("%s is a directory\n", path)
    }

### 未使用的导入和变量

如果你在程序中导入了一个包或声明了一个变量却没有使用的话,会引起编译错误。因为，导入未使用的包不仅会使程序变得臃肿，同时也降低了编译效率；初始化 一个变量却不使用，轻则造成对计算的浪费，重则可能会引起更加严重BUG。当一个程序处于开发阶段时，会存在一些暂时没有被使用的导入包和变量，如果为了 使程序编译通过而将它们删除，那么后续开发需要使用时，又得重新添加，这非常麻烦。空白标识符为上述场景提供了解决方案。

以下一段代码包含了两个未使用的导入包（fmt和io） 以及一个未使用的变量（fd），因此无法编译通过。我们可能希望这个程序现在就可以正确编译。

    package main

    import (
        "fmt"
        "io"
        "log"
        "os"
    )

    func main() {
        fd, err := os.Open("test.go")
        if err != nil {
            log.Fatal(err)
        }
        // TODO: use fd.
    }
    
为了禁止编译器对未使用导入包的错误报告，我们可以用空白标识符来引用一个被导入包中的符号。同样的，将未使用的变量fd赋值给一个空白标识符也可以禁止编译错误。这个版本的程序就可以编译通过了。

    package main

    import (
        "fmt"
        "io"
        "log"
        "os"
    )

    var _ = fmt.Printf // For debugging; delete when done.
    var _ io.Reader    // For debugging; delete when done.

    func main() {
        fd, err := os.Open("test.go")
        if err != nil {
            log.Fatal(err)
        }
        // TODO: use fd.
        _ = fd
    }
    
按照约定，用来临时禁止未使用导入错误的全局声明语句必须紧随导入语句块之后，并且需要提供相应的注释信息 —— 这些规定使得将来很容易找并删除这些语句。

### 副作用式导入

像上面例子中的导入的包，fmt或io，最终要么被使用，要么被删除：使用空白标识符只是一种临时性的举措。但有时，导入一个包仅仅是为了引入一些副作用，而不是为了真正使用它们。例如，net/http/pprof包会在其导入阶段调用init函数，该函数注册HTTP处理程序以提供调试信息。这个包中确实也包含一些导出的API，但大多数客户端只会通过注册处理函数的方式访问web页面的数据，而不需要使用这些API。为了实现仅为副作用而导入包的操作，可以在导入语句中，将包用空白标识符进行重命名：

    import _ "net/http/pprof"
    
这一种非常干净的导入包的方式，由于在当前文件中，被导入的包是匿名的，因此你无法访问包内的任何符号。（如果导入的包不是匿名的，而在程序中又没有使用到其内部的符号，那么编译器将报错。）

### 接口检查

正如我们在前面接口那章所讨论的，一个类型不需要明确的声明它实现了某个接口。一个类型要实现某个接口，只需要实现该接口对应的方法就可以了。在实际中，多数接口的类型转换和检查都是在编译阶段静态完成的。例如，将一个*os.File类型传入一个接受io.Reader类型参数的函数时，只有在*os.File实现了io.Reader接口时，才能编译通过。

但是，也有一些接口检查是发生在运行时的。其中一个例子来自encoding/json包内定义的Marshaler接口。当JSON编码器接收到一个实现了Marshaler接口的参数时，就调用该参数的marshaling方法来代替标准方法处理JSON编码。编码器利用类型断言机制在运行时进行类型检查：

    m, ok := val.(json.Marshaler)
   
假设我们只是想知道某个类型是否实现了某个接口，而实际上并不需要使用这个接口本身 —— 例如在一段错误检查代码中 —— 那么可以使用空白标识符来忽略类型断言的返回值：

    if _, ok := val.(json.Marshaler); ok {
        fmt.Printf("value %v of type %T implements json.Marshaler\n", val, val)
    }
    
在某些情况下，我们必须在包的内部确保某个类型确实满足某个接口的定义。例如类型json.RawMessage，如果它要提供一种定制的JSON格式，就必须实现json.Marshaler接口，但是编译器不会自动对其进行静态类型验证。如果该类型在实现上没有充分满足接口定义，JSON编码器仍然会工作，只不过不是用定制的方式。为了确保接口实现的正确性，可以在包内部，利用空白标识符进行一个全局声明：

    var _ json.Marshaler = (*RawMessage)(nil)
    
在该声明中，赋值语句导致了从*RawMessage到Marshaler的类型转换，这要求*RawMessage必须正确实现了Marshaler接口，该属性将在编译期间被检查。当json.Marshaler接口被修改后，上面的代码将无法正确编译，因而很容易发现错误并及时修改代码。

在这个结构中出现的空白标识符，表示了该声明语句仅仅是为了触发编译器进行类型检查，而非创建任何新的变量。但是，也不需要对所有满足某接口的类型都进行这样的处理。按照约定，这类声明仅当代码中没有其他静态转换时才需要使用，这类情况通常很少出现。

## 内嵌（Embedding）

Go没有提供经典的类型驱动式的派生类概念，但却可以通过内嵌其他类型或接口代码的方式来实现类似的功能。

接口的“内嵌”比较简单。我们之前曾提到过io.Reader和io.Writer这两个接口，以下是它们的实现：

    type Reader interface {
        Read(p []byte) (n int, err error)
    }

    type Writer interface {
        Write(p []byte) (n int, err error)
    }
    
在io包中，还提供了许多其它的接口，它们定义一类可以同时实现几个不同接口的类型。例如io.ReadWriter接口，它同时包含了Read和Write两个接口。尽管可以通过列出Read和Write两个方法的详细声明的方式来定义io.ReadWriter接口，但是以内嵌两个已有接口进行定义的方式会使代码显得更加简洁、直观：

    // ReadWriter is the interface that combines the Reader and Writer interfaces.
    type ReadWriter interface {
        Reader
        Writer
    }
    
这段代码的意义很容易理解：一个ReadWriter类型可以同时完成Reader和Writer的功能，它是这些内嵌接口的联合（这些内嵌接口必须是一组不相干的方法）。接口只能“内嵌”接口类型。

类似的想法也可以应用于结构体的定义，其实现稍稍复杂一些。在bufio包中，有两个结构体类型：bufio.Reader和 bufio.Writer，它们分别实现了io包中的类似接口。bufio包还实现了一个带缓冲的reader/writer类型，实现的方法是将reader和writer组合起来内嵌到一个结构体中：在结构体中，只列出了两种类型，但没有给出对应的字段名。

    // ReadWriter stores pointers to a Reader and a Writer.
    // It implements io.ReadWriter.
    type ReadWriter struct {
        *Reader  // *bufio.Reader
        *Writer  // *bufio.Writer
    }
    
内嵌的元素是指向结构体的指针，因此在使用前，必须将其初始化并指向有效的结构体数据。结构体ReadWriter可以被写作如下形式：

    type ReadWriter struct {
        reader *Reader
        writer *Writer
    }
    
为了使各字段对应的方法能满足io的接口规范，我们还需要提供如下的方法：

    func (rw *ReadWriter) Read(p []byte) (n int, err error) {
        return rw.reader.Read(p)
    }
    
通过对结构体直接进行“内嵌”，我们避免了一些复杂的记录。所有内嵌类型的方法可以不受约束的使用，换句话说，bufio.ReadWriter类型不仅具有bufio.Reader和bufio.Writer两个方法，同时也满足io.Reader，io.Writer和io.ReadWriter这三个接口。

在“内嵌”和“子类型”两种方法间存在一个重要的区别。当我们内嵌一个类型时，该类型的所有方法会变成外部类型的方法，但是当这些方法被调用时，其接收的参数仍然是内部类型，而非外部类型。在本例中，一个bufio.ReadWriter类型的Read方法被调用时，其效果和调用我们刚刚实现的那个Read方法是一样的，只不过前者接收的参数是ReadWriter的reader字段，而不是ReadWriter本身。

“内嵌”还可以用一种更简单的方式表达。下面的例子展示了如何将内嵌字段和一个普通的命名字段同时放在一个结构体定义中。

    type Job struct {
        Command string
        *log.Logger
    }
    
现在，Job类型拥有了Log，Logf以及*log.Logger的其他所有方法。当然，我们可以给Logger提供一个命名字段，但完全没有必要这样做。现在，当初始化结束后，就可以在Job类型上调用日志记录功能了。

    job.Log("starting now...")
    
Logger是结构体Job的一个常规字段，因此我们可以在Job的构造方法中按通用方式对其进行初始化：

    func NewJob(command string, logger *log.Logger) *Job {
        return &Job{command, logger}
    }
    
或者写成下面的形式：

    job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
    
如果我们需要直接引用一个内嵌的字段，那么将该字段的类型名称省略了包名后，就可以作为字段名使用，正如之前在ReaderWriter结构体的Read方法中实现的那样。可以用job.Logger访问Job类型变量job的*log.Logger字段。当需要重新定义Logger的方法时，这种引用方式就变得非常有用了。

    func (job *Job) Logf(format string, args ...interface{}) {
        job.Logger.Logf("%q: %s", job.Command, fmt.Sprintf(format, args...))
    }
    
内嵌类型会引入命字冲突，但是解决冲突的方法也很简单。首先，一个名为X的字段或方法可以将其它同名的类型隐藏在更深层的嵌套之中。假设log.Logger中也包含一个名为Command字段或方法，那么可以用Job的Command字段对其访问进行封装。

其次，同名冲突出现在同一嵌套层里通常是错误的；如果结构体Job本来已经包含了一个名为log.Logger的字段或方法，再继续内嵌log.Logger就是不对的。但假设这个重复的名字并没有在定义之外的地方被使用到，就不会造成什么问题。这个限定为在外部进行类型嵌入修改提供了保护；如果新加入的字段和某个内部类型的字段有命名冲突，但该字段名没有被访问过，那么就不会引起任何问题。