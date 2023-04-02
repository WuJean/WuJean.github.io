---
title: I2C工训摸鱼指北
date: 2023-03-31 10:23:54
updated:
tags:
categories:
keywords:
description:
top_img:
comments:
cover:
---

![image-20230331103106836](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230331103106836.png)

首先先甩个图上来，这些东西我真的是一点都没听，每次去工训就和傻子一样坐着，直到要验收了才想起居然要做东西。相信很多同学有和我一样的感受，所以我在验收前一周的工训课上决定发奋图强，一个小时速通工训，为后来者减少这个东西的恶心程度。（其实是I2C协议让我提起了一点学习的兴趣）

# 速通开始

![image-20230331103444041](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230331103444041.png)

如果仅仅是要过的话，只要完成单元测试，上课没去别被点到（第二次课开始可能点名），提交了实验报告就能过了。但是我们速通需要的是精致速通，如何花最少的时间学到一点点东西；拿到不算低的分数；收获一点点成就感，尤为重要。

我们看占比最多的实验报告到底包含了什么内容：

![image-20230331103740363](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230331103740363.png)

1. 编写TestBench的内容（一点点课后习题）可以加点仿真的内容
2. ModelSim工具的使用（操作的截图）可以和上一步合并
3. 分析代码
4. 实验总结

## 实验报告部分（1）

如何描述组合电路 、时序电路、状态机？

- 组合电路是一种最基本的电路，它的输出只取决于当前的输入，没有任何延迟或存储，因此在响应时间上非常快速。它通常用于执行简单的逻辑操作，例如布尔运算、逻辑比较和选择。

- 时序电路是一种电路，它包含了存储器元件，可以在内部存储一定数量的位或字，并且可以根据时钟信号进行读写。时序电路的输出不仅取决于当前输入，还取决于存储器的状态和历史，因此具有一定的延迟和存储能力。

- 状态机是一种特殊的时序电路，它具有多个状态和转移条件，可以响应不同的输入信号并在不同的状态之间转换。状态机通常用于控制系统，例如逻辑控制器、计数器和序列检测器等。状态机通常由时序电路和组合逻辑电路组成，可以描述为时序电路的一种应用。

它们之间的关系可以这样描述：组合电路是时序电路和状态机的基础，因为它们都可以由基本的逻辑门和它们的组合构成。时序电路是状态机的基础，因为状态机可以看作是一种具有特定状态转移规则的时序电路。状态机可以用于控制时序电路的状态转移，例如控制存储器的读写、控制数据通路的选择等。

如何编写TestBench？

编写TestBench需要以下步骤：

1. 定义测试向量：根据设计规格书，定义输入测试向量，并将其转化为二进制格式，以便仿真器能够读取和应用。
2. 编写测试脚本：使用测试脚本编写仿真环境，设置仿真时钟和仿真周期，以便在仿真器中运行测试向量，并捕获输出信号。测试脚本通常使用HDL（硬件描述语言）编写，例如Verilog、VHDL或SystemVerilog。
3. 仿真执行：运行仿真器，将测试向量提供给设计，并捕获输出信号。对于每个仿真周期，测试脚本会检查输出信号是否与预期的输出信号匹配。如果不匹配，则测试失败。
4. 结果分析：分析仿真结果，检查测试向量是否能够正确覆盖设计的所有状态和操作。如果测试失败，则需要调查问题，并修改设计或测试向量以确保测试覆盖到所有的设计情况。

## 安装软件

学习通资料中下载Modelsim&Crack，解压后按ReadMe.txt中的内容执行，有几点需要注意的：

- 在执行安装程序的最后一步提示你要重启的时候，选择否

- 破解软件要记得放到软件安装目录的win64文件夹下，再执行，否则会提示找不到dll文件
- 设置环境变量的时候要记得添加的是许可证的文件！
- 打开软件的时候记得使用管理员权限打开，否则会提示找不到许可证

## 实验报告部分（2）Modelsim的使用

我们直接使用自定义的FSM工程展示软件的使用

### 一、自定义说明

