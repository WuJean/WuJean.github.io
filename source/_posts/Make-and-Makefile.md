---
title: Make and Makefile
date: 2023-04-02 22:35:12
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover: https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402223938899.png
---

- ![image-20230402223644282](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402223644282.png)

# 一、如何编译一个程序

![image-20230402223708316](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402223708316.png)

要编译一个C语言程序，通常需要使用C编译器将源代码文件编译成目标文件，再将多个目标文件链接成可执行文件。以下是使用GCC编译器编译C语言程序的基本步骤：

1. 编写C语言程序的源代码文件，通常以.c为文件扩展名。例如，我们可以创建一个名为main.c的源代码文件，包含以下代码：

```C
#include <stdio.h>int main() {
    printf("Hello, world!\n");
    return 0;
}
```

1. 打开终端，并切换到C语言程序所在的目录。
2. 使用GCC编译器将源代码文件编译成目标文件。命令格式为：

```Bash
gcc -c source_file.c -o object_file.o
```

其中，source_file.c为源代码文件名，object_file.o为目标文件名。例如，我们可以使用以下命令将main.c编译成目标文件main.o：

```Bash
gcc -c main.c -o main.o
```

1. 如果程序包含多个源代码文件，则需要分别编译每个源代码文件，并将它们链接成可执行文件。命令格式为：

```Bash
gcc object_file1.o object_file2.o ... -o executable_file
```

其中，object_file1.o、object_file2.o等为各个目标文件名，executable_file为可执行文件名。例如，我们可以使用以下命令将main.o链接成可执行文件main：

```Bash
gcc main.o -o main
```

1. 在终端中执行可执行文件，即可运行C语言程序。

```Bash
./main
```

以上就是使用GCC编译器编译C语言程序的基本步骤。在实际开发中，可能需要添加编译选项、链接库等操作。

![image-20230402223736156](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402223736156.png)

**但是一个工程往往包含很多源文件以及很多依赖，如果我们仅仅靠手动指定编译命令，很容易犯错且效率低下。**

# 二、如何构建一个工程

![image-20230402223755063](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402223755063.png)

构建和管理C语言工程通常需要遵循一定的结构和规范，以方便项目的组织、维护和扩展。以下是一个简单的C语言工程结构示例：

```Plain
my_project/
├── include/
│   ├── module1.h
│   ├── module2.h
│   └── ...
├── src/
│   ├── main.c
│   ├── module1.c
│   ├── module2.c
│   └── ...
├── lib/
│   ├── libmodule1.a
│   ├── libmodule2.a
│   └── ...
├── bin/
│   └── my_program
└── Makefile
```

其中，include目录存放头文件，src目录存放源代码文件，lib目录存放静态库文件，bin目录存放可执行文件，Makefile为构建工具的配置文件。以下是一个简单的Makefile示例：

```Makefile
CC = gcc
CFLAGS = -Wall -Werror -I./include
LDFLAGS = -L./lib
LDLIBS = -lmodule1 -lmodule2

SRCS = $(wildcard src/*.c)
OBJS = $(patsubst %.c, %.o, $(SRCS))
EXECUTABLE = bin/my_program

all: $(EXECUTABLE)

$(EXECUTABLE):$(OBJS)
    $(CC) $(LDFLAGS) $(LDLIBS) $^ -o $@$
    
(OBJS): $(SRCS)
    $(CC) $(CFLAGS) -c $^ -o $@

clean:
    rm -rf $(OBJS) $(EXECUTABLE)
    
.PHONY: all clean
```

以上示例中，CC定义了编译器为GCC，CFLAGS定义了编译选项为-Wall -Werror -I./include，LDFLAGS定义了链接选项为-L./lib，LDLIBS定义了链接库为-lmodule1 -lmodule2。SRCS定义了源代码文件列表，OBJS定义了目标文件列表，EXECUTABLE定义了可执行文件名。规则定义中，all表示默认目标，$(EXECUTABLE)表示构建可执行文件的目标，$(OBJS)表示编译目标文件的目标，clean表示清理临时文件的目标。规则中的$^表示依赖文件列表，$@表示目标文件名，$<表示第一个依赖文件名。

![](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402223900552.png)

使用以上C语言工程结构和Makefile示例，可以按照以下步骤构建和管理C语言工程：

