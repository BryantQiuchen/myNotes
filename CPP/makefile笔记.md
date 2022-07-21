# makefile 介绍

## 示例

```makefile
edit : main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
    cc -o edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o

main.o : main.c defs.h
    cc -c main.c
kbd.o : kbd.c defs.h command.h
    cc -c kbd.c
command.o : command.c defs.h command.h
    cc -c command.c
display.o : display.c defs.h buffer.h
    cc -c display.c
insert.o : insert.c defs.h buffer.h
    cc -c insert.c
search.o : search.c defs.h buffer.h
    cc -c search.c
files.o : files.c defs.h buffer.h command.h
    cc -c files.c
utils.o : utils.c defs.h
    cc -c utils.c
clean :
    rm edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
```

`\` 表示换行，该目录下输入 `make` 就可以生成可执行文件 edit。

## 使用变量

声明一个变量，叫 `objects`、 `OBJS` 等等，然后在 makeflie 中定义：

```makefile
objects = main.o kbd.o command.o display.o \
     insert.o search.o files.o utils.o
```

然后就可以用 `$(objects)` 使用这个变量。

## Makefile 里有什么？

1. 显式规则。显式规则说明了如何生成一个或多个目标文件。这是由 Makefile 的书写者明显指出要生成的文件、文件的依赖文件和生成的命令。
2. 隐晦规则。由于我们的 make 有自动推导的功能，所以隐晦的规则可以让我们比较简略地书写 Makefile，这是由make 所支持的。
3. 变量的定义。在 Makefile 中我们要定义一系列的变量，变量一般都是字符串，这个有点像你 C 语言中的宏，当 Makefile 被执行时，其中的变量都会被扩展到相应的引用位置上。
4. 文件指示。其包括了三个部分，一个是在一个 Makefile 中引用另一个 Makefile，就像 C 语言中的 include 一样；另一个是指根据某些情况指定 Makefile 中的有效部分，就像 C 语言中的预编译 #if 一样；

在Makefile中的命令，必须要以 `Tab` 键开始。

## make 工作方式

GNU make 工作时执行步骤：

1. 读入所有的 Makefile。
2. 读入被 include 的其它 Makefile。
3. 初始化文件中的变量。
4. 推导隐晦规则，并分析所有规则。
5. 为所有的目标文件创建依赖关系链。
6. 根据依赖关系，决定哪些目标要重新生成。
7. 执行生成命令。

# 规则

规则包含两个部分：

- 依赖关系
- 生成目标的方法

在 Makefile 中，规则顺序是很重要的，因为，Makefile 中只应该有一个最终目标，其它的目标都是被这个目标所连带出来的，所以一定要让 make 知道你的最终目标是什么。一般来说，定义在 Makefile 中的目标可能会有很多，但是第一条规则中的目标将被确立为最终的目标。如果第一条规则中的目标有很多个，那么，第一个目标会成为最终的目标。make 所完成的也就是这个目标。

## 示例

```makefile
foo.o: foo.c defs.h       # foo模块
    cc -c -g foo.c
```

`foo.o` 是我们的目标，`foo.c` 和 `defs.h` 是目标依赖的源文件，只有一个命令 `cc -c -g foo.c`（Tab 键开头），这个规则告诉我们两件事：

1. 文件的依赖关系， `foo.o` 依赖于 `foo.c` 和 `defs.h` 的文件，如果 `foo.c` 和 `defs.h` 的文件日期要比 `foo.o` 文件日期要新，或是 `foo.o` 不存在，那么依赖关系发生。
2. 生成或更新 `foo.o` 文件，就是那个cc命令。它说明了如何生成 `foo.o` 这个文件。（当然，foo.c文件include了defs.h文件）

## 使用通配符

make 支持的三个通配符：`*, ?, ~`

`~` 字符在文件名中也有特殊的用途，如果是 `~/test` ，这就表示当前用户的 `$HOME` 目录下的test目录。（make 也支持），`*.c` 表示所有后缀为 c 的文件，如果我们的文件名中有通配符，可以用转义字符 `\`，如 `\*` 表示真实的字符 `*`。

```makefile
objects = *.o
```

这里的 objects 的值就是 `*.o`，如果让通配符在变量中展开，即让 objects 的值是所有 `.o` 的文件名的集合，可以改为：

```makefile
objects = $(wildcard *.o)
```

另给一个变量使用通配符的例子：

1. 列出一确定文件夹中的所有 `.c` 文件。

```makefile
objects := $(wildcard *.c)
```

2. 列出(1)中所有文件对应的 `.o` 文件，在（3）中我们可以看到它是由make自动编译出的:

```makefile
$(patsubst %.c,%.o,$(wildcard *.c))
```

3. 由(1)(2)两步，可写出编译并链接所有 `.c` 和 `.o` 文件

```makefile
objects := $(patsubst %.c,%.o,$(wildcard *.c))
foo : $(objects)
    cc -o foo $(objects)
