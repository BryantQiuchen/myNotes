# 基础

## 定义变量

使用 `var` 关键字

```go
//定义一个名称为“variableName”，类型为"type"的变量
var variableName type
```

定义变量并初始化值

```go
/*
	定义三个类型都是"type"的变量,并且分别初始化为相应的值
	vname1为v1，vname2为v2，vname3为v3
*/
var vname1, vname2, vname3 type= v1, v2, v3
```

直接使用简短声明，可以忽略声明（只能使用在函数内部，一般用 `var` 来定义全局变量）：

```go
vname1, vname2, vname3 := v1, v2, v3
```

Go 对于已声明但未使用的变量会在编译阶段报错，比如下面的代码就会产生一个错误：声明了`i`但未使用。

```go
package main

func main() {
	var i int
}
```

## 常量

所谓常量，也就是在程序编译阶段就确定下来的值，而程序在运行时无法改变该值。

```go
const constantName = value
//如果需要，也可以明确指定常量的类型：
const Pi float32 = 3.1415926
```

## 内置基础类型

**Boolean**

在 Go 中，布尔值的类型为 `bool`，值是 `true` 或 `false`，默认为 `false`。

```go
var isActive bool  // 全局变量声明
var enabled, disabled = true, false  // 忽略类型的声明
func test() {
	var available bool  // 一般声明
	valid := false      // 简短声明
	available = true    // 赋值操作
}
```

**数值类型**

整数的类型分为无符号和带符号两种，Go 同时支持 `int` 和 `uint`，这两种类型的长度相同，但具体长度取决于不同编译器的实现。

不同类型的变量之间不允许互相赋值或操作，不然会在编译时引起编译器报错。

```go
var a int8
var b int32
c := a + b
// 报错：invalid operation: a + b (mismatched types int8 and int32)
```

**字符串**

Go 中的字符串都是采用 `UTF-8` 字符集编码。字符串是用一对双引号（`""`）或反引号（``` ```）括起来定义，它的类型是 `string`。

Go 字符串时不可变的：

```go
var s string = "hello"
s[0] = 'c'
// 报错： cannot assign to s[0]
```

需要修改可以通过以下代码实现：

```go
s := "hello"
c := []byte(s)  // 将字符串 s 转换为 []byte 类型
c[0] = 'c'
s2 := string(c)  // 再转换回 string 类型
fmt.Printf("%s\n", s2)
```

或者通过 `+` 操作符来连接两个字符串来进行修改

```go
s := "hello"
s = "c" + s[1:] // 字符串虽然不能更改但可以进行切片操作
```

## 分组声明

在 Go 语言中，同时声明多个常量、变量，或者导入多个包时，可采用分组的方式进行声明。

下面的代码：

```go
import "fmt"
import "os"

const i = 100
const pi = 3.1415
const prefix = "Go_"

var i int
var pi float32
var prefix string
```

可以分组写成如下形式：

```go
import(
	"fmt"
	"os"
)

const(
	i = 100
	pi = 3.1415
	prefix = "Go_"
)

var(
	i int
	pi float32
	prefix string
)
```

## iota 枚举

Go里面有一个关键字 `iota`，这个关键字用来声明 `enum` 的时候采用，它默认开始值是0，const中每增加一行加1：

```go
package main

import (
	"fmt"
)

const (
	x = iota // x == 0
	y = iota // y == 1
	z = iota // z == 2
	w        // 常量声明省略值时，默认和之前一个值的字面相同。这里隐式地说w = iota，因此w == 3。其实上面y和z可同样不用"= iota"
)

const v = iota // 每遇到一个const关键字，iota就会重置，此时v == 0

const (
	h, i, j = iota, iota, iota //h=0,i=0,j=0 iota在同一行值相同
)

const (
	a       = iota //a=0
	b       = "B"
	c       = iota             //c=2
	d, e, f = iota, iota, iota //d=3,e=3,f=3
	g       = iota             //g = 4
)
```

## Go 程序设计规则

Go 之所以会那么简洁，是因为它有一些默认的行为：

