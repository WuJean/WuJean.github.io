---
title: CASPP-datalab
date: 2023-03-11 15:26:47
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

# 配置

实验环境为：

- WSL2 ubuntu22.04 64位
- i5-12400

# Begin

从该仓库拉取datalab

```
git clone https://github.com/WuJean/HNU-CSAPP.git
```

按照要求在lab目录下make btest，编译测试程序

如遇到编译错误可能是缺少在64位下运行32位的相关包，apt获取后再次make即可

具体表现为头找不到头文件的错误

做完每题后用 ./dlc 查看使用符号是否符合规范，再次make后执行 ./btest查看分数

## bitAnd

```
/* 
 * bitAnd - x&y using only ~ and | 
 *   Example: bitAnd(6, 5) = 4
 *   Legal ops: ~ |
 *   Max ops: 8
 *   Rating: 1
 */
int bitAnd(int x, int y) {
  return ~((~x)|(~y));
}
```

## getByte

```
/* 
 * getByte - Extract byte n from word x
 *   Bytes numbered from 0 (LSB) to 3 (MSB)
 *   Examples: getByte(0x12345678,1) = 0x56
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 6
 *   Rating: 2
 */
int getByte(int x, int n) {
  int mask = 0xff;
  return (x>>(n<<3)&mask);
}
```

位！=字节，4位一个字节

## logicalShift

```
/* 
 * logicalShift - shift x to the right by n, using a logical shift
 * Can assume that 0 <= n <= 31
 * Examples: logicalShift(0x87654321,4) = 0x08765432
 * Legal ops: ! ~ & ^ | + << >>
 * Max ops: 20
 * Rating: 3 
*/

int logicalShift(int x, int n) {
  int mask=((0x1<<(32+~n))+~0)|(0x1<<(32+~n));
  return (x>>n)&mask;
}
```

我们注意到移动的n是字节，示例中移动4个字节也就是移动了1位；c语言中使用的右移是算数右移，会保存符号位的信息，示例中给出的最高位0x8的二进制形式是1000，int类型的符号位为1，代表负数；故在算数右移的过程中会在左边填充1，很显然这不符合我们的要求；故需要设计一个掩码来实现这一操作：

- 待办

## bitCount

```
/*
 * bitCount - returns count of number of 1's in word
 * Examples: bitCount(5) = 2, bitCount(7) = 3
 * Legal ops: ! ~ & ^ | + << >>
 * Max ops: 40
 * Rating: 4
*/
   int bitCount(int x) {
    x = (x&0x55555555) + ((x>>1)&0x55555555);  
    x = (x&0x33333333) + ((x>>2)&0x33333333);  
    x = (x&0x0f0f0f0f) + ((x>>4)&0x0f0f0f0f);  
    x = (x&0x00ff00ff) + ((x>>8)&0x00ff00ff);  
    x = (x&0x0000ffff) + ((x>>16)&0x0000ffff);
    return x; 
   }
```

### 平行算法

1. 0x5555……这个换成二进制之后就是01 01 01 01 01 01 01 01…… 

2. 0x3333……这个换成二进制之后就是0011 0011 0011 0011……  

3. 0x0f0f………这个换成二进制之后就是00001111 00001111……

接下来的以此类推

该算法使用了分治的思想，我们将32位数分成32个段，段的取值能体现位是否为1，将32个段累加成的值就是该32位中1的个数；

`(n&0x55555555)+((n>>1)&0x55555555)` 将32位数中的32个段从左往右把相邻的两个段的值相加后放在2bits中，就变成了16个段，每段2位。同理`(n&0x33333333)+((n>>2)&0x33333333)`将16个段中相邻的两个段两两相加，存放在4bits中，就变成了8个段，每段4位。以此类推，最终求得数中1的个数就存放在一个段中，这个段32bits，就是最后的n。

例如1111进行第一次操作

(1111&0101)+((1111>>1)&0101) -> 0101+0101 = 1010	(代表高2位有10个1 低2位也有10个1)				//分别对奇偶位计算1的个数

下面的操作以此类推

#### what‘s more

如果考虑到int类型有符号，最高位为1的情况下，使用01（0在前的方法）可以有效避免算数右移带来的符号位增加，导致1的个数变多的情况。考虑更多的情况，若我们使用10（1在前的情况），会导致最低位的1在还未被统计的情况下被右移消除。

此外若改变一下思路，我们使用10与左移的操作，可以直接不考虑符号位的问题。

## bang

```
/* 
 * bang - Compute !x without using !
 *   Examples: bang(3) = 0, bang(0) = 1
 *   Legal ops: ~ & ^ | + << >>
 *   Max ops: 12
 *   Rating: 4 
 */
int bang(int x) {
  x=(x>>16)|x;
  x=(x>>8)|x;
  x=(x>>4)|x;
  x=(x>>2)|x;
  x=(x>>1)|x;
  return ~x&0x1;
}
```

