# Effective Go （十）
@(Go)[Go]


## 错误

向调用者返回某种形式的错误信息是库历程必须提供的一项功能。通过前面介绍的函数多返回值的特性，Go中的错误信息可以很容易同正常情况下的返回值一起返回给调用者。方便起见，错误通常都用内置接口error类型表示。

    type error interface {
        Error() string
    }
    
库开发人员可以通过实现该接口来丰富其内部功能，使其不仅能够呈现错误本身，还能提供更多的上下文信息。举例来说，os.Open函数会返回os.PathError错误。

    // PathError records an error and the operation and
    // file path that caused it.
    type PathError struct {
        Op string    // "open", "unlink", etc.
        Path string  // The associated file.
        Err error    // Returned by the system call.
    }

    func (e *PathError) Error() string {
        return e.Op + " " + e.Path + ": " + e.Err.Error()
    }
    
PathError的Error方法会生成类似下面给出的错误信息：

    open /etc/passwx: no such file or directory
    
这条错误信息包括了足够的信息：出现异常的文件名，操作类型，以及操作系统返回的错误信息等，因此即使它冒出来的时候距离真正错误发生时刻已经间隔了很 久，也不会给调试分析带来很大困难，比直接输出一句“no such file or directory” 要友好的多。

如果可能，描述错误的字符串应该能指明错误发生的原始位置，比如在前面加上一些诸如操作名称或包名称的前缀信息。例如在image包中，用来输出未知图片类型的错误信息的格式是这样的：“image: unknown format” 。

对于需要精确分析错误信息的调用者，可以通过类型开关或类型断言的方式查看具体的错误并深入错误的细节。就PathErrors类型而言，这些细节信息包含在一个内部的Err字段中，可以被用来进行错误恢复。

    for try := 0; try < 2; try++ {
        file, err = os.Create(filename)
        if err == nil {
            return
        }
        if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
            deleteTempFiles()  // Recover some space.
            continue
        }
        return
    }
    
在上面例子中，第二个if语句是另一种形式的类型断言。如该断言失败，ok的值将为false且e的值为nil。如果断言成功，则ok值为true，说明当前的错误，也就是e，属于*os.PathError类型，因而可以进一步获取更多的细节信息。

### 严重故障（Panic）

通常来说，向调用者报告错误的方式就是返回一个额外的error变量： Read方法就是一个很好的例子；该方法返回一个字节计数值和一个error变量。但是对于那些不可恢复的错误，比如错误发生后程序将不能继续执行的情况，该如何处理呢？

为了解决上述问题，Go语言提供了一个内置的panic方法，用来创建一个运行时错误并结束当前程序（关于退出机制，下一节还有进一步介绍）。该函数接受一个任意类型的参数，并在程序挂掉之前打印该参数内容，通常我们会选择一个字符串作为参数。方法panic还适用于指示一些程序中的不可达状态，比如从一个无限循环中退出。

    // A toy implementation of cube root using Newton's method.
    func CubeRoot(x float64) float64 {
        z := x/3   // Arbitrary initial value
        for i := 0; i < 1e6; i++ {
            prevz := z
            z -= (z*z*z-x) / (3*z*z)
            if veryClose(z, prevz) {
                return z
            }
        }
        // A million iterations has not converged; something is wrong.
        panic(fmt.Sprintf("CubeRoot(%g) did not converge", x))
    }
    
以上仅仅提供一个应用的示例，在实际的库设计中，应尽量避免使用panic。如果程序错误可以以某种方式掩盖或是绕过，那么最好还是继续执行而不是让整个程序终止。不过还是有一些反例的，比方说，如果库历程确实没有办法正确完成其初始化过程，那么触发panic退出可能就是一种更加合理的方式。

    var user = os.Getenv("USER")

    func init() {
        if user == "" {
            panic("no value for $USER")
        }
    }

### 恢复（Recover）

