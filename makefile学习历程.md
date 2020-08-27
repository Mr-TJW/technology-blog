# makefile 学习心得
----------------------------------------------------------------

## wildcard 展开通配符
将通配符中包括的所有文件展开。
示例1：不使用 wildcard 
```c
-rm *.o  -f
```
结果：
```c
rm *.o -f
```

示例2：使用 wildcard 
```c
-rm $(wildcard *.o)  -f
```
结果：
```c
rm data_queue.o dlt698.o main.o process_manager.o public.o tcp_client.o  -f
```

## VPATH 与 vpath 文件搜索
> 正常情况下，makefile 只会在当前目录下寻找依赖文件，但是有时候我们将依赖文件放在别的目标，这时就需要指定makefile 的搜索目录。

**注意：** makefile 永远优先在当前目录下搜索依赖文件，搜索不到才会去所指定的目录去寻找。

### VPATH
示例：`VPATH = src:../headers`
上面的的定义指定两个目录，"src" 和 "../headers"，make 会按照这个顺序进行搜索。目录由 "冒号" 分隔。（当然，当前目录永远是最高优先搜索的地方）

```c
cc = gcc
PROJECT = test

target : $(PROJECT)

VPATH =  ./src : ./include
.cpp.o .c.o:
	$(CC) -c $^
OBJ = test.o

$(PROJECT) : $(OBJ)
	$(CC) -o $@  $^

.PHONY : clean 
clean:
	-rm $(PROJECT) -f
	-rm *.o -f
```

### vpath
vpath 相比与 VPATH，使用起来更加灵活

如果 makefile 和源文件不在同目录下，可以使用 vpath 来指定路径。
示例： 
```c
vpath %.c ./src
```
该语句表示，要求 make 在 "./src" 目录下搜索所有以 ".c" 结尾的文件。（如果某文件在当前目录没有找到的话）

## 静态模式
语法：
```c
<targets ...>: <target-pattern>: <prereq-patterns ...>
<commands>
....
```
* **targets** 定义了一系列的目标文件，可以有通配符。是目标的一个集合。
* **target-parrtern** 是指明了 targets 的模式，也就是的目标集模式。
* **prereq-parrterns** 是目标的依赖模式，它对 target-parrtern 形成的模式再进行一次依赖目标的定义。

举个例子：`<target-parrtern>` 定义成 "%.o"，意思是我们的 `<target>` 集合中都是以 ".o" 结尾的，而如果我们的 `<prereq-parrterns>` 定义成 "%.c"， 意思是对 `<target-parrtern>` 所形成的目标集进行二次定义，其计算方法是，取 `<target-parrtern>` 模式中的 "%"（也就是去掉了 ".o" 这个结尾），并为其加上 ".c" 这个结尾，形成的新集合。

示例：
```c
objects = foo.o bar.o
$(objects): %.o: %.c
$(CC) -c $(CFLAGS) $< -o $@
```

等价于：
```c
foo.o : foo.c
$(CC) -c $(CFLAGS) foo.c -o $@
bar.o : bar.c
$(CC) -c $(CFLAGS) bar.c -o $@
```

但有时候，我们的目标集中并不是都是 ".o" 结尾的，因此我们需要过滤掉不是以 ".o" 结尾的目标，我们可以用 `filter` 函数。

示例：
```c
files = foo.elc bar.o lose.o
$(filter %.o,$(files)): %.o: %.c
$(CC) -c $(CFLAGS) $< -o $@
$(filter %.elc,$(files)): %.elc: %.el
emacs -f batch-byte-compile $<
```
`$(filter %.o,$(files))` 表示调用 Makefile 的 `filter` 函数，过滤 "$files" 集，只要其中模式为 "%.o" 的内容。

## -M -MM 选项 自动生成依赖性
如果是一个比较大型的工程，你必需清楚哪些 C 文件包含了哪些头文件，并且，你在加入或删除头文件时，也需要小心地修改 Makefile，这是一个很没有维护性的工作。为了避免这种繁重而又容易出错的事情，我们可以使用 C/C++ 编译的一个功能。大多数的C/C++ 编译器都支持一个 "-M" 的选项，即自动找寻源文件中包含的头文件，并生成一个依赖关系。例如，如果我们执行下面的命令：
`cc -MM main.c`
其输出为：
`main.o : main.c defs.h`

-M 和 -MM 区别是 -M 会将标准库的头文件也带进来。
注意：-M 或 -MM 选项只是帮助我们知晓依赖关系，并不参与编译功能。

## 命令执行顺序
如果你要让上一条命令的结果应用在下一条命令时，你应该使用分号分隔这两条命令。比如你的第一条命令是 cd 命令，你希望第二条命令得在 cd 之后的基础上运行，那么你就不能把这两条命令写在两行上，而应该把这两条命令写在一行上，用分号分隔。

示例1：
```c
exec:
cd /home/hchen
pwd
```
结果：pwd 会打印出当前的 Makefile 目录

示例2：
```c
exec:
cd /home/hchen; pwd
```
结果：pwd 会打印出 "/home/hchen"

## 命令出错
有些时候，命令的出错并不表示就是错误的。例如 mkdir 命令，我们一定需要建立一个目录，如果目录不存在，那么 mkdir 就成功执行，万事大吉，如果目录存在，那么就出错了。我们之所以使用 mkdir 的意思就是一定要有这样的一个目录，于是我们就不希望 mkdir 出错而终止规则的运行。
为了做到这一点，忽略命令的出错，我们可以在 Makefile 的命令行前加一个减号 "-"（在 Tab 键之后），标记为不管命令出不出错都认为是成功的。如：
```c
clean:
-rm -f *.o
```

## 嵌套执行 make
在一些大的工程中，我们会把我们不同模块或是不同功能的源文件放在不同的目录中，我们可以在每个目录中都书写一个该目录的 Makefile，这有利于让我们的 Makefile 变得更加地简洁，而不至于把所有的东西全部写在一个 Makefile 中，这样会很难维护我们的 Makefile，这个技术对于我们模块编译和分段编译有着非常大的好处。

例如，我们有一个子目录叫 subdir，这个目录下有个 Makefile 文件，来指明了这个目录下文件的编译规则。那么我们总控的 Makefile 可以这样书写：

`subsystem:`
`cd subdir && $(MAKE)`
其等价于：
`subsystem:`
`$(MAKE) -C subdir`

定义 `$(MAKE)` 宏变量的意思是，也许我们的 make 需要一些参数，所以定义成一个变量比较利于维护。这两个例子的意思都是先进入 "subdir" 目录，然后执行 make 命令。我们把这个 Makefile 叫做 `总控 Makefile`，总控 Makefile 的变量可以传递到下级的 Makefile 中（如果你显示的声明），但是不会覆盖下层的 Makefile 中所定义的变量，除非指定了 "-e" 参数。

如果你要传递变量到下级 Makefile 中，那么你可以使用这样的声明：
`export <variable ...>`

示例一：
`export variable = value`
其等价于：
`variable = value`
`export variable`
其等价于：
`export variable := value`
其等价于：
`variable := value`
`export variable`