1. 在my_project/include目录下创建头文件，如module1.h、module2.h等。
2. 在my_project/src目录下创建源代码文件，如main.c、module1.c、module2.c等。
3. 在my_project/lib目录下创建静态库文件，如libmodule1.a、libmodule2.a等。可以使用以下命令生成静态库文件：

```Bash
gcc -c module1.c -o module1.o
ar rcs libmodule1.a module1.o
```

其中，-c选项表示编译成目标文件，-o选项表示指定输出文件名，ar命令用于打包成静态库

1. 在my_project/bin目录下创建可执行文件目录。
2. 编写Makefile，并保存为my_project/Makefile。
3. 在my_project目录下运行make命令，即可自动构建可执行文件。可以使用以下命令：

```Bash
cd my_project
make
```

make会自动查找Makefile，并按照其中的规则构建目标文件和可执行文件。构建完成后，可执行文件将保存在my_project/bin目录下。

1. 可以使用make clean命令清理临时文件。可以使用以下命令：

```Bash
cd my_project
make clean
```

make clean会删除所有目标文件和可执行文件，以便重新构建。

1. 可以使用make install命令安装可执行文件到系统目录。可以使用以下命令：

```Bash
cd my_project
make install
```

make install会将可执行文件复制到/usr/local/bin目录下，以便全局使用。

以上是一个简单的C语言工程构建和管理示例，实际工程可能需要更复杂的结构和规则，具体需求可以根据实际情况进行调整。

**我们注意到编译一个工程与编译一个项目完全不同，但对于得到一份源码**并想要得到可执行文件的用户来说，他只需要在命令行中输入简单的几个make命令就完事了。

**makefile**文件与make命令到底是如何简化编译的过程的？

# 三、什么是Make

![image-20230402223938899](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402223938899.png)

​       Make是一种工具程序，它可以自动化构建（build）可执行文件、库文件等。Make通过读取一个名为Makefile的文件来获取构建相关信息，Makefile中包含了一系列规则，规定了如何编译源代码、链接目标文件、生成可执行文件等步骤。

​      要学习make工具，需要明白三个概念：目标、依赖、处理动作。

​      makefile所要进行的主要内容是明确目标、明确目标所依赖的内容、明确依赖条件满足时应该执行对应的处理        动作。

make的特性：

1. 自动化构建：Make可以自动化构建源代码，从而减少人工操作和提高效率。
2. 依赖关系管理：Make可以根据文件之间的依赖关系，自动判断哪些文件需要重新编译，从而避免不必要的重复工作。
3. 灵活性：Make可以根据具体的工程需求，定制不同的编译规则和构建流程。
4. 可扩展性：Make支持使用变量、函数等高级语法，可以根据需要编写自定义规则。
5. 可移植性：Make可以在不同的操作系统和编译器上使用，具有良好的跨平台性。
6. 易于维护：Make可以帮助开发者管理工程中的各个模块，使得整个工程更加清晰、易于维护。
7. 支持并行构建：Make支持并行构建，可以在多核CPU上提高构建速度。
8. 支持多语言：除了C语言外，Make还可以支持多种编程语言，包括C++、Java、Python等。

```Shell
$ make -h
Usage: make [options] [target] ...
Options:
  -b, -m                      Ignored for compatibility.
  -B, --always-make           Unconditionally make all targets.
  -C DIRECTORY, --directory=DIRECTORY
                              Change to DIRECTORY before doing anything.
  -d                          Print lots of debugging information.
  --debug[=FLAGS]             Print various types of debugging information.
  -e, --environment-overrides
                              Environment variables override makefiles.
  -E STRING, --eval=STRING    Evaluate STRING as a makefile statement.
  -f FILE, --file=FILE, --makefile=FILE
                              Read FILE as a makefile.
  -h, --help                  Print this message and exit.
  -i, --ignore-errors         Ignore errors from recipes.
  -I DIRECTORY, --include-dir=DIRECTORY
                              Search DIRECTORY for included makefiles.
  -j [N], --jobs[=N]          Allow N jobs at once; infinite jobs with no arg.
  -k, --keep-going            Keep going when some targets can't be made.
  -l [N], --load-average[=N], --max-load[=N]
                              Don't start multiple jobs unless load is below N.
  -L, --check-symlink-times   Use the latest mtime between symlinks and target.
  -n, --just-print, --dry-run, --recon
                              Don't actually run any recipe; just print them.
  -o FILE, --old-file=FILE, --assume-old=FILE
                              Consider FILE to be very old and don't remake it.
  -O[TYPE], --output-sync[=TYPE]
                              Synchronize output of parallel jobs by TYPE.
  -p, --print-data-base       Print make's internal database.
  -q, --question              Run no recipe; exit status says if up to date.
  -r, --no-builtin-rules      Disable the built-in implicit rules.
  -R, --no-builtin-variables  Disable the built-in variable settings.
  -s, --silent, --quiet       Don't echo recipes.
  --no-silent                 Echo recipes (disable --silent mode).
  -S, --no-keep-going, --stop
                              Turns off -k.
  -t, --touch                 Touch targets instead of remaking them.
  --trace                     Print tracing information.
  -v, --version               Print the version number of make and exit.
  -w, --print-directory       Print the current directory.
  --no-print-directory        Turn off -w, even if it was turned on implicitly.
  -W FILE, --what-if=FILE, --new-file=FILE, --assume-new=FILE
                              Consider FILE to be infinitely new.
  --warn-undefined-variables  Warn when an undefined variable is referenced.
```