- 大写字母开头的变量是可导出的，也就是其它包可以读取的，是公有变量；小写字母开头的就是不可导出的，是私有变量。
- 大写字母开头的函数也是一样，相当于 `class` 中的带 `public` 关键词的公有函数；小写字母开头的就是有 `private` 关键词的私有函数。

## `array`

array 就是数组，定义方式如下：

```go
var arr [n]type
```

n 表示数组长度，type 表示存储元素的类型，数组操作：

```go
var arr [10]int  // 声明了一个int类型的数组
arr[0] = 42      // 数组下标是从0开始的
arr[1] = 13      // 赋值操作
fmt.Printf("The first element is %d\n", arr[0])  // 获取数据，返回42
fmt.Printf("The last element is %d\n", arr[9]) //返回未赋值的最后一个元素，默认返回0
```

数组不能改变长度，`[3]int` 和 `[4]int` 是不同的类型，数组之间的赋值是值的赋值，当把一个数组作为参数传入参数时，实际上传入的时该数组的副本，而不是它的指针。如果要使用指针，要使用 `slice` 类型。

数组可以通过 `:=` 来声明：

```go
a := [3]int{1, 2, 3} // 声明了一个长度为3的int数组

b := [10]int{1, 2, 3} // 声明了一个长度为10的int数组，其中前三个元素初始化为1、2、3，其它默认为0

c := [...]int{4, 5, 6} // 可以省略长度而采用`...`的方式，Go会自动根据元素个数来计算长度
```

Go 支持嵌套数组，即多维数组：

```go
// 声明了一个二维数组，该数组以两个数组作为元素，其中每个数组中又有4个int类型的元素
easyArray := [2][4]int{{1, 2, 3, 4}, {5, 6, 7, 8}}
```

## `slice`

很多应用场景下，我们不知道需要多大的数组，因此就需要动态数组。

`slice `并不是真正意义上的动态数组，而是一个引用类型。`slice` 总是指向一个底层 `array`，`slice `的声明也可以像 `array` 一样，只是不需要长度。

我们可以声明一个 `slice`，并初始化数据：

```go
slice := []byte {'a', 'b', 'c', 'd'}
```

`slice` 可以从一个数组或一个已经存在的 `slice` 中再次声明，`slice` 通过 `array[i:j]` 来获取，其中 `i` 是开始位置，`j` 是结束为止，但不包含 `array[j]`，长度为 `j-i`。

```go
// 声明一个含有10个元素元素类型为byte的数组
var ar = [10]byte {'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j'}

// 声明两个含有byte的slice
var a, b []byte

// a指向数组的第3个元素开始，并到第五个元素结束，
a = ar[2:5]
//现在a含有的元素: ar[2]、ar[3]和ar[4]

// b是数组ar的另一个slice
b = ar[3:5]
// b的元素是：ar[3]和ar[4]
```

slice 有一些简洁的操作：

- `slice `的默认开始位置是0，`ar[:n]` 等价于 `ar[0:n]`
- `slice` 的第二个序列默认是数组的长度，`ar[n:]` 等价于 `ar[n:len(ar)]`
- 如果从一个数组里面直接获取 `slice`，可以这样 `ar[:]`，因为默认第一个序列是0，第二个是数组的长度，即等价于 `ar[0:len(ar)]`

```go
// 声明一个数组
var array = [10]byte{'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j'}
// 声明两个slice
var aSlice, bSlice []byte

// 演示一些简便操作
aSlice = array[:3] // 等价于aSlice = array[0:3] aSlice包含元素: a,b,c
aSlice = array[5:] // 等价于aSlice = array[5:10] aSlice包含元素: f,g,h,i,j
aSlice = array[:]  // 等价于aSlice = array[0:10] 这样aSlice包含了全部的元素

// 从slice中获取slice
aSlice = array[3:7]  // aSlice包含元素: d,e,f,g，len=4，cap=7
bSlice = aSlice[1:3] // bSlice 包含aSlice[1], aSlice[2] 也就是含有: e,f
bSlice = aSlice[:3]  // bSlice 包含 aSlice[0], aSlice[1], aSlice[2] 也就是含有: d,e,f
bSlice = aSlice[0:5] // 对slice的slice可以在cap范围内扩展，此时bSlice包含：d,e,f,g,h
bSlice = aSlice[:]   // bSlice包含所有aSlice的元素: d,e,f,g
```