示例二：
`export variable += value`
其等价于：
`variable += value `
`export variable`

如果你要传递所有的变量，那么，只要一个 export 就行了。后面什么也不用跟，表示传递所有的变量。

需要注意的是，有两个变量，一个是 SHELL，一个是 MAKEFLAGS，这两个变量不管你是否 export，其总是要传递到下层 Makefile 中，特别是 MAKEFILES 变量，其中包含了 make 的参数信息，如果我们执行 "总控 Makefile" 时有 make 参数或是在上层 Makefile 中定义了这个变量，那么 MAKEFILES 变量将会是这些参数，并会传递到下层 Makefile 中，这是一个系统级的环境变量。

## 变量操作符

### = 操作符
"="左侧是变量，右侧是变量的值，右侧变量的值可以定义在文件的任何一处，也就是说，右侧中的变量不一定非要是已定义好的值，其也可以使用后面定义的值，例如：

```c
foo = $(bar)
bar = $(ugh)
ugh = Huh?
all:
echo $(foo) 
```
我们执行 `make all` 将会打出变量 `$(foo)` 的值是 "Huh?",可见，变量是可以使用后面的变量来定义的。

这个功能有好的地方，也有不好的地方，好的地方是，我们可以把变量的真实值推到后面来定义，如：
`CFLAGS = $(include_dirs) -O`
`include_dirs = -Ifoo -Ibar`
当 `CFLAGS` 在命令中被展开时，会是 `-Ifoo -Ibar -O`。但这种形式也有不好的地方，那就是递归定义，如：
`CFLAGS = $(CFLAGS) -O`
或：
`A = $(B)`
`B = $(A)`
这会让 make 陷入无限的变量展开过程中去，当然，我们的 make 是有能力检测这样的定义，并会报错。还有就是如果在变量中使用函数，那么，这种方式会让我们的 make 运行时非常慢，更糟糕的是，他会使用得两个 make 的函数 `wildcard` 和 `shell` 发生不可预知的错误。因为你不会知道这两个函数会被调用多少次。

### := 操作符
为了避免上面的这种方法，我们可以使用 make 中的另一种用变量来定义变量的方法。这种方法使用的是 ":=" 操作符，如:
```c
x := foo
y := $(x) bar
x := later
```
其等价于：
```c
y := foo bar
x := later
```
这种方法，前面的变量不能使用后面的变量，只能使用前面已定义好了的变量。如果是这样：
```
y := $(x) bar
x := foo
```
那么，y 的值是 "bar"，而不是 "foo bar"。

### ?= 操作符
例如： `FOO ?= bar`
表示：如果 FOO 没有被定义过，那么变量 FOO 的值就是 "bar"，如果 FOO 先前被定义过，那么这条语将什么也不做。其等价于：
```c
ifeq ($(origin FOO), undefined)
FOO = bar
endif
```

### += 操作符
作用是**追加变量值**，如下：
```c
objects = main.o foo.o bar.o utils.o
objects += another.o
```
于是，我们的$(objects)值变成："main.o foo.o bar.o utils.o another.o"。

使用 "+=" 操作符，可以模拟为下面的这种例子：
```c
objects = main.o foo.o bar.o utils.o
objects := $(objects) another.o
```
所不同的是，用 "+=" 更为简洁。
如果变量之前没有定义过，那么，"+=" 会自动变成 "="，如果前面有变量定义，那么 "+=" 会继承于前次操作的赋值符。如果前一次的是 ":="， 那么 "+=" 会以 ":=" 作为其赋值符，如：
```c
variable := value
variable += more
```
等价于：
```c
variable := value
variable := $(variable) more
```

## override 指示符
如果有变量是通常 make 的命令行参数设置的，那么 Makefile 中对这个变量的赋值会被忽略。如果你想在 Makefile 中设置这类参数的值，那么，你可以使用 "override"  指示符。其语法是：
```c
override <variable> = <value>
override <variable> := <value>
```
命令行中的宏定义还是老大，Makefile 里加了 override 的那处需要根据是 ":="、"="，"?="，"+="，属性，要么收老大做小弟 (+=)，要么直接干掉老大 (:= 、=)，要么主动让位 (?=)，其他地方在老大面前还是啥都不是。

示例：
1. Makefile 中不使用 override，命令行中的宏定义是老大、Makefile 中的宏定义在老大面前啥都不是。
	```c
	EXTRA_CFLAGS := -fPIC -O3
	override EXTRA_CFLAGS += -DTEST
	all: clean
	.PHONY:clean
	clean:
		@echo $(EXTRA_CFLAGS)

	```
	输出：
	`>make`
	输出 -fPIC -O3 -DTEST

	`>make EXTRA_CFLAGS=AAAAAA`
	输出 AAAAAA -DTEST

	`>make EXTRA_CFLAGS:=AAAAAA`
	输出 AAAAAA -DTEST

	`>make EXTRA_CFLAGS?=AAAAAA`
	输出 AAAAAA -DTEST

	`>make EXTRA_CFLAGS+=AAAAAA`
	输出 AAAAAA -DTEST

2. 注意 override后面使用的是 "=" 或 ":="，这样也可以把老大给干掉
	```c
	EXTRA_CFLAGS := -fPIC -O3
	override EXTRA_CFLAGS = -DTEST
	all: clean
	.PHONY:clean
	clean:
		@echo $(EXTRA_CFLAGS)
	```
	`>make`
	输出 -DTEST

	`>make EXTRA_CFLAGS=AAAAAA`
	输出 -DTEST

	`>make EXTRA_CFLAGS:=AAAAAA`
	输出 -DTEST

	`>make EXTRA_CFLAGS?=AAAAAA`
	输出 -DTEST

	`>make EXTRA_CFLAGS+=AAAAAA`
	输出 -DTEST
3. override后面使用的是 "?=" ，这样主动让位
	```c
	EXTRA_CFLAGS := -fPIC -O3
	override EXTRA_CFLAGS ?= -DTEST
	all: clean
	.PHONY:clean
	clean:
		@echo $(EXTRA_CFLAGS)
	```
	`>make`
	输出 -fPIC -O3

	`>make EXTRA_CFLAGS=AAAAAA`
	输出 AAAAAA

	`>make EXTRA_CFLAGS:=AAAAAA`
	输出 AAAAAA

	`>make EXTRA_CFLAGS?=AAAAAA`
	输出 AAAAAA

	`>make EXTRA_CFLAGS+=AAAAAA`
	输出 AAAAAA

## 变量值的替换

我们可以替换变量中的共有的部分，其格式是` $(var:a=b)` 或是 `${var:a=b}`，其意思是，把变量 `var` 中所有以 "a" 字串结尾的 "a" 替换成 "b" 字串。例如：
```c
foo := a.o b.o c.o
bar := $(foo:.o=.c)
```
这个示例中，我们先定义了一个 `$(foo)` 变量，而第二行的意思是把 `$(foo)` 中所有以 ".o" 字串结尾全部替换成 ".c"，所以我们的 "$(bar)" 的值就是 "a.c b.c c.c"。

