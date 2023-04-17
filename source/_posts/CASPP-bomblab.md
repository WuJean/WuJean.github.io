---
title: CASPP-bomblab
date: 2023-04-17 09:44:26
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

开始写bomblab了，听说这是所有lab里最有趣的lab。反思一下写这篇博文既想当作实验报告，又想当成一个教程，边做边写总会出现一些纰漏，故前半部分记录了做lab的全过程，后半部分再做全局的总结和思考。

# 前置知识



# 实验一览



# Phase 1

反汇编出phase_1的汇编代码：

```
(gdb) disas phase_1
```

```
Dump of assembler code for function phase_1:
   0x08048b60 <+0>:     sub    $0x1c,%esp
   0x08048b63 <+3>:     movl   $0x804a284,0x4(%esp)
   0x08048b6b <+11>:    mov    0x20(%esp),%eax
   0x08048b6f <+15>:    mov    %eax,(%esp)
   0x08048b72 <+18>:    call   0x80490a4 <strings_not_equal>
   0x08048b77 <+23>:    test   %eax,%eax
   0x08048b79 <+25>:    je     0x8048b80 <phase_1+32>
   0x08048b7b <+27>:    call   0x80491b6 <explode_bomb>
   0x08048b80 <+32>:    add    $0x1c,%esp
   0x08048b83 <+35>:    ret
End of assembler dump.
```

- `sub $0x1c, %esp`：在栈上为该函数分配 28 字节的空间。
- `movl $0x804a284,0x4(%esp)`：将 0x804a284 这个地址值（在 .rodata 节）写入从 %esp 偏移量为 4 的位置，即第一个函数参数的位置。
- `mov 0x20(%esp),%eax`：将从 %esp 偏移量为 32 的位置，即第二个函数参数的位置处的值加载到寄存器 %eax 中。
- `mov %eax,(%esp)`：将 %eax 中的值存储到栈的顶部。
- `call 80490a4 <strings_not_equal>`：调用名为 `strings_not_equal` 的函数，该函数的地址为 0x80490a4。
- `test %eax,%eax`：将 %eax 寄存器与自身进行 AND 操作并设置标志寄存器。这是为了检查函数返回值是否为 0（两个字符串相等）。
- `je 8048b80 <phase_1+0x20>`：如果上一条指令设置了零标志，则跳转到地址 0x8048b80，即函数的结尾。
- `call 80491b6 <explode_bomb>`：如果字符串不相等，则调用名为 `explode_bomb` 的函数，该函数会使程序停止运行并输出错误信息。
- `add $0x1c,%esp`：将栈顶向上移动 28 个字节，即释放为该函数分配的栈空间。
- `ret`：从该函数返回。

参数是从右往左入栈的，第二个参数的长度为28，第一个参数为4。故第一个参数的地址为esp+4，第二个参数的地址为esp-32。

可以合理推测我们输入的字符串通过函数`phase_1`的第二个参数传入，将其放入eax寄存器。

将其与`strings_not_equal`函数中设定的答案对比，若相等则拆除炸弹成功。故我们需要使用gdb调试`strings_not_equal`函数，找到设定的答案。

我们在`strings_not_equal`函数处打上断点，运行程序并输入input1（test）：

```
(gdb) b strings_not_equal
Breakpoint 1 at 0x80490a4
(gdb) r
Starting program: /home/wujean/bomb216/bomb
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
test

Breakpoint 1, 0x080490a4 in strings_not_equal ()
```

我们为了查看ebx和esi的值，需要执行程序：

```
(gdb) si 6
0x080490bb in strings_not_equal ()
```

![image-20230417111505401](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230417111505401.png)

2-5行是被调用者保存寄存器 保存的过程。

```
0x080490b3 <+15>:    mov    0x14(%esp),%ebx
0x080490b7 <+19>:    mov    0x18(%esp),%esi
```

这两行代码读取函数的两个参数，我们使用gdb获取存储在这两个寄存器中参数的值。

```
(gdb) p/s (char*)$ebx
$1 = 0x804c3e0 <input_strings> "test"
(gdb) p/s (char*)$esi
$2 = 0x804a284 "Houses will begat jobs, jobs will begat houses."
```

这样一看就很清楚了，我们输入的test为函数的第二个参数，而第一个参数为一个定值：Houses will begat jobs, jobs will begat houses.

这便是我们需要比对的字符串，将其作为第一个输入，第一个炸弹成功拆除。

```
wujean@DESKTOP-8K47HOB:~/bomb216$ ./bomb
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Houses will begat jobs, jobs will begat houses.
Phase 1 defused. How about the next one?
```

# Phase 2

```
(gdb) disas phase_2
Dump of assembler code for function phase_2:
   0x08048b84 <+0>:     push   %ebx
   0x08048b85 <+1>:     sub    $0x38,%esp
   0x08048b88 <+4>:     lea    0x18(%esp),%eax
   0x08048b8c <+8>:     mov    %eax,0x4(%esp)
   0x08048b90 <+12>:    mov    0x40(%esp),%eax
   0x08048b94 <+16>:    mov    %eax,(%esp)
   0x08048b97 <+19>:    call   0x80492eb <read_six_numbers>
   0x08048b9c <+24>:    cmpl   $0x0,0x18(%esp)
   0x08048ba1 <+29>:    jns    0x8048ba8 <phase_2+36>
   0x08048ba3 <+31>:    call   0x80491b6 <explode_bomb>
   0x08048ba8 <+36>:    mov    $0x1,%ebx
   0x08048bad <+41>:    mov    %ebx,%eax
   0x08048baf <+43>:    add    0x14(%esp,%ebx,4),%eax
   0x08048bb3 <+47>:    cmp    %eax,0x18(%esp,%ebx,4)
   0x08048bb7 <+51>:    je     0x8048bbe <phase_2+58>
   0x08048bb9 <+53>:    call   0x80491b6 <explode_bomb>
   0x08048bbe <+58>:    add    $0x1,%ebx
   0x08048bc1 <+61>:    cmp    $0x6,%ebx
   0x08048bc4 <+64>:    jne    0x8048bad <phase_2+41>
   0x08048bc6 <+66>:    add    $0x38,%esp
   0x08048bc9 <+69>:    pop    %ebx
   0x08048bca <+70>:    ret
End of assembler dump.
```

