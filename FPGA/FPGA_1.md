**学习笔记：《FPGA学习笔记》索引**
[FPGA学习笔记#1 HLS简介及相关概念](https://blog.csdn.net/qq_38876396/article/details/142599792)
[FPGA学习笔记#2 基本组件——CLB、SLICE、LUT、MUX、进位链、DRAM、存储单元、BRAM](https://blog.csdn.net/qq_38876396/article/details/142761068)
[FPGA学习笔记#3 Vitis HLS编程规范、数据类型、基本运算](https://blog.csdn.net/qq_38876396/article/details/143643677)
[FPGA学习笔记#4 Vitis HLS 入门的第一个工程](https://blog.csdn.net/qq_38876396/article/details/143645004)
[FPGA学习笔记#5 Vitis HLS For循环的优化（1）](https://blog.csdn.net/qq_38876396/article/details/143662692)
[FPGA学习笔记#6 Vitis HLS For循环的优化（2）](https://blog.csdn.net/qq_38876396/article/details/143662824)
[FPGA学习笔记#7 Vitis HLS 数组优化和函数优化](https://blog.csdn.net/qq_38876396/article/details/143673251)
[FPGA学习笔记#8 Vitis HLS优化总结和案例程序的优化](https://blog.csdn.net/qq_38876396/article/details/143683331)

---

## 目录
* [1.HLS简介](#p1)
* [2.制导语句和约束](#p2)
* [3.SystemC](#p3)
* [4.Verilog和VHDL](#p4)
* [5.Verilog和C的区别](#p5)
* [6.FPGA的复位（重要！）](#p6)

****
# 1.HLS简介<a id="p1"></a>
**HLS（High Level Synthesis）**，高层次综合工具，可以使用户通过添加[制导语句（directives）和约束（constrains）](#2.制导语句和约束)，将C/C++/[System C](#3.SystemC)代码直接转为FPGA RTL（[Verilog，VHDL](#4.Verilog和VHDL)，SystemC）代码。

	Xilinx最新的HLS设计工具已经从Vivado HLS改为Vitis HLS
	
Vitis HLS支持将C、C++和OpenCL函数硬连线到器件逻辑互连结构和RAM/DSP块上。

****
# 2.制导语句和约束<a id="p2"></a>
**制导语句（directives、指导语句）**，依赖于源程序，开发中会在源程序中加入一些类似于注释的语句，称为制导语句。特别的在C语言中，可以使用 <font color=Red>**#pragma**</font> 来标记制导语句。常见的标准为OpenMP，较新的还有OpenACC、OpenHMPP。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/1/1.jpg"></div>



<font color=Red>**在适合并行的地方加入制导语句，可以提高计算设备的效能比**</font>。

约束（constrains），根据目标设备的资源限制，设置合适的资源约束，以限制资源使用量并优化性能。

****
# 3.SystemC<a id="p3"></a>
**系统建模语言**，基于C++开发的开源library，理论上和常见的boost库没有任何差别。只要环境支持C++，就可以进行SystemC建模和仿真。可以让软硬件并行开发，加快产品面世，其特点包括：
* 硬件架构探索，建立算法、性能模型；
* 验证工程师作为参考模型（通过DPI接口调用）；
* 设计工程师将其作为design spec，设计RTL；
* 软件工程师作为软件开发的硬件模型；
* 使多种提前测试成为可能；

在软硬件设计之前先开发**System C模型**（如TLM模型），**硬件部门将模型转化为RTL（寄存器传输级），软件部门在此模型上开发软件**，这样软硬件的任何错误都能被尽早发现。

	  TLM模型在各组件之间建立一个专用通道，让信息只在这个通道里流动，
	  因此避免了全局变量，public成员变量和config机制通讯时带来的不便

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/1/2.jpg"></div>

&emsp;&emsp;[图源-SystemC简介](https://blog.csdn.net/gsjthxy/article/details/126099348)

下图为SystemC框架图，**底层是C++及其标准库**，也可包括其他的C++库，如boost库。

<font color=Red>**SystemC library**</font>包含多种功能，最核心的有如下几项：
* **SystemC内核**：用于调度SystemC process，功能和Verilog、VHDL仿真内核非常相似；可以认为类似于Linux内核的极度简化版本。
* **SystemC process**：由用户定义，模拟硬件和软件的行为。其并行运行，由SystemC内核调度执行。
* **SystemC event**：在特定时间点发生的事件，SystemC内核和Verilog仿真内核一样是事件驱动型内核。
* **SystemC module**：用于将建模对象模块化、层次化；
* **SystemC channel/interface/port/export**：用于实现模块间通信

<div><img height=350 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/1/3.jpg"></div>

&emsp;&emsp;[图源-SystemC简介](https://blog.csdn.net/gsjthxy/article/details/126099348)

****
# 4.Verilog和VHDL<a id="p4"></a>
**HDL（Hardware Description Language，硬件描述语言）**，以文本形式描述数字系统硬件的结构和行为的语言，可以表示逻辑电路图、逻辑表达式，还可以表示数字逻辑系统所完成的逻辑功能。
Verilog HDL和VHDL是最流行的两种HDL。

**Verilog HDL和VHDL（Very-High-Speed Integrated Circuit Hardware Description Language，超高速集成电路硬件描述语言）** 是世界上最流行的两种硬件描述语言，都是在20世纪80年代中期开发出来的。前者由Gateway Design Automation公司（该公司于1989年被Cadence公司收购）开发，后者为美国军方组织开发。两种HDL均为IEEE标准。

	Tip: 中国、日本、美国大部分采用Verilog，而VHDL在欧洲应用较广泛
	
Verilog可以从五个层次对电路（系统）进行描述，包括:系统级、算法级、**寄存器传输级（即RTL级）**、门级、开关级。我们平时用的最多的为RTL级，故<font color=Red>**Verilog代码也经常被称为RTL代码**</font>。

****
# 5.Verilog和C的区别<a id="p5"></a>

**Verilog 是硬件描述语言**，在编译下载到 FPGA 之后，FPGA 会生成电路，所以 **Verilog 全部是并行处理与运行的**；**C 语言是软件语言**，编译下载到单片机/CPU 之后，还是软件指令，而单片机/CPU 处理软件指令需要取址、译码、执行，是**串行执行**的。

<font color=Red>**Verilog 和 C 的区别也是 FPGA 和单片机/CPU 的区别**</font>，由于 FPGA 全部并行处理，所以处理速度非常快，这个是 FPGA 的最大优势，这一点是单片机/CPU 替代不了的。

此外，Verilog学习难度要高得多（那是肯定的）。

---

### 6.FPGA的复位（重要！）<a id="p6"></a>
在 HLS 中，所有静态和全局变量都会被初始化为零（如果给定了初始化值，则初始化为其他值）。这包括 RAM，其中每个元素都被清除为零。

然而，**这种初始化只发生在 FPGA 首次编程时。任何后续处理器复位都不会触发初始化过程**。

如果需要清除设备的内部状态，那么应该包含某种复位协议（根据复位状态处理所需要的程序）。