另外一种变量替换的技术是以**静态模式**,如下：
```c
foo := a.o b.o c.o
bar := $(foo:%.o=%.c)
```
这依赖于被替换字串中的有相同的模式，模式中必须包含一个 "%" 字符，这个例子同样让 `$(bar)` 变量的值为 "a.c b.c c.c"。

## 环境变量
make 运行时的系统环境变量可以在 make 开始运行时被载入到 Makefile 文件中，但是如果 Makefile 中已定义了这个变量，或是这个变量由 make 命令行带入，那么系统的环境变量的值将被覆盖。（如果 make 指定了 "-e" 参数，那么，系统环境变量将覆盖 Makefile 中定义的变量）。

因此，如果我们在环境变量中设置了 "CFLAGS" 环境变量，那么我们就可以在所有的 Makefile 中使用这个变量了。这对于我们使用统一的编译参数有比较大的好处。如果 Makefile 中定义了 CFLAGS，那么则会使用 Makefile 中的这个变量，如果没有定义则使用系统环境变量的值，一个共性和个性的统一，很像 "全局变量" 和 "局部变量" 的特性。

## 条件判断
使用条件判断，可以让 make 根据运行时的不同情况选择不同的执行分支。条件表达式可以是比较变量的值，或是比较变量和常量的值。

`ifeq (<arg1>, <arg2>)` 判断两个参数是否相等

下面的例子，判断$(CC)变量是否"gcc"，如果是的话，则使用 GNU 函数编译目标。

```c
libs_for_gcc = -lgnu
normal_libs =
foo: $(objects)
ifeq ($(CC),gcc)
$(CC) -o foo $(objects) $(libs_for_gcc)
else
$(CC) -o foo $(objects) $(normal_libs)
endif
```
可见，在上面示例的这个规则中，目标 "foo" 可以根据变量 "$(CC)" 值来选取不同的函数库来编译程序。

我们可以从上面的示例中看到三个关键字：ifeq、else 和 endif。ifeq 的意思表示条件语句的开始，并指定一个条件表达式，表达式包含两个参数，以逗号分隔，表达式以圆括号括起。else 表示条件表达式为假的情况。endif 表示一个条件语句的结束，任何一个条件表达式都应该以 endif 结束。

1. `ifneq (<arg1>, <arg2>)` 判断两个参数是否不相等
2. `ifdef <variable-name>` 如果变量 `<variable-name>` 的值非空，那到表达式为真
3. `ifndef <variable-name>` 如果变量 `<variable-name>` 的值为空，那到表达式为真


## 使用函数
在 Makefile 中可以使用函数来处理变量，从而让我们的命令或是规则更为的灵活和具有智能。make 所支持的函数也不算很多，不过已经足够我们的操作了。函数调用后，函数的返回值可以当做变量来使用。

函数调用，很像变量的使用，也是以` "$" `来标识的，其语法如下：
`$(<function> <arguments>)`
或是
`${<function> <arguments>}`

这里，`<function>`就是函数名，make 支持的函数不多。`<arguments> `是函数的参数，参数间以逗号 "," 分隔，而函数名和参数之间以 "空格" 分隔。函数调用以` "$" `开头，以圆括号或花括号把函数名和参数括起。感觉很像一个变量，是不是？函数中的参数可以使用变量，为了风格的统一，函数和变量的括号最好一样，如使用 `$(subst a,b,$(x))` 这样的形式，而不是 `$(subst a,b,${x})` 的形式。因为统一会更清楚，也会减少一些不必要的麻烦。

还是来看一个示例：
```c
comma:= ,
empty:=
space:= $(empty) $(empty)
foo:= a b c
bar:= $(subst $(space),$(comma),$(foo))
```
在这个示例中，`$(comma)` 的值是一个逗号。`$(space)` 使用了`$(empty)` 定义了一个空格， `$(foo)` 的值是 "a b c" 。调用了函数 `subst`，这是一个替换函数，这个函数有三个参数，第一个参数是被替换字串，第二个参数是替换字串，第三个参数是替换操作作用的字串。这个函数也就是把 `$(foo)` 中的空格替换成逗号，所以 `$(bar)` 的值是 "a,b,c"。

### 字符串处理函数

1. **subst**
`$(subst <from>,<to>,<text>)`
名称：字符串替换函数。
功能：把字串 `<text> `中的 `<from>` 字符串替换成 `<to>`。
返回：函数返回被替换过后的字符串。
示例：
`$(subst ee,EE,feet on the street)，`
把 "feet on the street" 中的 "ee" 替换成 "EE"，返回结果是 "fEEt on the strEEt"。

1.  **patsubst**
`$(patsubst <pattern>,<replacement>,<text>)`
名称：模式字符串替换函数。
功能：查找 `<text>` 中的单词（单词以 "空格"、"Tab" 或 "回车" "换行"分隔）是否符合模式 `<pattern>` ，如果匹配的话，则以 `<replacement>` 替换。这里，`<pattern>` 可以包括通配符 "%"， 表示任意长度的字串。 如果 `<replacement>` 中也包含 "%"， 那么，`<replacement>` 中的这个 "%" 将是 `<pattern>` 中的那个 "%" 所代表的字串。（可以用 "\" 来转义， 以 "\%" 来表示真实含义的 "%" 字符）返回：函数返回被替换过后的字符串。
示例：
`$(patsubst %.c,%.o,x.c.c bar.c)`
把字串 "x.c.c bar.c" 符合模式 "%.c" 的单词替换成 "%.o"，返回结果是 "x.c.o bar.o"。
备注：这和我们前面"变量章节"说过的相关知识有点相似。
如：`$(var:<pattern>=<replacement>)`
相当于 `$(patsubst <pattern>,<replacement>,$(var))`,
而 `$(var: <suffix>=<replacement>)` 则相当于 `$(patsubst %<suffix>,%<replacement>,$(var))`,
例如有：`objects = foo.o bar.o baz.o`，
那么，`$(objects:.o=.c)` 和 `$(patsubst %.o,%.c,$(objects))"` 是一样的。

1. **strip**
`$(strip <string>)`
名称：去空格函数——strip。
功能：去掉 `<string>` 字串中开头和结尾的空字符。
返回：返回被去掉空格的字符串值。
示例：
`$(strip a b c )`
把字串 "a b c " 去掉开头和结尾的空格，结果是 "a b c"。

4. **findstring**
`$(findstring <find>,<in>)`
名称：查找字符串函数。
功能：在字串 `<in>` 中查找 `<find>` 字串。
返回：如果找到，那么返回 `<find>`，否则返回空字符串。
示例：
`$(findstring a,a b c)`
`$(findstring a,b c)`
第一个函数返回"a"字符串，第二个返回 "" 字符串（空字符串）

5. **filter**
`$(filter <pattern...>,<text>)`
名称：过滤函数。
功能：以 `<pattern>` 模式过滤 `<text>` 字符串中的单词，保留符合模式 `<pattern>` 的单词。可以
有多个模式。
返回：返回符合模式<pattern>的字串。
示例：
	```c
	sources := foo.c bar.c baz.s ugh.h
	foo: $(sources)
	cc $(filter %.c %.s,$(sources)) -o foo
	```
	`$(filter %.c %.s,$(sources))` 返回的值是 "foo.c bar.c baz.s"。