```

`wildcard`、`patsubst` 是 Makefile 的关键字。

## 文件搜寻

在大的工程中，有大量的源文件，通常做法是把这许多的源文件分类，存放在不同的目录中，当 make 需要找寻文件的依赖关系，可以在文件前加上路径，最好的方法是把一个路径告诉 make，让 make 自动去找。

Makefile 文件中的特殊变量 `VPATH` 就是完成这个功能的，如果没有指明这个变量，make 只会在当前的目录中去找寻依赖文件和目标文件。如果定义了这个变量，那么，make 就会在当前目录找不到的情况下，到所指定的目录中去找寻文件了。

```makefile
VPATH = src:../headers
```

上面指定了两个目录，“src”和“../headers”，make 会按照这个顺序进行搜索，目录由“冒号”分割。（当前目录永远是最高优先搜索的地方）

另一个设置文件搜索路径的方法是用 `vapth` 关键字（全小写），不是变量，是一个关键字，与上面的 `VPATH` 变量类似，但它更加灵活，使用方法：

- `vpath <pattern> <directories>`

    为符合模式 <pattern> 的文件指定搜索目录 <directories>

- `vpath <pattern>`

    清除符合模式 <pattern> 的文件的搜索目录

- `vpath`

    清除所有已被设置好了的文件搜索目录

`vpath` 使用方法中的 <pattern> 需要包含 `%` 字符。 `%` 的意思是匹配零或若干字符，例如， `%.h` 表示所有以 `.h` 结尾的文件。<pattern> 指定了要搜索的文件集，<directories> 指定了 <pattern> 文件集的搜索的目录。例：

```makefile
vpath %.h ../headers
```

表示 make 在 `../headers` 目录下搜索以 `.h` 结尾的文件，可以连续使用 vpath 语句，指定不同搜索策略。make 会按照 vpath 的先后顺序来执行搜索。

## 静态模式

静态模式下可以更加容易地定义多目标的规则，让我们的规则变得更加有弹性和灵活：

```makefile
<targets ...> : <target-pattern> : <prereq-patterns ...>
    <commands>
    ...
```

targets 定义了一系列的目标文件，是一个目标集合。

target-pattern 指明 target 的模式，目标集模式。

prereq-patterns 是目标的依赖模式，对 target-pattern 形成的模式在进行一次依赖目标的定义。

例：

```makefile
objects = foo.o bar.o

all: $(objects)

$(objects): %.o: %.c
    $(CC) -c $(CFLAGS) $< -o $@
```

指明目标从 `$(objects)` 中获取，`%.o` 表明要所有以`.o` 结尾的目标，也就是 `foo.o bar.o` ，也就是变量 `$(objects)` 集合的模式，依赖模式 `%.c` 取模式 `%.o` 中的 `%`，即 `foo bar`，并为其加上 `.c` 后缀，故依赖目标为 `foo.c bar.c`。而命令中的 `$<` 和 `$@` 则是自动化变量， `$<` 表示第一个依赖文件， `$@` 表示目标集（也就是“foo.o bar.o”）。于是，上面的规则展开后等价于下面的规则：

```makefile
foo.o : foo.c
    $(CC) -c $(CFLAGS) foo.c -o foo.o
bar.o : bar.c
    $(CC) -c $(CFLAGS) bar.c -o bar.o
