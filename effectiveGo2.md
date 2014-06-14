# Effective Go（二）
@(Go)[Go]

- 注释
- 名字
  - 程序包名
  - Get方法
  - 接口名
  - 混合大小写
- 分号


## 注释

Go提供了C风格的块注释`/* */`和C++风格的行注释`//`。通常为行注释；块注释大多数作为程序包的注释，但也可以用于一个表达式中，或者用来注释掉一大片代码。

程序，同时又是网络服务器：**godoc**。用来处理Go源文件，抽取有关程序包内容的文档。在顶层声明之前出现，并且中间没有换行的注释，会随着声明一起被抽取，作为该项的解释性文本。这些注释的本质和风格决定了godoc所产生文档的质量。

每个程序包都应该有一个包注释，一个位于package子句之前的块注释。对于有多个文件的程序包，包注释只需要出现在一个文件中，任何一个文件都可以。包注释应该用来介绍该程序包，并且提供与整个程序包相关的信息。它将会首先出现在godoc页面上，并会建立后续的详细文档。

    /*
    Package regexp implements a simple library for regular expressions.
    
    The syntax of the regular expressions accepted is:
    
        regexp:
            concatenation { '|' concatenation }
        concatenation:
            { closure }
        closure:
            term [ '*' | '+' | '?' ]
        term:
            '^'
            '$'
            '.'
            character
            '[' [ '^' ] character-ranges ']'
            '(' regexp ')'
    */
    package regexp

如果程序包很简单，则包注释可以非常简短。

    // Package path implements utility routines for
    // manipulating slash-separated filename paths.