6. **filter-out**
`$(filter-out <pattern...>,<text>)`
名称：反过滤函数。
功能：以 `<pattern>` 模式过滤 `<text>` 字符串中的单词，去除符合模式 `<pattern>` 的单词。可以
有多个模式。
返回：返回不符合模式 `<pattern>` 的字串。
示例：
	```c
	objects=main1.o foo.o main2.o bar.o
	mains=main1.o main2.o
	$(filter-out $(mains),$(objects))
	```
	`$(filter-out $(mains),$(objects))` 返回值是 "foo.o bar.o"。

7. **sort**
`$(sort <list>)`
名称：排序函数。
功能：给字符串 `<list>` 中的单词排序（升序）。
返回：返回排序后的字符串。
示例：`$(sort foo bar lose)`返回 "bar foo lose"。
备注：sort 函数会去掉 `<list>` 中相同的单词。

8. **word**
`$(word <n>,<text>)`
名称：取单词函数。
功能：取字符串 `<text>` 中第 `<n>` 个单词。（从一开始）
返回：返回字符串 `<text>` 中第 `<n>` 个单词。如果 `<n>` 比 `<text>` 中的单词数要大，那么返
回空字符串。
示例：`$(word 2, foo bar baz)` 返回值是 "bar"。

9. **wordlist**
`$(wordlist <s>,<e>,<text>)`
名称：取单词串函数。
功能：从字符串 `<text>` 中取从 `<s>` 开始到 `<e>` 的单词串。`<s>` 和 `<e>` 是一个数字。
返回：返回字符串 `<text>` 中从 `<s>` 到 `<e>` 的单词字串。如果 `<s>` 比 `<text>` 中的单词数要大，那
么返回空字符串。如果 `<e>` 大于 `<text>` 的单词数，那么返回从 `<s>` 开始，到 `<text>` 结束的单词
串。
示例： `$(wordlist 2, 3, foo bar baz)` 返回值是 "bar baz"。

10. **words**
`$(words <text>)`
名称：单词个数统计函数。
功能：统计 `<text>` 中字符串中的单词个数。
返回：返回 `<text>` 中的单词数。
示例：`$(words, foo bar baz)` 返回值是 "3"。
备注：如果我们要取 `<text>` 中最后的一个单词，我们可以这样：`$(word $(words <te
xt>),<text>)`。

11. **firstword**
`$(firstword <text>)`
名称：首单词函数。
功能：取字符串 `<text>` 中的第一个单词。
返回：返回字符串 `<text>` 的第一个单词。
示例：`$(firstword foo bar)` 返回值是 "foo"。
备注：这个函数可以用 `word` 函数来实现：`$(word 1,<text>)`。

**字符串函数实例**
`override CFLAGS += $(patsubst %,-I%,$(subst :, ,$(VPATH)))`
如果我们的 `$(VPATH)` 值是 "src:../headers"，那么 `$(patsubst %,-I%,$(subst :, ,$(VPATH)))` 将返回 "-Isrc -I../headers"，这正是 cc 或 gcc 搜索头文件路径的参数。

### 文件名操作函数
下面我们要介绍的函数主要是处理文件名的。每个函数的参数字符串都会被当做一个或是一系列的文件名来对待。

1. **dir**
`$(dir <names...>)`
名称：取目录函数。
功能：从文件名序列 `<names>` 中取出目录部分。目录部分是指最后一个反斜杠（"/"）之前
的部分。如果没有反斜杠，那么返回 "./"。
返回：返回文件名序列 `<names>` 的目录部分。
示例： `$(dir src/foo.c hacks)` 返回值是 "src/ ./"。

2. **notdir**
`$(notdir <names...>)`
名称：取文件函数。
功能：从文件名序列 `<names>` 中取出非目录部分。非目录部分是指最后一个反斜杠（"/"）
之后的部分。
返回：返回文件名序列 `<names>` 的非目录部分。
示例： `$(notdir src/foo.c hacks)` 返回值是 "foo.c hacks"。

3.  **suffix**
`$(suffix <names...>)`
名称：取后缀函数。
功能：从文件名序列 `<names>` 中取出各个文件名的后缀。
返回：返回文件名序列 `<names>` 的后缀序列，如果文件没有后缀，则返回空字串。
示例：`$(suffix src/foo.c src-1.0/bar.c hacks)` 返回值是 ".c .c"。

3. **basename**
`$(basename <names...>)`
名称：取前缀函数。
功能：从文件名序列 `<names>` 中取出各个文件名的前缀部分。
返回：返回文件名序列 `<names>` 的前缀序列，如果文件没有前缀，则返回空字串。
示例：`$(basename src/foo.c src-1.0/bar.c hacks)` 返回值是 "src/foo src-1.0/b
ar hacks"。

4. **addsuffix**
`$(addsuffix <suffix>,<names...>)`
名称：加后缀函数。
功能：把后缀 `<suffix>` 加到 `<names>` 中的每个单词后面。
返回：返回加过后缀的文件名序列。
示例：`$(addsuffix .c,foo bar)` 返回值是 "foo.c bar.c"。

5. **addprefix**
`$(addprefix <prefix>,<names...>)`
名称：加前缀函数。
功能：把前缀 `<prefix>` 加到 `<names>` 中的每个单词后面。
返回：返回加过前缀的文件名序列。
示例：`$(addprefix src/,foo bar)` 返回值是 "src/foo src/bar"。

6. **join**
`$(join <list1>,<list2>)`
名称：连接函数。
功能：把 `<list2>` 中的单词对应地加到 `<list1>` 的单词后面。如果 `<list1>` 的单词个数要比
 `<list2>` 的多，那么，`<list1>` 中的多出来的单词将保持原样。如果 `<list2>` 的单词个数要比
 `<list1>` 多，那么，`<list2>` 多出来的单词将被复制到 `<list2>` 中。
返回：返回连接过后的字符串。
示例：`$(join aaa bbb , 111 222 333)` 返回值是 "aaa111 bbb222 333"。

 ### 循环函数 foreach
` $(foreach <var>,<list>,<text>)`
功能：这个函数的意思是，把参数 `<list>` 中的单词逐一取出放到参数 `<var>` 所指定的变量中，然后再执行 `<text>` 所包含的表达式。每一次 `<text>` 会返回一个字符串，循环过程中，`<text>` 的所返回的每个字符串会以空格分隔，最后当整个循环结束时，`<text>` 所返回的每个字符串所组成的整个字符串（以空格分隔）将会是 foreach 函数的返回值。

示例:
```c
names := a b c d
files := $(foreach n,$(names),$(n).o)
```
上面的例子中，`$(name)` 中的单词会被挨个取出，并存到变量 "n" 中，`$(n).o` 每次根据 `$(n)` 计算出一个值，这些值以空格分隔，最后作为 foreach 函数的返回，所以，`$(files)` 的值是 "a.o b.o c.o d.o"。

注意：foreach 中的 `<var>` 参数是一个临时的局部变量，foreach 函数执行完后，参数 `<var>` 的变量将不在作用，其作用域只在 foreach 函数当中。