`slice` 是引用类型，所以当引用类型改变其中的值时，其他的所有引用都会改变，例如上面的 `aSlice` 和 `bSlice`，如果修改了 `aSlice` 中的值，那么 `bSlice` 中的值也会改变。

从概念上面来说 `slice` 像一个结构体，这个结构体包含了三个元素：

- 一个指针，指向数组中 `slice` 指定的开始位置
- 长度，即 `slice` 的长度
- 最大长度，也就是 `slice` 开始位置到数组的最后位置的长度

`slice` 内置函数

- `len` 获取 `slice` 的长度；
- `cap` 获取 `slice` 的最大容量；
- `append` 向 `slice` 里追加一个或多个元素，然后返回一个和 `slice` 一样类型的 `slice`。
- `copy` 从源 `slice` 的 `src` 中复制元素到目标 `dst`，返回复制元素的个数。

> `append `函数会改变 `slice` 所引用的数组的内容，从而影响到引用同一数组的其它 `slice`。 但当 `slice` 中没有剩余空间（即 `(cap-len) == 0`）时，此时将动态分配新的数组空间。返回的 `slice` 数组指针将指向这个空间，而原数组的内容将保持不变；其它引用此数组的 `slice` 则不受影响。

Go1.2 `slice` 可以指定容量

```go
var array [10]int
slice := array[2:4:7]
// 容量7-2=5，这个产生的slice无法访问到最后三个元素
```

## `map`

`map `的读取和设置也类似 `slice` 一样，通过 `key` 来操作，只是 `slice` 的 `index` 只能是 `int` 类型，而 `map` 多了很多类型，可以是 `int`，可以是 `string` 及所有完全定义了`==`与`!=`操作的类型。

```go
// 声明一个key是string，值为int的字典,这种方式的声明需要在使用之前使用make初始化
var numbers map[string]int
// 另一种map的声明方式
numbers = make(map[string]int)
numbers["one"] = 1  //赋值
numbers["ten"] = 10 //赋值
numbers["three"] = 3

fmt.Println("第三个数字是: ", numbers["three"]) // 读取数据
// 打印出来如:第三个数字是: 3
```

对于 `map` 需要注意：

- `map` 是无序的，每次打印的 `map` 都会不一样，必须通过 `key` 来获取；
- `map` 长度是不固定的，是一种引用类型；
- 内置函数 `len` 同样适用于 `map`，返回 `map` 拥有的 `key` 的数量；
- `map` 和其他的基本类型不同，不是线程安全的，在多个 go-routine 存取时，必须使用 mutex lock 机制。

## `make`、`new` 操作

`make` 用于内建类型的内存分配（`map, slice, channel`），`new` 用于各种类型的内存分配。

内建函数 `new(T)` 分配了零值填充的 `T` 类型的内存空间，并且返回其地址，即一个 `*T` 类型的值（返回一个指针）

内建函数 `make(T, args)` 与 `new(T)` 有着不同的功能，make只能创建 `slice`、`map`和`channel`，并且返回一个有初始值(非零)的`T`类型，而不是`*T`。

## 流程控制

### `if`

Go 的 `if ` 允许在条件判断语句里面声明一个变量，这个变量的作用域只能在该条件逻辑块内，其他地方就不起作用了，如下所示

```go
// 计算获取值x,然后根据x返回的大小，判断是否大于10。
if x := computedValue(); x > 10 {
    fmt.Println("x is greater than 10")
} else {
	fmt.Println("x is less than 10")
}

//这个地方如果这样调用就编译出错了，因为x是条件里面的变量
fmt.Println(x)
```

### `for`

语法：

```go
for exp1; exp2; exp3 {
    // ...
}
```

`for` 配合 `range` 可以读取 `slice` 和 `map` 的数据

```go
for k, v := range map {
	fmt.Println("map's key:",k)
	fmt.Println("map's val:",v)
}
```

### `switch`