# 四、如何写makefile

## 4.1 编写的基本规则

makefile文件由一组**依赖关系**和**规则**构成。每个依赖关系都由一个目标（即将要创建的文件）和一个该目标所依赖的源文件组成；规则描述了如何通过这些依赖文件创建目标。写法如下：

```Plain
target: prerequisites
    command1
    command2
    ...
```

- target是即将要创建的目标（通常是一个可执行文件），target 后面紧跟一个冒号。
- prerequisite 是生成该目标所需要的源文件（依赖），一个目标所依赖的文件可以有多个，依赖文件与目标之间以及各依赖文件之间用空格或制表符 Tab 隔开，这些元素组成了一个**依赖关系**。
- command 是**规则，**也就是 make 需要执行的命令，它可以是任意的 shell 命令。
- 在makefile文件中，注释以 # 开头，一直延续到改行结束。

### 4.1.1 依赖关系

**依赖关系定义了最终应用程序里的每个文件与源文件之间的关系**。一个依赖关系列表由目标和该目标的零个或多个依赖组成，语法是：

```Plain
target: prerequisite1 prerequisite2 prerequisite3 ...
```

依赖关系表明了一件事：要生成 target，需要有这几个依赖文件的存在，而且，若其中一个依赖文件发生了改变，则需要重新生成 target。

目标所依赖的文件可以有一个或多个，也可以没有依赖文件 —— 该目标总被认为是过时的，在执行 make 命令时，若指定了该目标，则该目标所对应的规则将总被执行（如目标 clean）。

目标以来往往还有另一种用法，用于执行其他目标。如：

```SQL
.PHONY: all clean target

all: target clean

target: helloworld.o
        gcc helloworld.o -o helloworld

helloworld.o:
        gcc -c helloworld.c

clean:
        rm helloworld.o
```

执行`all`目标的时候，依赖另外两个目标`target`和`clean`。在执行`all`目标前，会优先执行目标`target`和`clean`。

怎么判断`all`依赖的是**目标**还是**文件**？

```SQL
.PHONY: all

all: test
        @echo in all

test:
        @echo in test
```

执行这个Makefile时，当前目录下有无test文件会有两个不同的执行结果

![image-20230402223600526](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402223600526.png)

没有test文件时，执行make命令，会先后执行all和test两个目标；

有test文件时，执行make命令，只执行all一个目标。

总结来说，判断依赖是**目标**还是**文件**，有以下两个规则：

1. 优先检查当前目录下是否有同名文件，有则文件，无则目标
2. .PHONY 标识的都是(伪)目标，不再检查文件是否存在

makefile 文件中可以有很多个目标，每个目标都有自己对应的规则。make 命令默认创建的是 makefile 文件中的第一个目标。也可以自己指定一个目标让 make 命令去创建，只需要将该目标的名字作为参数放到 make 命令之后即可（如常用的 make clean）。

更好的做法是，**将** **makefile** **文件中的第一个目标定义为 all，然后再 all 后面列出其他从属目标，这将告诉 make 命令，在未指定特定目标时，默认情况下将创建哪个目标**。

### 4.1.2 规则

规则的内容可以是任意的 shell 命令。关于规则，有以下要点：

1. **规则所在行必须以制表符 tab 开头，不能用空格**；
2. 规则所在行最好不要以空格结尾，可能会导致 make 命令执行失败；
3. 如果一行不足以写下所有内容，需要在每行代码的结尾加上一个反斜杠符 “\”，以让所有的命令在逻辑上处于同一行。