### if 函数
`$(if <condition>,<then-part>)`
或是
`$(if <condition>,<then-part>,<else-part>)`

功能：if 函数可以包含 else 部分，或是不含。即 if 函数的参数可以是两个，也可以是三个。`<condition>` 参数是 if 的表达式，如果其返回的为非空字符串，那么这个表达式就相当于返回真，于是，`<then-part>` 会被计算，否则 `<else-part>` 会被计算。

返回值：如果 `<condition>` 为真（非空字符串），那个 `<then-part>` 会是整个函数的返回值，如果 `<condition>` 为假（空字符串），那么 `<else-part>` 会是整个函数的返回值，此时如果 `<else-part>` 没有被定义，那么，整个函数返回空字串。所以，`<then-part>` 和 `<else-part>` 只会有一个被计算。

### call 函数
`$(call <expression>,<parm1>,<parm2>,<parm3>...)`

功能：call 函数是唯一一个可以用来创建新的参数化的函数。你可以写一个非常复杂的表达式，这个表达式中，你可以定义许多参数，然后你可以用 call 函数来向这个表达式传递参数。当 make 执行这个函数时，`<expression>` 参数中的变量，如$(1)，$(2)，$(3)等，会被参数 `<parm1>`，`<parm2>`，`<parm3>` 依次取代。而 `<expression>` 的返回值就是 call 函数的返回值。

示例：
```c
reverse = $(1) $(2)
foo = $(call reverse,a,b)
```

### origin 函数
`$(origin <variable>)`

功能：origin 函数不像其它的函数，他并不操作变量的值，他只是告诉你你的这个变量是哪里来的。

注意：`<variable>` 是变量的名字，不应该是引用。所以你最好不要在 `<variable>` 中使用 "$" 字符。Origin 函数会以其返回值来告诉你这个变量的 "出生情况"，下面，是 origin 函数的返回值:
* **undefined**: 如果 `<variable>` 从来没有定义过，origin 函数返回这个值 "undefined"。
* **default**: 如果 `<variable>` 是一个默认的定义，比如 "CC" 这个变量，这种变量我们将在后面讲述。
* **environment**: 如果 `<variable>` 是一个环境变量， 并且当 Makefile 被执行时， "-e" 参数没有被打开。
* **file**: 如果 `<variable>` 这个变量被定义在 Makefile 中。
* **command line**: 如果 `<variable>` 这个变量是被命令行定义的。
* **override**: 如果 `<variable>` 是被 override 指示符重新定义的。
* **automatic**: 如果 `<variable>` 是一个命令运行中的自动化变量

示例：
这些信息对于我们编写 Makefile 是非常有用的，例如，假设我们有一个 Makefile 其包了一个定义文件 Make.def，在 Make.def 中定义了一个变量 "bletch"，而我们的环境中也有一个环境变量 "bletch"，此时，我们想判断一下，如果变量来源于环境，那么我们就把之重定义了，如果来源于 Make.def 或是命令行等非环境的，那么我们就不重新定义它。于是，在我们的 Makefile 中，我们可以这样写：
```c
ifdef bletch
ifeq "$(origin bletch)" "environment"
bletch = barf, gag, etc.
endif
endif
```

### shell 函数
shell 函数也不像其它的函数。顾名思义，它的参数应该就是操作系统 Shell 的命令。它和反引号 "`" 是相同的功能。这就是说， shell 函数把执行操作系统命令后的输出作为函数返回。于是，我们可以用操作系统命令以及字符串处理命令 awk，sed 等等命令来生成一个变量，如：
```c
contents := $(shell cat foo)
files := $(shell echo *.c)
```
注意，这个函数会新生成一个 Shell 程序来执行命令，所以你要注意其运行性能，如果你的 Makefile 中有一些比较复杂的规则，并大量使用了这个函数，那么对于你的系统性能是有害的。特别是 Makefile 的隐晦的规则可能会让你的 shell 函数执行的次数比你想像的多得多。

### 控制 make 运行函数
make 提供了一些函数来控制 make 的运行。通常，你需要检测一些运行 Makefile 时的运行时信息，并且根据这些信息来决定，你是让 make 继续执行，还是停止。

1. **error**
	`$(error <text ...>)`
	产生一个致命的错误，**<text ...>** 是错误信息。注意，error 函数不会在一被使用就会产生错误信息，所以如果你把其定义在某个变量中，并在后续的脚本中使用这个变量，那么也是可以的。

	示例一：
	```c
	ifdef ERROR_001
	$(error error is $(ERROR_001))
	endif
	```

	示例二：
	```
	ERR = $(error found an error!)
	.PHONY: err
	err: ; $(ERR)
	```
	示例一会在变量 ERROR_001 定义了后执行时产生 error 调用，而示例二则在目录 err 被执行时才发生 error 调用。

2. **warning**
	`$(warning <text ...>)`
	这个函数很像 error 函数，只是它并不会让 make 退出，只是输出一段警告信息，而 make 继续执行。

## make 的运行
一般来说，最简单的就是直接在命令行下输入 make 命令，make 命令会找当前目录的 makefile 来执行，一切都是自动的。但也有时你也许只想让 make 重编译某些文件，而不是整个工程，而又有的时候你有几套编译规则，你想在不同的时候使用不同的编译规则。

### make 的退出码

make 命令执行后有三个退出码：
* 0 - 表示成功执行。
* 1 - 如果 make 运行时出现任何错误，其返回 1。
* 2 - 如果你使用了 make 的 "-q" 选项，并且 make 使得一些目标不需要更新，那么返回 2。

### 指定 makefile

GNU make 找寻默认的 Makefile 的规则是在当前目录下依次找三个文件 : `GNUmakefile`、`makefile` 和 `Makefile`。其按顺序找这三个文件，一旦找到，就开始读取这个文件并执行。

当然，我们也可以给 make 命令指定一个特殊名字的 Makefile。要达到这个功能，我们要使用 make 的 "-f" 或是 "--file" 参数（"--makefile" 参数也行）。例如，我们有个makefile 的名字是 "hchen.mk"，那么，我们可以这样来让 make 来执行这个文件：
`make –f hchen.mk`
如果在 make 的命令行是， 你不只一次地使用了 "-f" 参数，那么， 所有指定的 makefile将会被连在一起传递给 make 执行。

### 指定目标
一般来说，make 的最终目标是 makefile 中的第一个目标，而其它目标一般是由这个目标连带出来的。这是 make 的默认行为。当然，一般来说，你的 makefile 中的第一个目标是由许多个目标组成，你可以指示 make，让其完成你所指定的目标。要达到这一目的很简单，需在 make 命令后直接跟目标的名字就可以完成（如前面提到的 "make clean" 形式）

示例：
```c
CC = g++

PROJECT = dlt698
OBJ = $(wildcard *.cpp *.c)

target : $(PROJECT)

$(PROJECT) :  $(OBJ)
	$(CC) -o $@ $(CFLAGS)  $^ 
	-cp $@ $(LOCALDIR)/../target
```
在此 makefile 中，输入 make 和 输入 make target 或者 make dlt698 是相同的行为。