```

## 自动生成依赖性

大多数 C/C++ 编译器都支持一个 "-M" 选项，自动找寻源文件中包含的头文件，并生成一个依赖关系，例：

```makefile
cc -M main.c
```

输出：

```makefile
main.o : main.c defs.h
```

# 书写命令

每条规则中的命令和操作系统 Shell 中的命令行是一致的，make 会按照顺序一条一条执行，每条命令的开头必须以 `Tab` 键开头，除非，命令是紧跟在依赖规则后面的分号后的。在命令行之间中的空格或是空行会被忽略，但是如果该空格或空行是以 `Tab` 键开头的，那么 make 会认为其是一个空命令。

## 命令执行

如果要让上一条命令的结果应用在下一条命令上，应该使用分号分隔这两条命令。比如你的第一条命令是cd命令，你希望第二条命令得在cd之后的基础上运行，那么你就不能把这两条命令写在两行上，而应该把这两条命令写在一行上，用分号分隔。

## 定义命令包

如果 Makefile 中出现一些相同命令序列，那么我们可以为这些相同的命令序列定义一个变量。定义这种命令序列的语法以 `define` 开始，以 `endef` 结束，如:

```makefile
define run-yacc
yacc $(firstword $^)
mv y.tab.c $@
endef
```

run-yacc 就是包的名字，不能和 makefile 中的变量重名，包中第一个命令就是运行 yacc 程序，这个程序总会生成 y.tab.c 文件，第二行把这个文件改名字，使用示例：

```makefile
foo.c : foo.y
    $(run-yacc)
```

我们可以看见，要使用这个命令包，我们就好像使用变量一样。在这个命令包的使用中，命令包 run-yacc 中的 `$^` 就是 `foo.y` ， `$@` 就是 `foo.c`，make 在执行命令包时，命令包中的每个命令会被依次独立执行。

# 使用变量

Makefile 中定义的变量，代表一个文本字串，在 makefile 中执行时候会自动原模原样地展开在所使用的地方，像 C/C++ 中的宏一样，但 makefile 中可以改变其值。

变量的命名字可以包含字符、数字，下划线（可以是数字开头），但不应该含有 `:` 、 `#` 、 `=` 或是空字符（空格、回车等）。变量是大小写敏感的，“foo”、“Foo”和“FOO”是三个不同的变量名。

有一些变量是很奇怪字串，如 `$<` 、 `$@` 等，这些是自动化变量。

## 变量基础

变量声明时需要赋初值，使用时在变量名前加 `$` 符号，最好用小括号或大括号包含起来。

例：

```makefile
objects = program.o foo.o utils.o
program : $(objects)
    cc -o program $(objects)

$(objects) : defs.h
```

## 变量中的变量

在定义变量的值时，可以用其他变量来构造变量的值，有两种方法。

第一种，使用 `=` 号，等号右侧的值可以定义在文件的任何一处，不一定是已定义好的值（也可以使用后面定义的值）。如：

```makefile
foo = $(bar)
bar = $(ugh)
ugh = Huh?

all:
    echo $(foo)
```

我们执行 make all 将会打出变量 `$(foo)` 的值是 `Huh?` （ `$(foo)` 的值是 `$(bar)` ， `$(bar)` 的值是 `$(ugh)` ， `$(ugh)` 的值是 `Huh?` ）可见，变量是可以使用后面的变量来定义的。

这种定义方式可能会产生递归定义，当然，make 是有能力检测这样的定义，并会报错。

为了避免上面的这种方法，可以使用 make 中的另一种变量来定义变量的方法，使用 `:=` 操作符，如：

```makefile
x := foo
y := $(x) bar
x := later
```

等价于

```makefile
y := foo bar
x := later
```

使用这种方法，前面的变量不能使用后面的变量，只能使用前面已经定义好的变量。

## 变量的替换

我们可以替换变量中的共有的部分，其格式是 `$(var:a=b)` 或是 `${var:a=b}` ，其意思是，把变量“var”中所有以“a”字串“结尾”的“a”替换成“b”字串。这里的“结尾”意思是“空格”或是“结束符”。

```makefile
foo := a.o b.o c.o
bar := $(foo:.o=.c)
```

首先定义了 `foo` 变量，第二行把 `$(foo)` 中所有以 `.o` 结尾的替换成 `.c` 