下面是一个状态机，描述一个人一天的活动，包括上课、自习、开会、吃饭等活动。状态不少于10种，不少于5个判断选择。

```
// fsm.v

module fsm (
    input clk,    // 时钟信号
    input rst,    // 复位信号
    input alarm,  // 闹钟信号
    input class,  // 上课信号
    input study,  // 自习信号
    input meet,   // 开会信号
    input lunch,  // 中餐信号
    input dinner, // 晚餐信号
    output reg[3:0] state // 状态输出
);

// 状态定义
parameter S0 = 0;
parameter S1 = 1;
parameter S2 = 2;
parameter S3 = 3;
parameter S4 = 4;
parameter S5 = 5;
parameter S6 = 6;
parameter S7 = 7;
parameter S8 = 8;
parameter S9 = 9;
parameter S10 = 10;

// 状态寄存器
reg[3:0] current_state;

// 初始化状态
initial begin
    current_state = S0;
end

// 状态转移
always@(posedge clk, posedge rst) begin
    if (rst) begin
        current_state <= S0;
    end 
    else begin
        case(current_state)
            S0: if (alarm) current_state <= S1;
            S1: if (class) current_state <= S2;
            S2: if (study) current_state <= S3;
            S3: if (meet) current_state <= S4;
            S4: if (lunch) current_state <= S5;
            S5: if (alarm) current_state <= S6;
            S6: if (class) current_state <= S7;
            S7: if (study) current_state <= S8;
            S8: if (meet) current_state <= S9;
            S9: if (dinner) current_state <= S10;
        endcase
    end
end

// 输出状态
always@(current_state) begin
    state = current_state;
end

endmodule
```

下面是一个测试程序，用于测试状态机的功能。我们可以通过改变输入信号的值，观察状态机的输出状态，以验证状态机的正确性。

```
// fsm_tb.v

module fsm_tb;

// 时钟信号
reg clk;
always #5 clk = ~clk;

// 复位信号
reg rst;

// 输入信号
reg alarm;
reg class;
reg study;
reg meet;
reg lunch;
reg dinner;

// 输出信号
wire [3:0] state;

// 实例化被测试模块
fsm dut(
.clk(clk),
.rst(rst),
.alarm(alarm),
.class(class),
.study(study),
.meet(meet),
.lunch(lunch),
.dinner(dinner),
.state(state)
);

// 初始化
initial begin
clk = 0;
rst = 1;
alarm = 0;
class = 0;
study = 0;
meet = 0;
lunch = 0;
dinner = 0;
#5 rst = 0; // 释放复位
#250 $finish; // 结束仿真
end

// 测试用例
always@(posedge clk) begin
case($time)
5: alarm = 1;
15: alarm = 0;
25: class = 1;
35: class = 0;
45: study = 1;
55: study = 0;
65: meet = 1;
75: meet = 0;
85: lunch = 1;
95: lunch = 0;
105: alarm = 1;
115: alarm = 0;
125: class = 1;
135: class = 0;
145: study = 1;
155: study = 0;
165: meet = 1;
175: meet = 0;
185: lunch = 1;
195: lunch = 0;
205: dinner = 1;
215: dinner = 0;
endcase
end

// 输出状态
always@(posedge clk) begin
$display("Time=%d, State=%d", $time, state);
end

endmodule
```

 我们可以按照以下步骤进行仿真：

1. 建立工程 

2. 添加代码文件 `fsm.v` 和 `fsm_tb.v` 

3. 编译代码 

4. 启动仿真

5. 添加波形信号 

6. 执行仿真流程 

   最后，我们可以观察波形图，以验证状态机的功能。波形图应该显示出输入信号和输出状态随时间的变化。

编写完fsm.v以及fsm_tv.v后我们新建工程：

![image-20230402132931877](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402132931877.png)



将两个文件添加到工程中：

![image-20230402133013295](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402133013295.png)

编译所有程序：

![image-20230402133202091](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402133202091.png)

在view中找到找到library，在work中找到\_db文件，右键选择仿真。 （一定要在这里打开！）

![image-20230402154037002](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402154037002.png)

