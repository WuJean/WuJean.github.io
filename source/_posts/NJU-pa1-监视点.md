---
title: NJU-pa1-监视点
date: 2023-03-14 21:15:52
updated:
tags: NJU-pa
categories:
keywords:
description: 
top_img:
comments:
cover:
---

# 拓展表达式求值的功能

我们在前一章中已经实现了基础的表达式四则运算的功能，但是表达式中还有许多的指令还未实现，本章主要实现了这些指令并详解整个表达式求值的构造。

```
<expr> ::= <decimal-number>
  | <hexadecimal-number>    # 以"0x"开头
  | <reg_name>              # 以"$"开头
  | "(" <expr> ")"
  | <expr> "+" <expr>
  | <expr> "-" <expr>
  | <expr> "*" <expr>
  | <expr> "/" <expr>
  | <expr> "==" <expr>
  | <expr> "!=" <expr>
  | <expr> "&&" <expr>
  | "*" <expr>              # 指针解引用
```

## 拓展表达式

我们查看底层的数据结构：

```
static struct rule {
 const char *regex;		//正则表达式的匹配字符
 int token_type;		//该字符的类别
 int pre;		//操作符的优先级
}
```

我们修改了原本的数据结构，将优先级用一个int类型的数字表示。

数字越小优先级越高，我们在解析表达式时可以用比较操作符之间的优先级，从而按规则完成表达式求值。

我们需要在本模块完成的几个操作符：

```
  | <hexadecimal-number>    # 以"0x"开头
  | <reg_name>              # 以"$"开头
  | <expr> "!=" <expr>
  | <expr> "&&" <expr>
```

这几个操作数都是直接可以解析出来参与运算的，属于优先级最高的，我们在rules数组中将这几个操作符定义如下：

```
 {"!=",TK_NEQ,0},
 {"&&",'&',0},
 {"^\$",'$',0},   //reg
 {"^\0x",'0x',0},   //
```

至于其中如何处理乘法符号 *和指针解引用 *如何分辨，pa文档中直接给出了解决方法：

我们知道表达式求值过程中我们是通过token的类型来识别并传递到运算函数中，我们只需要在切割token的时候通过规则加以识别就可以找出两哥类型符号的区别。

```
for (int i = 0; i < nr_token; i++)
 {
  if(tokens[i].type=='*'&&(i!=0||tokens[i-1].pre>2)){
   tokens[i].type = DEREF;
  }
  switch (tokens[i].type)
  {
  	//处理
  }
 }
```

解引用后的指针是一个值，值之前的操作符一定是一个符号，我们就可以通过这样的规则来判断并重置该token的类型，从而实现判断的操作。

## 新增表达式求值