对于一些隐式的运行时错误，如切片索引越界、类型断言错误等情形下，panic方法就会被调用，它将立刻中断当前函数的执行，并展开当前Goroutine的调用栈，依次执行之前注册的defer函数。当栈展开操作达到该Goroutine栈顶端时，程序将终止。但这时仍然可以使用Go的内建recover方法重新获得Goroutine的控制权，并将程序恢复到正常执行的状态。

调用recover方法会终止栈展开操作并返回之前传递给panic方法的那个参数。由于在栈展开过程中，只有defer型函数会被执行，因此recover的调用必须置于defer函数内才有效。

在下面的示例应用中，调用recover方法会终止server中失败的那个Goroutine，但server中其它的Goroutine将继续执行，不受影响。

    func server(workChan <-chan *Work) {
        for work := range workChan {
            go safelyDo(work)
        }
    }

    func safelyDo(work *Work) {
        defer func() {
            if err := recover(); err != nil {
                log.Println("work failed:", err)
            }
        }()
        do(work)
    }
    
在这里例子中，如果do(work)调用发生了panic，则其结果将被记录且发生错误的那个Goroutine将干净的退出，不会干扰其他Goroutine。你不需要在defer指示的闭包中做别的操作，仅需调用recover方法，它将帮你搞定一切。

只有直接在defer函数中调用recover方法，才会返回非nil的值，因此defer函数的代码可以调用那些本身使用了panic和recover的库函数而不会引发错误。还用上面的那个例子说明：safelyDo里的defer函数在调用recover之前可能调用了一个日志记录函数，而日志记录程序的执行将不受panic状态的影响。

有了错误恢复的模式，do函数及其调用的代码可以通过调用panic方法，以一种很干净的方式从错误状态中恢复。我们可以使用该特性为那些复杂的软件实现更加简洁的错误处理代码。让我们来看下面这个例子，它是regexp包的一个简化版本，它通过调用panic并传递一个局部错误类型来报告“解析错误”（Parse Error）。下面的代码包括了Error类型定义，error处理方法以及Compile函数：

    // Error is the type of a parse error; it satisfies the error interface.
    type Error string
    func (e Error) Error() string {
        return string(e)
    }

    // error is a method of *Regexp that reports parsing errors by
    // panicking with an Error.
    func (regexp *Regexp) error(err string) {
        panic(Error(err))
    }

    // Compile returns a parsed representation of the regular expression.
    func Compile(str string) (regexp *Regexp, err error) {
        regexp = new(Regexp)
        // doParse will panic if there is a parse error.
        defer func() {
            if e := recover(); e != nil {
                regexp = nil    // Clear return value.
                err = e.(Error) // Will re-panic if not a parse error.
            }
        }()
        return regexp.doParse(str), nil
    }
    
如果doParse方法触发panic，错误恢复代码会将返回值置为nil—因为defer函数可以修改命名的返回值变量；然后，错误恢复代码会对返回的错误类型进行类型断言，判断其是否属于Error类型。如果类型断言失败，则会引发运行时错误，并继续进行栈展开，最后终止程序 —— 这个过程将不再会被中断。类型检查失败可能意味着程序中还有其他部分触发了panic，如果某处存在索引越界访问等，因此，即使我们已经使用了panic和recover机制来处理解析错误，程序依然会异常终止。

有了上面的错误处理过程，调用error方法（由于它是一个类型的绑定的方法，因而即使与内建类型error同名，也不会带来什么问题，甚至是一直更加自然的用法）使得“解析错误”的报告更加方便，无需费心去考虑手工处理栈展开过程的复杂问题。

    if pos == 0 {
        re.error("'*' illegal at start of expression")
    }
    
上面这种模式的妙处在于，它完全被封装在模块的内部，Parse方法将其内部对panic的调用隐藏在error之中；而不会将panics信息暴露给外部使用者。这是一个设计良好且值得学习的编程技巧。