编译出波形（妈的我调了两个小时）这边建议不要调直接抄我的

![image-20230402152735372](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402152735372.png)

第二个任务到此结束，这边建议如果要自己写的话用异步的，且判断跳转最好是直接跳转到下一状态，不然很容易弄混掉，调试起来极其困难。

本来想摸鱼来着的现在把这东西的操作和原理学的差不多了。。。。

## 实验报告（3）EEPRROM读写代码及仿真

### 写在前面

I2C（Inter-Integrated Circuit）是一种串行通信协议，通常用于连接多个数字电路芯片。它由飞利浦（Philips）公司在1980年代开发，现在已成为一种标准的总线协议。I2C协议需要两个信号线：SCL（串行时钟）和SDA（串行数据）。I2C总线支持多主机配置，即多个主机可以访问同一组设备。

EEPROM（Electrically Erasable Programmable Read-Only Memory）是一种可以在电子设备上存储数据的非易失性存储器。它与普通的ROM（Read-Only Memory）不同，可以通过电子擦除和编程操作来修改存储的数据。EEPROM通常被用作存储系统配置信息和其他重要的数据。

在一个I2C总线上连接一个EEPROM设备需要指定一个设备地址。EEPROM可以通过I2C总线进行读写操作。写操作需要指定要写入的内存地址和写入的数据。读操作需要指定要读取的内存地址，并从设备中读取数据。对EEPROM的读写操作通常被称为“Byte Write”和“Random Read”。在“Byte Write”操作中，单个字节被写入EEPROM。在“Random Read”操作中，从EEPROM中读取单个字节的数据。

#### I2C是如何工作的

![](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402172437019.png)

可以理解为：

- SCL提供时钟信号
- SDA根据时钟信号来传输数据

I2C通信协议是一个基于起始条件和停止条件的序列化协议。下面是一个基本的I2C通信过程：

1. 主机发送起始条件信号（START）。
2. 主机发送设备地址，以及读/写操作位（RW）。
3. 设备返回应答信号（ACK）。
4. 主机发送内存地址。
5. 设备返回应答信号。
6. 主机发送数据。
7. 设备返回应答信号。
8. 主机发送停止条件信号（STOP）。

对于“Byte Write”操作，主机发送起始条件信号后，发送设备地址和写操作位。设备返回应答信号后，主机发送内存地址，并等待设备返回应答信号。然后，主机发送要写入的单个字节，并等待设备返回应答信号。最后，主机发送停止条件信号。

对于“Random Read”操作，主机发送起始条件信号后，发送设备地址和写操作位。设备返回应答信号后，主机发送内存地址，并等待设备返回应答信号。主机然后发送重复起始条件信号（RESTART），并发送设备地址和读操作位。设备返回应答信号后，设备发送一个字节的数据。主机返回应答信号后，设备可以继续发送更多的数据或发送停止条件信号。

#### 如何与EEPRROM结合

要与EEPROM结合使用，需要编写一个I2C总线驱动程序和一个EEPROM设备驱动程序。

I2C总线驱动程序负责管理I2C总线的通信，发送起始条件信号、设备地址、读/写操作位、内存地址和数据，并等待设备返回应答信号。主机可以使用这个驱动程序与多个设备进行通信。

EEPROM设备驱动程序负责解析主机发送的命令，执行相应的读写操作，并返回数据或状态信息。EEPROM驱动程序通常包括以下函数：

1. 读取字节函数：根据指定的内存地址从EEPROM中读取一个字节的数据。
2. 写入字节函数：将一个字节的数据写入指定的内存地址。
3. 读取数据函数：从EEPROM中读取指定长度的数据。
4. 写入数据函数：将指定长度的数据写入EEPROM。

这些函数可以根据具体的EEPROM型号和应用需求进行实现。主机可以通过这些函数进行“Byte Write”和“Random Read”操作。

在编写EEPROM设备驱动程序时，需要注意EEPROM的地址范围和操作时序。不同型号的EEPROM可能有不同的地址范围和操作时序，需要仔细查看其数据手册。此外，还需要考虑对数据的保护和错误检测机制，以确保数据的完整性和可靠性。