```go
i := 10
switch i {
case 1:
	fmt.Println("i is equal to 1")
case 2, 3, 4:
	fmt.Println("i is equal to 2, 3 or 4")
case 10:
	fmt.Println("i is equal to 10")
default:
	fmt.Println("All I know is that i is an integer")
}
```

Go里面 `switch` 默认相当于每个 `case` 最后带有 `break`，匹配成功后不会自动向下执行其他 `case`，而是跳出整个 `switch`, 但是可以使用 `fallthrough `强制执行后面的 `case` 代码。

```go
integer := 6
switch integer {
case 4:
	fmt.Println("The integer was <= 4")
	fallthrough
case 5:
	fmt.Println("The integer was <= 5")
	fallthrough
case 6:
	fmt.Println("The integer was <= 6")
	fallthrough
case 7:
	fmt.Println("The integer was <= 7")
	fallthrough
case 8:
	fmt.Println("The integer was <= 8")
	fallthrough
default:
	fmt.Println("default case")
}
/*
The integer was <= 6
The integer was <= 7
The integer was <= 8
default case
*/
```

## `struct`

创建一个自定义类型：

```go
type person struct {
    name string
    age int
}
```

具体使用：

```go
package main

import "fmt"

// 声明一个新的类型
type person struct {
	name string
	age int
}

// 比较两个人的年龄，返回年龄大的那个人，并且返回年龄差
// struct也是传值的
func Older(p1, p2 person) (person, int) {
	if p1.age>p2.age {  // 比较p1和p2这两个人的年龄
		return p1, p1.age-p2.age
	}
	return p2, p2.age-p1.age
}

func main() {
	var tom person

	// 赋值初始化
	tom.name, tom.age = "Tom", 18

	// 两个字段都写清楚的初始化
	bob := person{age:25, name:"Bob"}

	// 按照struct定义顺序初始化值
	paul := person{"Paul", 43}

	tb_Older, tb_diff := Older(tom, bob)
	tp_Older, tp_diff := Older(tom, paul)
	bp_Older, bp_diff := Older(bob, paul)

	fmt.Printf("Of %s and %s, %s is older by %d years\n",
		tom.name, bob.name, tb_Older.name, tb_diff)

	fmt.Printf("Of %s and %s, %s is older by %d years\n",
		tom.name, paul.name, tp_Older.name, tp_diff)

	fmt.Printf("Of %s and %s, %s is older by %d years\n",
		bob.name, paul.name, bp_Older.name, bp_diff)
}
// Output
Of Tom and Bob, Bob is older by 7 years
Of Tom and Paul, Paul is older by 25 years
Of Bob and Paul, Paul is older by 18 years
```

## `method`

`method` 是附属在一个给定的类型上，语法和函数的声明相同，只是在 `func` 后面增加了 receiver（method 依附的主体），语法如下：

```go
func (r ReceiverType) funcName(parameters) (results)
```

举例：

```go
package main

import (
	"fmt"
	"math"
)

type Rectangle struct {
	width, height float64
}

type Circle struct {
	radius float64
}

func (r Rectangle) area() float64 {
	return r.width*r.height
}

func (c Circle) area() float64 {
	return c.radius * c.radius * math.Pi
}

func main() {
	r1 := Rectangle{12, 2}
	r2 := Rectangle{9, 4}
	c1 := Circle{10}
	c2 := Circle{25}

	fmt.Println("Area of r1 is: ", r1.area())
	fmt.Println("Area of r2 is: ", r2.area())
	fmt.Println("Area of c1 is: ", c1.area())
	fmt.Println("Area of c2 is: ", c2.area())
}
```

在使用 method 的时候重要注意几点

- 虽然 method 的名字一模一样，但是如果接收者不一样，那么 method 就不一样
- method 里面可以访问接收者的字段
- 调用 method 通过`.`访问，就像 struct 里面访问字段一样

method 可以作用于任何你自定义的类型、内置类型、struct 等各种类型上。

## `interface`

interface 是一组 method 签名的组合，我们通过 interface 来定义对象的一组行为。

如果定义了一个 interface 的变量，那么这个变量里面可以存实现这个 interface 的任意类型的对象。