**两个特殊字符 - 和 @:**

- 在规则中，若命令之前加上了符号 “-”，则表明 make 命令将忽略该命令产生的所有错误；
- 若在命令之前加上了符号“@”，则表明 make 在执行该命令前，不会将该命令显示在标准输出上。

**两个特殊的目标：clean 和 install** 

　　目标 clean 和 install 是两个特殊的目标，它们并不用于创建文件，而是有其他用途。

- 目标 clean 在前面已经提到过，它使用 rm 命令来删除目标文件。rm 命令通常以减号 - 开头，表示让 make 命令忽略该命令的执行结果，这意味着，即使由于文件不存在而导致 rm 命令返回错误，命令 make clean 也能成功执行。
- 目标 install 用于按照命令的执行顺序将应用程序安装到指定的目录。

### 4.1.3 makefile文件中的宏

在 makefile 文件中定义一个宏很简单，如下：

```Plain
MACRONAME=value
```

这里定义了一个宏 MACRONAME，引用宏的方法是使用 $(MACRONAME) 或 ${MACRONAME} 。使用宏定义，可以让 makefile 文件的可移植性更强。除了自己定义一些宏以外，make 命令还内置了一些特殊的宏定义，使得 makefile 文件变得更加简洁：

| $?   | 表示所有比目标文件更新的依赖项列表，用空格分隔 |
| ---- | ---------------------------------------------- |
| $@   | 表示目标文件的名称                             |
| $<   | 表示依赖项列表中的第一个文件名称               |
| $*   | 表示目标文件的名称，但不包含扩展名             |

除了在 makefile 文件里面定义宏以外，还可以调用 make 命令时，在命令行上给出宏定义。命令行上的宏定义将 覆盖在 makefile 文件中的宏定义。需要注意的是，在 make 命令后接宏定义时，宏定义必须以单个参数的形式传递，因此，需要避免在宏定义中使用空格或加引号。

## 4.2 具体的例子

csapp中datalab的makefile如下：

```JavaScript
#
# Makefile that builds btest and other helper programs for the CS:APP data lab
# 
CC = gcc
CFLAGS = -O -Wall -m32
LIBS = -lm

all: btest fshow ishow

btest: btest.c bits.c decl.c tests.c btest.h bits.h
    $(CC) $(CFLAGS) $(LIBS) -o btest bits.c btest.c decl.c tests.c

fshow: fshow.c
    $(CC) $(CFLAGS) -o fshow fshow.c

ishow: ishow.c
    $(CC) $(CFLAGS) -o ishow ishow.c

# Forces a recompile. Used by the driver program. 
btestexplicit:
    $(CC) $(CFLAGS) $(LIBS) -o btest bits.c btest.c decl.c tests.c 

clean:
    rm -f *.o btest fshow ishow *~
```

下面是详细的Makefile分析：

```Makefile
CC = gcc
CFLAGS = -O -Wall -m32
LIBS = -lm
```

这三行定义了Makefile中将要使用的变量，`CC`是指定使用的编译器，`CFLAGS`是指定编译器使用的参数，`LIBS`是指定需要链接的库。

```CSS
all: btest fshow ishow
```

这一行定义了一个名为`all`的默认目标，其依赖于`btest`、`fshow`和`ishow`三个目标，即执行`make`命令时，会默认编译这三个程序。

```Makefile
btest: btest.c bits.c decl.c tests.c btest.h bits.h
    $(CC) $(CFLAGS) $(LIBS) -o btest bits.c btest.c decl.c tests.c
```

这一段定义了`btest`目标。它依赖于`btest.c`、`bits.c`、`decl.c`、`tests.c`和`btest.h`、`bits.h`两个头文件。在编译这个目标时，使用了`$(CC)`变量指定的编译器，`$(CFLAGS)`变量指定的编译器参数和`$(LIBS)`变量指定的链接库。这一行中最后的`-o btest`指定输出的可执行文件名为`btest`。

```Makefile
fshow: fshow.c
    $(CC) $(CFLAGS) -o fshow fshow.c
```

这一段定义了`fshow`目标，依赖于`fshow.c`文件。编译时同样使用了`$(CC)`变量指定的编译器和`$(CFLAGS)`变量指定的编译器参数，最后的`-o fshow`指定输出的可执行文件名为`fshow`。

