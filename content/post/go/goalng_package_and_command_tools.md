---
title: "goalng包和命令工具"
date: "2019-06-29T00:00:00+08:00"
tags: 
- go
showToc: true
---


## 包简介

任何包系统设计的目的都是为了简化大型程序的设计和维护工作，通过将一组相关的特性放进一个独立的单元以便于理解和更新，在每个单元更新的同时保持和程序中其它单元的相对独立性。这种模块化的特性允许每个包可以被其它的不同项目共享和重用，在项目范围内、甚至全球范围统一的分发和复用。

每个包一般都定义了一个不同的名字空间用于它内部的每个标识符的访问。每个名字空间关联到一个特定的包，让我们给类型、函数等选择简短明了的名字，这样可以在使用它们的时候减少和其它部分名字的冲突。

每个包还通过控制包内名字的可见性和是否导出来实现封装特性。通过限制包成员的可见性并隐藏包API的具体实现，将允许包的维护者在不影响外部包用户的前提下调整包的内部实现。通过限制包内变量的可见性，还可以强制用户通过某些特定函数来访问和更新内部变量，这样可以保证内部变量的一致性和并发时的互斥约束。

## 包的导入

每个包是由一个全局唯一的字符串所标识的导入路径定位。出现在import语句中的导入路径也是字符串。

```go
import (
    "fmt"
    "math/rand"
    "encoding/json"
    "golang.org/x/net/html"
    "github.com/go-sql-driver/mysql"
)
```

## 包声明

在每个Go语言源文件的开头都必须有包声明语句。包声明语句的主要目的是确定当前包被其它包导入时默认的标识符（也称为包名）。

通常来说，默认的包名就是包导入路径名的最后一段，因此即使两个包的导入路径不同，它们依然可能有一个相同的包名。例如，math/rand包和golang.org/x/exp/rand包的包名都是rand。

```go
package main
import (
	"fmt"
	"math/rand"  
)
	// 导入rand包 用rand包里的方法函数等
func main() {
	fmt.Println(rand.Int63n(10000))  // 9410
}
```
```go
package main
import (
	"fmt"
	"golang.org/x/exp/rand"
)
func main() {
	fmt.Println(rand.Int63n(10000)) // 9351
}

```

## 导入声明

如果我们想同时导入两个有着名字相同的包，例如math/rand包和golang.org/x/exp/rand包，那么导入声明必须至少为一个同名包指定一个新的包名以避免冲突。这叫做导入包的重命名。

```go
package main
import (
	"fmt"
	rand1 "math/rand"
	rand2 "golang.org/x/exp/rand"
)
func main() {
	fmt.Println(rand1.Int63n(10000)) // 9401
	fmt.Println(rand2.Int63n(10000)) // 9351
}
```

## 包的匿名导入

如果只是导入一个包而并不使用导入的包将会导致一个编译错误。但是有时候我们只是想利用导入包而产生的副作用：它会计算包级变量的初始化表达式和执行导入包的init初始化函数。这时候我们需要抑制“unused import”编译错误，我们可以用下划线_来重命名导入的包。比如序号gorm包是要带入sql驱动

```go
import (
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/mysql"
)
func main() {
  db, err := gorm.Open("mysql", "user:password@/dbname?charset=utf8&parseTime=True&loc=Local")
  defer db.Close()
}
```

## 包和命名

当创建一个包，**一般要用短小的包名，但也不能太短导致难以理解。**标准库中最常用的包有bufio、bytes、flag、fmt、http、io、json、os、sort、sync和time等包。

包名一般采用单数的形式。标准库的bytes、errors和strings使用了复数形式，这是为了避免和预定义的类型冲突，同样还有go/types是为了避免和type关键字冲突。

## 工具

Go语言工具箱的具体功能，包括如何下载、格式化、构建、测试和安装Go语言编写的程序。

### go命令

```go
$ go
...
    build            compile packages and dependencies
    clean            remove object files
    doc              show documentation for package or symbol
    env              print Go environment information
    fmt              run gofmt on package sources
    get              download and install packages and dependencies
    install          compile and install packages and dependencies
    list             list packages
    run              compile and run Go program
    test             test packages
    version          print Go version
    vet              run go tool vet on packages
Use "go help [command]" for more information about a command.
...
```

### 下载包

使用Go语言工具箱的go命令，不仅可以根据包导入路径找到本地工作区的包，甚至可以从互联网上找到和更新包。

使用命令go get可以下载一个单一的包或者用...下载整个子目录里面的每个包。Go语言工具箱的go命令同时计算并下载所依赖的每个包，这也是前一个例子中golang.org/x/net/html自动出现在本地工作区目录的原因。

### 构建包

go build命令编译命令行参数指定的每个包。如果包是一个库，则忽略输出结果；这可以用于检测包是可以正确编译的。如果包的名字是main，go build将调用链接器在当前目录创建一个可执行程序；以导入路径的最后一段作为可执行程序的名字。

### 包文档

Go语言的编码风格鼓励为每个包提供良好的文档。包中每个导出的成员和包声明前都应该包含目的和用法说明的注释。

Go语言中的文档注释一般是完整的句子，第一行通常是摘要说明，以被注释者的名字开头。注释中函数的参数或其它的标识符并不需要额外的引号或其它标记注明。例如，下面是fmt.Fprintf的文档注释。

```go
// Fprintf formats according to a format specifier and writes to w.
// It returns the number of bytes written and any write error encountered.
func Fprintf(w io.Writer, format string, a ...interface{}) (int, error)
```

首先是go doc命令，该命令打印其后所指定的实体的声明与文档注释，该实体可能是一个包：

```go
$ go doc time
package time // import "time"
Package time provides functionality for measuring and displaying time.
const Nanosecond Duration = 1 ...
func After(d Duration) <-chan Time
func Sleep(d Duration)
func Since(t Time) Duration
func Now() Time
type Duration int64
type Time struct { ... }
...many more...
```

或者是某个具体的包成员：

```go
$ go doc time.Since
func Since(t Time) Duration
    Since returns the time elapsed since t.
    It is shorthand for time.Now().Sub(t).
```

或者是一个方法：

```go
$ go doc time.Duration.Seconds
func (d Duration) Seconds() float64
    Seconds returns the duration as a floating-point number of seconds.
```

### 内部包

在Go语言程序中，包是最重要的封装机制。没有导出的标识符只在同一个包内部可以访问，而导出的标识符则是面向全宇宙都是可见的。

```go
net/http
net/http/internal/chunked
net/http/httputil
net/url
```

### 查询包

go list命令可以查询可用包的信息。其最简单的形式，可以测试包是否在工作区并打印它的导入路径：

```go
$ go list github.com/go-sql-driver/mysql
github.com/go-sql-driver/mysql
```

go list命令还可以获取每个包完整的元信息，而不仅仅只是导入路径，这些元信息可以以不同格式提供给用户。其中-json命令行参数表示用JSON格式打印每个包的元信息。可以用"..."表示匹配相关包的包的导入路径。

```go
$ go list ...mysql
github.com/astaxie/beego/session/mysql
github.com/go-sql-driver/mysql
```

参考文章:

*   《go语言圣经》