取非操作，若该数不为0，则该数至少有一位为1；我们用压缩的思想，将每一位的信息依次压缩，直到到最低有效位，再与0x1相与得到最终的返回值。

## tmin

```
/* 
 * tmin - return minimum two's complement integer 
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 4
 *   Rating: 1
 */
int tmin(void) {
  return 0x1<<31;
}
```

获取最小的二进制补码，直接左移31位。

## fitsBits

```
/* 
 * fitsBits - return 1 if x can be represented as an 
 *  n-bit, two's complement integer.
 *   1 <= n <= 32
 *   Examples: fitsBits(5,3) = 0, fitsBits(-4,3) = 1
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 2
 */
int fitsBits(int x, int n) {
    int tmp = x>>(~(~n+1));
    tmp = (tmp+1)>>1;
    return !(tmp);
}
```

因为n位二进制的最高位为符号位，一个数能否被n位二进制表示与符号位并无关系，我们只需要关注n-1位的内容是否与x的数据位匹配；其实从某种意义上来说，n-1位的内容也是无关紧要的，我们更需要关注的是n-1位以后的位是否有我们无法表示的信息；

当x为正数时，我们假设x能被n位二进制数表示，那么n-1位后将不包含任何信息，也就是全为0；

当x为负数时同理，只不过n-1位后全为1；

我们现在要寻找一种能将这一特性转化成为逻辑值的过程函数；首先我们将无关紧要的低n-1位右移出我们的工作区，注意~n(~n)可以表示n-1；随后我们将右移过后的数加上1，这是为了将负数与正数的判断条件达成统一，若负数满足表示的条件那么加一的操作会产生连锁反应将高位全置为0；至此第n位的part到此结束，我们将其右移，判断剩下的式子是非为全0就可完成判断。

在本题中我们知道了对于负数来说高位的1就相当于正数高位的0，对于另一种方法而言：

- 代办

本题在64位下运行过不了示例，但是相信你的想法，你做完了就是对的（其实是测试用例错了）

我们查看错误时可以看到当n为32时理论上对于任何的x都应该返回1，但是测试用例返回了0；我们可以在工作区文件夹中找到tests.c文件，查看其中生成fitbits测试结果的函数：

```
int test_fitsBits(int x, int n)
{
  if(n==32) return 1;
  int TMin_n = -(1 << (n-1));
  int TMax_n = (1 << (n-1)) - 1;
  return x >= TMin_n && x <= TMax_n;
}
```

乍一看是没错的，但是由于编译优化，这段代码中的某些指令被简化了导致结果出错，具体的错误过程理解一下就好，我们只需要在函数的开头加上:

```
if(n==32) return 1;
```

## divpwr2

```
/* 
 * divpwr2 - Compute x/(2^n), for 0 <= n <= 30
 *  Round toward zero
 *   Examples: divpwr2(15,1) = 7, divpwr2(-33,4) = -2
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 15
 *   Rating: 2
 */
int divpwr2(int x, int n) {
  int bias=(x>>31)&((0x1<<n)+~0);
  return (x+bias)>>n;
}
```

这题计算 x/(2^n) ，注意不能直接右移，直接右移是向下舍入的，题目要求是向零舍入，也就是正数向下舍入，负数向上舍入，这里参照 CS:APP 书上的做法，给负数加上一个偏正的因子 `(0x1<<n)+~0)` ，判断负数直接看符号位。

## negate

```
/* 
 * negate - return -x 
 *   Example: negate(1) = -1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 5
 *   Rating: 2
 */
int negate(int x) {
  return ~x+1;
}
```

这题求 -x ，直接利用补码的性质 `-x=~x+1` 就可以了。

## isPositive

```
/* 
 * isPositive - return 1 if x > 0, return 0 otherwise 
 *   Example: isPositive(-1) = 0.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 8
 *   Rating: 3
 */
int isPositive(int x) {
    return !(!x)&!((x>>31)&0x1);
}
```

直接判断符号位，注意排除0的情况。

## isLessOrEqual

```
/* 
 * isLessOrEqual - if x <= y  then return 1, else return 0 
 *   Example: isLessOrEqual(4,5) = 1.
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 24
 *   Rating: 3
 */
int isLessOrEqual(int x, int y) {
  int val=!!((x+~y)>>31);
  x=x>>31;
  y=y>>31;
  return (!!x|!y)&((!!x&!y)|(val));
}
```