使用指定终极目标的方法可以很方便地让我们编译我们的程序，例如下面这个例子：
```c
.PHONY: all
all: prog1 prog2 prog3 prog4
```
从这个例子中，我们可以看到， 这个 makefile 中有四个需要编译的程序： "prog1"，"prog2"， "prog3" 和 "prog4"，我们可以使用 "make all" 命令来编译所有的目标（如果把 all 置成第一个目标，那么只需执行 "make"），我们也可以使用 "make prog2" 来单独编译目标 "prog2"。

即然 make 可以指定所有 makefile 中的目标，那么也包括"伪目标"，于是我们可以根据这种性质来让我们的 makefile 根据指定的不同的目标来完成不同的事。在 Unix 世界中，软件发布时，特别是 GNU 这种开源软件的发布时，其 makefile 都包含了编译、安装、打包等功能。我们可以参照这种规则来书写我们的 makefile 中的目标。
1. **all**
	这个伪目标是所有目标的目标，其功能一般是编译所有的目标。
2. **clean**
	这个伪目标功能是删除所有被 make 创建的文件。
3. **install**
	这个伪目标功能是安装已编译好的程序，其实就是把目标执行文件拷贝到指定的目标中去。
4. **print**
	这个伪目标的功能是例出改变过的源文件。
5. **tar**
	这个伪目标功能是把源程序打包备份。也就是一个 tar 文件。
6. **dist**
	这个伪目标功能是创建一个压缩文件，一般是把 tar 文件压成 Z 文件。或是 gz 文件
7. **TAGS**
   	这个伪目标功能是更新所有的目标，以备完整地重编译使用。
8. **check** 和 **test**
   	这两个伪目标一般用来测试 makefile 的流程。

当然一个项目的 makefile 中也不一定要书写这样的目标，这些东西都是 GNU 的东西，但是我想，GNU 搞出这些东西一定有其可取之处（等你的 UNIX 下的程序文件一多时你就会发现这些功能很有用了），这里只不过是说明了，如果你要书写这种功能，最好使用这种名字命名你的目标，这样规范一些，规范的好处就是——不用解释，大家都明白。而且如果你的 makefile 中有这些功能，一是很实用，二是可以显得你的 makefile 很专业（不是那种初学者的作品）。

### 规则检查
有时候，我们不想让我们的 makefile 中的规则执行起来，我们只想检查一下我们的命令，或是执行的序列。于是我们可以使用 make 命令的下述参数：
* "-n"
* "--just-print"
* "--dry-run"
* "--recon"
不执行参数，这些参数只是打印命令，不管目标是否更新，把规则和连带规则下的命令打印出来，但不执行，这些参数对于我们调试 makefile 很有用处。
示例：
```c
root@atmel:/mnt/hgfs/Mr Tang/GitHub/Dlt698MsgDeal/src# ls
daemon.pid      dlt698.cpp  makefile             public.cpp
data_queue.cpp  main.cpp    process_manager.cpp  tcp_client.cpp

root@atmel:/mnt/hgfs/Mr Tang/GitHub/Dlt698MsgDeal/src# make -n
g++ -c data_queue.cpp  -I./include -I./../include 
g++ -c dlt698.cpp  -I./include -I./../include 
g++ -c main.cpp  -I./include -I./../include 
g++ -c process_manager.cpp  -I./include -I./../include 
g++ -c public.cpp  -I./include -I./../include 
g++ -c tcp_client.cpp  -I./include -I./../include 
g++ -o dlt698 -fPIC   data_queue.o dlt698.o main.o process_manager.o public.o tcp_client.o -I./include -I./../include  -lpthread -L./../lib  
cp dlt698 ./../target

root@atmel:/mnt/hgfs/Mr Tang/GitHub/Dlt698MsgDeal/src# ls
daemon.pid      dlt698.cpp  makefile             public.cpp
data_queue.cpp  main.cpp    process_manager.cpp  tcp_client.cpp
```
从上述示例中可以看出，make -n 并没有真正运行 make 命令，而只是显示 make 执行的过程。

* "-t"
* "--touch"
这个参数的意思就是把目标文件的时间更新，但不更改目标文件。也就是说，make 假装编译目标，但不是真正的编译目标，只是把目标变成已编译过的状态。

* "-q"
* "--question"
这个参数的行为是找目标的意思，也就是说，如果目标存在，那么其什么也不会输出，当然也不会执行编译，如果目标不存在，其会打印出一条出错信息。

* "-W <file>"
* "--what-if=<file>"
* "--assume-new=<file>"
* "--new-file=<file>"
这个参数需要指定一个文件。一般是是源文件（或依赖文件），Make 会根据规则推导来运行依赖于这个文件的命令，一般来说，可以和 "-n" 参数一同使用，来查看这个依赖文件
所发生的规则命令。另外一个很有意思的用法是结合"-p"和"-v"来输出 makefile 被执行时的信息（这个将在后面讲述）。

### make 的参数

* "-b"
* "-m"
这两个参数的作用是忽略和其它版本 make 的兼容性。

* "-B"
"--always-make"
认为所有的目标都需要更新（重编译）。

* "-C `<dir>`"
* "--directory=`<dir>` "
	指定读取 makefile 的目录。如果有多个 "-C" 参数，make 的解释是后面的路径以前面的作为相对路径，并以最后的目录作为被指定目录。如： `"make –C ~hchen/test –C prog"` 等价于 `"make –C ~hchen/test/prog"`。

	示例：
	makefile 在 "/mnt/hgfs/Mr Tang/GitHub/Dlt698MsgDeal/src" 路径下， 可在别的路径下运行 make 命令，我这里在 makefile 上一级运行 make。
	```c
	root@atmel:/mnt/hgfs/Mr Tang/GitHub/Dlt698MsgDeal# make -C ./src
	make: Entering directory `/mnt/hgfs/Mr Tang/GitHub/Dlt698MsgDeal/src'
	g++ -c data_queue.cpp  -I./include -I./../include 
	g++ -c dlt698.cpp  -I./include -I./../include 
	g++ -c main.cpp  -I./include -I./../include 
	g++ -c process_manager.cpp  -I./include -I./../include 
	g++ -c public.cpp  -I./include -I./../include 
	g++ -c tcp_client.cpp  -I./include -I./../include 
	g++ -o dlt698 -fPIC   data_queue.o dlt698.o main.o process_manager.o public.o tcp_client.o -I./include -I./../include  -lpthread -L./../lib  
	cp dlt698 ./../target
	make: Leaving directory `/mnt/hgfs/Mr Tang/GitHub/Dlt698MsgDeal/src'
	root@atmel:/mnt/hgfs/Mr Tang/GitHub/Dlt698MsgDeal# 
	```

* "-I `<dir>`"
* "--include-dir=`<dir>`"
指定一个被包含 makefile 的搜索目标。可以使用多个 "-I" 参数来指定多个目录。

* "-r"
禁止 make 使用任何隐含规则。

* "-R"
禁止 make 使用任何作用于变量上的隐含规则。

* "-s"
在命令运行时不输出命令的输出。

## 隐含规则

