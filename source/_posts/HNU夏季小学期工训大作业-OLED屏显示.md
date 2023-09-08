---
title: HNU夏季小学期工训大作业-OLED屏显示
date: 2023-09-07 14:56:11
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

大二夏小学期第二阶段要求基于STC板完成一个课程设计，本质上就是抄来抄去换名字水过去，助教的验收更是水之又水，通过验收时到达的顺序来划分分数段。反正水过去就完了，抄来抄去也是没意思，想起以前买过一个oled小屏幕，于是决定基于这个屏幕来完成我的大作业。写这篇文章记录一下完成这个项目的过程，希望可以让有缘人在使用屏幕的时候少走一些弯路。

项目源代码地址：https://github.com/WuJean/HNU/tree/main/%E5%B7%A5%E8%AE%AD

## 项目准备

### 硬件准备

1. 某宝0.96寸oled屏 4针接口

   ![image-20230908161910294](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230908161910294.png)

2. 公对母杜邦线

   ![image-20230908161854264](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230908161854264.png)

由于OLED自带的接针和STC板上的接口并不是直接对应，故需要杜邦线来对应接针

具体的对应关系为：

| OLED | STC  |
| ---- | :--: |
| GND  | GND  |
| VCC  | VCC  |
| SCL  | P1.0 |
| SDA  | P1.1 |

### 资料准备

参考0.96寸OLED程序源码-51例程

源码中为我们提供了几个调用OLED的基本库：

oled.c	oled.h	oledfont.h	bmp.h

其中oled.c中实现了通过提前定义在oledfont.h以及bmp.h中的字模或图片，完成显示的各种功能；以及对oled屏幕的各种操作

```
//OLED控制用函数
void delay_ms(unsigned int ms);
void OLED_ColorTurn(u8 i);
void OLED_DisplayTurn(u8 i);
void OLED_WR_Byte(u8 dat,u8 cmd);
void OLED_Set_Pos(u8 x, u8 y);
void OLED_Display_On(void);
void OLED_Display_Off(void);
void OLED_Clear(void);
void OLED_ShowChar(u8 x,u8 y,u8 chr,u8 sizey);
u32 oled_pow(u8 m,u8 n);
void OLED_ShowNum(u8 x,u8 y,u32 num,u8 len,u8 sizey);
void OLED_ShowString(u8 x,u8 y,u8 *chr,u8 sizey);
void OLED_ShowChinese(u8 x,u8 y,u8 no,u8 sizey);
void OLED_DrawBMP(u8 x,u8 y,u8 sizex, u8 sizey,u8 BMP[]);
```

我们只需要include这些库，调用其中的函数便可在oled屏上显示想要的内容

### OLED显示原理与取模方式

参考 https://blog.csdn.net/u010858987/article/details/103362144

水平方向分布了128个像素点，垂直方向分布了64个像素点（如图一所示）。而驱动芯片在点亮像素点的时候，是以8个像素点为单位的。官方的例程推荐的是垂直扫描的方式，也就是先画垂直方向的8个像素点（如下图二所示），所以我们在画点的时候Y的取值为0-7，X的取值为0-127.

```
//存放格式如下.
//[0]0 1 2 3 ... 127    
//[1]0 1 2 3 ... 127    
//[2]0 1 2 3 ... 127    
//[3]0 1 2 3 ... 127    
//[4]0 1 2 3 ... 127    
//[5]0 1 2 3 ... 127    
//[6]0 1 2 3 ... 127    
//[7]0 1 2 3 ... 127 
————————————————
```

我们使用软件进行图片和汉字的取模，下面分别展示两款软件的取模预设：

#### 图片取模

我们使用image2Lcd这款软件来完成对图片的取模 

下载地址：http://www.ddooo.com/softdown/134212.htm

![image-20230908163647220](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230908163647220.png)

我们采用数据水平 字节垂直的取模方式，并设置图片的最大宽高为我们oled屏的最大宽高，导入图片后点击保存，便可生成一个c语言数组，格式如下：