当y大于等于x时候返回1，由于不能使用减号，我们直接使用~x，y+(~x)相当于y+1-x，故直接判断符号位就可以实现判断。

我们需要额外考虑的就是减法溢出的情况，正数一定大于负数，但是正数减负数会出现溢出的情况；所以对于一些显而易见的操作我们直接返回判断的值；对于正数减正数再用相减的值进行判断。

`(!!x|!y)&((!!x&!y)|(val))`  有空再分析

## ilog2

```
/*
 * ilog2 - return floor(log base 2 of x), where x > 0
 *   Example: ilog2(16) = 4
 *   Legal ops: ! ~ & ^ | + << >>
 *   Max ops: 90
 *   Rating: 4
 */
int ilog2(int x) {
  int res = 0;
  res = (!!(x>>(16)))<<4;
  res = res+((!!(x>>(8+res)))<<3);
  res = res+((!!(x>>(4+res)))<<2);
  res = res+((!!(x>>(2+res)))<<1);
  res = res+((!!(x>>(1+res)))<<0);
  return res;
}
```

一个数以二进制表示，最高 1 位所对应的 2 的次方，就是logx向下取整的值。所以我们只需要找到最高位的1，它的位置就是对数值。由于我们受到符号位的限制，所以我们需要找出其他的方法找到最高位的1并将其转化成计数。

我们首先将x右移16位，找高16位是否存在1，若高16位存在1则下一步在高八位继续找1；若不存在1则在低八位找1；

对res的加减操作不仅代表了找1的范围，还代表了1的位置信息；最后返回res即为正确答案。

## float_neg

```
/* 
 * float_neg - Return bit-level equivalent of expression -f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representations of
 *   single-precision floating point values.
 *   When argument is NaN, return argument.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 10
 *   Rating: 2
 */
unsigned float_neg(unsigned uf)
{
  int c = 0x00ffffff;
  if ((~(uf << 1)) < c){
    return uf;
  }
  else{
    return uf ^ (0x80000000);
  }
}
```

符号位直接取反，但要注意uf为NaN的情况。

## float_i2f

```
/* 
 * float_i2f - Return bit-level equivalent of expression (float) x
 *   Result is returned as unsigned int, but
 *   it is to be interpreted as the bit-level representation of a
 *   single-precision floating point values.
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned float_i2f(int x) {  
  int sign = (x>>31)&0x1;
  int exp=0,frac=0,delta=0;
  int frac_mask = 0x7fffff; 
  if (!x){
    return x;
  }
  else if(x==0x80000000){
    exp = 158;
  }
  else{
    if(sign) x = -x;
    int index = 30;
    while(!(x>>index)){
      index--;
    }
    exp = index+127;
    x = x<<(31-index);
    frac = (x>>8)&frac_mask;
    x = x&0xff;
    delta = x>0x80 || ((x==0x80)&&(frac&0x1));
    frac+=delta;
    if(frac>>23){
      exp+=1;
      frac = frac&frac_mask;
    }
  }
  return (sign<<31)|(exp<<23)|frac;
}
```

- 判断符号位
- 判断x为0 以及x为0x80000000的特殊情况
- 去除前导0
- 获取23位frac
- 对舍去的8位判断舍入
- 判断舍入后的结果frac是否超过23位
- 返回拼接后的结果

## float_twice

```
/* 
 * float_twice - Return bit-level equivalent of expression 2*f for
 *   floating point argument f.
 *   Both the argument and result are passed as unsigned int's, but
 *   they are to be interpreted as the bit-level representation of
 *   single-precision floating point values.
 *   When argument is NaN, return argument
 *   Legal ops: Any integer/unsigned operations incl. ||, &&. also if, while
 *   Max ops: 30
 *   Rating: 4
 */
unsigned float_twice(unsigned uf) {
  int sign = uf>>31&&0x01;
  int exp = uf>>23 & 0xff;
  int frac = uf & 0x7fffff;
  if(exp!=0xff){
    if(!exp)  frac=frac<<1;
    else{
      exp += 1;
      if(exp==0xff)
        frac=0;
    }
  }
  return sign<<31|exp<<23|frac;
}
```

草草结束

# 结语

![image-20230324213239814](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230324213239814.png)

写到很多很妙的方法，但是由于没有集中精力去写，加上对int类的补码表示不怎么理解，导致浪费了很多不必要的时间，不过这次的lab之旅最大的收获也是浪费这些时间的过程；巧妙的位运算，戴着枷锁起舞，深入理解神奇的补码....

半写半抄完成，与同学深入交流不同的写法，手动copy代码，理解每一处细节。我承认这次没做得很好，这篇博客也仅仅是起一个记录得作用，如果后来人先看到了这里，希望你可以珍惜第一次写datalab的懵懂。