在我们使用 Makefile 时，有一些我们会经常使用，而且使用频率非常高的东西，比如，我们编译 C/C++的源程序为中间目标文件（Unix 下是 [.o] 文件，Windows 下是 [.obj] 文件）。这里讲述的就是一些在 Makefile 中的 "隐含的"，早先约定了的，不需要我们再写出来的规则。

"隐含规则" 也就是一种惯例，make 会按照这种“惯例”心照不喧地来运行，那怕我们的 Makefile 中没有书写这样的规则。例如，把 [.c] 文件编译成 [.o] 文件这一规则，你根本就不用写出来，make 会自动推导出这种规则，并生成我们需要的 [.o] 文件。

"隐含规则" 会使用一些我们系统变量，我们可以改变这些系统变量的值来定制隐含规则的运行时的参数。如系统变量"CFLAGS" 可以控制编译时的编译器参数。

我们还可以通过 "模式规则" 的方式写下自己的隐含规则。用 "后缀规则" 来定义隐含规则会有许多的限制。使用 "模式规则" 会更回得智能和清楚，但 "后缀规则" 可以用来保证我们 Makefile 的兼容性。

我们了解了 "隐含规则"，可以让其为我们更好的服务，也会让我们知道一些 "约定俗成" 了的东西，而不至于使得我们在运行 Makefile 时出现一些我们觉得莫名其妙的东西。当然，任何事物都是矛盾的，水能载舟，亦可覆舟，所以，有时候 "隐含规则" 也会给我们造成不小的麻烦。只有了解了它，我们才能更好地使用它。

### 使用隐含规则

如果要使用隐含规则生成你需要的目标，你所需要做的就是不要写出这个目标的规则。那么，make 会试图去自动推导产生这个目标的规则和命令，如果 make 可以自动推导生成这个目标的规则和命令，那么这个行为就是隐含规则的自动推导。当然，隐含规则是 make 事先约定好的一些东西。例如，我们有下面的一个 Makefile：

```c
foo : foo.o bar.o
cc –o foo foo.o bar.o $(CFLAGS) $(LDFLAGS)
```

我们可以注意到，这个 Makefile 中并没有写下如何生成 foo.o 和 bar.o 这两目标的规则和命令。因为 make 的 "隐含规则" 功能会自动为我们自动去推导这两个目标的依赖目标和生成命令。

make 会在自己的 "隐含规则" 库中寻找可以用的规则，如果找到，那么就会使用。如果找不到，那么就会报错。在上面的那个例子中，make 调用的隐含规则是，把 [.o] 的目标的依赖文件置成[.c]，并使用 C 的编译命令 "cc –c $(CFLAGS) [.c]" 来生成 [.o] 的目标。

也就是说，我们完全没有必要写下下面的两条规则：

```c
foo.o : foo.c
cc –c foo.c $(CFLAGS)
bar.o : bar.c
cc –c bar.c $(CFLAGS)
```
因为，这已经是 "约定" 好了的事了， make 和我们约定好了用 C 编译器 "cc" 生成 [.o] 文件的规则，这就是隐含规则。

当然，如果我们为 [.o] 文件书写了自己的规则，那么 make 就不会自动推导并调用隐含规则，它会按照我们写好的规则忠实地执行。

还有，在 make 的 "隐含规则库" 中，每一条隐含规则都在库中有其顺序，越靠前的则是越被经常使用的，所以，这会导致我们有些时候即使我们显示地指定了目标依赖，make 也不会管。如下面这条规则（没有命令）：
`foo.o : foo.p`
依赖文件 "foo.p"（Pascal 程序的源文件）有可能变得没有意义。如果目录下存在了 "foo.c" 文件，那么我们的隐含规则一样会生效，并会通过 "foo.c" 调用 C 的编译器生成 foo.o 文件。因为，在隐含规则中，Pascal 的规则出现在 C 的规则之后，所以，make 找到可以生成 foo.o 的 C 的规则就不再寻找下一条规则了。如果你确实不希望任何隐含规则推导，那么，你就不要只写出 "依赖规则"，而不写命令。

### 隐含规则一览

这里我们将讲述所有预先设置（也就是 make 内建）的隐含规则，如果我们不明确地写下规则，那么，make 就会在这些规则中寻找所需要规则和命令。当然，我们也可以使用 make 的参数 "-r" 或 "--no-builtin-rules" 选项来取消所有的预设置的隐含规则。

当然，即使是我们指定了 "-r" 参数，某些隐含规则还是会生效，因为有许多的隐含规则都是使用了“后缀规则”来定义的，所以，只要隐含规则中有 "后缀列表"（也就一系统定义在目标.SUFFIXES 的依赖目标），那么隐含规则就会 生效。默认的后缀列表是：.out,.a, .ln, .o, .c, .cc, .C, .p, .f, .F, .r, .y, .l, .s, .S, .mod, .sym,.def, .h, .info, .dvi, .tex, .texinfo, .texi, .txinfo, .w, .ch .web, .sh, .elc, .el。

1. 编译 C 程序的隐含规则
"<n>.o" 的目标的依赖目标会自动推导为 "<n>.c" ，并且其生成命令是 `$(CC) –c $(CPPFLAGS) $(CFLAGS)`

2. 编译 C++ 程序的隐含规则
"<n>.o" 的目标的依赖目标会自动推导为 "<n>.cc" 或是 "<n>.C"， 并且其生成命令是 `$(CXX) –c $(CPPFLAGS) $(CFLAGS)`。

### 定义模式规则

* 模式规则介绍

	你可以使用模式规则来定义一个隐含规则。一个模式规则就好像一个一般的规则，只是在规则中，目标的定义需要有 "%" 字符。"%" 的意思是表示一个或多个任意字符。在依赖目标中同样可以使用 "%"，只是依赖目标中的 "%" 的取值，取决于其目标。

	有一点需要注意的是，"%" 的展开发生在变量和函数的展开之后，变量和函数的展开发生在 make 载入 Makefile 时，而模式规则中的 "%" 则发生在运行时。

	模式规则中，至少在规则的目标定义中要包含 "%"，否则，就是一般的规则。目标中的 "%" 定义表示对文件名的匹配，"%" 表示长度任意的非空字符串。例如："%.c" 表示以".c" 结尾的文件名（文件名的长度至少为 3），而 "s.%.c" 则表示以 "s." 开头， ".c" 结尾的文件名（文件名的长度至少为 5）。

	如果 "%" 定义在目标中，那么，目标中的 "%" 的值决定了依赖目标中的 "%" 的值，也就是说，目标中的模式的 "%" 决定了依赖目标中"%"的样子。例如有一个模式规则如下：

	`%.o : %.c ; <command ......>`

	其含义是，指出了怎么从所有的 [.c] 文件生成相应的 [.o] 文件的规则。如果要生成的目标是 "a.o b.o"，那么 "%c" 就是 "a.c b.c"。

	一旦依赖目标中的 "%" 模式被确定，那么，make 会被要求去匹配当前目录下所有的文件名，一旦找到，make 就会执行规则下的命令，在模式规则中，目标可能会是多个的，如果有模式匹配出多个目标，make 就会产生所有的模式目标，此时，make 关心的是依赖的文件名和生成目标的命令这两件事。