### 实验内容

1. 在ModelSim建立工程，在波形窗口分析各个信号的作用；
2. 能结合ModelSim仿真波形理解每一行代码的作用；
3. 参考设计文件仅实现了“Byte Write”和“Random Read"功能，如能实现更多读写模式将加分。

既然是摸鱼那就只做前两个就行了

1. 编写编写设计文件和TestBench文件，实现对EEPROM的读写功能验证
   1. 设计文件（i2c.v)：实现对EEPROM模型的“Byte Write”和“Random Read"功能；
   2. 测试文件（i2c_tb.v)：实现“Byte Write”和“Random Read"的功能验证；
   3. 建立完整的工程，编译正确，波形能证明读写功能的正确性。
   4. 工程目录如下图所示：

![image-20230402160221027](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402160221027.png)

这边我们直接使用参考的设计文件：（看起来好像要自己手打）

### 端口

```
`timescale 1ns / 1ps //时间单位1ns，精度1ps
module i2c(
input clk, //时钟
input rstn, //复位
input write_op, //写操作
input [7:0]write_data, //需要写入的数据
input read_op, //读操作
output reg [7:0]read_data, //读出的数据
input [7:0]addr, //数据地址
output op_done, //操作结束
output reg scl, //scl
inout sda //sda
);
```

### 状态

Byte Write : START + DEVICE +ACK + ADDR + ACK + DATA + ACK + STOP         

Random Read : START + DEVICE + ACK +ADDR + ACK+START + DEVICE + DATA + NO ACK + STOP        

每个状态用8位16进制数表示，对照上面图中SDA上数据传输情况，每一位数据传输都有一个对应的状态，多出了一些等待状态。

```
parameter IDLE =8'h00,
WAIT_WTICK0=8'h01,
WAIT_WTICK1=8'h02,
W_START=8'h03,
W_DEVICE7=8'h04,
W_DEVICE6=8'h05,
W_DEVICE5=8'h06,
W_DEVICE4=8'h07,
W_DEVICE3=8'h08,
W_DEVICE2=8'h09,
W_DEVICE1=8'h0a,
W_DEVICE0=8'h0b,
W_DEVACK=8'h0c,
W_ADDRES7=8'h0d,
W_ADDRES6=8'h0e,
W_ADDRES5=8'h0f,
W_ADDRES4=8'h10,
W_ADDRES3=8'h11,
W_ADDRES2=8'h12,
W_ADDRES1=8'h13,
W_ADDRES0=8'h14,
W_AACK=8'h15,
W_DATA7=8'h16,
W_DATA6=8'h17,
W_DATA5=8'h18,
W_DATA4=8'h19,
W_DATA3=8'h1a,
W_DATA2=8'h1b,
W_DATA1=8'h1c,
W_DATA0=8'h1d,
W_DACK=8'h1e,
WAIT_WTICK3=8'h1f,
R_START=8'h20,
R_DEVICE7=8'h21,
R_DEVICE6=8'h22,
R_DEVICE5=8'h23,
R_DEVICE4=8'h24,
R_DEVICE3=8'h25,
R_DEVICE2=8'h26,
R_DEVICE1=8'h27,
R_DEVICE0=8'h28,
R_DACK=8'h29,
R_DATA7=8'h2a,
R_DATA6=8'h2b,
R_DATA5=8'h2c,
R_DATA4=8'h2d,
R_DATA3=8'h2e,
R_DATA2=8'h2f,
R_DATA1=8'h30,
R_DATA0=8'h31,
R_NOACK=8'h32,
S_STOP=8'h33,
S_STOP0=8'h34,
S_STOP1=8'h35,
W_OPOVER=8'h36;
reg [7:0]i2c,next_i; //当前状态，下一状态
```

### SCL同步

SCL同步是指当主设备在SCL线上产生时钟脉冲时，从设备能够正确地识别这些脉冲并将数据正确地传输到主设备。

在I2C总线中，主设备控制时钟线，从设备根据主设备的时钟脉冲来同步数据传输。主设备通过发送起始信号和地址信号来选定一个从设备，并向从设备发送或接收数据。当主设备发出一个时钟脉冲时，从设备在接收到脉冲后采样SDA线上的数据。如果数据被正确地采样，从设备会发出一个确认信号（ACK）。

SCL同步对于I2C通信的正确性非常重要，因为如果从设备不能正确地同步时钟脉冲，数据传输可能会出错。在设计I2C接口电路时，需要仔细考虑SCL同步问题，以确保从设备能够正确地识别时钟脉冲并进行数据传输。

```
reg [7:0]div_cnt; //时钟计数器
wire scl_tick;
//计数，一个时钟周期div_cnt+1
always @(posedge clk or negedge rstn)
    if(!rstn) div_cnt <=8'd0;
    else if((i2c==IDLE)|scl_tick) div_cnt <=8'd0;
    else div_cnt<=div_cnt+1'b1;