另一种变量替换的技术是以“静态模式”定义的，如：

```makefile
foo := a.o b.o c.o
bar := $(foo:%.o=%.c)
```

## 变量的值再变成变量

```makefile
x = y
y = z
a := $($(x))
```

在这个例子中，`$(x)` 的值是 `y`，所以 `$($(x))` 就是 `$(y)`，于是 `$(a)` 的值就是 `z`。（注意，是 `x=y`，而不是 `x=$(y)`）

再来看一个例子：

```makefile
x = $(y)
y = z
z = Hello
a := $($(x))
```

这里的 `$($(x))` 被替换成了 `$($(y))` ，因为 `$(y)` 值是 `z`，所以，最终结果是： `a:=$(z)` ，也就是 `Hello`。

还可以使用多个变量来组成一个变量的名字，其值：

```makefile
first_second = Hello
a = first
b = second
all = $($a_$b)
```

这里的 `$a_$b` 组成了“first_second”，于是，`$(all)` 的值就是“Hello”。

## 多行变量

使用 define 关键字设置可以有换行，用于定义一系列的命令。

define 指示符后面跟变量名字，重起一行定义变量的值，以 endif 关键字结束，命令需要以 [Tab] 开头，用 define 定义的命令变量没有以 [Tab] 开头，make 就不会认为它是命令。

```makefile
define two-lines
echo foo
echo $(bar)
endef
```

# 使用条件判断

判断 `$CC` 变量是否 `gcc`

```makefile
libs_for_gcc = -lgnu
normal_libs =

foo: $(objects)
ifeq ($(CC),gcc)
    $(CC) -o foo $(objects) $(libs_for_gcc)
else
    $(CC) -o foo $(objects) $(normal_libs)
endif
```

在上面示例的这个规则中，目标 `foo` 可以根据变量 `$(CC)` 值来选取不同的函数库来编译程序。

`ifeq` 的意思表示条件语句的开始，并指定一个条件表达式，表达式包含两个参数，以逗号分隔，表达式以圆括号括起。 `else` 表示条件表达式为假的情况。 `endif` 表示一个条件语句的结束，任何一个条件表达式都应该以 `endif` 结束。

# 使用函数

## 调用语法

```makefile
$(<function> <arguments>)
	或
${<function> <arguments>}
```

`<function>` 就是函数名，`<argument>` 以 `,` 分隔

## 字符串处理函数

### subst

```makefile
$(subst <from>,<to>,<text>)
```

- 功能：把字串 `<text>` 中的 `<from>` 字符串替换成 `<to>` 。
- 返回：函数返回被替换过后的字符串。

### patsubst

```makefile
$(patsubst <pattern>,<replacement>,<text>)
```

