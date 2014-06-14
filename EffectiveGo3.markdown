# Effective Go（三）
@(Go)

## 控制结构

Go的控制结构与C的相似，但是有重要的区别。没有`do`或者`while`循环，只有一个稍微广义的`for`；`switch`更加灵活；`if`和`switch`接受一个像`for`那样可选的初始化语句；`break`和`continue`语句接受一个可选的标号来指定中断或继续什么；还有一些新的控制结构，包括类型`switch`和多路通信复用器（multiway communications multiplexer）`select`。语句也稍微有些不同：没有圆括号，并且控制结构体必须总是由大括号包裹。

### If

Go中，简单的if看起来是这样的：

    if x > 0 {
        return y
    }
    
强制的大括号可以鼓励大家在多行中编写简单的if语句。不管怎样，这是一个好的风格，特别是当控制结构体包含了一条控制语句，例如return或者break。

既然if和switch接受一个初始化语句，那么常见的方式是用来建立一个局部变量。

    if err := file.Chmod(0664); err != nil {
        log.Print(err)
        return err
    }
    
在Go的库中，你会发现当if语句不会流向下一条语句时—也就是说，控制结构体结束于`break`，`continue`，`goto`或者`return`，则不必要的else会被省略掉。

    f, err := os.Open(name)
    if err != nil {
        return err
    }
    codeUsing(f)
    
这个例子是一种常见的情况，代码必须判断一系列的错误条件。顺着代码从上往下，就是成功的控制流的执行方向，在这个过程中处理掉错误条件，这样让代码非常易读。由于错误情况往往会结束于`return`语句，因此在这种情况下不需要有`else`语句。

    f, err := os.Open(name)
    if err != nil {
        return err
    }
    d, err := f.Stat()
    if err != nil {
        f.Close()
        return err
    }
    codeUsing(f, d)

### 重新声明和重新赋值

上一章节的最后一个例子，展示了`:=`短声明形式的工作细节，该声明调用了`os.Open`进行读取。

    f, err := os.Open(name)

该语句声明了两个变量，f和err。几行之后，又调用了f.Stat进行读取，

    d, err := f.Stat()
    
这看起来像是又声明了`d`和`err`。但是，注意`err`在两条语句中都出现了。这种重复是合法的：`err`是在第一条语句中被声明，而在第二条语句中只是被重新赋值。这意味着使用之前已经声明过的`err`变量调用`f.Stat`，只是赋给其一个新的值。

在`:=`声明中，变量v即使已经被声明过，也可以出现，前提是：

* 该声明和v已有的声明在相同的作用域中（如果v已经在外面的作用域里被声明了，则该声明将会创建一个新的变量 §）
* 初始化中相应的值是可以被赋给v的，并且声明中至少有其它一个变量将被声明为一个新的变量
* 这种不寻常的属性纯粹是从实用主义方面来考虑的。例如，这会使得在一个长的if-else链中，很容易地使用单个err值。你会经常看到这种用法。

**值得一提的是，在Go中，函数参数和返回值的作用域与函数体的作用域是相同的，虽然它们在词法上是出现在包裹函数体的大括号外面。**

### For

Go的for循环类似于`C`但又不等同于`C`。它统一了`for`和`while`，并且没有`do-while`。有三种形式，其中只有一种具有分号。

    // Like a C for
    for init; condition; post { }
    
    // Like a C while
    for condition { }
    
    // Like a C for(;;)
    for { }
    
短声明使得在循环中很容易正确的声明索引变量。

    sum := 0
    for i := 0; i < 10; i++ {
        sum += i
    }
    
如果你是在*数组*，*切片*，*字符串*或者*map*上进行循环，或者从*channel*中进行读取，则可以使用`range`子句来管理循环。

    for key, value := range oldMap {
        newMap[key] = value
    }
    
如果你只需要`range`中的第一项（key或者index），则可以丢弃第二个：

    for key := range m {
        if key.expired() {
            delete(m, key)
        }
    }
    
如果你只需要range中的第二项（value），则可以使用空白标识符，一个下划线，来丢弃第一个：

    sum := 0
    for _, value := range array {
        sum += value
    }
    