```
const unsigned char gImage_1[512] = { /* 0X02,0X01,0X40,0X00,0X40,0X00, */
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X1C,0X38,0X32,0X36,0X30,0X36,0X16,0X1A,0X1E,0X18,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X07,0X0F,0X0F,0X0F,0X1F,0X1F,0XBF,0XFF,0XFF,0X7F,0X7F,0X3F,0X3F,0X7F,
0X7F,0X7F,0X7F,0X3F,0X3F,0X3F,0X3F,0X0F,0X03,0X03,0X01,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0XFC,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,
0XFF,0XFF,0XFF,0XFF,0XFF,0XDF,0X87,0XC1,0XC0,0XC0,0X80,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0XC0,0XF0,0XFC,0XFE,0XF8,0XFC,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,
0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XFF,0XE0,0X40,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X80,0XE0,0XFE,0XFF,0XFF,0XFF,
0XFD,0XFF,0XFF,0XFF,0XFF,0XFE,0X80,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X0C,0X0C,0X1C,0X38,0XF8,0XF0,0XF0,
0XE0,0XA0,0XF0,0XF8,0XFC,0X38,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,0X00,
};
```

将其放入bmp.h中待用，注意STC板的写入内存有限，过多的图片可能会导致写入失败

#### 汉字取模

我们采用字模软件PCtoLCD，进行以下预设：

![image-20230908164851749](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230908164851749.png)

输入想要展示的汉字，点击生成字模：

![image-20230908164552847](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230908164552847.png)

生成字模如下：

```
 湖(0) 南(1) 大(2) 学(3)
{
{0x10,0x60,0x02,0x8C,0x00,0x88,0x88,0xFF,0x88,0x88,0x00,0xFE,0x22,0x22,0xFE,0x00},
{0x04,0x04,0x7E,0x01,0x00,0x1F,0x08,0x08,0x08,0x9F,0x60,0x1F,0x42,0x82,0x7F,0x00},/*"湖",0*/
{0x04,0xE4,0x24,0x24,0x64,0xA4,0x24,0x3F,0x24,0xA4,0x64,0x24,0x24,0xE4,0x04,0x00},
{0x00,0xFF,0x00,0x08,0x09,0x09,0x09,0x7F,0x09,0x09,0x09,0x48,0x80,0x7F,0x00,0x00},/*"南",1*/
{0x20,0x20,0x20,0x20,0x20,0x20,0x20,0xFF,0x20,0x20,0x20,0x20,0x20,0x20,0x20,0x00},
{0x80,0x80,0x40,0x20,0x10,0x0C,0x03,0x00,0x03,0x0C,0x10,0x20,0x40,0x80,0x80,0x00},/*"大",2*/
{0x40,0x30,0x11,0x96,0x90,0x90,0x91,0x96,0x90,0x90,0x98,0x14,0x13,0x50,0x30,0x00},
{0x04,0x04,0x04,0x04,0x04,0x44,0x84,0x7E,0x06,0x05,0x04,0x04,0x04,0x04,0x04,0x00},/*"学",3*/
};
```

我们可以通过字模的编号来按顺序调用OLED_ShowChinese函数在给定的位置显示想要的汉字

## 项目实现

理解完显示原理后一切就很简单了，无非就是结合板子的功能在屏幕上显示一些东西，由于调用板子的功能需要写很多的回调函数，故本项目只基于K1 K2 K3三个按键实现了简单的菜单、确认、返回功能，并实现了动画播放以及菜单的功能。

需要注意的是

- 在开始项目的时候要**预先定义好各种状态**，并在执行显示功能的时候注意状态的变化，这样可以**避免内容反复刷新**
- 记得将重复部分的代码打包成函数，这样会让整个项目变得简洁（我写的很臃肿）
- 最好不要热插拔屏幕，可能会烧屏

其他详细的部分都放在github仓库中，实验报告中也写的比较详细了

如果想实现更多的功能也可以参考：https://blog.csdn.net/qq_51684393/article/details/126669757

​																https://wolfvoid.github.io/
