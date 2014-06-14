# Effective Go（五）
@(Go)[Go]

## 数据

### 使用new进行分配

Go有两个分配原语，内建函数new和make。它们所做的事情有所不同，并且用于不同的类型。这会有些令人混淆，但规则其实很简单。我们先讲下new。这是一个用来分配内存的内建函数，但是不像在其它语言中，它并不初始化内存，只是将其置零。也就是说，new(T)会为T类型的新项目，分配被置零的存储，并且返回它的地址，一个类型为*T的值。在Go的术语中，其返回一个指向新分配的类型为T，值为零的指针。

由于new返回的内存是被置零的，这会有助于你将数据结构设计成，每个类型的零值都可以使用，而不需要进一步初始化。这意味着，数据结构的用户可以使用new来创建数据，并正确使用。例如，bytes.Buffer的文档说道，"Buffer的零值是一个可以使用的空缓冲"。类似的，sync.Mutex没有显式的构造器和Init方法。相反的，sync.Mutex的零值被定义为一个未加锁的互斥。

“零值可用”的属性是可以传递的。考虑这个类型声明。

    type SyncedBuffer struct {
        lock    sync.Mutex
        buffer  bytes.Buffer
    }
    
SyncedBuffer类型的值也可以在分配或者声明之后直接使用。在下一个片段中，p和v都不需要进一步的处理便可以正确地工作。

    p := new(SyncedBuffer)  // type *SyncedBuffer
    var v SyncedBuffer      // type  SyncedBuffer

### 构造器和复合文字

有时候零值并不够好，需要一个初始化构造器（constructor），正如这个源自程序包os的例子。

    func NewFile(fd int, name string) *File {
        if fd < 0 {
            return nil
        }
        f := new(File)
        f.fd = fd
        f.name = name
        f.dirinfo = nil
        f.nepipe = 0
        return f
    }
    
有许多这样的模版。我们可以使用复合文字（composite literal）进行简化，其为一个表达式，在每次求值的时候会创建一个新实例。

    func NewFile(fd int, name string) *File {
        if fd < 0 {
            return nil
        }
        f := File{fd, name, nil, 0}
        return &f
    }
    
注意，不像C，返回一个局部变量的地址是绝对没有问题的；变量关联的存储在函数返回之后依然存在。实际上，使用复合文字的地址也会在每次求值时分配一个新的实例，所以，我们可以将最后两行合并起来。

    return &File{fd, name, nil, 0}
复合文字的域按顺序排列，并且必须都存在。然而，通过field:value显式地为元素添加标号，则初始化可以按任何顺序出现，没有出现的则对应为零值。因此，我们可以写成

    return &File{fd: fd, name: name}
    
作为一种极端情况，如果复合文字根本不包含域，则会为该类型创建一个零值。表达式new(File)和&File{}是等价的。