拆了第一个炸弹后发现需要格外注意其中调用的函数，第二个炸弹中有两个函数：`read_six_numbers`、`explode_bomb`

## read_six_numbers

```
(gdb) disas read_six_numbers
Dump of assembler code for function read_six_numbers:
   0x080492eb <+0>:     sub    $0x2c,%esp
   0x080492ee <+3>:     mov    0x34(%esp),%eax
   0x080492f2 <+7>:     lea    0x14(%eax),%edx
   0x080492f5 <+10>:    mov    %edx,0x1c(%esp)
   0x080492f9 <+14>:    lea    0x10(%eax),%edx
   0x080492fc <+17>:    mov    %edx,0x18(%esp)
   0x08049300 <+21>:    lea    0xc(%eax),%edx
   0x08049303 <+24>:    mov    %edx,0x14(%esp)
   0x08049307 <+28>:    lea    0x8(%eax),%edx
   0x0804930a <+31>:    mov    %edx,0x10(%esp)
   0x0804930e <+35>:    lea    0x4(%eax),%edx
   0x08049311 <+38>:    mov    %edx,0xc(%esp)
   0x08049315 <+42>:    mov    %eax,0x8(%esp)
   0x08049319 <+46>:    movl   $0x804a4b7,0x4(%esp)
   0x08049321 <+54>:    mov    0x30(%esp),%eax
   0x08049325 <+58>:    mov    %eax,(%esp)
   0x08049328 <+61>:    call   0x8048870 <__isoc99_sscanf@plt>
   0x0804932d <+66>:    cmp    $0x5,%eax
   0x08049330 <+69>:    jg     0x8049337 <read_six_numbers+76>
   0x08049332 <+71>:    call   0x80491b6 <explode_bomb>
   0x08049337 <+76>:    add    $0x2c,%esp
   0x0804933a <+79>:    ret
End of assembler dump.
```

可以由函数名以及前几行代码的操作看出，本函数读取了六个参数，若读取的参数不足6个则直接调用`explode_bomb`，炸弹爆炸。



回到`phase_2`函数中，与第一问，答案存储在内存中的某个地址中不同，本题主要为分析汇编代码得出六个数的规律。

```
(gdb) disas phase_2
Dump of assembler code for function phase_2:
   0x08048b84 <+0>:     push   %ebx
   0x08048b85 <+1>:     sub    $0x38,%esp
   0x08048b88 <+4>:     lea    0x18(%esp),%eax
   0x08048b8c <+8>:     mov    %eax,0x4(%esp)		//预留六个int类型的空间，将其地址作为参数传入read_six_numbers
   0x08048b90 <+12>:    mov    0x40(%esp),%eax
   0x08048b94 <+16>:    mov    %eax,(%esp)
   0x08048b97 <+19>:    call   0x80492eb <read_six_numbers>		//输入六个数字（检验是否为6个）
   0x08048b9c <+24>:    cmpl   $0x0,0x18(%esp)		//a[0]>0
   0x08048ba1 <+29>:    jns    0x8048ba8 <phase_2+36>
   0x08048ba3 <+31>:    call   0x80491b6 <explode_bomb>		//否则爆炸
 ->0x08048ba8 <+36>:    mov    $0x1,%ebx						--------------------------------
   0x08048bad <+41>:    mov    %ebx,%eax						int i=1,j=1;i<6;i++,j++
   0x08048baf <+43>:    add    0x14(%esp,%ebx,4),%eax			if(a[i-1]+j!=a[i])
   0x08048bb3 <+47>:    cmp    %eax,0x18(%esp,%ebx,4)
   0x08048bb7 <+51>:    je     0x8048bbe <phase_2+58>
   0x08048bb9 <+53>:    call   0x80491b6 <explode_bomb>			爆炸
   0x08048bbe <+58>:    add    $0x1,%ebx
   0x08048bc1 <+61>:    cmp    $0x6,%ebx
 <-0x08048bc4 <+64>:    jne    0x8048bad <phase_2+41>			---------------------------------
   0x08048bc6 <+66>:    add    $0x38,%esp
   0x08048bc9 <+69>:    pop    %ebx
   0x08048bca <+70>:    ret
End of assembler dump.
```

故我们只需要保证输入的第一个数大于0，且后面的数按照 +1 +2 +3 +4 +5的规律递增，就可成功拆除炸弹。

```
wujean@DESKTOP-8K47HOB:~/bomb216$ ./bomb
Welcome to my fiendish little bomb. You have 6 phases with
which to blow yourself up. Have a nice day!
Houses will begat jobs, jobs will begat houses.
Phase 1 defused. How about the next one?
2 3 5 8 12 17
That's number 2.  Keep going!
```