```Makefile
ishow: ishow.c
    $(CC) $(CFLAGS) -o ishow ishow.c
```

这一段定义了`ishow`目标，依赖于`ishow.c`文件。编译时同样使用了`$(CC)`变量指定的编译器和`$(CFLAGS)`变量指定的编译器参数，最后的`-o ishow`指定输出的可执行文件名为`ishow`。

```Makefile
btestexplicit:
    $(CC) $(CFLAGS) $(LIBS) -o btest bits.c btest.c decl.c tests.c
```

这一段定义了`btestexplicit`目标，和`btest`目标类似，但它并不是默认目标，它的主要作用是强制重新编译`btest`程序。

```Bash
clean:
    rm -f *.o btest fshow ishow *~
```

这一段定义了一个名为`clean`的目标，用于删除所有生成的目标文件和可执行文件。`rm -f`指令用于强制删除，`*.o`表示所有的目标文件，`btest`、`fshow`和`ishow`是三个可执行文件的名字，`*~`表示所有以`~`结尾的文件，这些通常是一些编辑器留下的备份文件。

总之，这个Makefile的作用是编译三个程序：`btest`、`fshow`和`ishow`。其中，`btest`程序依赖于`bits.c`、`decl.c`、`tests.c`和`btest.h`、`bits.h`两个头文件，用于测试一个位级别操作的实现是否正确。`fshow`和`ishow`程序分别用于显示浮点数和整数的二进制表示。同时，`btestexplicit`目标可以用于强制重新编译`btest`程序。`clean`目标用于删除所有生成的目标文件和可执行文件，以便重新编译或清理空间。

## 4.3 库的链接

在Linux下，Makefile通过在编译链接命令中指定库的位置来找到库。通常，库文件分为静态库和共享库两种类型，分别以`.a`和`.so`为文件扩展名。

对于标准库和系统库，编译器和链接器会自动搜索其默认的路径，无需指定路径。对于外部库，则需要在Makefile中指定库的搜索路径。在Makefile中指定库的搜索路径的方式有多种。

1. 在编译链接命令中指定库的搜索路径

可以使用`-L`选项来指定库的搜索路径，例如：

```Bash
gcc -o myapp myapp.o -L/usr/local/lib -lmylib
```

这个命令会在`/usr/local/lib`目录中搜索名为`libmylib.so`的共享库文件，并将其链接到`myapp`可执行文件中。

1. 设置环境变量LD_LIBRARY_PATH

LD_LIBRARY_PATH是Linux系统中用来指定共享库搜索路径的环境变量，可以在Makefile中通过定义这个环境变量来指定库的搜索路径。例如：

```JavaScript
export LD_LIBRARY_PATH=/usr/local/lib
```

这个命令会将`/usr/local/lib`目录添加到共享库搜索路径中。

1. 在Makefile中定义变量

可以在Makefile中定义一个变量来指定库的搜索路径，然后在编译链接命令中使用这个变量。例如：

```Makefile
LIB_DIR = /usr/local/lib

myapp: myapp.o
    gcc -o myapp myapp.o -L$(LIB_DIR) -lmylib
```

这个Makefile会将`/usr/local/lib`目录添加到库的搜索路径中，并在编译链接命令中使用`$(LIB_DIR)`变量来指定库的搜索路径。

需要注意的是，如果库文件不在默认的搜索路径中，就必须指定库的搜索路径。否则，链接器会报错，提示找不到库文件。

另外，在Makefile中，可以使用`pkg-config`命令来自动获取库的编译链接参数，避免手动指定参数。这需要库的开发者提供一个`.pc`文件，指定库的编译链接参数。然后，在Makefile中可以使用如下命令获取编译链接参数：

```Makefile
CFLAGS = $(shell pkg-config --cflags mylib)
LDFLAGS = $(shell pkg-config --libs mylib)
```

这个命令会自动获取`mylib`库的编译链接参数，并将它们分别保存在`CFLAGS`和`LDFLAGS`变量中。然后，在编译链接命令中使用这些变量即可。

**那么当我们指定了一个搜索路径，想要找其中的某个库，链接器不会****递归****的搜索库所依赖的其他库，也就是我们需要手动指定库的依赖关系；当依赖关系过于复杂的时候常常会出现依赖问题导致编译失败。**

那么该怎么解决呢？

这时候就出现了自动化构建工具。我们将介绍两种自动化构建工具：Cmake与Xmake。

# 五、Cmake