空白标识符有许多用途，这在后面的章节中会有介绍。

对于字符串，range会做更多的事情，通过解析UTF-8来拆分出单个的Unicode字符。错误的编码会消耗一个字节，产生一个替代的符文（rune）U+FFFD。`rune`是Go的一个内建类型，用于指定一个单独的Unicode字符。详情参见《Go语言编程规范》。循环：

    for pos, char := range "日本\x80語" { // \x80 is an illegal UTF-8 encoding
        fmt.Printf("character %#U starts at byte position %d\n", char, pos)
    }
    
会打印出

    character U+65E5 '日' starts at byte position 0
    character U+672C '本' starts at byte position 3
    character U+FFFD '�' starts at byte position 6
    character U+8A9E '語' starts at byte position 7
    
在循环中，Go没有逗号操作符，并且++和--是语句而不是表达式。因此，如果你想在for中运行多个变量，你需要使用并行赋值（尽管这样会阻碍使用++和--）。

    // Reverse a
    for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
        a[i], a[j] = a[j], a[i]
    }

### Switch

Go的switch要比C的更加通用。表达式不需要为常量，甚至不需要为整数，case是按照从上到下的顺序进行求值，直到找到匹配的。如果switch没有表达式，则对true进行匹配。因此，可以按照语言习惯将`if-else-if-else`链写成一个`switch`。

    func unhex(c byte) byte {
        switch {
        case '0' <= c && c <= '9':
            return c - '0'
        case 'a' <= c && c <= 'f':
            return c - 'a' + 10
        case 'A' <= c && c <= 'F':
            return c - 'A' + 10
        }
        return 0
    }
    
`switch`不会自动从一个case子句跌落到下一个case子句。但是case可以使用逗号分隔的列表。

    func shouldEscape(c byte) bool {
        switch c {
        case ' ', '?', '&', '=', '#', '+', '%':
            return true
        }
        return false
    }
    
虽然和其它类C的语言一样，使用break语句来提前中止switch在Go中几乎不怎么常见。不过，有时候是需要中断包含它的循环，而不是switch。在Go中，可以通过在循环上加一个标号，然后“breaking”到那个标号来达到目的，该例子展示了这些用法。

    Loop:
	    for n := 0; n < len(src); n += size {
		    switch {
		    case src[n] < sizeOne:
			    if validateOnly {
				    break
			    }
			    size = 1
			    update(src[n])

		    case src[n] < sizeTwo:
			    if n+1 >= len(src) {
				    err = errShortInput
				    break Loop
			    }
			    if validateOnly {
				    break
			    }
			    size = 2
			    update(src[n] + src[n+1]<<shift)
		    }
	    }
	    
当然，continue语句也可以接受一个可选的标号，但是只能用于循环。

作为这个章节的结束，这里有一个对字节切片进行比较的程序，使用了两个switch语句：

    // Compare returns an integer comparing the two byte slices,
    // lexicographically.
    // The result will be 0 if a == b, -1 if a < b, and +1 if a > b
    func Compare(a, b []byte) int {
        for i := 0; i < len(a) && i < len(b); i++ {
            switch {
            case a[i] > b[i]:
                return 1
            case a[i] < b[i]:
                return -1
            }
        }
        switch {
        case len(a) > len(b):
            return 1
        case len(a) < len(b):
            return -1
        }
        return 0
    }

### 类型switch

`switch`还可用于获得一个接口变量的动态类型。这种类型switch使用类型断言的语法，在括号中使用关键字`type`。如果`switch`在表达式中声明了一个变量，则变量会在每个子句中具有对应的类型。比较符合语言习惯的方式是在这些case里重用一个名字，实际上是在每个case里声名一个新的变量，其具有相同的名字，但是不同的类型。

    var t interface{}
    t = functionOfSomeType()
    switch t := t.(type) {
    default:
        fmt.Printf("unexpected type %T", t)       // %T prints whatever type t has
    case bool:
        fmt.Printf("boolean %t\n", t)             // t has type bool
    case int:
        fmt.Printf("integer %d\n", t)             // t has type int
    case *bool:
        fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
    case *int:
        fmt.Printf("pointer to integer %d\n", *t) // t has type *int
    }