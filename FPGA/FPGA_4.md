本笔记使用的Vitis HLS版本为2022.2，在windows11下运行，仿真part为xcku15p_CIV-ffva1156-2LV-e，这一篇终于没有再大量使用别人的内容，是我自己从头捋到尾的结果，不过之后的笔记还是要参照别人的教程就是了。
## 目录
* [1.工程的创建](#p1)
* [2.添加程序文件](#p2)
* [3.程序编写](#p3)
* [4.指定Top函数（第一次执行）](#p4)
* [5.程序运行](#p5)
* &nbsp;&nbsp;&nbsp;&nbsp;[5.1.C仿真](#p51)
* &nbsp;&nbsp;&nbsp;&nbsp;[5.2.C综合](#p52)
* &nbsp;&nbsp;&nbsp;&nbsp;[5.3.C/RTL联合仿真](#p53)
* [6.简单的优化](#p6)

---
# 1.工程的创建<a id="p1"></a>
Vitis的安装请自行查找教程，本教程使用的Xilinx IDE为Vitis HLS，比较早的版本则可以使用Vivado HLS，功能都差不多。

首先点击Create Project创建工程。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/1.jpg"></div>

输入工程名和工程目录，点击Next。

<div><img height=400 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/2.jpg"></div>

无视添加Design文件和Testbench文件，我们之后再加，直接点Next。

<div><img height=300 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/3.jpg"></div>

选择FPGA型号，根据自己需要写，如果想完整复现结果就选一样的，选完后点OK，再点Finish完成创建。

<div><img height=400 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/4.jpg"></div>

<div><img height=400 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/5.jpg"></div>

工程创建结束。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/6.jpg"></div>

---

# 2.添加程序文件<a id="p2"></a>
在工程界面左上角是资源管理器，右键Source->New Source File，创建top.h和top.cpp（一次只能创建一个文件），然后同样的方式右键Test Bench->New Test Bench File，创建test.cpp文件。文件存储位置任意，可以放在工程根目录。

<div><img height=200 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/7.jpg"></div>

<div><img height=100 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/8.jpg"></div>

创建完成后如下图所示：

<div><img height=200 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/9.jpg"></div>

至于为什么不和其他教程里叫VectorAdd.h和.cpp，因为这样以后用这个工程直接跑别的测试没有啥违和感。

---

# 3.程序编写<a id="p3"></a>
我们用一个经典的测试例程，数组A的每一个元素+数据t，其结果输出到数组B中。

top.h：

```cpp
// top.h
#define N 5
typedef int data_t;
void VectorAdd(data_t A[N],data_t t,data_t B[N]);
```

top.cpp：

```cpp
// top.cpp
#include "top.h"

void VectorAdd(data_t A[N],data_t t,data_t B[N])
{
	unsigned int i;
	myloop:
	for(i=0;i<N;i++)
	{
		B[i] = A[i] + t;
	}
}
```

test.cpp：

```cpp
#include <iostream>
#include <iomanip>
#include "top.h"

using namespace std;

int main(){
	data_t A[N] = {-4,-3,0,1,2};
	data_t c = 5;
	data_t B[N] = {0};
	data_t RefB[N] = {1,2,5,6,7};
	unsigned int i = 0;
	unsigned int errcnt = 0;

	VectorAdd(A,c,B);

	cout<<setfill('-')<<setw(30)<<'-'<<'\n';
	cout<<setfill(' ')<<setw(10)<<left<<"A";
	cout<<setfill(' ')<<setw(10)<<left<<"C";
	cout<<setfill(' ')<<setw(10)<<left<<"B"<<'\n';
	cout<<setfill('-')<<setw(30)<<'-'<<'\n';

	for ( i = 0;i<N;i++)
	{
		cout<<setfill(' ')<<setw(10)<<left<<A[i];
		cout<<setfill(' ')<<setw(10)<<left<<c;
		cout<<setfill(' ')<<setw(10)<<left<<B[i];
		if(B[i] == RefB[i])
		{
			cout<<'\n';
		}
		else
		{
			cout << "(" << RefB[i] << ")" << '\n';
			errcnt ++ ;
		}
	}

	cout << setfill('-') << setw(30) << '-' <<'\n';

	if(errcnt > 0)
	{
		cout << "Test Failed" << '\n';
		return 1;
	}
	else{
		cout<< "Test Passed" << '\n';
		return 0;
	}
}
```

---

# 4.指定Top函数（第一次执行）<a id="p4"></a>

第一次执行前，我们要先配置程序的Top函数，HLS的程序分为内核代码（Top函数及其依赖）和主机代码（Test Bench及其依赖）。

点击菜单栏Project->Project Settings。

<div><img height=300 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/10.jpg"></div>

点击Synthesis->Browse，选择仿真的Top Fuction。

<div><img height=450 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/11.jpg"></div>

选择编写的VectorAdd函数，依次点击OK后配置完成。

<div><img height=500 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/12.jpg"></div>

---

# 5.程序运行<a id="p5"></a>

## 5.1.C仿真<a id="p51"></a>

点击左下角的Run C Simulation，执行C仿真，得到代码的执行结果。

<div><img height=300 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/13.jpg"></div>

直接点OK。

<div><img height=300 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/14.jpg"></div>

打印内容为下，说明仿真执行成功。

<div><img height=250 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/15.jpg"></div>

---

## 5.2.C综合<a id="p52"></a>

点击Run C Synthesis执行C综合。

<div><img height=250 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/16.jpg"></div>

点击OK。

<div><img height=300 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/17.jpg"></div>

可以看到C代码被综合成RTL代码后，会给出其性能指标、使用的硬件资源量，以及下面会给出接口等信息。

循环迭代延迟就是从上一轮循环结束到判断是否进入下一轮循环的延迟，比如for(int i=0; i<N; i++)中i++和i<N所需的时钟周期数，Interval就是循环之间的间隔周期数。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/18.jpg"></div>

---

## 5.3.C/RTL联合仿真<a id="p53"></a>

进行RTL和C的联合仿真CoSimulation。

<div><img height=300 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/19.jpg"></div>

注意这里Dump Trace要选择port，仿真后才可以查看波形。

<div><img height=450 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/20.jpg"></div>

联合仿真结束后会打印报告，主要作用为显示执行性能指标Latency，同时如果Dump Trace为port，则左上角的波形图图标会亮起，点击可以查看仿真波形。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/21.jpg"></div>

打开波形图后，首先将原来的展示部分全部删除，然后只将Top函数（VectorAdd）生成的结果拖动到波形窗口，然后点击图中的图标展开所有信号。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/22.jpg"></div>

然后可以看到有一些值我们直接看看不懂，那是因为它默认按照十六进制显示，右键对应的数据，然后选择Signed Decimal，因为我们使用的是有符号数据，所以选择这个。

也可以设置Signal Color来配置不同数据的颜色，看起来更加方便一些。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/23.jpg"></div>

从这张图可以很清晰的看到A数组、B数组取出的数据，用于取出数据的地址，以及运算得到的结果。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/24.jpg"></div>

---
# 6.简单的优化<a id="p6"></a>
接下来我们对Top函数进行简单的优化，因为新版本的Vitis中对每个循环默认添加了PIPELINE（流水线）优化，因此我们对top.cpp的loop进行UNROLL（展开）优化。

首先New一个Solution：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/25.jpg"></div>

编写solution的名称，并确定是否从其他solution中拷贝约束，因为solution1中没有配置任何约束，所以勾不勾选都可以。

<div><img height=500 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/26.jpg"></div>

添加solution后会自动切换到该solution，正在使用的solution图标左上角会有一个对钩。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/27.jpg"></div>

选择top.cpp，点击右侧的Directive（没有的可以在Window->Show View->Directive打开），右键myloop标签，添加约束。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/28.jpg"></div>

选择UNROLL，并点击OK，完成添加，这里选择Directive File会将约束语句放在solution_name/constraints/directive.tcl中，选择Source File会将约束语句直接写在程序中，这里以Source File为例，因为之后也会用到在程序中直接编写约束的情况。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/29.jpg"></div>

添加Source约束后会在for循环左大括号的下一行添加#pragma HLS UNROLL，这也是我为什么目前更喜欢添加到源文件，因为学习起来更加直观一些，添加到directive文件的话测试移植等则更加方便。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/30.jpg"></div>

再次执行C仿真，可以得到结果如下，应该是很明显有效率的提升，那我们该怎么更方便的对比呢？Vitis HLS提供了多个Solution仿真报告的比较功能。

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/31.jpg"></div>

点击Project->Compare Reports。

<div><img height=300 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/32.jpg"></div>

将左侧的可选报告双击或Add到右边，上下顺序在对比报告中会对应左右顺序，可以在这里调整。

<div><img height=350 src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/33.jpg"></div>

生成的对比报告可以更清楚的区分和对比各个solution的性能，因为它把运行时间、Latency、资源消耗都分开展示，所以这个功能也可以用来分析单个报告，可以更直观一些。

性能对比：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/34.jpg"></div>

资源使用量对比：

<div><img src="https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/fpga/4/35.jpg"></div>

可以看到性能提升很明显，同时LUT的使用量也变多了，是用资源换取性能的操作。