* 模式规则示例
	下面这个例子表示了,把所有的 [.c] 文件都编译成 [.o] 文件:
	```c
	%.o : %.c
	$(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@
	```
* 自动化变量

	> 所谓自动化变量，就是这种变量会把模式中所定义的一系列的文件自动地挨个取出，直至所有的符合模式的文件都取完了。这种自动化变量只应出现在规则的命令中。

	在上述的模式规则中，目标和依赖文件都是一系例的文件，那么我们如何书写一个命令来完成从不同的依赖文件生成相应的目标？ 因为在每一次的对模式规则的解析时，都会是不同的目标和依赖文件。

	自动化变量就是完成这个功能的。在前面，我们已经对自动化变量有所提涉，相信你看到这里已对它有一个感性认识了。
	
	下面是所有的自动化变量及其说明：

	**`$@`** :表示规则中的目标文件集。在模式规则中，如果有多个目标，那么，"$@" 就是匹配于目标中模式定义的集合。

	**`$%`** :仅当目标是函数库文件中，表示规则中的目标成员名。例如，如果一个目标是 `foo.a (bar.o)`，那么，`$%` 就是 "bar.o"，`$@` 就是 "foo.a"。

	**`$<`** :依赖中的第一个依赖名字。如果依赖目标是以模式（即 "%" ）定义的，那么 "$<" 将是符合模式的一系列的文件集。注意，其是一个一个取出来的。

	**`$?`** ：所有比目标新的依赖目标的集合。以空格分隔。
	
	**`$^`** ：所有的依赖目标的集合。以空格分隔。如果在依赖目标中有多个重复的，那个这个变量会去除重复的依赖目标，只保留一份。
	
	**`$+`** ：这个变量很像 "$^"，也是所有依赖目标的集合。只是它不去除重复的依赖目标。
	
	**`$*`** ：这个变量表示目标模式中 "%" 及其之前的部分。如果目标是 `dir/a.foo.b`，并且目标的模式是 `a.%.b`，那么，`$*` 的值就是 `dir/a.foo`。这个变量对于构造有关联的文件名是比较有较。如果目标中没有模式的定义，那么 `$*` 也就不能被推导出，但是，如果目标文件的后缀是 make 所识别的，那就是除了后缀的那一部分。例如：如果目标是 "foo.c"，因为 ".c"是 make 所能识别的后缀名，所以，`$*` 的值就是 "foo"。这个特性是 GNU make 的，很有可能不兼容于其它版本的 make，所以，你应该尽量避免使用 `$*`，除非是在隐含规则或是静态模式中。如果目标中的后缀是 make 所不能识别的，那么 `$*` 就是空值。

	当你希望只对更新过的依赖文件进行操作时，`$?` 在显式规则中很有用，例如，假设有一个函数库文件叫 "lib"，其由其它几个 object 文件更新。那么把 object 文件打包的比较有效率的 Makefile 规则是：

	```c
	lib : foo.o bar.o lose.o win.o
	ar r lib $?
	```

	在上述所列出来的自动量变量中。四个变量 `$@、$<、$%、$*` 在扩展时只会有一个文件，而另三个的值是一个文件列表。这七个自动化变量还可以取得文件的目录名或是在当前目录下的符合模式的文件名，只需要搭配上 "D" 或 "F" 字样。这是 GNU make 中老版本的特性，在新版本中，我们使用函数 "dir" 或 "notdir" 就可以做到了。"D" 的含义就是 Directory，就是目录，"F" 的含义就是 File，就是文件。下面是对于上面的七个变量分别加上"D"或是"F"的含义：

	下面是对于上面的七个变量分别加上"D"或是"F"的含义：
	
	**`$(@D)`** ：表示 `$@` 的目录部分（不以斜杠作为结尾），如果 `$@` 值是"dir/foo.o"，那么 `$(@D)` 就是 "dir"，而如果 `$@` 中没有包含斜杠的话，其值就是"."（当前目录）。
	
	**`$(@F)`** : 表示 `$@` 的文件部分，如果 `$@` 值是 "dir/foo.o"， 那么 `$(@F)` 就是 "foo.o"，`$(@F)` 相当于函数 `$(notdir $@)`。

	**`"$(*D)"`** **`"$(*F)"`** : 	和上面所述的同理，也是取文件的目录部分和文件部分。对于上面的那个例子， `$(*D)` 返回 "dir"，而 `$(*F)` 返回 "foo"

	**`$(%D)`** **`$(%F)`** : 分别表示了函数包文件成员的目录部分和文件部分。这对于形同 "archive(member)" 形式的目标中的 "member" 中包含了不同的目录很有用。

	**`$(<D)`** **`$(<F)`** : 分别表示依赖文件的目录部分和文件部分。
	
	**`$(^D)`** **`$(^F)`**	: 分别表示所有依赖文件的目录部分和文件部分。

* 重载内建隐含规则
	你可以重载内建的隐含规则（或是定义一个全新的），例如你可以重新构造和内建隐含规则不同的命令，如：
	```c
	%.o : %.c
	$(CC) -c $(CPPFLAGS) $(CFLAGS) -D$(date)
	```
	你可以取消内建的隐含规则，只要不在后面写命令就行。如：
	`%.o : %.s`
	同样，你也可以重新定义一个全新的隐含规则，其在隐含规则中的位置取决于你在哪里写下这个规则。朝前的位置就靠前。

* 老式风格的 "后缀规则"
	后缀规则是一个比较老式的定义隐含规则的方法。后缀规则会被模式规则逐步地取代。因为模式规则更强更清晰。为了和老版本的 Makefile 兼容，GNU make 同样兼容于这些东西。后缀规则有两种方式："双后缀"和"单后缀"。

	双后缀规则定义了一对后缀：目标文件的后缀和依赖目标（源文件）的后缀。**如 ".c.o" 相当于 "%o : %c"**。单后缀规则只定义一个后缀，也就是源文件的后缀。如 ".c" 相当于 "% :%.c"。

	后缀规则中所定义的后缀应该是 make 所认识的，如果一个后缀是 make 所认识的，那么这个规则就是单后缀规则，而如果两个连在一起的后缀都被 make 所认识，那就是双后缀规则。例如：".c"和".o"都是 make 所知道。因而，如果你定义了一个规则是".c.o"那么其就是双后缀规则，意义就是".c"是源文件的后缀，".o"是目标文件的后缀。如下示例：
	
	```c
	.c.o:
	$(CC) -c $(CFLAGS) $(CPPFLAGS) -o $@ $<
	```

	后缀规则不允许任何的依赖文件，如果有依赖文件的话，那就不是后缀规则，那些后缀统统被认为是文件名，如：
	
	```c
	.c.o: foo.h
	$(CC) -c $(CFLAGS) $(CPPFLAGS) -o $@ $<
	```

	这个例子，就是说，文件".c.o"依赖于文件"foo.h"，而不是我们想要的这样：
	
	```c
	%.o: %.c foo.h
	$(CC) -c $(CFLAGS) $(CPPFLAGS) -o $@ $<
	```