顺便说一下，当确实有错误发生时，我们习惯采取的“重新触发panic”（re-panic）的方法会改变panic的值。但新旧错误信息都会出现在崩溃 报告中，引发错误的原始点仍然可以找到。所以，通常这种简单的重新触发panic的机制就足够了—所有这些错误最终导致了程序的崩溃—但是如果只想显示最 初的错误信息的话，你就需要稍微多写一些代码来过滤掉那些由重新触发引入的多余信息。这个功能就留给读者自己去实现吧！

### 一个web服务示例

让我们以一个完整的Go程序示例 —— 一个web服务 —— 来作为这篇文档的结尾。事实上，这个例子其实是一类“Web re-server”，也就是说它其实是对另一个Web服务的封装。谷歌公司提供了一个用来自动将数据格式化为图表或图形的在线服务，其网址是：http://chart.apis.google.com。 这个服务使用起来其实有点麻烦 —— 你需要把数据添加到URL中作为请求参数，因此不易于进行交互操作。我们现在的这个程序会为用户提供一个更加友好的界面来处理某种形式的数据：对于给定的 一小段文本数据，该服务将调用图标在线服务来产生一个QR码，它用一系列二维方框来编码文本信息。可以用手机摄像头扫描该QR码并进行交互操作，比如将 URL地址编码成一个QR码，你就省去了往手机里输入这个URL地址的时间。

下面是完整的程序代码，后面会给出详细的解释。

    package main

    import (
        "flag"
        "html/template"
        "log"
        "net/http"
    )

    var addr = flag.String("addr", ":1718", "http service address") // Q=17,     R=18

    var templ = template.Must(template.New("qr").Parse(templateStr))

    func main() {
        flag.Parse()
        http.Handle("/", http.HandlerFunc(QR))
        err := http.ListenAndServe(*addr, nil)
        if err != nil {
            log.Fatal("ListenAndServe:", err)
        }
    }

    func QR(w http.ResponseWriter, req *http.Request) {
        templ.Execute(w, req.FormValue("s"))
    }

    const templateStr = `
    <html>
    <head>
    <title>QR Link Generator</title>
    </head>
    <body>
    {{if .}}
    <img src="http://chart.apis.google.com/chart?    chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />
    <br>
    {{.}}
    <br>
    <br>
    {{end}}
    <form action="/" name=f method="GET"><input maxLength=1024     size=70
    name=s value="" title="Text to QR Encode"><input type=submit
    value="Show QR" name=qr>
    </form>
    </body>
    </html>
    `
    
main函数之前的部分很容易理解。包flag用来构建我们这个服务默认的HTTP端口。从模板变量templ开始进入了比较好玩的部分，它的功能是用来构建一个HTML模板，该模板被我们的服务器处理并用来显式页面信息；我们后面还会看到更多细节。

main函数使用我们之前介绍的机制来解析flag，并将函数QR绑定到我们服务的根路径。然后调用http.ListenAndServe方法启动服务；该方法将在服务器运行过程中一直处于阻塞状态。

QR函数用来接收包含格式化数据的请求信息，并以该数据s为参数对模板进行实例化操作。

模板包html/template的功能非常强大；上述程序仅仅触及其冰山一角。本质上说，它会根据传入templ.Execute方法的参数，在本例中是格式化数据，在后台替换相应的元素并重新生成HTML文本。在模板文本（templateStr）中，双大括号包裹的区域意味着需要进行模板替换动作。在{{if .}}和{{end}}之间的部分只有在当前数据项，也就是.，不为空时才被执行。也就是说，如果对应字符串为空，内部的模板信息将被忽略。

代码片段{{.}}表示在页面中显示传入模板的数据 —— 也就是查询字符串本身。HTML模板包会自动提供合适的处理方式，使得文本可以安全的显示。

模板串的剩余部分就是将被加载显示的普通HTML文本。如果你觉得这个解释太笼统了，可以进一步参考Go文档中，关于模板包的深入讨论。

看，仅仅用了很少量的代码加上一些数据驱动的HTML文本，你就搞定了一个很有用的web服务。这就是Go语言的牛X之处：用很少的一点代码就能实现很强大的功能。