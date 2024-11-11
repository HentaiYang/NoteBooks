本笔记使用的Vitis HLS版本为2022.2，在windows11下运行，仿真part为xcku15p_CIV-ffva1156-2LV-e，主要根据教程：[跟Xilinx SAE 学HLS系列视频讲座-高亚军](https://www.bilibili.com/video/BV1bt41187RW)进行学习

## 目录
* [1.数组优化](#p1)
* &nbsp;&nbsp;&nbsp;&nbsp;[1.1.双端口内存](#p11)
* &nbsp;&nbsp;&nbsp;&nbsp;[1.2.数组分割](#p12)
* &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.2.1.一维数组分割](#p121)
* &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.2.2.多维数组分割](#p122)
* &nbsp;&nbsp;&nbsp;&nbsp;[1.3.数组合并](#p13)
* &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.3.1.横向合并](#p131)
* &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.3.2.纵向合并](#p132)
* &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.3.3.数组分割和数组合并的结合使用](#p133)
* &nbsp;&nbsp;&nbsp;&nbsp;[1.4.数组转换（reshape）](#p14)
* &nbsp;&nbsp;&nbsp;&nbsp;[1.5.数组分割、合并和转换的对比](#p15)
* &nbsp;&nbsp;&nbsp;&nbsp;[1.6.其他数组优化](#p16)
* &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.6.1.定义ROM](#p)
* &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.6.2.数组初始化](#p)
* &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[1.6.3.FPGA的复位（一定要知道！）](#p)
* [2.函数优化](#p)
* &nbsp;&nbsp;&nbsp;&nbsp;[2.1.数据类型优化](#p21)
* &nbsp;&nbsp;&nbsp;&nbsp;[2.2.inline](#p22)
* &nbsp;&nbsp;&nbsp;&nbsp;[2.3.Allocation](#p23)
* &nbsp;&nbsp;&nbsp;&nbsp;[2.4.DATAFLOW](#p24)

---
# 1.数组优化<a id="p1"></a>
数组的优化对程序性能的提升是非常重要的，一个合理的内存结构可以提高程序的并行度，我们要找到资源和性能的权衡点，最大化FPGA使用率。

如果数组为top函数传入的参数，会表现为外部memory；如果设计在内部，则会用内部RAM、LUTRAM、UltraRAM、寄存器等形式表示。

## 1.1.双端口内存<a id="p11"></a>

考虑如下图所示的例程，top函数接收一个输入数组mem[N]（N=4），将每三个连续数据求和，输出到sum[N-2]数组中，共2轮循环。

很明显，在每一次循环中需要读取mem数组三次，写入sum数组一次，可以看到右侧的时序图，完成一次写操作需要3个时钟周期。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/1.jpg"></div>


对于这样输出只用一次、输入使用多次，我们可以使用RESOURCE directive配置为双端口内存，提高对内存的读取效率：

```cpp
#pragma HLS RESOURCE variable=mem core=RAM_2P_BRAM
```

并同时对loop进行UNROOL：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/2.jpg"></div>

对应的资源消耗和时序如下，现在一个时钟周期可以读取2个数据，UNROLL出的2组电路在4个时钟周期便完成计算。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/3.jpg"></div>

---

## 1.2.数组分割<a id="p12"></a>
### 1.2.1.一维数组分割<a id="p121"></a>

如下图是对于一个6长度数组的不同memory分配，这三种方式分别对应了一个数组对连续块进行处理（常见的可以考虑前一半和后一半分别有不同的处理，block/factor=2）、一个数组对间隔数据进行处理（常见的可以考虑奇偶数据，cyclic/factor=2）、最高效率并发处理。

```cpp
Block/Factor=3方式分割：数组等分成三份，相邻2数据为1组
#pragma HLS ARRAY_PARTITION variable=mem block factor=3 dim=1
Cyclic/Factor=3方式分割：数组等分为三份，03一组14一组25一组
#pragma HLS ARRAY_PARTITION variable=mem cyclic factor=3 dim=1
Register方式分割：数组等分为6份，一个数据一组（占一个寄存器）
#pragma HLS ARRAY_PARTITION variable=mem complete dim=1
```

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/4.jpg"></div>

不同分割方法的时序如下：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/5.jpg"></div>

对于上面例程中的1输出-3输入运算关系，使用Block/Factor=2，分为2个数组，再使用双端口RAM，一次最多可以读取4数据，完全够用，无需开到Factor=3。

---

### 1.2.2.多维数组分割<a id="p122"></a>

以多维数组My_array[10][6][4]为例，10是一维，6是二维，4是三维，使用的约束和一维数组的一样，只需要改变dim的值：

```cpp
#pragma HLS ARRAY_PARTITION variable=mem block factor=3 dim=?
#pragma HLS ARRAY_PARTITION variable=mem cyclic factor=3 dim=?
#pragma HLS ARRAY_PARTITION variable=mem complete dim=?
```

对于指定不同dim，HLS会对不同的数组进行整体分割，如图为dim=3和dim=1所针对的数据，如果对dim3使用block/factor=2，则My_array_0和1一组，2和3一组进行分配。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/6.jpg"></div>


我们以矩阵加法为例，如图有两个4*5矩阵mat_a和mat_b，其相加后结果存储在sum矩阵中：
<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/7.jpg"></div>

我们可以三个数组采用单端口BRAM存储，均对第一维使用Block/Factor=4的分割，即每个矩阵分割为4个长度为5的一维数组，并对循环添加PIPELINE、UNROLL约束来进行优化：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/8.jpg"></div>

我们看一下Factor=4和2的情况，当Factor<该维数据个数时，会将对应数据依次拼接，总之一定会将一个N维数组变为一个N-1维数组。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/9.jpg"></div>

不同factor的时序图如下，可以通过地址线确定其在内存中也是顺序排列：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/10.jpg"></div>

---

## 1.3.数组合并<a id="p13"></a>
将多个小数组合并为一个大数组不仅可以减少资源用量，在合理使用下也能提高性能。

数组合并使用的约束是ARRAY_MAP，其分为横向和纵向。

### 1.3.1.横向合并<a id="p131"></a>
横向/水平方向合并（Horizontal mapping）使用的约束语句如下，其扩展数组长度，而不扩展数组位宽。
```cpp
#pragma ARRAY_MAP variable=A instance=ab_array horizontal
#pragma ARRAY_MAP variable=B instance=ab_array horizontal
```

如下图，有数组A[N]和B[M]，使用该约束后会合并为AB[N+M]，低地址为0 ~ N-1，高地址为N ~ N+M-1。

这一操作会扩展B（数据位宽较小者）的位宽，拼接后的数组以所有数组中最大位宽为准，长度为各个数组相加。

可以看出，这一操作所针对的场景和数组分割中的Block相似。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/11.jpg"></div>

得到的数组的内存分配如下：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/12.jpg"></div>

---

### 1.3.2.纵向合并<a id="p132"></a>
纵向/垂直方向合并（Vertical mapping）的依赖语句如下：

```cpp
#pragma ARRAY_MAP variable=A instance=ab_array vertical
#pragma ARRAY_MAP variable=B instance=ab_array vertical
```

如下图中，N>M，则扩展A数组的位宽为sizeof(A[0])+sizeof(B[0])，将B数组的对应数据和A数组对应数据拼接，A数组的长度不变。

这一操作针对的场景和数组分割中的cyclic相似。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/13.jpg"></div>

数组合并后的内存分配如下：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/14.jpg"></div>

---
### 1.3.3.数组分割和数组合并的结合使用<a id="p133"></a>
ARRAY_PARTITION和ARRAY_MAP是可以结合使用的，考虑如下例子：

对于2个数组m_accum和v_accum，我们在一个循环中分别使用其奇数下标的数据，在另一个循环中分别使用其偶数下标的数据。

这是我们从单个数组角度上，我们需要使用cyclic进行数据分割；从2个循环的角度上，我们需要将奇数据和奇数据水平拼接，偶数据和偶数据水平拼接，我们就可以使用如下的约束语句：

```cpp
#pragma HLS ARRAY_PARTITION variable=m_accum cyclic factor=2 dim=1
#pragma HLS ARRAY_PARTITION variable=v_accum cyclic factor=2 dim=1
#pragma HLS ARRAY_MAP variable=m_accum[0] instance=mv_accum horizontal
#pragma HLS ARRAY_MAP variable=v_accum[0] instance=mv_accum horizontal
#pragma HLS ARRAY_MAP variable=m_accum[1] instance=mv_accum_1 horizontal
#pragma HLS ARRAY_MAP variable=v_accum[1] instance=mv_accum_1 horizontal
```

---

## 1.4.数组转换（reshape）<a id="p14"></a>
对于单个数组，如果我们想让其从数据上仍为一个数组，但从结构上改变其数据排布，那么可以使用ARRAY_RESHAPE约束进行数组的reshape。

以一维数组为例，如下是3种reshape方法：

```cpp
Block/Factor=2：将数组一分为二，然后进行拼接，长度减半，数据位宽翻倍
#pragma HLS ARRAY_RESHAPE dim=1 factor=2 type=block variable=arr
Cyclic/Factor=2：将数据奇偶分组（对2取模为组号）进行拼接，长度减半，位宽翻倍
#pragma HLS ARRAY_RESHAPE dim=1 factor=2 type=cyclicvariable=arr
Complete：将所有数据拼到一个数据中，长度最短，位宽最长
#pragma HLS ARRAY_RESHAPE dim=1 type=complete variable=arr
```

其内存中的数据分配如下图：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/15.jpg"></div>

---

## 1.5.数组分割、合并和转换的对比<a id="p15"></a>
有例程如下图，数组A[N] B[M]，第一个循环分别处理A数组的奇偶数据，第二个循环分别处理B数组的前半、后半数据，第三、四个循环分别依次读取A、B，并写入到pa、pb

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/16.jpg"></div>

可以发现，第一个循环更适合对A启用cyclic/factor=2，第二个循环更适合对B启用block/factor=2。
我们对A和B尝试以下这几种约束，分别将A和B横向ARRAY_MAP拼接、将A和B纵向ARRAY_MAP拼接、对A启用ARRAY_RESHAPE cyclic/factor=2并对B启用ARRAY_RESHAPE block/factor=2。

因此，我们分别指定如下约束，其中最右侧为我们刚才分析的约束：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/17.jpg"></div>

最终得到结果如下，可以看到，使用RESHAPE的性能最高，但资源消耗也远超其他约束。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/18.jpg"></div>

---

## 1.6.其他数组优化<a id="p16"></a>

### 1.6.1.定义ROM<a id="p161"></a>
**方法1**：使用const + 初始化值

缺点：初始值较多时较繁琐，代码管理不便

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/19.jpg"></div>


**方法2**：使用头文件，将初始值放在文件中，代码/工程管理较方便

该头文件内结构应如下所示：

```cpp
1,
2,
3,
4,
5
```

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/20.jpg"></div>

需要注意的是使用下面的方式是不行的，因为这种方法是使用了#include展开文件到声明位置的原理，#include "xxx.h"必须是所在行唯一的语句。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/21.jpg"></div>

**方法3**：如果一个数据的value是通过数学计算得到的，并且在程序中不被更新，Vitis HLS会自动将其定义为ROM，如下图的sin_table：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/22.jpg"></div>

默认情况下ROM的输出latency是2，我们可以在RESOURCE directive中将其配置为自定义值。

---

### 1.6.2.数组初始化<a id="p162"></a>
如果想要将数组映射为RTL的memory，那么需要加static进行修饰，这样既可以保证数组映射为memory，而且还能节省初始化的时间。

如果想要将数组映射为ROM，就需要用到上面的三种定义ROM的方式。

---

### 1.6.3.FPGA的复位（一定要知道！）<a id="p163"></a>
这一点在第一篇笔记中提到过，这里讲到了静态变量，所以再说一遍。

在 HLS 中，所有静态和全局变量都会被初始化为零（如果给定了初始化值，则初始化为其他值）。这包括 RAM，其中每个元素都被清除为零。

然而，**这种初始化只发生在 FPGA 首次编程时。任何后续处理器复位都不会触发初始化过程**。

如果需要清除设备的内部状态，那么应该包含某种复位协议（根据复位状态处理所需要的程序）。

---

# 2.函数优化<a id="p2"></a>
## 2.1.数据类型优化<a id="p21"></a>
对于参数等数据使用的数据类型，如果可以提前确认某一个值的上限，使用ap_int<>等类型可以有效减少资源的使用量，如下图：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/23.jpg"></div>

---

## 2.2.inline<a id="p22"></a>
HLS中有INLINE约束，给函数添加INLINE约束后，在编译时会自动将inline修饰的函数展开到调用位置（硬件层面上），可以减少调用函数带来的开销。

```cpp
#pragma HLS INLINE
```

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/24.jpg"></div>

如下图，启用INLINE后最终只产生了一个模块：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/25.jpg"></div>

同时，可以在directive中关闭inline：

```cpp
#pragma INLINE off
```

---
## 2.3.Allocation<a id="p23"></a>
在上篇笔记中有过介绍，使用ALLOCATION约束将Accumulator函数实例化2次

```cpp
#pragma ALLOCATION instances=Accumulator limit=2 function
```

对默认、limit=1、limit=2的情形进行了测试，可以看到limit=2带来了更高的性能，但会消耗更多资源：	

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/26.jpg"></div>

---

## 2.4.DATAFLOW<a id="p24"></a>
应用于函数的DATAFLOW，可以将多个调用间的依赖关系形成流水线并行执行

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/7/27.jpg"></div>