注释不需要额外的格式，例如星号横幅。生成的输出甚至可能会不按照固定宽度的字体进行展现，所以不要依靠用空格进行对齐—godoc，gofmt会处理这些事情。注释是不作解析的普通文本，所以HTML和其它注解，例如`_this_`，将会被逐字地复制。对于缩进的文本，godoc确实会进行调整，来按照固定宽度的字体进行显示，这适合于程序片段。[fmt package](http://golang.org/pkg/fmt) 包的注释就使用了这种方式来获得良好的效果。

根据上下文，godoc可能不会重新格式化注释，所以要确保它们看起来非常直接：使用正确的拼写，标点，以及语句结构，将较长的行进行折叠，等等。

在程序包里面，任何直接位于顶层声明之前的注释，都会作为该声明的文档注释。程序中每一个被导出的（大写的）名字，都应该有一个文档注释。

文档注释作为完整的语句可以工作的最好，可以允许各种自动化的展现。第一条语句应该为一条概括语句，并且使用被声明的名字作为开头。

    // Compile parses a regular expression and returns, if successful, a Regexp
    // object that can be used to match against text.
    func Compile(str string) (regexp *Regexp, err error) {

如果都是使用名字来起始一个注释，那么就可以通过grep来处理godoc的输出。设想你正在查找正则表达式的解析函数，但是不记得名字“Compile”了，那么，你运行了命令

    $ godoc regexp | grep parse

如果程序包中所有的文档注释都起始于"This function..."，那么grep将无法帮助你想起这个名字。但是，因为程序包文档是使用名字作为起始注释，所以你将会看到类似这样的信息，这将使你想起你要查找的单词。

    $ godoc regexp | grep parse

        Compile parses a regular expression and returns, if successful, a Regexp
        parsed. It simplifies safe initialization of global variables holding
        cannot be parsed. It simplifies safe initialization of global variables
    $
    
Go的声明语法允许对声明进行组合。单个的文档注释可以用来介绍一组相关的常量或者变量。由于展现的是整个声明，这样的注释通常非常简洁。
    
    // Error codes returned by failures to parse an expression.
    var (
        ErrInternal      = errors.New("regexp: internal error")
        ErrUnmatchedLpar = errors.New("regexp: unmatched '('")
        ErrUnmatchedRpar = errors.New("regexp: unmatched ')'")
        ...
    )

分组还可以用来指示各项之间的关系，例如一组实际上由一个互斥进行保护的变量。

    var (
        countLock   sync.Mutex
        inputCount  uint32
        outputCount uint32
        errorCount  uint32
    )
    
## 名字

和其它编程语言一样，名字在Go中是非常重要的。它们甚至还具有语义的效果：一个名字在程序包之外的可见性是由它的首字符是否为大写来确定的。因此，值得花费一些时间来讨论Go程序中的命名约定。

### 程序包名

当一个程序包被导入，程序包名便可以用来访问它的内容。

    import "bytes"

当执行上述语句之后，导入的程序便可以访问到bytes.Buffer。如果每个使用程序包的人都可以使用相同的名字来引用它的内容，这会是很有帮助的。这意味着程序包名要很好：简短，简明，能引起共识的。按照惯例，程序包名使用小写，一个单词的名字；不需要使用下划线或者混合大小写。要力求简短，因为每个使用你的程序包的人都将敲入那个名字。不用担心会与先前的有冲突。程序包名只是导入的缺省名字；其不需要在所有源代码中是唯一的。对于很少出现的冲突情况下，导入的程序包可以选择一个不同的名字在本地使用。不管怎样，冲突是很少的，因为导入的文件名确定了所要使用的程序包。

另一种约定是，程序包名为其源路径的文件名；在src/pkg/encoding/base64中的程序包，是作为"encoding/base64"来导入的，但是名字为base64，而不是`encoding_base64`或`encodingBase64`。

程序包的导入者将使用名字来引用其内容，因此在程序包中被导出的名字可以利用这个事实来避免口吃现象。（不要使用import .标记，这将会简化那些必须在程序包之外运行，本不应该避免的测试）例如，在bufio程序包中的带缓冲的读入类型叫做Reader，而不是BufReader，因为用户看到的是bufio.Reader，一个清晰，简明的名字。而且，因为被导入的实体总是通过它们的程序包名来寻址，所以bufio.Reader和io.Reader并不冲突。类似的，为ring.Ring创建一个新实例的函数，在Go中是定义一个构造器—通常会被叫做NewRing，但是由于Ring是程序包导出的唯一类型，由于程序包叫做ring，所以它只叫做New。这样，程序包的客户将会看到ring.New。使用程序包结构可以帮助你选择好的名字。

另一个小例子是once.Do；`once.Do(setup)`很好读，写成`once.DoOrWaitUntilDone(setup)`并不会有所改善。长名字并不会自动使得事物更易理解。具有帮助性的文档注释往往会比格外长的名字更有用。

### Get方法

Go不提供对Get方法和Set方法的自动支持。你自己提供Get方法和Set方法是没有错的，通常这么做是合适的。但是，在Get方法的名字前加上`Get`，是不符合语言习惯的，并且也没有必要。如果你有一个域叫做owner（小写，不被导出），则Get方法应该叫做Owner（大写，被导出），而不是GetOwner。对于要导出的，使用大写名字，提供了区别域和方法的钩子。Set方法，如果需要，则可以叫做SetOwner。这些名字在实际中都很好读：

owner := obj.Owner()
if owner != user {
    obj.SetOwner(user)
}

### 接口名

按照约定，单个方法的接口使用方法名加上“er”后缀来命名，或者类似的修改来构造一个施动者名词：Reader，Writer，Formatter，CloseNotifier等。

有许多这样的名字，最有效的方式就是理解它们，以及它们所体现的函数名字。Read，Write，Close，Flush，String等，都具有规范的签名和含义。为了避免混淆，不要为你的方法使用这些名字，除非其具有相同的签名和含义。反过来讲，如果你的类型实现了一个和众所周知的类型具有相同含义的方法，那么就使用相同的名字和签名；例如，为你的字符串转换方法起名为String，而不是ToString。

### 混合大小写

最后，Go约定使用MixedCaps或者mixedCaps的形式，而不是下划线来书写多个单词的名字。

## 分号

类似于C，Go的规范语法是使用分号来终结语句的。但是与C不同的是，这些分号并不在源码中出现。词法分析器会在扫描时，使用简单的规则自动插入分号，因此输入文本中大部分是没有分号的。

规则是这样的，如果在换行之前的最后一个符号为一个标识符（包括像int和float64这样的单词），一个基本的文字，例如数字或者字符串常量，或者如下的一个符号

> `break` `continue` `fallthrough` `return` `++` `--` `)` `}`

则词法分析器总是会在符号之后插入一个分号。这可以总结为**如果换行出现在可以结束一条语句的符号之后，则插入一个分号**。

紧挨着右大括号之前的分号也可以省略掉，这样，语句

    go func() { for { dst <- <-src } }()
    
就不需要分号。地道的Go程序只在for循环子句中使用分号，来分开初始化，条件和继续执行，这些元素。分号也用于在一行中分开多条语句，这也是你编写代码应该采用的方式。

分号插入规则所导致的一个结果是，你不能将控制结构（if，for，switch或select）的左大括号放在下一行。如果这样做，则会在大括号之前插入一个分号，这将会带来不是想要的效果。应该这样编写

    if i < f() {
        g()
    }
    
而不是这样

    if i < f()  // wrong!
    {           // wrong!
        g()
    }