- 功能：查找 `<text>` 中的单词（单词以“空格”、“Tab”或“回车”“换行”分隔）是否符合模式 `<pattern>` ，如果匹配的话，则以 `<replacement>` 替换。这里， `<pattern>` 可以包括通配符 `%` ，表示任意长度的字串。如果 `<replacement>` 中也包含 `%` ，那么， `<replacement>` 中的这个 `%` 将是 `<pattern>` 中的那个 `%` 所代表的字串。（可以用 `\` 来转义，以 `\%` 来表示真实含义的 `%` 字符）
- 返回：函数返回被替换过后的字符串。

### strip

```makefile
$(strip <string>)
```

- 功能：去掉 `<string>` 字串中开头和结尾的空字符。
- 返回：返回被去掉空格的字符串值。

### findstring

```makefile
$(findstring <find>,<in>)
```

- 功能：在字串 `<in>` 中查找 `<find>` 字串。
- 返回：如果找到，那么返回 `<find>` ，否则返回空字符串。

### filter

```makefile
$(filter <pattern...>,<text>)
```

- 功能：以 `<pattern>` 模式过滤 `<text>` 字符串中的单词，保留符合模式 `<pattern>` 的单词。可以有多个模式。
- 返回：返回符合模式 `<pattern>` 的字串。

### filter-out

```makefile
$(filter-out <pattern...>,<text>)
```

- 功能：以 `<pattern>` 模式过滤 `<text>` 字符串中的单词，去除符合模式 `<pattern>` 的单词。可以有多个模式。
- 返回：返回不符合模式 `<pattern>` 的字串。

## 文件名操作函数

### dir

```makefile
$(dir <names...>)
```

- 功能：从文件名序列 `<names>` 中取出目录部分。目录部分是指最后一个反斜杠（ `/` ）之前的部分。如果没有反斜杠，那么返回 `./` 。
- 返回：返回文件名序列 `<names>` 的目录部分。

### join

```makefile
$(join <list1>,<list2>)
```

- 功能：把 `<list2>` 中的单词对应地加到 `<list1>` 的单词后面。如果 `<list1>` 的单词个数要比 `<list2>` 的多，那么， `<list1>` 中的多出来的单词将保持原样。如果 `<list2>` 的单词个数要比 `<list1>` 多，那么， `<list2>` 多出来的单词将被复制到 `<list1>` 中。
- 返回：返回连接过后的字符串。

## foreach 函数

```makefile
$(foreach <var>,<list>,<text>)
```

把参数 `<list>` 中的单词逐一取出放到参数 `<var>` 所指定的变量中，然后再执行 `<text>` 所包含的表达式。每一次 `<text>` 会返回一个字符串，循环过程中， `<text>` 的所返回的每个字符串会以空格分隔，最后当整个循环结束时， `<text>` 所返回的每个字符串所组成的整个字符串（以空格分隔）将会是foreach函数的返回值。

`<var>` 最好是一个变量名， `<list>` 可以是一个表达式，而 `<text>` 中一般会使用 `<var>` 这个参数来依次枚举 `<list>` 中的单词。

foreach 中的 `<var>` 参数是一个临时的局部变量， foreach 函数执行完毕后，参数就不再起作用。

## shell 函数

shell函数把执行操作系统命令后的输出作为函数返回。于是，我们可以用操作系统命令以及字符串处理命令awk，sed等等命令来生成一个变量。如：

```makefile
contents := $(shell cat foo)
files := $(shell echo *.c)
```

# 自动化变量

自动化变量，就是这种变量会把模式中所定义的一系列的文件自动地挨个取出，直至所有的符合模式的文件都取完了。这种自动化变量只应出现在规则的命令中。

下面是所有的自动化变量及其说明：

- `$@` : 表示规则中的目标文件集。在模式规则中，如果有多个目标，那么， `$@` 就是匹配于目标中模式定义的集合。
- `$%` : 仅当目标是函数库文件中，表示规则中的目标成员名。例如，如果一个目标是 `foo.a(bar.o)` ，那么， `$%` 就是 `bar.o` ， `$@` 就是 `foo.a` 。如果目标不是函数库文件，那么，其值为空。
- `$<` : 依赖目标中的第一个目标名字。如果依赖目标是以模式（即 `%` ）定义的，那么 `$<` 将是符合模式的一系列的文件集。注意，其是一个一个取出来的。
- `$?` : 所有比目标新的依赖目标的集合。以空格分隔。
- `$^` : 所有的依赖目标的集合。以空格分隔。如果在依赖目标中有多个重复的，那么这个变量会去除重复的依赖目标，只保留一份。
- `$+` : 这个变量很像 `$^` ，也是所有依赖目标的集合。只是它不去除重复的依赖目标。
- `$*` : 这个变量表示目标模式中 `%` 及其之前的部分。如果目标是 `dir/a.foo.b` ，并且目标的模式是 `a.%.b` ，那么， `$*` 的值就是 `dir/foo` 。这个变量对于构造有关联的文件名是比较有效。如果目标中没有模式的定义，那么 `$*` 也就不能被推导出，但是，如果目标文件的后缀是make所识别的，那么 `$*` 就是除了后缀的那一部分。例如：如果目标是 `foo.c` ，因为 `.c` 是make所能识别的后缀名，所以， `$*` 的值就是 `foo` 。这个特性是GNU make的，很有可能不兼容于其它版本的make，所以，你应该尽量避免使用 `$*` ，除非是在隐含规则或是静态模式中。如果目标中的后缀是make所不能识别的，那么 `$*` 就是空值。