//scl时间
wire scl_ls =(div_cnt==8'd0); //scl low
wire scl_lc = (div_cnt==8'd7); //scl low center
wire scl_hs =(div_cnt==8'd15); //scl high
wire scl_hc = (div_cnt==8'd22); //scl high center
assign scl_tick = (div_cnt==8'd29); //一个周期结束
```

使用SCL同步的好处是，它可以确保数据在传输时被正确地解释和接收，从而减少数据传输错误的发生。此外，SCL还可以用来控制I2C总线上的设备数目，以确保总线上只有一个设备处于活动状态，避免多个设备同时进行数据传输而产生干扰

### 状态更新

```
//状态
always @(posedge clk or negedge rstn)
if(!rstn) i2c <=0;
else i2c <= next_i;
```

### 读写命令的判断

读写命令是通过端口输入write_op和read_op确定的，这两个信号是低电平有效，用了wr_op和rd_op两个寄存器把输入的write_op和 read_op取反，这样如果有读或写命令，wr_op或rd_op为1，复位和操作结束(wr_opover)时都要清零，接下来就用wr_op和rd_op判断是否读写了，这里仅仅是把原来低电平有效的信号替换为两个高电平有效信号。

```
reg wr_op,rd_op; //读写操作
always @ (posedge clk or negedge rstn)
    if(!rstn) wr_op <= 0;
    else if (i2c==IDLE) wr_op <= ~write_op;
    else if(i2c==W_OPOVER) wr_op <=1'b0;
always @(posedge clk or negedge rstn)
    if(!rstn) rd_op <= 0;
    else if (i2c==IDLE) rd_op <= ~read_op;
    else if(i2c==W_OPOVER) rd_op <=1'b0;
wire d5ms_over; //等待
```

### 下一状态的判断

代码定义了多个状态，从IDLE（空闲）状态开始。当有读写操作时，状态将转换到WAIT_WTICK0状态，然后转移到W_START状态，再到W_DEVICE0到W_DACK状态，以完成写入操作。同样，当进行读取操作时，状态将转换到WAIT_WTICK3状态，然后转移到R_START状态，再到R_DEVICE0到R_DACK状态，以完成读取操作。

在代码中，SCL_TICK信号表示SCL时钟信号的状态。WR_OP和RD_OP信号表示写入操作和读取操作的状态。状态之间的转换由next_i变量控制。

各个状态的具体含义：

- IDLE（空闲）状态：系统处于空闲状态，没有进行任何读写操作。
- WAIT_WTICK0状态：等待写操作的信号WTICK0，表示系统已准备好接收写入数据。
- W_START状态：开始进行写操作。
- W_DEVICE0状态：写操作的目标设备为设备0。
- W_DACK状态：等待写操作的确认信号DACK，表示写操作已完成。
- WAIT_WTICK3状态：等待读操作的信号WTICK3，表示系统已准备好读取数据。
- R_START状态：开始进行读操作。
- R_DEVICE0状态：读操作的目标设备为设备0。
- R_DACK状态：等待读操作的确认信号DACK，表示读操作已完成。

```
always@(*)
case (i2c)
IDLE: begin next_i = IDLE;if(wr_op|rd_op) next_i = WAIT_WTICK0;end //有读写操作跳出空闲状态
//wait tick
WAIT_WTICK0:begin next_i = WAIT_WTICK0;if(scl_tick) next_i=WAIT_WTICK1;end
WAIT_WTICK1:begin next_i = WAIT_WTICK1;if(scl_tick) next_i = W_START;end
//START:SCL=1,SDA=1->0(scl_lc)
W_START:begin next_i=W_START;if(scl_tick) next_i=W_DEVICE7;end
//DEVICE ADDRESS（1010_000_0(WRITE)）
W_DEVICE7:begin next_i = W_DEVICE7;if(scl_tick) next_i=W_DEVICE6;end
W_DEVICE6:begin next_i = W_DEVICE6;if(scl_tick) next_i=W_DEVICE5;end
W_DEVICE5:begin next_i = W_DEVICE5;if(scl_tick) next_i=W_DEVICE4;end
W_DEVICE4:begin next_i = W_DEVICE4;if(scl_tick) next_i=W_DEVICE3;end
W_DEVICE3:begin next_i = W_DEVICE3;if(scl_tick) next_i=W_DEVICE2;end
W_DEVICE2:begin next_i = W_DEVICE2;if(scl_tick) next_i=W_DEVICE1;end
W_DEVICE1:begin next_i = W_DEVICE1;if(scl_tick) next_i=W_DEVICE0;end
W_DEVICE0:begin next_i = W_DEVICE0;if(scl_tick) next_i=W_DEVACK;end
//ACK
W_DEVACK:begin next_i=W_DEVACK;if(scl_tick) next_i=W_ADDRES7;end
//WORD ADDRESS
W_ADDRES7 :begin next_i = W_ADDRES7;if(scl_tick) next_i=W_ADDRES6;end
W_ADDRES6 :begin next_i = W_ADDRES6;if(scl_tick) next_i=W_ADDRES5;end
W_ADDRES5 :begin next_i = W_ADDRES5;if(scl_tick) next_i=W_ADDRES4;end
W_ADDRES4 :begin next_i = W_ADDRES4;if(scl_tick) next_i=W_ADDRES3;end
W_ADDRES3 :begin next_i = W_ADDRES3;if(scl_tick) next_i=W_ADDRES2;end
W_ADDRES2 :begin next_i = W_ADDRES2;if(scl_tick) next_i=W_ADDRES1;end
W_ADDRES1 :begin next_i = W_ADDRES1;if(scl_tick) next_i=W_ADDRES0;end
W_ADDRES0 :begin next_i = W_ADDRES0;if(scl_tick) next_i=W_AACK;end
//ACK
W_AACK:begin next_i = W_AACK;
if(scl_tick&wr_op) next_i=W_DATA7; //wr_op即写命令，开始写数据
else if(scl_tick&rd_op) next_i=WAIT_WTICK3; //rd_op读命令,则下一状态为WAIT_WTICK3
end
//WRITE DATA[7:0]
W_DATA7:begin next_i=W_DATA7;if(scl_tick)next_i=W_DATA6;end
W_DATA6:begin next_i=W_DATA6;if(scl_tick)next_i=W_DATA5;end
W_DATA5:begin next_i=W_DATA5;if(scl_tick)next_i=W_DATA4;end
W_DATA4:begin next_i=W_DATA4;if(scl_tick)next_i=W_DATA3;end
W_DATA3:begin next_i=W_DATA3;if(scl_tick)next_i=W_DATA2;end
W_DATA2:begin next_i=W_DATA2;if(scl_tick)next_i=W_DATA1;end
W_DATA1:begin next_i=W_DATA1;if(scl_tick)next_i=W_DATA0;end
W_DATA0:begin next_i=W_DATA0;if(scl_tick)next_i=W_DACK;end
//ACK
W_DACK:begin next_i=W_DACK; if(scl_tick) next_i=S_STOP;end
//Current Address Read
//START: SCL=1,SDA=1->0(scl_lc)
WAIT_WTICK3:begin next_i=WAIT_WTICK3; if(scl_tick) next_i=R_START;end
R_START:begin next_i=R_START; if(scl_tick)next_i=R_DEVICE7;end
//DEVICE ADDRESS(1010_000_1(READ))
R_DEVICE7:begin next_i=R_DEVICE7; if(scl_tick) next_i=R_DEVICE6;end
R_DEVICE6:begin next_i=R_DEVICE6; if(scl_tick) next_i=R_DEVICE5;end
R_DEVICE5:begin next_i=R_DEVICE5; if(scl_tick) next_i=R_DEVICE4;end
R_DEVICE4:begin next_i=R_DEVICE4; if(scl_tick) next_i=R_DEVICE3;end
R_DEVICE3:begin next_i=R_DEVICE3; if(scl_tick) next_i=R_DEVICE2;end
R_DEVICE2:begin next_i=R_DEVICE2; if(scl_tick) next_i=R_DEVICE1;end
R_DEVICE1:begin next_i=R_DEVICE1; if(scl_tick) next_i=R_DEVICE0;end
R_DEVICE0:begin next_i=R_DEVICE0; if(scl_tick) next_i=R_DACK;end
//ACK
R_DACK:begin next_i=R_DACK;if(scl_tick) next_i=R_DATA7;end
//READ DATA[7:0], SDA:input
R_DATA7:begin next_i=R_DATA7;if(scl_tick) next_i=R_DATA6;end
R_DATA6:begin next_i=R_DATA6;if(scl_tick) next_i=R_DATA5;end
R_DATA5:begin next_i=R_DATA5;if(scl_tick) next_i=R_DATA4;end
R_DATA4:begin next_i=R_DATA4;if(scl_tick) next_i=R_DATA3;end
R_DATA3:begin next_i=R_DATA3;if(scl_tick) next_i=R_DATA2;end
R_DATA2:begin next_i=R_DATA2;if(scl_tick) next_i=R_DATA1;end
R_DATA1:begin next_i=R_DATA1;if(scl_tick) next_i=R_DATA0;end
R_DATA0:begin next_i=R_DATA0;if(scl_tick) next_i=R_NOACK;end
//NO ACK
R_NOACK:begin next_i=R_NOACK;if(scl_tick) next_i=S_STOP;end
//STOP
S_STOP:begin next_i=S_STOP;if(scl_tick) next_i=S_STOP0;end
S_STOP0:begin next_i=S_STOP0;if(scl_tick) next_i=S_STOP1;end
S_STOP1:begin next_i=S_STOP1;if(scl_tick) next_i=W_OPOVER;end
//WAIT write_op=0,read_op=0;
W_OPOVER:begin next_i = W_OPOVER;if(d5ms_over)next_i=IDLE;end //操作结束回到空闲状态
default:begin next_i= IDLE;end
endcase
```

### SCL同步实现

```
//SCL
assign clr_scl=scl_ls&(i2c!=IDLE)&(i2c!=WAIT_WTICK0)& //clr_scl，scl置0信号
(i2c != WAIT_WTICK1)&(i2c!=W_START)&(i2c!=R_START)
&(i2c!=S_STOP0)&(i2c!=S_STOP1)&(i2c!=W_OPOVER);
always @(posedge clk or negedge rstn)
if(!rstn) scl <= 1'b1; //复位，scl为高电平
else if(clr_scl) scl <= 1'b0; //scl 1->0
else if(scl_hs) scl <=1'b1; //scl 0->1，high start，clk15
```

### SDA实现

SDA线是I2C总线上的双向数据线，用于在主设备和从设备之间传输数据。当主设备想要向从设备发送数据时，它将数据写入SDA线，并将时钟信号发送到SCL（Serial Clock）线上。从设备在接收到时钟信号后读取SDA线上的数据。同样地，当从设备想要向主设备发送数据时，它将数据写入SDA线，并等待主设备发送时钟信号

![image-20230402172437019](https://raw.githubusercontent.com/WuJean/Picgo-blog/main/image-20230402172437019.png)