interface 就是一组抽象方法的集合，它必须由其他非 interface 类型实现，而不能自我实现。

### 空 `interface`

空 interface 不包含任何 method，正因为如此，所有的类型都是实现了空 interface，在存储任意类型的数值时非常有用，因为它可以存储任意类型的数值：

```go
var a interface{}
var i int = 5
s := "hello"
// a 可以存储任意类型的值
a = i
a = s
```

## 反射

Go 语言实现了反射，所谓反射就是能够检查程序在运行时的状态。

使用 reflect 一般分为三步：要去反射是一个类型的值(这些值都实现了空 interface)，首先需要把它转化成 reflect 对象(reflect.Type 或者 reflect.Value，根据不同的情况调用不同的函数)。这两种获取方式如下：

```go
t := reflect.TypeOf(i)    // 得到类型的元数据,通过t我们能获取类型定义里面的所有元素
v := reflect.ValueOf(i)   // 得到实际的值，通过v我们获取存储在里面的值，还可以去改变值
```

转化成 reflect 对象之后我们就可以进行一些操作，将 reflect 对象转化成相应的值：

```go
tag := t.Elem().Field(0).Tag  //获取定义在struct里面的标签
name := v.Elem().Field(0).String()  //获取存储在第一个字段里面的值
```

获取反射值能返回相应的类型和数值

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```

## 并发

### `goroutine`

goroutine 是 Go 并行设计的核心。goroutine 说到底其实就是协程，但是它比线程更小，十几个 goroutine 可能体现在底层就是五六个线程，Go 语言内部帮你实现了这些 goroutine 之间的内存共享。执行 goroutine 只需极少的栈内存(大概是4~5KB)，当然会根据相应的数据伸缩。也正因为如此，可同时运行成千上万个并发任务。goroutine 比 thread 更易用、更高效、更轻便。

goroutine 是通过 Go 的 runtime 管理的一个线程管理器。goroutine 通过 `go` 关键字实现了，其实就是一个普通的函数。

### `channels`

goroutine 运行在相同的地址空间，因此访问共享内存必须做好内存，goroutine 之间的数据通信的机制：channel。channel 可以与 UNIX shell 中的双向管道做类比：可以通过它发送或者接收值。只能是一个特定的类型：channel 类型。必须使用 make 创建 channel：

```go
c1 := make(chan int)
cs := make(chan string)
cf := make(chan interface{})
```

channel 通过操作符 `<-` 来接收和发送数据

```go
ch <- v    // 发送v到channel ch.
v := <-ch  // 从ch中接收数据，并赋值给v
```

### Buffered channels

Go 允许指定 channels 的缓冲大小，就是 channel 可以存储多少元素。`ch := make(chan bool, 4)`，创建了可以存储4个元素的 bool 型 channel，在这个 channel 中，前4个元素可以无阻塞的写入。当写入第5个元素时，代码将会阻塞，直到其他 goroutine 从 channel 中读取一些元素，腾出空间。

```go
ch := make(chan type, value)
```

当 value = 0 时，channel 是无缓冲阻塞读写的，当 value > 0 时，channel 有缓冲、是非阻塞的，直到写满 value 个元素才阻塞写入。

### `Select`

如果存在多个 channel 的时候，该如何操作，Go 里提供了关键字 `select`，通过 `select` 可以监听 channel 上的数据流动，默认是阻塞的，只有当坚挺的 channel 中有发送或接收时才会运行，多个 channel 都准备好时，select 会随机选择一个执行。

```go
package main

import "fmt"

func fibonacci(c, quit chan int) {
    x, y := 1, 1
    for {
        select {
        case c <- x:
            x, y = y, x + y
        case <-quit:
            fmt.Println("quit")
            return
        }
    }
}

func main() {
    c := make(chan int)
    quit := make(chan int)
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(<-c)
        }
        quit <- 0
    }()
    fibonacci(c, quit)
}
```

在`select`里面还有default语法，`select`其实就是类似switch的功能，default就是当监听的channel都没有准备好的时候，默认执行的（select不再阻塞等待channel）。

```go
select {
case i := <-c:
    // use i
default:
    // 当c阻塞的时候执行这里
}
```