复合文字还可用于arrays，slices和maps，域标号使用适当的索引或者map key。下面的例子中，不管Enone，Eio和Einval的值是什么，只要它们不同，初始化就可以工作。

    a := [...]string   {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
    s := []string      {Enone: "no error", Eio: "Eio", Einval: "invalid argument"}
    m := map[int]string{Enone: "no error", Eio: "Eio", Einval: "invalid     argument"}

### 使用make进行分配

回到分配的话题。内建函数make(T, args)与new(T)的用途不一样。它只用来创建slice，map和channel，并且返回一个初始化的(而不是置零)，类型为T的值（而不是*T）。之所以有所不同，是因为这三个类型的背后是象征着，对使用前必须初始化的数据结构的引用。例如，slice是一个三项描述符，包含一个指向数据（在数组中）的指针，长度，以及容量，在这些项被初始化之前，slice都是nil的。对于slice，map和channel，make初始化内部数据结构，并准备好可用的值。例如，

    make([]int, 10, 100)
    
分配一个有100个int的数组，然后创建一个长度为10，容量为100的slice结构，并指向数组前10个元素上。（当创建slice时，容量可以省略掉，更多信息参见slice章节。）对应的，new([]int)返回一个指向新分配的，被置零的slice结构体的指针，即指向nilslice值的指针。

这些例子阐释了new和make之间的差别。

    var p *[]int = new([]int)       // allocates slice structure; *p == nil; rarely     useful
    var v  []int = make([]int, 100) // the slice v now refers to a new array of 100 ints

    // Unnecessarily complex:
    var p *[]int = new([]int)
    *p = make([]int, 100, 100)

    // Idiomatic:
    v := make([]int, 100)
    
记住make只用于map，slice和channel，并且不返回指针。要获得一个显式的指针，使用new进行分配，或者显式地使用一个变量的地址。

### 数组

数组可以用于规划内存的精细布局，有时利于避免分配，不过从根本上讲，它们是切片的基本构件，这是下一章节的话题。作为铺垫，这里介绍一下数组。

在Go和C中，数组的工作方式有几个重要的差别。在Go中，

数组是值。将一个数组赋值给另一个，会拷贝所有的元素。
特别是，如果你给函数传递一个数组，其将收到一个数组的拷贝，而不是它的指针。
数组的大小是其类型的一部分。类型[10]int和[20]int是不同的。
数组为值这样的属性，可以很有用处，不过也会有代价；如果你希望类C的行为和效率，可以传递一个数组的指针。

    func Sum(a *[3]float64) (sum float64) {
        for _, v := range *a {
            sum += v
        }
        return
    }

    array := [...]float64{7.0, 8.5, 9.1}
    x := Sum(&array)  // Note the explicit address-of operator
    
不过，这种风格并不符合Go的语言习惯。相反的，应该使用切片。

### 切片

切片（slice）对数组进行封装，提供了一个针对串行数据，更加通用，强大和方便的接口。除了像转换矩阵这样具有显式维度的项，Go中大多数的数组编程都是通过切片完成，而不是简单数组。

切片持有对底层数组的引用，如果你将一个切片赋值给另一个，二者都将引用同一个数组。如果函数接受一个切片参数，那么其对切片的元素所做的改动，对于调用者是可见的，好比是传递了一个底层数组的指针。因此，Read函数可以接受一个切片参数，而不是一个指针和一个计数；切片中的长度已经设定了要读取的数据的上限。这是程序包os中，File类型的Read方法的签名：

func (file *File) Read(buf []byte) (n int, err error)
该方法返回读取的字节数和一个错误值，如果存在的话。要读取一个大缓冲b中的前32个字节，可以将缓冲进行切片（这里是动词）。

    n, err := f.Read(buf[0:32])
    
这种切片很常见，而且有效。实际上，如果先不考虑效率，下面的片段也可以读取缓冲的前32个字节。

    var n int
    var err error
    for i := 0; i < 32; i++ {
        nbytes, e := f.Read(buf[i:i+1])  // Read one byte.
        if nbytes == 0 || e != nil {
            err = e
            break
        }
        n += nbytes
    }
    
只要还符合底层数组的限制，切片的长度就可以进行改变；直接将其赋值给它自己的切片。切片的容量，可以通过内建函数cap访问，告知切片可以获得的最大长度。这里有一个函数可以为切片增加数据。如果数据超出了容量，则切片会被重新分配，然后返回新产生的切片。该函数利用了一个事实，即当用于nil切片时，len和cap是合法的，并且返回0.

    func Append(slice, data[]byte) []byte {
        l := len(slice)
        if l + len(data) > cap(slice) {  // reallocate
            // Allocate double what's needed, for future growth.
            newSlice := make([]byte, (l+len(data))*2)
            // The copy function is predeclared and works for any slice type.
            copy(newSlice, slice)
            slice = newSlice
        }
        slice = slice[0:l+len(data)]
        for i, c := range data {
            slice[l+i] = c
        }
        return slice
    }
    
我们必须在后面返回切片，尽管Append可以修改slice的元素，切片本身（持有指针，长度和容量的运行时数据结构）是按照值传递的。

为切片增加元素的想法非常有用，以至于实现了一个内建的append函数。不过，要理解该函数的设计，我们还需要一些更多的信息，所以我们放到后面再说。

### 二维切片

Go的数组和切片都是一维的。要创建等价的二维数组或者切片，需要定义一个数组的数组或者切片的切片，类似这样：

    type Transform [3][3]float64  // A 3x3 array, really an array of arrays.
    type LinesOfText [][]byte     // A slice of byte slices.
    
因为切片是可变长度的，所以可以将每个内部的切片具有不同的长度。这种情况很常见，正如我们的LinesOfText例子中：每一行都有一个独立的长度。

    text := LinesOfText{
	    []byte("Now is the time"),
	    []byte("for all good gophers"),
	    []byte("to bring some fun to the party."),
    }
    
有时候是需要分配一个二维切片的，例如这种情况可见于当扫描像素行的时候。有两种方式可以实现。一种是独立的分配每一个切片；另一种是分配单个数组，为其 指定单独的切片们。使用哪一种方式取决于你的应用。如果切片们可能会增大或者缩小，则它们应该被单独的分配以避免覆写了下一行；如果不会，则构建单个分配 会更加有效。作为参考，这里有两种方式的框架。首先是一次一行：

    // Allocate the top-level slice.
    picture := make([][]uint8, YSize) // One row per unit of y.
    // Loop over the rows, allocating the slice for each row.
    for i := range picture {
	    picture[i] = make([]uint8, XSize)
    }
然后是分配一次，被切片成多行：

    // Allocate the top-level slice, the same as before.
    picture := make([][]uint8, YSize) // One row per unit of y.
    // Allocate one large slice to hold all the pixels.
    pixels := make([]uint8, XSize*YSize) // Has type []uint8 even though     picture is [][]uint8.
    // Loop over the rows, slicing each row from the front of the remaining pixels slice.
    for i := range picture {
	    picture[i], pixels = pixels[:XSize], pixels[XSize:]
    }

### Maps

Map是一种方便，强大的内建数据结构，其将一个类型的值（key）与另一个类型的值（element或value） 关联一起。key可以为任何定义了等于操作符的类型，例如整数，浮点和复数，字符串，指针，接口（只要其动态类型支持等于操作），结构体和数组。切片不能 作为map的key，因为它们没有定义等于操作。和切片类似，map持有对底层数据结构的引用。如果将map传递给函数，其对map的内容做了改变，则这 些改变对于调用者是可见的。

Map可以使用通常的复合文字语法来构建，使用分号分隔key和value，这样很容易在初始化的时候构建它们。

    var timeZone = map[string]int{
        "UTC":  0*60*60,
        "EST": -5*60*60,
        "CST": -6*60*60,
        "MST": -7*60*60,
        "PST": -8*60*60,
    }
    
赋值和获取map的值，在语法上看起来跟数组和切片类似，只不过索引不需要为一个整数。

    offset := timeZone["EST"]
    
尝试使用一个不在map中的key来获取map值，将会返回map中元素相应类型的零值。例如，如果map包含的是整数，则查找一个不存在的key将会返回0。可以通过值类型为bool的map来实现一个集合。将map项设置为true，来将值放在集合中，然后通过简单的索引来进行测试。

    attended := map[string]bool{
        "Ann": true,
        "Joe": true,
        ...
    }

    if attended[person] { // will be false if person is not in the map
        fmt.Println(person, "was at the meeting")
    }
    
有时你需要区分开没有的项和值为零的项。是否有一个项为"UTC"，或者由于其根本不在map中，所以为空字符串？你可以通过多赋值的形式来进行辨别。

    var seconds int
    var ok bool
    seconds, ok = timeZone[tz]
    
这被形象的称作为“comma ok”用法。在这个例子中，如果tz存在，seconds将被设置为适当的值，ok将为真；如果不存在，seconds将被设置为零，ok将为假。这有个例子，并增加了一个友好的错误报告：

    func offset(tz string) int {
        if seconds, ok := timeZone[tz]; ok {
            return seconds
        }
        log.Println("unknown time zone:", tz)
        return 0
    }
    
如果只测试是否在map中存在，而不关心实际的值，你可以将通常使用变量的地方换成空白标识符（_）

    _, present := timeZone[tz]
    
要删除一个map项，使用delete内建函数，其参数为map和要删除的key。即使key已经不在map中，这样做也是安全的。

    delete(timeZone, "PDT")  // Now on Standard Time

### 打印输出

Go中的格式化打印使用了与C中printf家族类似的风格，不过更加丰富和通用。这些函数位于fmt程序包中，并具有大写的名字：fmt.Printf，fmt.Fprintf，fmt.Sprintf等等。字符串函数（Sprintf等）返回一个字符串，而不是填充到提供的缓冲里。

你不需要提供一个格式串。对每个Printf，Fprintf和Sprintf，都有另外一对相应的函数，例如Print和Println。这些函数不接受格式串，而是为每个参数生成一个缺省的格式。Println版本还会在参数之间插入一个空格，并添加一个换行，而Print版本只有当两边的操作数都不是字符串的时候才增加一个空格。在这个例子中，每一行都会产生相同的输出。

    fmt.Printf("Hello %d\n", 23)
    fmt.Fprint(os.Stdout, "Hello ", 23, "\n")
    fmt.Println("Hello", 23)
    fmt.Println(fmt.Sprint("Hello ", 23))
    
格式化打印函数fmt.Fprint等，接受的第一个参数为任何一个实现了io.Writer接口的对象；变量os.Stdout和os.Stderr是常见的实例。

接下来这些就和C不同了。首先，数字格式，像%d，并不接受正负号和大小的标记；相反的，打印程序使用参数的类型来决定这些属性。

    var x uint64 = 1<<64 - 1
    fmt.Printf("%d %x; %d %x\n", x, x, int64(x), int64(x))
    
会打印出

    18446744073709551615 ffffffffffffffff; -1 -1
    
如果只是想要缺省的转换，像十进制整数，你可以使用通用格式%v（代表“value”）；这正是Print和Println所产生的结果。而且，这个格式可以打印任意的的值，甚至是数组，切片，结构体和map。这是一个针对前面章节中定义的时区map的打印语句

    fmt.Printf("%v\n", timeZone)  // or just fmt.Println(timeZone)
    
其会输出

    map[CST:-21600 PST:-28800 EST:-18000 UTC:0 MST:-25200]
    
当然，map的key可能会按照任意顺序被输出。当打印一个结构体时，带修饰的格式%+v会将结构体的域使用它们的名字进行注解，对于任意的值，格式%#v会按照完整的Go语法打印出该值。

    type T struct {
        a int
        b float64
        c string
    }
    t := &T{ 7, -2.35, "abc\tdef" }
    fmt.Printf("%v\n", t)
    fmt.Printf("%+v\n", t)
    fmt.Printf("%#v\n", t)
    fmt.Printf("%#v\n", timeZone)
    
会打印出

    &{7 -2.35 abc   def}
    &{a:7 b:-2.35 c:abc     def}
    &main.T{a:7, b:-2.35, c:"abc\tdef"}
    map[string] int{"CST":-21600, "PST":-28800, "EST":-18000, "UTC":0, "MST":-25200}
    
（注意符号&）还可以通过%q来实现带引号的字符串格式，用于类型为string或[]byte的值。格式%#q将尽可能的使用反引号。（格式%q还用于整数和符文，产生一个带单引号的符文常量。）还有，%x用于字符串，字节数组和字节切片，以及整数，生成一个长的十六进制字符串，并且如果在格式中有一个空格（% x），其将会在字节中插入空格。

另一个方便的格式是%T，其可以打印出值的类型。

    fmt.Printf("%T\n", timeZone)
    
会打印出

    map[string] int
    
如果你想控制自定义类型的缺省格式，只需要对该类型定义一个签名为String() string的方法。对于我们的简单类型T，看起来可能是这样的。

    func (t *T) String() string {
        return fmt.Sprintf("%d/%g/%q", t.a, t.b, t.c)
    }
    fmt.Printf("%v\n", t)
    
会按照如下格式打印

    7/-2.35/"abc\tdef"
    
（如果你需要打印类型为T的值，同时需要指向T的指针，那么String的接收者必须为值类型的；这个例子使用了指针，是因为这对于结构体类型更加有效和符合语言习惯。更多信息参见下面的章节pointers vs. value receivers）

我们的String方法可以调用Sprintf，是因为打印程序是完全可重入的，并且可以按这种方式进行包装。然而，对于这种方式，有一个重要的细节需要明白：不要将调用Sprintf的String方法构造成无穷递归。如果Sprintf调用尝试将接收者直接作为字符串进行打印，就会导致再次调用该方法，发生这样的情况。这是一个很常见的错误，正如这个例子所示。

    type MyString string

    func (m MyString) String() string {
        return fmt.Sprintf("MyString=%s", m) // Error: will recur forever.
    }
    
这也容易修改：将参数转换为没有方法函数的，基本的字符串类型。

    type MyString string
    func (m MyString) String() string {
        return fmt.Sprintf("MyString=%s", string(m)) // OK: note conversion.
    }
    
在初始化章节，我们将会看到另一种避免该递归的技术。

另一种打印技术，是将一个打印程序的参数直接传递给另一个这样的程序。Printf的签名使用了类型...interface{}作为最后一个参数，来指定在格式之后可以出现任意数目的（任意类型的）参数。

    func Printf(format string, v ...interface{}) (n int, err error) {
    
在函数Printf内部，v就像是一个类型为[]interface{}的变量，但是如果其被传递给另一个可变参数的函数，其就像是一个正常的参数列表。这里有一个对我们上面用到的函数log.Println的实现。其将参数直接传递给fmt.Sprintln来做实际的格式化。

    // Println prints to the standard logger in the manner of fmt.Println.
    func Println(v ...interface{}) {
        std.Output(2, fmt.Sprintln(v...))  // Output takes parameters (int, string)
    }
    
我们在嵌套调用Sprintln中v的后面使用了...来告诉编译器将v作为一个参数列表；否则，其会只将v作为单个切片参数进行传递。

除了我们这里讲到的之外，还有很多有关打印的技术。详情参见godoc文档中对fmt的介绍。

顺便说下，...参数可以为一个特定的类型，例如...int，可以用于最小值函数，来选择整数列表中的最小值：

    func Min(a ...int) int {
        min := int(^uint(0) >> 1)  // largest int
        for _, i := range a {
            if i < min {
                min = i
            }
        }
        return min
    }

### append内建函数

现在，我们需要解释下append内建函数的设计了。append的签名与我们上面定制的Append函数不同。简略地讲，类似于这样：

    func append(slice []T, elements ...T) []T
    
其中T为任意给定类型的占位符。你在Go中是无法写出一个类型T由调用者来确定的函数。这就是为什么append是内建的：它需要编译器的支持。

append所做的事情是将元素添加到切片的结尾，并返回结果。需要返回结果，是因为和我们手写的Append一样，底层的数组可能会改变。这个简单的例子

    x := []int{1,2,3}
    x = append(x, 4, 5, 6)
    fmt.Println(x)
    
会打印出[1 2 3 4 5 6]。所以append的工作方式有点像Printf，搜集任意数目的参数。

但是，如果我们想按照我们的Append那样做，给切片增加一个切片，那么该怎么办？简单：在调用点使用...，就像我们在上面调用Output时一样。这个片段会产生和上面相同的输出。

    x := []int{1,2,3}
    y := []int{4,5,6}
    x = append(x, y...)
    fmt.Println(x)
    
如果没有...，则会因为类型错误而无法编译；y不是int型的。