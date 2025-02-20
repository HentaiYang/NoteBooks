本笔记为《Ceph设计原理与实现》书籍阅读笔记，主要记录Ceph中关键的技术细节，大部分内容基于2017年1月发布的Kraken稳定版。

---

# 导论
ceph:OpenStack默认存储后端
ceph是软件定义存储，几乎所有linux和类UNIX都能运行，2016年ceph从x86移植到ARM

ceph为分布式架构，可以管理上千节点、PB以上存储的大规模集群，扁平寻址使客户端可以直接和任意服务端节点通信。

ceph是统一存储系统，支持块、文件存储协议（SAN、NAS...），也支持对象存储（S3、Swift...）

ceph具备高可扩展性、高可靠性和高性能。

Ceph基于C++开发，大量采用STL和Boost库中的高级特性。

Ceph整体架构如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241120201517.png)

RADOS是ceph的支撑组件，除了ceph当前三大核心应用组件RBD、RGW和CephFS之外（分别对应块、对象和文件访问接口），基于RADOS和其派生librados标准库可以开发任意类型其他组件。

---

# 笔记导读

笔记的顺序如下：

1.CRUSH，Ceph两大核心设计之一，具有计算寻址、高并发和动态数据均衡、可定制的副本策略等特性。

2.BlueStore，于Jewel版本开始取代FileStore作为文件存储底层。

3.纠删码存储池，传统的三副本会占用大量空间，而纠删码将采用任意k+m备份策略消耗的额外空间控制在1倍以内。

4.PG，Ceph最核心和最复杂的概念之一，最重要的是它可以在OSD之间自由进行迁移

5.QoS，服务质量，对集群资源进行合理统筹。

6.RBD，新型分布式块存储服务组件，其先于最开始的CephFS，伴随OpenStack进入公众视野。

7.RGW，对象存储接口，以S3协议为代表，通过互联网（HTTP）进行传输，顺应云计算技术而生。

8.CephFS，文件存储接口，文件系统是计算机中最基本、经典的概念之一，其采用树状结构管理数据、基于查表进行寻址，与Ceph的扁平方式数据管理、计算寻址的理念不同，而支持文件系统又需要Ceph引入元数据管理服务器，与去中心化、横向扩展的理念冲突，使得CephFS一直未能有所建树。

9.Ceph应用，通过生产环境中的各种案例，将理论应用于实践。





---

# 第一章 计算为王——基于可扩展哈希的受控副本分布策略CRUSH

大型分布式存储系统中，数据写入后很少会再次移动，这就导致随着时间推移，新设备的加入，老设备的退出，会使系统数据分布趋于不均衡。

如果数据的**粒度足够小并随机分布**，最终是可以趋于均衡的。一般用**哈希函数**可以做到，但存在两个问题：

	1.系统中存储设备数量发生变化时，如何最小化数据迁移量，以尽快平衡系统
	2.数据包含多个备份，更高的数据可靠性需要更合理的备份分布。

因此，Ceph需要**扩展哈希函数**，称为**CRUSH(Controlled Replication Under Scalable Hashing)**，这是一种基于哈希的数据分布算法。

CRUSH通过计算寻址，支持多种备份策略（镜像、RAID、纠删码等），并能控制多个备份映射到集群不同区域的设备上。这使得CRUSH适用于对**可扩展性、性能及可靠性有极高要求**的大型分布式存储系统。

# 1.straw

## 1.1.straw的诞生

Ceph最开始用于管理大型分级存储网络，不同层级有不同容灾程度，被称为**容灾域（安全域）**。一个典型的Ceph集群层级结构如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241111173501.png)

每个主机包含多个磁盘，每个机架包含多个主机，一个机架就是一个容灾域。高可靠性需要数据的多个副本放在不同机架的磁盘上。因此，**CRUSH首先得是一种基于层级的深度优先遍历算法**。

一般而言层数越高变化的可能性就越小，例如一个集群大多数情况只会对应一个数据中心，而主机或磁盘则可能会经常变化。因此，**CRUSH还应该能针对不同层级设置不同选择算法**。

Sage Weil最初设计了4种基本选择算法：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241111185623.png)

综合考虑爆炸式增长的存储需求（添加元素），和大规模集群中的常态故障（删除元素），以及可靠性需求（副本存在更高级别容灾域，例如不同数据中心），在实际环境中除straw算法外的3种算法形同虚设。

---

## 1.2.straw算法
首先再确定一遍，straw算法的作用是在多个中心、机架、主机上，为一个数据选择一个磁盘，其有如下需求：

	公平性：让系统的数据趋于平衡；
	可靠性：让数据副本尽可能存储到不同容灾域；
	灵活性：保证添加和删除设备后算法仍可用。

顾名思义，straw算法将所有元素比作稻草（从一把稻草中选出最长的，我不懂书里为啥具象成吸管），针对一个输入，**为每个元素随机计算一个长度，然后选择最长的元素作为输出**，这个过程也被称为**抽签（draw）**，对应元素的长度称为**签长**。

CRUSH还引入了权重，权重越大（容量大），分担的数据越多，我们倾向于让权重越大的元素有更大的签长。

因此，straw算法的结果取决于三个因素：**固定输入、元素编号和元素权重**，其中元素编号起随机种子的作用。所以针对固定输入，straw算法实际只受元素权重影响，而每个元素的签长只和自身权重相关时，straw算法的添加和删除元素都是最优的，下面以添加为例

**以下用“磁盘”代指“元素”，“数据”代指“输入”，也可能用原文叫法，更容易理解一些，请注意，原文使用的是元素和输入，说明这个算法的通用性。**

	1) 假设集群包含n个磁盘:(e1,e2,...,en)
	2) 向集合中添加1个磁盘en+1:(e1,e2,...en,en+1)
	3) 此时要重新平衡所有磁盘的所有数据，对于任意一个数据x，对e1-en有签长d1-dn，假定其中的最大值为dmax，在添加en+1前，每个数据都有dmax
	4) 因为签长只和自身编号和自身权重相关，因此计算dn+1不受现有磁盘的影响
	5) 因此对于每个数据x，只需要比较dmax和dn+1，如果dn+1较大，则更新dmax，x会被重新映射到磁盘en+1

这样，straw算法会**随机将原有磁盘中的数据重新映射到新加入的磁盘**；同理在**删除磁盘时**，straw算法会**将该磁盘中的数据随机映射到其他磁盘中**，其他磁盘中的数据的dmax肯定大于数据对被删磁盘计算的签长，所以不用变动。

照这样看，**无论是添加en+1还是删除en+1，都不会引起e1-en之间任两个磁盘的数据交换**。然而在straw算法诞生8年后，社区还是发现了添加或删除OSD会引起不相关的数据迁移。

原straw算法伪代码如下：

```python
max_x = -1
max_item = -1
for each item:
    x = hash(input, r)
    x = x * item_straw
    if x > max_x:
        max_x = x
        max_item = item
return max_item
```

可见，算法计算的签长取决于输入(input)、随机因子(r)和权重item_straw，item_straw通过权重计算获得：

```python
reverse = rearrange all weights in reverse order
straw = -1
weight_diff_prev_total = 0
for each item:
    item_straw = straw * 0x10000
    weight_diff_prev = (reverse[current_item] - reverse[prev_item]) * items_remain
    weight_diff_prev_total += weight_diff_prev
    weight_diff_next = (reverse[next_item] - reverse[current_item]) * items_remain
    scale = weight_diff_prev_total / (weight_diff_prev_total + weight_diff_next)
    straw *= pow(1/scale, 1/items_remain)
```

按照伪代码中的逻辑，其首先根据weights（容量）反向排序，然后逐个计算磁盘的权重，计算过程中前面磁盘的权重对当前磁盘权重有很大影响，因此前面提到的**“签长只和自身编号和自身权重相关”**就不成立了，添加或删除一个磁盘en+1后，每个数据对磁盘e1-en的签长d1-dn都会变化，就产生了不相干的数据迁移。

（这么一看每次添加/删除元素之后，x都会重新计算对每个元素的签长？没看过源代码，现在有存dmax吗？不存的原因是啥？）

---

## 1.3.straw2算法

出于兼容性考虑，Sage引入了一种新的算法straw2对原有算法进行修正，这一算法计算签长时仅使用元素自身的权重，同时计算也更加简单：

```python
max_x = -1
max_item = -1
for each item:
    x = hash(input, r)
    x = ln(x / 65536) / weight
    if x > max_x:
        max_x = x
        max_item = item
return max_item
```

上述逻辑中，对输入和随机因子哈希后，结果落在0-65535，对其除以65536必定小于1，对其取自然对数ln后结果为负，再进一步除以自身权重后，权重越大则结果越大（负得越少）。

---

# 2.CRUSH算法

CRUSH算法基于权重将数据映射到所有存储设备中，过程受控且高度依赖于集群的拓扑描述——**cluster map**，不同的**数据分布策略**通过不同的**placement rule**实现。

**placement rule是一组包括最大副本数、容灾级别等的自定义约束**，如上一节中的集群架构图，我们可以自定义将互为镜像的3个数据副本写入位于不同机架的磁盘上（Ceph的默认备份策略）。

CRUSH使用**输入x、cluster map和placement rule**，输出一个包含n个不同存储对象的集合。因为自定义约束通常不会变化，所以**只要cluster map不变，CRUSH的结果就是确定的**。

---

## 2.1.Cluster Map

Cluster Map是Ceph集群拓扑结构的逻辑描述形式，通常用树来实现——叶子是device(存储设备)；中间节点为bucket(主机、机架等)；根节点为root，是整个集群的入口。节点有唯一ID、类型、权重，只有device拥有非负ID，表明其用于承载数据，非叶子节点的权重是其所有孩子节点的权重之和。

以下是Cluster Map中常见的节点（层级）类型

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241111205343.png)

层级类型的数量和名称都可以修改和裁剪，假设所有磁盘权重（规格）一致，可以给出第一节中集群的部分Cluster Map描述如下（图太长了，下半部分很容易推出来）：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241111205835.png)



---



## 2.2.Placement Rule

每条Placement Rule可以包含多个操作，共三种类型：

**take**：从cluster map选取指定bucket，并作为后续步骤的输入，默认策略选取的是root节点

**select**：从bucket中随机选择指定类型和数量的条目(items)，对于不同备份策略——多副本和纠删码，有两种select——firstn和indep。如果item故障、过载或者与已选中item冲突，都会触发select重新执行，因此需要设置最大尝试次数。

	两种算法都是深度优先，不同的是纠删码要求结果有序
	对于数量无法满足的情况（例如需要4个item）
	firstn会返回[1,2,4]这种结果
	indep会返回[1,2,CRUSH_ITEM_NONE,4]
	indep会保证数量正确，不存在的item用空穴填充。

**emit**：输出最终选择结果给上级调用者并返回。

可见最重要的是select操作。select也支持容灾域模式，算法会保证返回的叶子设备位于不同的、指定类型的容灾域下。因此，在容灾域模式下，一条最简单的placement rule可以只包含3个操作：

```
take(root)
select(replicas, type)
emit(void)
```

type为容灾域类型，例如rack，则选择结果都会位于不同机架；如果是host，则保证结果位于不同主机。

---

## 2.3.select流程

以firstn为例的select流程如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241111211955.png)

·**（1）bucket选取下一个item**

构建cluster map时会根据bucket特点指定选择算法（如straw2），算法结果取决于输入标识符x和随机因子r，因此如果选择失败则需要调整r。

为了防止死循环，需要限制select的**全局尝试次数(choose_total_tries)**；同时容灾域模式下会产生递归调用，为避免尝试次数指数型增长，采用布尔变量**chooseleaf_descend_once**控制，为真时，下一级调用至多重试一次，反之则由调用者重试。可以用**当前重试次数（或2^(N-1)倍，N=chooseleaf_vary_r）**对递归调用的r进行调整，这样递归调用的初始r取决于待选择副本编号和传入的随机因子(**parent_r**)。

老的CRUSH实现中有如下机制，因为大量人工干预破坏了CRUSH的公平性，因此**在Ceph的第一个正式发行版就被移除**：CRUSH在中间bucket因为冲突选择失败时，可以在当前bucket重试，对应局部重试次数(choose_local_retries)，对于这种情况还有一个备用选择算法，其重排bucket所有item，并不断排除选择过的条目，最终保证选中一个不冲突的item，也可以限制该算法重试次数(choose_local_fallback_retries)。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241113093430.png)

**（2）冲突**

冲突指选中的条目已经存在于输出条目列表中。

**（3）OSD过载（或失效）**

虽然CRUSH理论上能保证磁盘间的数据平衡，但实际上：

	集群规模小、整体容量有限时，集群PG总数有限，CRUSH输入样本容量不够。
	CRUSH存在缺陷：straw2每次都计算选中单个item的独立概率，但容灾域需要计算多个副本的条件概率，因此CRUSH无法处理多副本下的均匀分布问题。

这导致Ceph集群，特别是异构集群中，出现大量数据分布悬殊的情况，因此需要对CRUSH结果进行人工调整。调整会对每个磁盘（OSD）设置额外权重reweight，算法选中OSD后，再基于reweight对该OSD进行一次过载测试，失败则拒绝选择。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241113094456.png)

根据测试过程可见，**reweight越高（越接近0x10000)，选中概率就越高，直到100%可选**。在实际应用中，降低过载OSD、增加空闲OSD的reweight都可以触发数据的重新分布，初始所有OSD的reweight都是0x10000。

这样的另一个好处是能**区分OSD的暂时失效和永久删除**：

**暂时失效**：例如**需要拔下一块磁盘**（被拔出一定时间后Ceph将其设置为out），可以**调整reweight为0**将其从候选条目中淘汰，从而将其中数据迁移到其他OSD，当**插回该磁盘时，再将reweight调整为0x10000**就可以迁回属于该OSD的数据；

**永久删除**：删除OSD会删除其条目，即便该设备被**重新添加，编号也会不同，承载的数据也会变化**。

---

# 3.调制CRUSH

Ceph中，涉及客户端进行数据寻址的场景都需要基于CRUSH计算，可以通过上一节表1-3的参数**调整CRUSH**（虽然里面只有choose_total_tries可以调整），花费尽可能小的代价。为使参数正常生效，**客户端和服务器的Ceph版本应严格一致**。

调制CRUSH最简单的办法就是用现成的、有预先验证的模板(profile)：

```bash
ceph osd crush tunables {profile}
```

一些系统预先定义的模板如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241113100305.png)

---

## 3.1.编辑CRUSH Map

为了将CRUSH作为一个整体进行管理（因为内核客户端也需要用到），Ceph将cluster map和所有的placement rule合并成一张CRUSH map，可以独立实施数据备份及分布策略。一般**通过CLI即可方便的在线修改CRUSH**，也可以直接编辑CRUSH map：

**（1）获取CRUSH map**

集群创建后一般会自动生成CRUSH map，可以通过如下命令获取：

```bash
ceph osd getcrushmap -o {compiled-crushmap-filename}
```

该命令将输出CRUSH map到指定文件。

出于测试或其他目的，也可以手动创建CRUSH map：

```bash
crushtool -o {compiled-crushmap-filename} --build --num_osds N layer1 ...
```

其中，--num_osds N layer1 ...将N个OSD从0开始编号，并在指定层级中平均分布，每个layer采用<name,algorithm,size>（size指每种类型bucket下条目个数）的三元组进行描述，并按照从低（靠近叶子结点）到高（靠近根节点）的顺序排列，如下命令生成如2.1.小节中的集群CRUSH map：

```bash
crushtool -o mycrushmap -buile --numosds 27 host straw2 3 rack straw2 3 root unifrom 0
```

上面输出的CRUSH map都需要反编译才能被正常编辑。

**（2）反编译CRUSH map**

执行命令：

```bash
crushtool -d {compiled-crushmap-filename} -o {decompiled-crushmap-filename}
```

例如：

```bash
crushtool -d mycrushmap -o mycrushmap.txt
```

**（3）编辑CRUSH map**

反编译后，可以直接以文本形式打开和编辑CRUSH map：

```bash
vi mycrushmap.txt

# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable straw_calc_version 1
```

也可以编辑placement rule

```bash
#rules
rule replicated_ruleset {
	ruleset 0
	type replicated
	min_size 1
	max_size 10
	step take root
	step chooseleaf firstn 0 type host
	step emit
}
```

placement rule中各个参数含义如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241113162623.png)

因此上述规则指示“采用多副本数据备份策略，副本必须位于不同主机的磁盘之上”。

**（4）编译CRUSH map**

修改后的CRUSH map需要编译才能被Ceph识别：

```bash
crushtool -c {decompiled-crushmap-filename} -o {compiled-crushmap-filename}
```

**（5）模拟测试**

在新的CRUSH map生效之前，可以先进行模拟测试，以验证对应的修改是否符合预期，例如如下命令打印输入范围为[0,9]、副本数为3、采用编号为0的ruleset的映射结果：

```bash
crushtool -i mycrushmap --test --min-x 0 --max-x 9 --num-rep 3 --ruleset 0 --show_mappings
```

打印结果：

```bash
CRUSH rule 0x0 [19,11,3]
CRUSH rule 0x1 [15,7,21]
CRUSH rule 0x2 [26,5,14]
CRUSH rule 0x3 [8,25,13]
CRUSH rule 0x4 [5,13,21]
CRUSH rule 0x5 [7,25,16]
CRUSH rule 0x6 [17,25,8]
CRUSH rule 0x7 [13,4,25]
CRUSH rule 0x8 [18,5,15]
CRUSH rule 0x9 [26,3,16]
```

也可以仅统计结果分布概况（输入为[0,100000]）：

```bash
crushtool -i mycrushmap --test --min-x 0 --max-x 100000 --num-rep 3 --ruleset 0 --show_utilization
```

打印结果为每个设备实际存储数据量和理想情况数据量：

```bash
device 0:		stored : 11243	expected : 11111.2
device 1:		stored : 11064	expected : 11111.2
...
```

**（6）注入集群**

新的CRUSH map验证充分后，用如下命令将其注入集群：

```bash
ceph osd setcrushmap -i {compiled-crushmap-filename}
```

---

## 3.2.定制CRUSH规则

如何灵活定制CRUSH规则以满足特定需求？以第一张图中的集群为例。

最常见的场景是需要提升容灾域，默认为host，即存在不同主机上，可以提升为rack，即副本存储在不同机架上，修改或新建ruleset：

```bash
vi mycrushmap.txt
# rules
rule replicated_ruleset {
	ruleset 0
	type replicated
	min_size 1
	max_size 10
	step take root
	step chooseleaf firstn 0 type rack
	step emit
}
```

测试结果：

```bash
crushtool -i mycrushmap --test --min-x 0 --max-x 9 --num-rep 3 --ruleset 0 --show_mappings
CRUSH rule 0x0 [19,15,3]
CRUSH rule 0x1 [15,2,18]
CRUSH rule 0x2 [26,5,14]
CRUSH rule 0x3 [8,20,13]
CRUSH rule 0x4 [5,13,19]
CRUSH rule 0x5 [7,25,10]
CRUSH rule 0x6 [17,25,5]
CRUSH rule 0x7 [13,4,18]
CRUSH rule 0x8 [18,5,11]
CRUSH rule 0x9 [26,1,16]
```

可见此时3个副本都存在不同机架的磁盘上。

也可以限制只选择某个特定机架下的OSD：

```bash
rule replicated_ruleset {
	ruleset 0
	type replicated
	min_size 1
	max_size 10
	step take rack2
	step chooseleaf firstn 0 type host
	step emit
}
```

测试结果：

```bash
crushtool -i mycrushmap --test --min-x 0 --max-x 9 --num-rep 3 --ruleset 0 --show_mappings
CRUSH rule 0x0 [19,21,26]
CRUSH rule 0x1 [20,23,26]
CRUSH rule 0x2 [26,20,22]
CRUSH rule 0x3 [22,25,18]
CRUSH rule 0x4 [21,26,18]
CRUSH rule 0x5 [21,25,19]
CRUSH rule 0x6 [19,25,23]
CRUSH rule 0x7 [21,18,25]
CRUSH rule 0x8 [18,24,21]
```

可见所有副本都被限制在rack2[18,26]的范围内的OSD上。

此外，在CRUSH map中，除了OSD（叶子结点）外的层级关系都是虚拟的，这提供了更大的灵活性，例如下面的极端例子中，我们新建一个虚拟的主机virtualhost，配置OSD编号0、9、18到这个虚拟主机上，并限制所有副本必须分布在这个主机上：

```bash
vi mycrushmap.txt
host virtualhost {
	id -14			# do not change unnecessarily
	# weight 3.000
	alg straw2
	hash 0  # rjenkins1
	item osd.0	weight 1.000
	item osd.9	weight 1.000
	item osd.18	weight 1.000
}
# rules
rule customized_ruleset {
	ruleset 1
	type replicated
	min_size 1
	max_size 10
	step take virtualhost
	step choose leaf firstn 0 type osd
	step emit
}
```

实际测试结果：

```bash
crushtool -i mycrushmap --test --min-x 0 --max-x 9 --num-rep 1 --ruleset 1 --show_mappings
CRUSH rule 0x0 [0]
CRUSH rule 0x1 [9]
CRUSH rule 0x2 [9]
CRUSH rule 0x3 [0]
CRUSH rule 0x4 [18]
CRUSH rule 0x5 [18]
CRUSH rule 0x6 [18]
CRUSH rule 0x7 [18]
CRUSH rule 0x8 [18]
CRUSH rule 0x9 [9]
```

---

## 3.3.数据重平衡

我们调整每个OSD的reweight可以触发PG在OSD之间的迁移，以恢复数据平衡。这一操作可以逐个OSD也可以批量进行。

首先查看集群空间利用率统计：

```bash
ceph osd df tree
```

对空间利用率较高的OSD逐个执行：

```bash
ceph osd reweight {osd_numeric_id} {reweight}
```

上述两个参数的含义：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241113170404.png)

批量调整有2种模式：

	reweight-by-utilization：按照OSD当前空间利用率
	reweight-by-pg：按照PG在OSD之间的分布

为防止影响前端，可以先测试执行命令后将触发PG迁移数量的统计，以下用test-reweight-by-utilization为例：

```
ceph osd test-reweight-by-utilization {overload} {max_change}\{max_osdx} {--no-increasing}
```

下表是上述命令的参数含义：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241113171625.png)

例如：

```bash
ceph osd test-reweight-by-utilization 105 .2 4 --no-increasing
no change
move 197 / 11016 (1.78831%)
avg 344.25
stddev 90.4592 -> 92.0923 (expected baseline 18.2618)
min osd.11 with 61 -> 60 pgs (0.177197 -> 0.174292 * mean)
max osd.31 with 424 -> 379 pgs (1.23166 -> 1.10094 * mean)

oload 105
max_change 0.2
max_change_osds 4
average 0.156612
overload 0.164443
osd.20 weight 1.000000 -> 0.844574
osd.27 weight 1.000000 -> 0.869125
osd.31 weight 1.000000 -> 0.890121
osd.13 weight 1.000000 -> 0.895248
```

由结果可知，真正执行本命令将导致：

	197个PG发生迁移
	每个OSD承载的平均PG数目为344.25，本次调整后方差从90.4592变为92.0923
	负载最轻的为osd.11，61个PG，调整后将承载60个PG
	负载最重的为osd.31，承载424个PG，调整后承载379个PG
	四个OSD的reweight进行了调整

以下命令将确认执行调整（就是把test-去掉）：

```bash
ceph osd reweight-by-utilization 105 .2 4 --no-increasing
```

---

# 4.总结

CRUSH最为人诟病之处在其引入扩展的副本策略支持之后导致的数据不均衡问题，Ceph支持可能具有复杂的层级拓扑结构使得这个问题难以交给CRUSH自身解决，只能依托于用户进行人工干预。

同时，社区中不断有反映异构集群中CRUSH选不出足够副本数，对CRUSH调参来解决困难且复杂，而且会牺牲计算性能。直到书籍撰写的2017年1月的Ceph版本仍有这个问题，我之后会查看这几年的主要更新日志有关CRUSH的更新，如果有解决，我将更新这篇笔记。

---

# 第二章 性能之巅——新型对象存储引擎BlueStore

BlueStore最早在Jewel版本引入，以替代传统的FileStore，其具备如下特性：

	SSD适配：BlueStore充分考虑了下一代全SSD和全NVMe SSD闪存阵列的适配，例如将LevelDB替换为RocksDB，对SSD存储做了大量优化。
	
	裸设备管理：FileStore需要通过本地文件系统间接管理磁盘，而BlueStore绕过文件系统，自身接管裸设备，直接进行对象操作，大大缩短I/O路径。
	
	元数据分离：元数据的索引效率对性能有着致命影响，BlueStore允许元数据单独采用高速固态设备存储，从而提高性能。
	
	位图管理：相比传统机械磁盘的扇区，SSD普遍采用4K或更大的块大小，这既可以使位图大小大大缩小，而且相较段索引方式碎片化程度根底、效率更高。

---

# 1.BlueStore设计理念

## 1.1.数据可靠性

存储系统中，**所有读操作都是同步的**，除非命中缓存，否则都要从磁盘中读取到指定内容后才能返回到前端。而**写操作**一般都会预先在内存中缓存，再**批量写入磁盘**，因此理论上写到缓存后就可以返回，但内存掉电会丢失数据，这样无法保证**数据可靠性**。

一种可行的方案是将数据先写入高性能、掉电数据不丢失的中间设备，数据落盘后再释放中间设备空间，称为**写日志**，中间设备被称为**日志设备**，一般用NVRAM或SSD等设备。这样即便掉电也可以**通过日志重放进行恢复**，可以在不影响数据可靠性的基础上加速写操作，这样的存储系统也叫**日志型存储系统**。

这样会**消耗额外的硬件资源**，但因为日志空间可重复使用，所以不需要很大容量，多个磁盘共享一个日志设备也是可行的。除此之外，由于同一份数据先后在日志设备和磁盘写了两次，造成了**“双写”**问题，当日志设备和普通磁盘合一时，这一问题尤其严重。

---

## 1.2.数据一致性

数据一致性，即数据的修改操作要么全部完成，要么没有任何变化（All or nothing），不能是任何中间状态。符合这一语义的存储系统也称为**事务型存储系统**，其所有修改操作符合事务(Transaction)的ACID语义：原子性(Atomictiy)、一致性(Consistency)、隔离性(Isolation)、持久性(Durability)。

---

## 1.3.BlueStore面临的问题

BlueStore可以理解为事务型的本地日志对象存储系统，保证数据可靠性和一致性的同时，要尽可能减小“双写”带来的影响，提升写性能。当前**全闪存阵列普遍使用普通SSD当数据盘，日志设备由NVRAM或NVMe SSD等高速设备充当**。全闪存阵列的主要开销不是寻址时间，而是数据传输时间，因此写入量超过一定规模时，日志盘的引入不再具有明显优势：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241117172205.png)

引入增量日志可以对这个问题进行改进，针对大范围覆盖写，只在前后非块大小对齐的部分使用日志，其他部分直接执行重定向写（分配新区域写入新值，再释放旧区域），首先对相关专业术语进行介绍：

	block-size(块大小)：对磁盘操作的最小粒度（原子粒度），例如普通机械盘是512字节，SSD普遍为4096(4K)字节。
	
	RMW(Read-Modify-Write)：改写已有内容时，如果改写内容不足一个块大小，则需要先读块，再合并修改内容，最后写入原先位置。RMW带来了额外的读操作，并且对原内容的覆盖写如果掉电，会有数据损坏风险。
	
	COW(Copy-On-Write)：当覆盖写发生时，COW会分配一块新空间存放新写入的内容，也称为写时重定向。新空间写入结束后，即可释放原数据所在空间。

COW可以解决RMW带来的两个问题，但同样存在缺陷：

**1.破坏数据的物理连续性。**多次COW之后，前端的大范围顺序读在后端都会变成随机读，这对于机械磁盘是灾难性的。但因为SSD的顺序和随机读写没有数量级差异，并且COW也有将随机写转化为顺序写的能力，所以对写性能有一定补偿。

**2.小块覆盖写的低性能**。对于非块大小对齐的小块覆盖写（例如覆盖写一块4096字节中的567-1456字节）。首先执行COW后**原来的块仍有有效内容**，不能全部释放，之后的范围读取则需要**至少两次读操作，再Merge**拼凑出完成数据，大大降低性能；其次COW涉及**空间重分配和地址指针重定向**，引入更多元数据，**元数据过多则缓存命中率下降**，性能也受影响。

---

## 1.4.BlueStore的设计策略

BlueStore针对**写操作综合运用了RMW和COW策略**，将任何写请求根据块大小分为3部分，即首尾非块大小对齐部分和中间块大小对齐部分，例如：

	写入地址：0x1234，写入总大小：20000字节，块大小：0x1000(4096字节)
	第1部分（首部部分）：0x1234 - 0x1FFF
	第2部分（中间部分）：0x2000 - 0x5FFF（4个块）
	第3部分（尾部部分）：0x6000 - 0x6053

对于**中间部分使用COW策略**，对于**首尾部分使用RMW策略**，**对RMW部分引入日志**，避免数据损坏的隐患。

BlueStore对上层提供了读写两种类型的多线程访问接口，都是基于PG粒度。PG内部使用读写锁，以实现**读请求并发、写请求（泛指修改操作）排他**。并且因为读请求是同步的，而写请求一般为异步，所以对每个PG设计OpSequencer队列，**对所有操作该PG的写请求进行保序**。不同ObjectStore对OpSequencer的实现不同，BlueStore的实现主要包含**2个FIFO队列**，分别对2个阶段的写请求保序。

如前所述，BlueStore将**所有写请求划分为普通和带日志**两种。带日志的写请求会先后执行写日志-覆盖写数据，覆盖写数据使用独立线程池，所以需要**一个额外队列将覆盖写数据阶段的带日志写请求再次排序**。所有写请求通过ObjectStore定义的标准queue_transactions接口提交至BlueStore处理，该接口提交多个事务为一个事务组，整体符合ACID语义。**当前受限于PG实现，一个事务组并不能针对同一个PG的多个对象操作。**

通过BlueStore的接口来操作PG下的某个对象，首先要找到对应的PG上下文和对象上下文，对应其元数据信息，并且BlueStore通过kvDB固化这两类上下文至磁盘，所以我们首先要了解这两类上下文的磁盘数据结构（On-Disk Format）。

---

# 2.磁盘数据结构

BlueStore因为要管理裸设备，其磁盘数据结构极其复杂，除了PG和对象管理结构，还需要大量用于组织磁盘数据的结构。每种数据结构一般都有磁盘和内存两种格式：

	磁盘格式：以"_t"结尾，全部小写
	内存格式：不以"_t"结尾，只有首字母大写。

此外，元数据被设计成可以和用户数据分开存放，统一以键值对的形式存放在kvDB中。

---

## 2.1.PG

### 2.1.1.pool和PG

Ceph对集群所有存储资源池化管理。**资源池（pool，也称存储池）**是一个虚拟概念，表示**一组约束条件**。例如针对一个pool制定CRUSH规则，以限制存储资源或容灾域等；或针对不同pool制定副本策略，例如对时延敏感的采用多副本备份，不重要的数据则采用纠删码备份；甚至分别为pool指定独立scrub、压缩、校验策略等。

Ceph将**前端数据都抽象为对象**，每个对象可以生成**全局唯一的OID(Object ID)**，形成一个**扁平的寻址空间**。

为了实现不同pool之间的策略隔离，Ceph引入了**中间结构PG**，实现**两级映射**。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241117202746.png)

**两级映射均使用伪随机哈希函数**，以保证均匀性。**第一级为静态映射**，将**任何前端类型的应用数据**按照固定大小切割、编号后，作为哈希输入**映射到PG**；**第二级实现PG到OSD的映射**，将PGID和集群拓扑作为哈希输入，并使用CRUSH进行调整，进而实现可靠性、自动平衡等（第二级都是**第一章中讲到的**）。最终，**pool以PG作为基本单位进行组织**，其实际是**一些对象的逻辑载体**。

---

### 2.1.2.PGID的生成

集群的pool由Monitor统一管理，**每个pool也有pool-id**（为保证pool-id的全局唯一性，Monitor也要求实现分布式一致性，目前采用Paxos实现），因此**PG只需要分配一个pool内唯一的编号，然后和pool-id结合**即可保证PGID的全局唯一性，例如pool-id为1，该pool指定了256个PG，则PGID为1.0, 1.1, ..., 1.255形式，这样就**维持了扁平寻址空间**。

Ceph通过C/S(Client/Server)模式实现外部应用和存储集群的数据交互，客户段访问集群时，**需要由特定类型的client和操作对象名**（例如RBD应用，形如rbd_data.12d72ae8944a.00000000000002a7j的字符串)，**计算出32位哈希值**，**取模后根据归属的pool就能找到对象所在PG及PGID**。假定pool-id为1的某个对象计算哈希值为0x4979FA12，PG数量为256，则有：

```
0x4979FA12 mod 256 = 18
```

可知该对象由编号为18的PG负责承载，PGID为1.18。

将对象映射置PG一般不会用到全部32位哈希值（这需要有2^32个PG），就上面的例子而言，我们很容易验证pool内的如下对象同样会被映射到PGID为1.18的PG上：

```
0x4979FB12 mod 256 = 18
0x4979FC12 mod 256 = 18
0x4979FD12 mod 256 = 18
...
```

我们仅使用了32位哈希值的后8位来确定PG编号，并且**PG编号等于低8位的数值**，即0x12=18。因此pool内的PG数目为2^n时（例如256=2^8），我们可以直接使用**低n比特掩码**(2^n-1)进行高速的**按位与运算**，在本例中就是0xFF，即255：

```
0x4979FA12 & 0xFF = 0x12 = 18
```

相反，**PG数目不能被2整除**时，我们假定其具有n位（例如PG数目为12=0b1100，n=4），此时进行按位与运算时，**会将某些对象映射到实际并不存在的PG**，例如0x112E & 0xF = 0xE = 14，共12个PG却得到PG编号14。

我们假定**PG数目cnt由n位二进制表示**，则可以确定必然有**cnt>=2^(n-1)**，例如对于cnt=12，则2^(n-1)=2^3=8，对2^(n-1)相与的结果必定在pool中。

但如果我们无脑相与，就会浪费[2^(n-1), cnt-1]的PG，那么我们就退而求其次，保证相对每个PG而言，映射到该PG下的所有对象的低n-1比特都是相同的，此映射方法称为stable_mod：

```c++
if ((hash & (2^n - 1)) < pg_num)
    return (hash & (2^n - 1));
else
    return (hash & (2^(n-1) - 1));
```

这样就将原本**与运算得到的不存在的PG**进行了**重新映射**，比如PG数量为12的pool中(n=4)，就将低4位数值为12-15的对象映射到了[0,7]，具体而言，将12(0b1100)映射到了4(0b0100), ... , 15(0b1111)映射到了7(0b0111)：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241117212947.png)

注意上图**仅对[12,15]进行了重定向，[8,11]并没有变化**（如果对于每个对象都和2^(n-1)-1相与，那么[8,11]的确会被重定向到[0,4]），从结果来讲等同于将空穴平移来进行重定向。因此无论PG数目能否被2整除，stable_mod都可以产生一个**最好的、相对稳定的结果**，这是**PG分裂的重要理论基础**。很明显的，该例子中[4,7]承受了2倍于其他PG的数据，我们假设n=8，对PG总数进行更换，可以得到：

```
极大情况：cnt=255, 实际范围[0,254] 掩码1映射范围[0,255] 掩码2将[255,255]重定向到[127,127]
极小情况：cnt=129，实际范围[0,128] 掩码1映射范围[0,255] 掩码2将[129,255]重定向到[1,127]
中间情况：cnt=192，实际范围[0,191] 掩码1映射范围[0,255] 掩码2将[192,255]重定向到[64,127]
```

可见，PG数目越偏向于2^n-1或2^n，则承载一倍数据和两倍数据的PG数量之差越大。在两端时，1个PG承载一倍/两倍数据，cnt-1个PG承载两倍/一倍数据；在最中间时，一半的PG承载一倍数据，另一半承载两倍数据。总体而言，**PG总数越接近2的整数倍，stable_mod越稳定**。

---

### 2.1.3.PG分裂

爆炸式的数据增长使扩容成为常态，Ceph的高可扩展性就是针对这一点。集群扩容后，负载需要在所有OSD之间重新均衡——让新OSD立即参与负荷分担，这一特性**通过PG自动迁移和重平衡实现**。

Ceph提供了一种手动添加PG数目的机制，因为stable_mod输入之一的**PG数目发生变化**，所以部分**老PG中的对象会重新映射到新PG中**，因此需要**同步转移这些对象**（很熟悉？第一章的CRUSH是PG-OSD的映射算法，处理的是OSD的数量变化；而stable_mod是对象-PG的映射算法，处理的是PG的数量变化）。这一过程称为**PG分裂**，其步骤如下：

**1）**Monitor检测到pool中PG数目发生修改，发起并完成信息同步，随后将包含变更信息的**新OSDMap推送至相关OSD**。

**2）**OSD对比新老OSDMap，如果**某个pool的PG数目发生变化**（Ceph目前只支持添加PG数目），则需要**执行分裂**。

**3）**假定pool(（id为1）中**原PG数目为2^4、新的PG数目为2^6**，以某个老PG为例，很明显分裂后**只需要关注哈希值的低6位**，容易验证该PG中已有对象根据低6位可以分为这4个类型：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241118192757.png)

对这四种类型的对象执行stable mod，得到如下结果：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241118192903.png)

可见添加PG后，**仅有第一种类型仍留在老PG上，并且需要创建编号分别为1/2/3 * 16 + X的新PG**。对所有老PG执行该过程后，最终会将存储池中的PG数目调整为原来的4倍，即2^6，而因为这一过程总是基于老的PG（祖先PG）产生新的孩子PG，并且孩子PG最初的对象全部源自祖先PG，所以称为PG分裂。

FileStore实现中PG对应一个目录，其下对象用文件保存，并处于索引效率和PG分裂的考虑，对目录下文件分层管理。采用stable_mod执行对象-PG的映射后，因为一个PG下的对象可以保证**哈希值低(n-1)位相同**，那么很明显**逆序的哈希值可以作为目录分层的依据**，假定某个对象的哈希值为0xA4CEE0D2，则一个可能的目录层次为./2/D/0/E/E/C/4/A/0xA4CEE0D2。再假定一个**PGID为Y.D2的PG归属的pool分裂前有256个PG**，则某时刻一个可能的目录为：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241118195043.png)

如果分裂为4096个PG，则每个祖先PG产生15个孩子PG，PGID为Y.D2的PG产生的孩子PG的PGID为：

```
Y.1D2
Y.2D2
...
Y.ED2
Y.FD2
```

可知需要迁移到新PG的对象分别存储在：

```
./2/D/1/
./2/D/2/
...
./2/D/E/
./2/D/F/
```

因此，我们**不需要实际移动对象**（毕竟所有PG都在同一块OSD上），**直接修改目录的归属就可以完成底层的对象转移**。

那么就存在一个新问题，新增的孩子PG的PGID都不相同，**仍使用PGID作为CRUSH输入时，将引发大量新增PG在OSD之间的迁移**（老PG不会迁移，第一章讲过CRUSH的结果只和元素本身相关）。但**分裂之前**PG在OSD之间的分布就已经**趋于均衡**（是我们手动添加导致的PG分裂），所以我们希望**分裂后让祖先和其孩子PG保持相同分布**。

为此，每个存储池除了记录当前的PG数目外，还要记录**分裂之前祖先PG的个数**，称为**PGP数目(pgp_num)**。则利用CRUSH将PG分布到不同安全域的OSD上时，我们不直接使用PGID，而是**使用其在pool内的唯一编号对pgp_num执行stable_mod，再和pool-id一起哈希之后作为CRUSH的特征输入(x)**，这样就能保证孩子PG和其祖先有相同的CRUSH计算结果（本例中即仍使用4位掩码，后续再次分裂时则使用6位掩码，此时才会触发本该在本次分裂引发的PG在OSD之间的迁移）。

新的BlueStore实现中，因为同一OSD下所有对象共享一个扁平的寻址空间，所以PG分裂时只修改目录归属不会影响后续性能，因而更加高效。PG对应的磁盘数据结构为bluestore_cnode_t（后续简称cnode），其定义如下，目前仅包含一个字段，以指示stable_mod执行时PG的掩码位数。

---

## 2.2.对象

### 2.2.1.逻辑段

Bluestore中的对象非常类似于文件，每个对象都有BlueStore实例内唯一的编号、独立大小、从0开始逻辑编址、支持扩展属性等，对象的**组织形式**也类似文件，**基于逻辑段（extent）**进行的。

每个extent都可以写成{offset, length, data}的三元组形式：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119101321.png)

data用于从磁盘上索引对应逻辑段的数据。磁盘空间碎片化严重时，逻辑上连续的段不一定能分配物理上也连续的空间，所以data主要包含一些物理空间段的集合，每个段对应磁盘的一块独立存储空间，称为**bluestore_pextent_t**（简称pextent），其成员表如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119101912.png)

pextent与物理设备相关，因此offset和length必须是块大小对齐。如果extent中的offset不是块大小对齐的，对齐约束会使extent的逻辑起始地址和对应物理起始地址产生偏移，同样extent的length不是块大小对齐时，结束地址之间也会产生偏移，这两个偏移会大大增加数据处理难度。

---

### 2.2.2.逻辑段扩展功能

extent是对象内的基本数据管理单元，因此很多扩展功能都基于extent粒度实现：

**（1）数据校验**

计算机组件都可能产生**静默数据错误(Silent Data Corruption)**，顾名思义，就是不能被计算机组件自身察觉的错误，其潜在危害性极大。单个组件静默数据错误出现概率在1e-7左右，极容易被忽略，然而因为每个组件都可能发生，长时间海量数据场景下的**静默数据错误的发生几乎是必然的**。

解决静默数据错误的主要手段是**校验和**，原始数据作为输入得到校验和，将校验和与数据分开存储，读取时分别读取数据与校验和，并重新计算校验和，如果相等则认为数据正确，反之则出错。BlueStore**直接用kvDB保存校验和，ACID特性保证其可靠性**，因此**校验和不一致我们可以断定是原始数据出错**。

良好的检验算法应当有极低的冲突概率（不同输入得到相同输出的概率），并且同一种算法，输出长度越长，冲突概率要低得多。但更低的冲突概率一般对应更复杂的计算过程，我们要**在冲突概率和执行效率之间权衡**。

当前主流的校验算法有**CRC、xxhash**等（BlueStore默认都支持），可以根据配置项灵活选择。

**（2）数据压缩**

数据压缩**将原始数据转换为长度更短的输出**，以节省空间，其过程必然可逆，即**将输出作为输入时可以得到原始数据，称为数据解压缩**。常见的压缩算法大多基于模式匹配，一般其迭代次数决定了压缩收益，因此我们同样需要在**压缩收益和执行效率之间权衡**。

数据压缩有两个关键信息——一是选用的压缩算法，而是压缩后的数据长度。BlueStore用**压缩头bluestore_compression_header_t**表示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119112534.png)

压缩头和压缩后的数据一起保存，并一起计算压缩收益，即压缩率。当**压缩率小于某个阈值**（目前为0.875，即要求至少压缩掉1/8的数据），此时压缩净收益太小，**BlueStore会拒绝压缩**。

引入数据压缩会影响不完全覆盖写，假定我们要对一个压缩后的extent执行不完全覆盖写，一般而言有两种方案：**一是读出数据、解压缩，然后正常覆盖写**；**二是直接执行重定向写**，即重新分配一个extent，并允许其和原extent有部分重合的内容（因为是不完全覆盖）。**方案一严重影响了写效率**；**方案二即浪费了空间又增加了元数据，影响了读效率**。并且新extent还可以执行压缩，多次不完全覆盖写之后，部分内容可能被多个extent重复保存，进一步浪费空间且降低性能。

目前**BlueStore基于方案二实现数据压缩策略**，但这远非一种完美的压缩策略，所以**默认关闭**。

**（3）数据共享**

指extent在不同对象间的共享，一般由**对象克隆操作**产生共享。某个extent的数据被多个extent共享时，需要**记录共享的起始地址、数据长度和共享次数**，形如**{offset, length, refs}**的三元组，BlueStore称为**bluestore_shared_blob_t**，主要是一张基于extent的引用计数表，其与对象分开**单独存储**。

extent之间还有一种特殊的共享方式——**存储空间共享**。BlueStore可以自定义最小可分配空间，**一般为块大小**，但块大小越大位图越小，索引速度更快，也**可以将其提升为块大小的整数倍**。例如块大小4K、最小可分配空间16K，此时**即便写入1字节，也要分配16K空间**，且**只有一个4K的块被使用**，存在被共享的可能。

---

### 2.2.3.extent数据结构

以上，我们讨论了extent三元组的**data抽象数据**目前期望支持的特性，可以定义其磁盘数据结构为**bluestore_blob_t（简称blob）**，如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119150020.png)

对应的**extent磁盘数据结构**如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119152933.png)

extent除blob之外的成员表示**逻辑段抽象数据，其与物理段（blob）之间的关系**如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119153140.png)

---

### 2.2.4.extent map

一个对象下所有extent组成一个extent map，以索引对象下所有数据，其中的extent可以不连续（前一个的逻辑结束地址要小于后一个的逻辑起始地址），因此**extent map中可能存在空穴**。extent过多或磁盘碎片化都可能导致extent map臃肿，影响kvDB效率，所以要**对过大的extent map分片**，并将信息单独保存，称为shard_info：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119154439.png)

分片编码后的长度不需要很精确，实现中总是基于上一次的分片信息预估当前extent map中每个extent编码后的平均大小，再以此计算分片逻辑地址。因为不是很精确，所以分片可能需要进行多次。

此外，如果某个blob被两个相邻的extent共享，这两个extent又恰好被分片分开，那么将优先对这个blob执行分裂操作，如果分裂失败（如blob被压缩、设置了FLAG_SHARED等），则将该blob加入spanning_blob_map，后续将整个spanning_blob_map和对象的其他元数据（例如onode）一并作为单条记录进行保存。

分片还可以实现对extent的按需加载，读取所需分片并解码还原所有extent到extent map；同样在修改时，也只需更新分片。这样减少了元数据产生，也减少了操作使用的元数据。

综上我们定义extent map的磁盘数据结构如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119165653.png)

至此，我们分析了一个对象的所有关键元素。

---

### 2.2.5.对象数据结构

BlueStore中对象的磁盘数据结构为bluestore_onode_t，简称onode。

BlueStore的对象和上层的对象并不相同。**BlueStore原则上只需要通过一个唯一索引和上层对象建立联系**即可，比如OID，但**随着上层中命名空间、快照等特定信息的追加，影响了OID的全局唯一性**（如快照会使两个对象拥有相同OID，但有不同快照ID），因此**额外信息也需要在上层对象-onode的映射中考虑到**。

BlueStore将对象名转义，并作为**kvDB表的唯一索引**（对象元数据存储在kvDB）。转义过程**一是要尽可能减少转义后的长度，二是要区分对象名的定长和非定长部分**——OID、命名空间等长度不定，哈希、快照ID等为定长整数。转义过程如下：

	1.所有定长整型转为等宽字符串。例如32位哈希0x30313233编码后为"1234"（不能将0x00作为kvDB的默认字符串结束符）。
	
	2.所有变长字符串进行转义。将任意小于等于'#'的字符转换为长度固定为3、形如"#XY"的字符串；任意大于等于'~'的字符转换为长度3、"~XY"的字符串；转换后的字符串用'!'作为结束符（'!'小于'#'，因此原始字符串不可能出现'!'字符）。
	
	3.拼接1和2得到的字符串。

onode包含四个部分：**数据、扩展属性、omap头部和omap条目**。数据和扩展属性和文件相关概念类似；omap类似扩展属性且无限制，可以和扩展属性有相同key但有不同内容。可以认为**扩展属性用于保存少量小型属性对**，**omap用于保存大量大型属性对**。因此扩展属性和onode一并保存，omap则分开保存，下表为kvDB中需要存储的onode相关条目：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119173456.png)

其中extent map的索引策略考虑了局部性原理——加载onode后，大概率继续加载extent map，因此onode和对应extent map相邻存储可以提升性能。

onode的磁盘数据结构bluestore_onode_t如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119173905.png)

---

# 3.缓存管理

## 3.1.常见的缓存淘汰算法

缓存最重要的问题就是有限的缓存空间被填满后，新的缓存页面该替换哪个页面？这一算法称为缓存淘汰（替换）算法，常见的算法有两种：

**（1）LRU（Least Recently Used）**

LRU算法淘汰最长时间未使用的页面，基于时间局部性。如果请求队列有很好的时间局部特性，则LRU是最优算法。此外LRU也容易实现。

**（2）LFU（Least Frequently Used）**

LFU算法淘汰最低访问频率的页面，基于空间局部性。但很显然的是，访问频率低可能代表页面刚刚加入；访问频率高可能代表页面基本用完。因此LFU算法一般需要兼顾频率的时效性。

**（3）ARC（Adaptive Replacement Cache）**

LRU和LFU算法基于局部性原理，并不通用。基于此诞生了ARC算法，其使用两个队列对缓存页面进行管理：**MRU（Most Recently Used）队列保留最近访问过的页面；MFU（Most Frequently Used）保留最近至少被访问两次的页面**。

ARC算法的**关键在于两个队列长度可变，会根据请求序列的特征自动调整**：时间局部性明显时，MFU队列长度为0，ARC退化为LRU；空间局部性明显时，MRU低劣长度为0，ARC退化为LFU。其**缺点在于维护队列众多**（MRU、MFU及对应的影子队列，共4个）、**执行效率低**，在时延敏感系统——例如数据库系统中并不特别适用。

**（4）2Q**

基于LRU/2（本质是LFU算法）发展而来，是一种针对**数据库**，特别是**关系型数据库系统**优化的淘汰算法。数据库经常存在多个事务操作同一块热点数据的场景，因此算法主要聚焦**多并发事务的数据相关性**。

2Q使用队列A1in、A1out和Am管理缓存，这些队列都是**LRU队列**，**A1in和Am是真正的缓存队列**，**A1out则是影子队列**，保存相关页面的管理结构而不保存真实数据。算法流程如下：

	1.新页面总是先加入A1in，对A1in的命中2Q认为访问相关，不增加热度，页面被淘汰时加入到A1out；
	
	2.A1out中某个页面被命中时，2Q认为访问不相关，将其加入Am队列头部提升热度；
	
	3.Am队列中页面再次被命中时，同样将其转转移到Am队列头部提升热度；
	
	4.Am队列中淘汰的页面也进入A1out。

2Q用A1in队列识别热点数据——如果一个页面在短时间内被连续索引，将被视作相关，这个时间段被称为**“相关索引间隔”（Correlated Reference Period）**，取决于**A1in队列容量**。**A1out的容量**决定了一个页面累计的访问热度能持续多长时间，称为**“热度保留间隔”（Retained Information Period）**。2Q实现中，这**两个容量的调整基于缓存容量自动进行**，其实现极其高效，适合时延敏感的系统。

---

## 3.2.BlueStore中的缓存管理

BlueStore使用了LRU和2Q算法。其中2Q的A1in和Am队列容量配比推荐为1:1，A1out队列容量则推荐为A1in和Am队列容量之和，这样配置较为通用。为减少锁碰撞，BlueStore会实例化多个缓存（不同PG请求可以并发处理，因此OSD会设置多个PG工作队列，缓存实例数与之对应）。

缓存实例都基于Cache基类，然后派生出TwoQCache类和LRUCache类，LRUCache比较简单且可视为TwoQCache的一种特例，故不展开分析。

---

### 3.2.1.Cache类

Cache的基本成员如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119181901.png)

Cache可以缓存用户数据或元数据，而**BlueStore使用2Q作为默认缓存类型**，因此应**尽量用来缓存元数据**（缓存元数据的比重受bluestore_cache_meta_ratio控制，默认为0.9，即90%的cache用来缓存元数据）。

BlueStore大的元数据类型有两种：**Collection和Onode**。**Collection**为PG的内存管理结构，因为体积小巧且数量有限（Ceph推荐每个OSD承载约100个PG）被设计成**常驻内存**。Onode数量庞大，需要有淘汰机制。因此**Cache设计主要面向用户数据和Onode**。

类似其他元数据（如OSDMap），**Onode直接采用LRU队列管理**（所以**2Q只管理数据而不管理元数据**，这有违引入2Q的初衷）。理论上将所有Onode放在单独的LRU队列中效率最高，但每个**Onode唯一归属特定的Collection**，需要一个中间结构建立这一联系，方便Collection级的操作。中间结构称为**OnodeSpace**，其关键数据成员如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119183729.png)

同样的，对**用户数据也要建立和上层管理结构的对应关系**，Onode中Extent是管理用户数据的基本单元，**Blob则执行用户数据到磁盘空间的映射**，因此基于Blob引入中间结构——**BufferSpace**，建立**Blob到缓存的二级索引**，其关键成员如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119184141.png)

BufferSpace管理的基本单元是Buffer，一个**Buffer管理Blob的一段数据**（不一定干净）。其关键成员如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119184308.png)

Buffer的状态转换比较简单，如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119184521.png)

因为Blob是用户数据的基本单位，所以**对缓存的操作一般基于BufferSpace的粒度进行**，BufferSpace中的关键方法如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241119184645.png)

BlueStore的2Q实现中，针对Onode使用LRU，针对Buffer才是真正的2Q，TwoQCache的管理结构如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241120135732.png)

虽然TwoQCache内部**对Onode和Buffer分开管理**，但**缓存空间是共享的**，因此需要指定一个合适的分割比例。实际测试中，因为用户数据的缓存和操作系统以及磁盘级的缓存有重叠，所以**也有提议全部用作Onode缓存**的，实际效果有所验证。除了这些通用缓存，还有一些特殊目的引入的缓存，我们之后再介绍。

---

# 4.磁盘空间管理

## 4.1.常见磁盘空间管理模式

块是磁盘操作数据的最小单位，每个块任意时刻只能是“空闲”或“占用”两种状态之一，因此**一个比特就可以表示一个块**。我们将磁盘空间按照块进行切分、编号，一位表示一块，那么**所有位会形成一个有序比特流**，可以以块为粒度索引磁盘中任意空间，这种管理方式叫做**位图**。

位图大小和磁盘空间成正比，从管理效率考量，位图必须全部装入内存。早期计算机因为磁盘容量有限（位图远小于内存空间）可以用位图管理，但**一个存储系统的内存和磁盘比例相当悬殊**——如VMAXe最大支持内存和磁盘容量分别为512GB和1.3PB（1:2662），整个系统需要300GB以上的位图，显然这样的位图作为整体的效率极其低下。

一个改进策略是**段式磁盘空间管理**，每个段包含**offset和length**，指示起始地址和长度，假定offset和length是64位无符号整数，那么单个段占用16字节空间，管理空间为[0, 16EB]，对于不同的管理空间**可能取得极高收益，也可能开销远大于位图**。段管理较灵活，适用于**申请范围比较宽泛的场景**。

磁盘管理的索引效率一般和**内存组织形式**有关，如位图的顺序排列和二叉树排列，数据量足够大时效率相差几个数量级。此外，上层应用的需求各异，不同的空间分配策略也会大相径庭。例如段管理，常见有3种空间分配策略：

	首次拟合法（First Fit）：查找所有段中第一个满足所需空间大小的段，进行分配后返回。
	
	最佳拟合法（Best Fit）：查找所有段中和所需空间大小最接近的段，进行分配后返回。
	
	最差拟合法（Worst Fit）：查找所有段中空间最大的段，进行分配后返回。

一般而言，**最佳拟合法适合请求空间较为广泛的系统**，使系统中所有段的空间相差甚远。相反，**最差拟合法适合请求空间范围较窄的系统**，可以使系统中所有段的空间趋于一致。而**首次拟合法较为随机**，适用场景介于两者之间，常见于**事先无法预测请求空间范围的通用系统**。最佳拟合法和最差拟合法都需要对段进行排序，而首次拟合法按照段的起始地址自然排序（为应对空间碎片化，会将空闲的、物理上连续的段合并为大段），因此**首次拟合法效率最高**。

**Ceph因为面向分布式设计**，每个节点的磁盘空间管理相对独立，且主要面向SSD等块较大的磁盘，所以**默认采用位图**（也支持段管理，可以切换）。磁盘管理分为**空闲空间列表和已分配空间列表**两部分，但显然这两部分强相关，磁盘上任意块同一时刻只会存在于两个列表的其中一个， **从一张列表可以推出另一张列表**。

因此，BlueStore考虑每个Onode已经详细记录了存放数据对应的物理空间，并且便于合并段为大段的前提下，选择**将空闲空间列表存盘**。这样系统上电时，加载空闲空间列表就可以在内存中还原出已分配空间列表。

BlueStore中，**FreelistManager组件负责管理空闲空间列表**，**Allocator组件负责管理已分配空间列表**，两个组件都有段和位图的实现，我们只介绍BlueStore默认的位图实现，对应**BitmapFreelistManager和BitmapAllocator**。

---

## 4.2.BitmapFreelistManager

空闲空间管理以块为粒度，将连续的固定数量块组成段，从而将磁盘划分为多个段进行管理，段的起始地址编号是BlueStore内唯一的索引，因而**可以使用kvDB固化所有段信息**。

每个段的值是一个长度固定的比特流，置位为分配，反之为空闲，状态的切换可以适用对应位置1的掩码，然后进行异或操作，例如一个长度为8的段：

	原状态: 00100111
	下标为0,1的块切换为已分配状态，6,7切换为空闲状态
	掩码: 11000011
	运算过程: 00100111 ^ 11000011 = 11100100

此外因为使用kvDB存储段信息，所以需要合理调整段大小（过大则值长度变大，过小则键值对数目增多），以充分发挥kvDB性能。

BitmapFreelistManager定义的公共接口如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241120164531.png)

---

## 4.3.BitmapAllocator

BitmapAllocator是一个块粒度的内存版本磁盘空间分配器，因为不需要存盘，所以可以非扁平组织，以提升索引效率。实现中块是以树状形式组织的：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241120165904.png)

**BitmapEntry：**BitmapAllocator的**最小可操作单位**，例如定义为64位无符号整数，则可以记录64个相邻块的状态。

**BitmapZone：**由一定数量的BitmapEntry组成，是**单次最大可分配单位**，大小可配置，拥有独立的锁逻辑，并且所有API都是原子的，因此不同BitmapZone的操作可以并发。

**BitmapArea：**BitmapAllocator管理空间过大时，可以进一步引入中间结构BitmapArea，其叶子节点称为BitmapAreaLeaf，一个叶子包含固定数量的BitmapZone。

**BitmapAreaInternal：**由多个BitmapAreaLeaf组成，多个BitmapAreaInternal中间节点可以再次作为上一层BitmapAreaInternal的孩子节点，最终归属于根节点BitmapAllocator。

整个BitmapAllocator的拓扑结构都可配置，主要受两个参数影响：

	BitmapZone大小：决定单次可分配的最大连续空间，和并发操作的粒度。
	BitmapArea单个（中间或叶子）节点的跨度（span-size）：决定了BitmapArea的高度，进而决定了索引效率。

BitmapAllocator允许**自定义最小可分配空间**，即**分配的最小粒度**，也必须配置为**块大小的整数倍**。实现中总以最小可分配空间为单位进行分配，**每找到一个最小可分配空间视为完成一次分配**，直到满足空间需求，**物理上相邻的空间**会在以段形式返回给上层应用时**自动进行合并**。

**默认情况下最小可分配空间和块大小相同**，此时空间分配实际是在逐位扫描，这样实际只在**空间碎片化程度高的时候有效**，反之如果**碎片化程度低并且有大量大空间需求时，则效率非常低**。一个**改进方向**是分配空间时不是找到最小可分配空间即完成一次分配，而是**找到尽可能多、同时为最小可分配空间整数倍的连续空间才算完成一次分配**。

BitmapAllocator定义的公共接口如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241120173919.png)

因为BlueStore将不同类型的数据严格区分并允许用不同设备存储，所以一个BlueStore实例中可能存在多个BitmapAllocator实例。

---

# 5.BlueFS

## 5.1.RocksDB与BlueFS

### 5.1.1.RocksDB

2011年诞生的**LevelDB基于Google的BitTable数据库系统**，是**键值对类型的日志型非关系数据库**，为存储**海量键值对**设计。理论上LevelDB的键和值只受存储容量限制，可以为任意长度的字节流，并严格按照键排序。LevelDB继承并发展了**LSM-Tree**和**SSTable**（Sorted String Table，对所有修改重定向，即从不覆盖写），将sst在磁盘上分级存储。

但LevelDB的单线程压缩（Compaction）以及用mmap将SSTable读入内存等做法无法发挥SSD的性能。因此基于LevelDB诞生了RocksDB，为新型高性能固态存储提供卓越的键值对类型数据库访问性能。下表展示了二者的性能对比。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241120194458.png)

RocksDB具有如下特性：

* 专为使用**本地闪存存储**、**数据库容量不超过几TB**的应用程序设计，是一种非**内嵌的分布式数据库**。

* 适合存储**小型/中型键值对**——**性能随键值对长度上升快速下降**。

* 性能随CPU核数及后端存储的I/O能力呈线性扩展。

* 支持列簇（Column Families）、备份和还原点、Merge操作符等。

RocksDB采用C++开发，设计灵活，**几乎所有组件都可替换**，例如固化SSTable和WAL（Write Ahead Log，日志）的本地文件系统。**BlueStore默认采用RocksDB**取代LevelDB作为**元数据存储引擎**。同时，因为多数本地文件系统（如XFS、ext4、ZFS等）的**很多功能非必须**，需要裁剪，所以为RocksDB定制一个本地文件系统解决的更加彻底，BlueFS应运而生。

---

### 5.1.2.BlueFS

BlueFS是个简易的用户态日志型文件系统，其实现了RocksDB::Env的所有接口，用于固化RocksDB运行过程中产生的.sst（SSTable）和.log（WAL）文件。WAL对RocksDB的性能至关重要，所以BlueFS支持将.sst和.log分开存储，方便用更高速的设备存储。

引入BlueFS后，BlueStore将所有存储空间从逻辑上分为三个层次：

**（1）慢速（Slow）空间**

主要存储**对象数据**，可由普通大容量机械磁盘提供，由**BlueStore自行管理**。

**（2）高速（DB）空间**

存储BlueStore元数据（如onode），可由普通SSD提供，容量需求比（1）小。BlueStore的元数据交给RocksDB管理，而RocksDB最终通过BlueFS将数据存盘，因此由BlueFS直接管理。

**（3）超高速（WAL）空间**

主要用于存储RocksDB产生的.log文件，可由更高性能SSD充当，容量需求和（2）相当（也取决于RocksDB的配置）。也由BlueFS直接管理。

高速、超高速空间需求不是固定的，如果这两个空间规划的较为保守，BlueFS也允许用慢速空间进行数据转存。因此，**BlueStore会共享一部分慢速空间给BlueFS**，并**实时监控和动态调整**，具体策略为：BlueStore用一个周期性唤醒的同步线程查询BlueFS的可用空间，如果**BlueFS的可用空间**在整个BlueStore的可用空间中**占比过小**，则**分配一定量空间到BlueFS**（空间不足设定最小值时，追加到最小值），**反之则从BlueFS中回收一部分空间到自身**。

综上，如果3类空间分别用不同设备，则一般情况下BlueFS中有3种可用空间。

	.log文件及BlueFS日志（BlueFS也是日志型本地文件系统）：优先用WAL空间，其次用DB空间，均不存在/空间不足则用Slow空间。
	
	.sst文件：优先用DB空间，不存在/空间不足则用Slow空间。

**Slow类型空间均由BlueStore管理**，分配给BlueFS的段用**bluefs_extents**管理，并从Allocator中扣除。bluefs_extents中的每个元素对应Slow设备的一个段，每次**更新后会作为BlueStore的元数据存盘**。这样BlueFS上电时，通过预加载bluefs_extents，就可以正确初始化对应的Allocator。DB设备和WAL设备由BlueFS负责初始化，因此**BlueFS上电时通常会初始化3个Allocator实例**。

虽然BlueStore已经将所有已分配空间信息保存在每个Onode中，但Onode数目可能十分庞大，为了加速上电过程，**BlueStore会额外固化一张空闲空间列表**，但实际这两类信息是重复的。

BlueFS则不同，其存储规模十分有限，一般为BlueStore的千分之一到百分之一；另一方面RocksDB生成的.sst文件大小固定且从不修改。因此**BlueFS**不保存任何额外的分配信息，而是**在上电时遍历所有文件元数据，来生成Allocator**。

---

## 5.2.磁盘数据结构

BlueFS是日志型文件系统，磁盘数据主要包括文件、目录和日志三种类型。

层级（树状）结构组织目录和文件的数据格式较复杂，而且在小规模数据时效率低于扁平结构。BlueFS因为只用于存储单个BlueStore（对应一块磁盘）的元数据，文件规格较统一（绝大多数为sst）且数量有限，所以**采用扁平结构组织**。

BlueFS使用两类表来追踪所有文件及目录层级关系：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121160858.png)

BlueFS定位一个文件**共需两次查找**：第一次通过dir_map找到文件所在最底层目录；第二次通过目录的file_map找到对应文件。**dir_map中描述的都是文件的绝对路径，条目之间没有隶属关系**。

每个文件采用类似inode的结构管理，称为bluefs_fnode_t（简称fnode）。file_map建立的是文件名和fnode之间的映射关系：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121161214.png)

其中，因为文件可能来自多个不同块设备（WAL,DB,Slow）的空间，因此extents中的单个**extent还需记录归属的块设备标识**（其他和bluestore_pextent_t相同），用数据结构bluefs_extent_t表示，如下表：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121161644.png)

BlueFS的**所有修改操作都基于日志**，每个修改操作会生成一个独立的日志事务，再通过**flush_log()批量固化日志**。例如mkdir()会生成如下日志，然后向上层返回操作成功：

```
op_code: OP_DIR_CREATE
op_data: dir(string)
```

BlueFS采用**增量日志**，日志膨胀会浪费空间，并且初始化速度慢，需要定期进行**日志压缩**。将最新dir_map和所有file_map通过一个独立日志事务存盘，通过这个日志即可还原完整的dir_map和file_map，亦即可以作为后续增量的基准，在基准日志之前的日志都可以删除（通过序列号比较）。

早期日志压缩的**完全同步造成了写停顿问题**（write stalls），改进后**仅在生成内存基准日志事务时严格同步**（通过BlueFS全局的排他锁），而其他过程异步，大大改善了写停顿。综上，我们定义**日志事务的磁盘数据结构bluefs_transaction_t**如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121163636.png)

上电时，我们总是通过**日志重放**来得到**BlueFS的所有元数据**，需要一个索引日志位置的固定入口，BlueFS总是将其写到自身接管的**DB设备的第二个4K存储空间**，结构为**bluefs_super_t**，如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121164612.png)

BlueFS提供的API与传统文件类似，不再赘述。我们可以得到数据从OSD到落盘的路径如下图所示，其中BlueFS->Block Device除元数据外还包括日志数据：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241120201718.png)

---

## 5.3.块设备

BlueStore管理的Slow、DB和WAL空间可以分别使用不同类型的块设备（Block Device）提供。Linux中块设备被当做文件管理，访问通过驱动程序实现，且只能以块为粒度进行。

块设备的传输速率取决于制造工艺和使用的**总线（传输）标准**，例如SATA 3.0理论上限为6Gbps，PCIe 3.0X4上限则高达32Gbps。SATA总线对应的**AHCI接口专为机械硬盘设计**，不适合SSD。**NVMe（Non-Volatile Memory express）**与AHCI类似，是**逻辑设备接口标准**，基于PCIe的NVMe SSD有着数十倍以上的IOPS，时延则降低几十倍。

SPDK（Storage Performance Development Kit）是高性能、可扩展、用户态的存储类应用程序的工具套，它将必需的驱动程序移动到用户态，用主动轮询替代中断来避免内核开销。SPDK自带NVMe驱动，目前BlueStore**基于SPDK实现了对NVMe块设备的支持**，后续计划增加NVRAM的支持，以提供更高的性能。

---

# 6.具体实现

## 6.1.mkfs

mkfs负责**固化用户指定的配置项到磁盘**，后续**BlueStore上电时直接从磁盘读取**，不会再受配置文件改变的影响（每个BlueStore实例的配置项都可以不同）。因为不同配置下BlueStore的磁盘组织形式不同，如果两次上电有不同配置项可能导致永久性损坏。mkfs中需要写入的**典型配置项**如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121192003.png)

---

## 6.2.mount

mount在**OSD进程上电**时完成**正常上电前的检查和准备工作**，处理逻辑如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121192252.png)

其主要包含以下步骤：

**（1）校验ObjectStore类型**

校验ObjectStore和磁盘实际使用的ObjectStore类型相同，防止将数据写坏。

**（2）fsck或deep-fsck**

fsck扫描并校验BlueStore实例中的**所有元数据**，deep-fsck会**进一步校验所有对象数据**。这一操作可以通过Ceph的工具选择在业务空闲、或例行维护时手动进行。mount的**fsck选项默认关闭**。

**（3）加载并锁定fsid**

**fsid唯一标识一个BlueStore实例**，锁定fsid是为了防止该磁盘被多个OSD进程的BlueStore实例同时打开和访问，影响数据一致性。

**（4）加载主块设备**

用作存储对象数据，一般是大容量慢速磁盘，由BlueStore直接管理。

**（5）加载数据库，读取元数据**

mount过程从数据库读取的主要元数据如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121193218.png)

**（6）加载Collection**

如前所述，Collection数量有限，所以可以常驻内存。

**（7）其他处理**

随后，mount**创建工作线程**（如同步线程，执行回调的finisher线程，监控线程等），如果上次下电不是优雅下电，那么**可能需要执行日志重放**来恢复数据。最后会**设置校验算法、压缩算法等全局参数**后，mount全部完成，上层应用可以正常读写BlueStore当中的数据。

---

## 6.3.read

读取对象指定范围内的数据，目前BlueStore实现的read接口是同步的，其处理逻辑如下：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121193751.png)

read逻辑比较简单，主要涉及查找Collection、查找Onode、读缓存和读磁盘4个步骤：

**（1）查找Collection**

Collection数量有限且全在内存里，耗时相对后续几乎可以忽略。如果Collection存在，则获取其读写锁的读锁，即针对同一个Collection的读操作可以并发。

**（2）查找Onode**

Onode使用LRU队列管理，如果LRU没有命中，则需要基于其磁盘格式在内存中重建。Onode的管理结构如下图所示，每个Onode包含一张ExtentMap，ExtentMap包含数个Extent，每个Extent管理一段逻辑范围并关联一个Blob，Blob通过若干个pextent将数据映射到磁盘。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121195708.png)

每个Blob包含一个**SharedBlob**，是一张**基于pextent的引用计数表**，用于实现Extent之间的**数据共享**。此外，SharedBlob还包含BufferSpace，用于缓存Blob数据并纳入Cache管理。

每个Blob有shared和!shared状态，为**shared状态**时表明Blob被多个Onode共享，此时SharedBlob**关联了有效的引用计数表**，和ExtentMap一样，这张表是按需加载的，只有需要使用时才从kvDB加载到SharedBlob，此时**SharedBlob状态变为loaded**（对应有未加载引用计数表的!loaded状态），BlueStore会同步将其**加入Collection的全局SharedBlob缓存**。

因此，如果**Onode没有在缓存命中**，为防止重建Onode的读取数据过多，**ExtentMap、SharedBlob等管理结构都是动态、根据需要进行加载的**。

**（3）读缓存**

Onode**在缓存命中**时，说明部分数据已经在全局Cache中，此时BlueStore会尝试从Cache读取，并产生**已命中数据区域**和**未命中数据区域**两张列表。

**（4）读磁盘**

**有数据未在缓存命中**时需要读磁盘，我们根据（3）中的未命中数据区域列表**加载对应的Extent到内存**，然后**读取磁盘对应位置**。BlueStore会根据配置**校验数据**（防止静默数据错误），并**解压数据**（如果数据被压缩），最后通过**拼凑和填充**（全0）的方式得到完整数据并返回。

---

## 6.4.write

### 6.4.1.queue_transactions

write在内的**所有涉及数据修改的操作**，都是**通过queue_transactions接口以事务组的形式提交到BlueStore**的。queue_transactions处理逻辑如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121201909.png)

OpSequencer在本章第1节介绍过，用于对同一个PG提交的多个事务组保序。对于**同一个事务组的多个操作**，BlueStore会创建一个**事务上下文TransContext**，并**顺序添加每个修改操作**后批量处理，添加完成后（此时修改操作对应的内存版本完成），TransContext**汇总操作数及波及字节数**，再提交给Throttle（BlueStore内部的一种流控机制）判断**是否进行写抑制**，没有过载则不需要写抑制，开始执行TransContext。

所有queue_transactions提交的事务组都是异步执行的，需要**若干类型的回调上下文**，事务组执行到特定阶段后，执行回调上下文来唤醒**上层应用执行相应操作**。常见的回调上下文有两种：

	on_readable：写日志完成应答，修改操作写入日志设备后进行的回调。
	
	on_commit：数据写入主要存储设备后进行的回调，也称on_disk或on_safe。

BlueStore的写操作只会产生少量WAL日志，并和其他元数据一并用kvDB保存，所以写日志要先于写数据完成，在对应事务组完成后，BlueStore会先执行on_commit，然后执行on_readable（出于兼容性考虑，BlueStore实际并不需要on_readable）***（存疑？这里为什么矛盾？）***

---

### 6.4.2.write

将write加入TransContext的流程如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241121205448.png)

write不操作任何已有数据称为**新写**，反之则均为**覆盖写**。

#### 6.4.2.1.新写

新写需要按照逻辑地址范围进行**MAS（最小可分配空间）对齐**，分为**头尾非MAS对齐写**和**中间MAS对齐写**。

**MAS对齐的新写**的处理逻辑如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241123110751.png)

对大量数据写入分为多个Extent主要是防止生成的校验数据过大，影响kvDB的索引效率。

**非MAS对齐的新写**和上图流程类似，区别在于：

* 每次至多需要分配一个Extent。
* Extent中blob_offset不再为0。
* 待写入数据需要块对齐，无效部分填充0，防止干净数据被污染。

#### 6.4.2.2.覆盖写

因为BlueStore对于MAS对齐部分总是执行COW，所以**MAS对齐的覆盖写**处理逻辑和新写类似，但需要找出所有波及范围已经存在的ExtentMap、Extent和对应空间，在同步完成后一并释放。

对于**头尾非MAS对齐的覆盖写**则较复杂，需要考虑如下因素：

**（1）能否直接跳过，执行COW**

常见于Extent被压缩过，此时RMW代价太大，直接采用COW。

**（2）能否直接复用已有Extent的unused块**

如果MAS大于基本块大小，那么Extent中可能产生块粒度的空穴，被标记为unused。如果写入内容不足一个unused块，则头尾无效部分填充0。

**（3）是否需要执行WAL写**

此时**RMW操作无法避免**，为避免写坏数据，需要**使用WAL**。下图展示了WAL的常见情况（假定MAS为基本块大小，如果不为基本块大小，则其头尾非基本块对齐部分处理逻辑同下）：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241124165935.png)

WAL写包括**补齐读、合并、全0填充3**个步骤，BlueStore随后基于此数据生成一个**日志事务**，并加入对应的**TransContext**。此日志事务**用kvDB保存**，写入kvDB后才能**对原区域数据执行覆盖写**，效率很低。

---

### 6.4.3.TransContext

所有的修改操作都添加到TransContext时，就可以开始处理TransContext了，其处理过程分为3阶段：

**（1）等待所有在途的写I/O完成**

主要是COW产生的I/O，将直接写入新磁盘空间，因此可以和对应事务的产生**同步执行**。如果掉电，那么**新分配空间和待释放的旧空间尚未更新到kvDB**，没有影响。

**（2）同步所有涉及的元数据修改到kvDB**

包括**已申请空间和待释放空间**（只需固化FreelistManager），通过一个事务同步提交至**kvDB**。因为kvDB的ACID属性，本阶段可以保证数据一致性。

**（3）通过WAL对应的日志事务执行覆盖写**

WAL日志事务已经写入数据库，**通过重放安全的执行覆盖写**。覆盖写完成后再**生成一个释放该WAL日志事务条目的事务**，同步提交至kvDB并等待完成，最后将所有**待释放空间加入Allocator**。本阶段为**可选**，没有WAL写时在（2）阶段完成后即释放到Allocator。

---

# 7.使用指南

## 7.1.部署BlueStore

OSD需要预留少量空间存储OSD启动的引导数据，并用本地文件系统格式化（此时后端的ObjectStore还未正常上电，无法直接访问），较大的分区则以裸设备的形式充当Slow设备。我们可以得到一个OSD模型如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241125094316.png)

部署BlueStore时，首先将主设备划分为大小两个分区，小分区（例如100MB）存放OSD的启动引导数据，同时作为OSD启动后的工作目录，用本地文件系统格式化；大分区直接作为裸的块设备，和另外两个裸块设备（或分区）通过符号链接挂载到OSD的工作目录下。全SSD或测试情况下，**可以用同一块设备的4个分区完成BlueStore的部署**。

所有设备就绪时，修改ceph.conf即可完成BlueStore部署：

```bash
[osd.0]
host = ceph-01
osd data = /var/lib/ceph/osd/ceph-0/
bluestore block path = /dev/disk/by-partlabel/osd-device-0-block
bluestore db path = /dev/disk/by-partlabel/osd-device-0-db
bluestore wal path = /dev/disk/by-partlabel/osd-device-0-wal
```

这样BlueStore上电时将在OSD工作目录下创建block、block.db、block.wal三个文件，然后通过ceph.conf通过符号链接指向分区/设备路径。

通过ceph-disk可以自动部署BlueStore，为此需要通过ceph.conf规划各个分区大小：

```bash
[global]
bluestore block db size = 67108864
bluestore block wal size = 134217728
bluestore block size = 5368709120
```

然后执行：

```bash
sudo ceph--disk prepare --bluestore /dev/sdb
```

将在sdb上创建4个分区：

```bash
sudo sgdisk -p /dev/sdb
...
Number	Start	End			Size		Code	Name
1		2048	206847		100.0 MiB	FFFF	ceph data
2		206848	337919		64.0 MiB	FFFF	ceph block.db
3		337920	600063		128MiB		FFFF	ceph block.wal
4		600064	11085823	5.0GiB		FFFF	ceph block
```

也支持用多个块设备：

```bash
sudo ceph-disk prepare --bluestore /dev/sdb --block.db /dev/sdc --block-wal /dev/sdc
```

上述命令将在sdb创建2个分区，分别为ceph的工作目录和block；block.db和block.wal则分别指向sdc的第1、2个分区。

进一步分离block.db和block.wal：

```bash
sudo ceph-disk prepare --bluestore /dev/sdb --block.db /dev/sdc1 --block.wal /dev/sdd1
```

目前的推荐容量配置为Slow:DB:WAL=100:1:1进行配置，更大的DB和WAL效果更佳。

目前（2017年）BlueStore和RocksDB仍被标记为实验性质，需要添加如下配置才能正常启用：

```bash
[global]
enable experimental unrecoverable data corrupting features = bluestore rocksdb
osd objectstore = bluestore
```

---

## 7.2.配置参数

所有和BlueStore相关的配置参数如下。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241125100516.png)

---

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241125100551.png)

---

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241125100607.png)

---

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241125100620.png)

---

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241125100645.png)

---

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241125100658.png)

---

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241125100711.png)

---

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/2/20241125100734.png)

---





# 第三章 时空博弈——纠删码原理与overwrites支持

**组件失效导致的数据丢失风险**是所有存储系统面临的共同考研，小到静默数据错误，大到主机或整个机柜异常掉电。**纠删码（Erasure Coding，EC）**是常用的数据保护机制，对原始数据**分片**并**编码**生成备份数据，最后分别写入不同存储介质。数据恢复是编码的逆运算，称为**解码**。

纠删码的**容错能力取决于备份数据的份数**，也是系统中**允许同时失效的最大存储介质数目**。通常备份数据份数越多，编码就越复杂，额外空间消耗越多。实际中**很少使用阶数（备份数据份数）高于2的纠删码**。

最简单的纠删码就是完全复制，但开销过大。RAID（Redundant Array of Independent Disk，指RAID5或RAID6），以及衍生的更具一般性的RS-RAID（Reed-Solomon RAID），以更复杂的编解码策略降低存储空间成本。

---

# 1.RAID技术概述

单个磁盘存储数据有访问速度慢、容量小、安全性差的固有缺陷，**RAID技术尝试使用多个磁盘联合提供存储服务**。

磁盘数据以扇区为基本单位，RAID抽象出一个类似的最小数据访问单位——**条带**，在多个磁盘间建立逻辑映射，抽象为一个容量更大、I/O并发能力更高的虚拟磁盘，以条带为基本单位管理，如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125102105.png)

主流的RAID有如下几种基本模式（假定所有磁盘规格相同）：

**（1）RAID0**

将多个磁盘以条带为单位重新划分，形成一个逻辑连续的虚拟磁盘，**I/O能力和可用空间是所有磁盘之和**，但**不具备容错能力**。如下图所示。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125192000.png)

**（2）RAID1**

将同一份数据**重复写入多个磁盘**，每个磁盘存储完全相同，也称为**镜像**，提供的**I/O能力和存储空间都只有1/N**。容错能力和镜像个数N成正比。下图为N=2的RAID1系统：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125193023.png)

**（3）RAID5**

以一个或多个基本块作为**条带深度均分数据**，通过**异或计算数据块大小的校验块**，I/O能力及可用空间为所有磁盘的(N-1)/N，N为磁盘个数，即条带宽度。数据出错可以通过异或恢复数据，因此RAID5具有容错能力，最多允许一块磁盘异常。校验块的位置会不停地在磁盘间跳动。下图为一个条带宽度为4、深度为4个基本块的RAID5系统。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125193614.png)

RAID5条带宽度和深度通常是固定的，但ZFS自带的**RAID5（RAIDZ）的条带宽度和深度可以根据I/O大小动态调整**，从而避免浪费和提升I/O并发能力（很明显固定的大小很容易产生空穴）。下图为RAIDZ的条带分布：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125194846.png)

**（4）RAID6**

进一步提升了容错能力，**允许同时两块磁盘故障**，代价是每个条带包含**两个校验块**，空间和I/O利用率都比RAID5要低，算法也更复杂。

上述形式中，**RAID0最快，可以与其他RAID形式组合**，常见组合有RAID10、RAID50等。

---

# 2.RS-RAID和Jerasure

RAID5基于条带将数据均匀切分为多个数据块di（i=1,2,...,n），然后尝试建立数据块之间的联系，例如用普通的加法：

```
d1+d2+...+dn=c
```

c称为基于加法生成的校验块，这样通过解方程就可以恢复数据块。但加法并不适合计算机，实际中用的是**异或运算**：首先异或是**极其高效的位运算**；其次，异或**存在逆运算且与正向运算完全相同**；最后，**异或不会产生进位**，校验块大小与数据块相同。

RAID6或更高阶的RAID原理相似，都是利用条带中n个数据块通过编码得到m个校验块（即系统允许同时m个磁盘故障），通过**解码**就可以还原所有缺失数据块。

将上述过程转为更一般的描述形式：**基于n个可变输入，构造m个等式，使m元一次方程组有唯一解**（解码时，故障磁盘数对应未知数）。通信领域的Reed-Solomon codes就是解决的这个问题，将其引入存储系统后成为**RS-RAID**，简单来说包含如下3个方面：

* 基于范德蒙德矩阵**计算校验和**
* 基于高斯消元法进行**数据恢复**
* 基于伽罗华域执行**编解码**过程中所要求的算术运算

---

## 2.1.计算校验和

假定有n个数据块d1,d2,...,dn，定义Fi为所有数据块的线性组合，则根据Fi可以计算得到校验块ci（i=1,2,...,m）：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125202619.png)

如果我们用向量D和C分别表示数据块和校验块的集合，Fi表示矩阵F中的每一行，则有FD=C。

因此F称为编码矩阵，RS-RAID使用m*n范德蒙德矩阵充当编码矩阵F，有：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125202807.png)

因此上述等式变为：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125202820.png)

---

## 2.2.数据恢复

定义矩阵A和向量E，满足：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125202942.png)

于是有AD=E成立：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125203404.png)

此时对于任意一个块，矩阵A和向量E都有一行与之对应，A被称为分布矩阵（后来也称生成矩阵）。假定某个数据块失效，则将相应行从矩阵A和向量E删除，得到：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125203609.png)

恰好有m个数据块失效时，可知A'为一个n*n矩阵，因为F是范德蒙德矩阵，所以A的任意n行都是线性独立的，即非奇异矩阵，因此A'是可逆的，所以D中所有未知数都可以通过高斯消元法求解，即所有数据块都可修复。

---

## 2.3.算术运算

上述恢复使用的高斯消元法需要使用除法运算，但计算机的除法精读有限，且乘法运算容易产生进位，增加了校验块空间。

我们定义一个整型集合：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125204220.png)

集合对应w位数据能表示的数值范围。我们定义基于该集合的加减乘除四则运算，并且在集合内封闭。我们将满足上述约束的集合称为伽罗华域，也称有限域，数学表达式为![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125204437.png)。

RS-RAID实现的加法就是异或运算，而异或的逆运算就是异或本身，因此有限域的减法运算也是异或。容易验证GF(2)={0,1}中异或运算是封闭的：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125204734.png)

为推导乘法运算的一般形式，我们先定义基于GF(2)={0,1}的多项式基本运算规则:

* 多项式系数全部来自GF(2)（只能取0或1）。
* 多项式中次数相同的项，可以基于GF(2)的加法（异或运算）进行合并。

```
假定：
r(x) = x + 1
s(x) = x
则有：
r(x) + s(x) = x + 1 + x = (1 + 1)x + 1
			=(1 ^ 1)x + 1 = 0 + 1 = 0 ^ 1 = 1
```

基于该规则，我们可以定义多项式的模运算规则如下：

```
如果r(x) = q(x)t(x) + s(x)，其中：
	s(x)和t(x)为任意多项式，且s(x)的次数小于q(x)
则：
	r(x) mod q(x) = s(x)
```

例如：

```
假定：
r(x) = x² + x
q(x) = x² + 1
所以：
r(x) mod q(x) = x + 1
```

如果多项式q(x)满足：

* 次数为w。
* 所有项的系数来自GF(2)。
* 不能被因式分解

那么可以基于q(x)生成![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125204437.png)中所有元素，过程如下：

1）前三个元素固定为0、1、x。

2）将前一个元素乘以x，然后对q(x)取模。

3）重复步骤2），直到结果为1。

满足上述条件的q(x)称为![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125204437.png)的生成多项式，记作：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125210128.png)

常见的生成多项式：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125210150.png)

针对w=4，使用q(x)=x^4+x+1生成GF(2^4)所有元素的过程如下（注意加法和减法都是异或运算，用加号表示）：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125210308.png)

为了在RS-RAID中使用![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125204437.png)，需要将上述多项式中每项系数与对应二进制对应。以w=4为例：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125210617.png)

据此可以定义乘法运算规则：

1）将乘数和被乘数的二进制（十进制）表示转换为对应的多项式表示。

2）执行多项式乘法，将得到的结果针对q(x)取模。

3）将2）得到的结果转换为对应的二进制（十进制）表示。

例如：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125210828.png)

或者根据表3-1有：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125210917.png)

由对数运算性质：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125210935.png)

可见，引入对数运算可将乘除法转化为加减法。我们用gflog和gfilog分别表示![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125204437.png)的对数运算和其逆运算，则乘除法可简化为，其中mod(2^w-1)等价于对q(x)取模：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125211053.png)

如果能预先计算所有元素的gflog和gfilog表，就可以快速执行任两个元素的乘除法运算。而![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125204437.png)中每个元素都由原生多项式x^i得来，因为：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125211314.png)

所以有：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125211343.png)

例如针对GF(2^4)，能得到：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125211412.png)

可以得到完整的gflog和gfilog表：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125211428.png)

由此我们可以方便的进行任意两个元素的乘除法运算：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125211526.png)

至此我们完整实现了四则运算，可以完成RS-RAID编解码过程中所需的全部算术运算。

---

## 2.4.缺陷与改进

James发现**分布矩阵A并不具备“删除任意m行所得到n\*n子矩阵仍然可逆”的特性**，进行了修正。修正的分布矩阵（称为分布矩阵B）总是可以通过如下的(n+m)\*n范德蒙德矩阵经过有限步的初等变换，将前n行转化为单位矩阵之后得到：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241125212004.png)

可以证明删除任意m行都是可你的，由此得到的分布矩阵B仍有此特性。

但**直接采用范德蒙德矩阵作为编码矩阵复杂度太高**，一个改进方案的思路为：**一是引入位矩阵（bit matrix）将编码的乘法运算进一步转化为异或运算**；**二是用性能更好的柯西矩阵作为编码矩阵**（可以作为编码矩阵的**柯西矩阵不唯一**），因为范德蒙德矩阵求逆运算时间复杂度为O(n³)而柯西矩阵仅为O(n²)，理论上解码可以快一个数量级。

因为柯西矩阵不唯一，**“好”的柯西矩阵和“坏”柯西矩阵导致的性能差异平均为10%，而极端情况可达83%**，如何生成一个“好”的柯西矩阵的一般性算法发表在《Optimizing Cauchy Reed-Solomon Codes for Fault-Tolerant Storage Applications》一文当中。

---

## 2.5.Jerasure

2007年，James基于上述理论给出一个**RS-RAID的开源实现**，称为**Jerasure**。

Jerasure实现的纠删码是水平方式的，即同时需要**k+m块不同的磁盘**分别承载和校验数据（数据盘和校验盘）。如果存储系统**能够容忍任意m块磁盘同时故障**，那么称此时采用的纠删码为**MDS（Maximum Distance Separable）类型的纠删码**。

Jerasure库的大部分纠删码还引入了字长w参数，即一次编码的长度，如果w=8，则每次至多完成k字节编码；w=16，则每次至多完成k*2字节编码等。部分低阶纠删码，如RAID5，因为只用到异或运算，所以可以直接用天然字长sizeof(long)作为运算的基本单位，此时每次编码的单位称为包（packet），包大小（packetsize）为字长的整数倍即可。

如果包过大，导致切分到某些盘的数据不足一个包，则用全0填充（异或的性质）。下表为Jerasure常见参数及含义：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241127161000.png)

针对业界最普遍使用的“m=2“（RAID6），Jerasure进行了两类优化：一类优化针对范德蒙德矩阵，因为矩阵中只包含1和2，所以可以对乘2运算优化；另一类则直接改造编码矩阵，使用最小密度RAID6（Minimal Density RAID6）的编码方法集，使用位矩阵，且要求非0元素尽可能少。目前支持3种最小密度RAID6：

* Liberation：要求w必须是素数。
* Blaum-Roth：要求w+1必须是素数。
* Liber8tion：要求w必须等于8。

Jerasure目前所支持的编解码方式如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241127174508.png)

---

# 3.纠删码在Ceph中的应用

Ceph的数据保护方式有多副本和纠删码两种，其中**纠删码以插件的形式**提供服务。Ceph支持的纠删码实现包括**Jerasure（默认）、ISA**（Intel STorage Acceleration，专用于x86 CPU）等。在Ceph中使用纠删码首先要指定**详细配置模板**，再与对应存储池绑定，模板参数如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241127175007.png)

上表中键值对部分需要根据纠删码的官方文档进一步进行配置，以Jerasure为例：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241127175526.png)

由此，我们可以创建一个纠删码模板：

```bash
ceph osd erasure-code-profile set my-ec-profile plugin=jerasure k=4 m=2 technique=liber8tion ruleset-failure-domain=host
```

创建了一个采用liber8tion算法（此时m必须为2）、容灾域为主机级别（k+m=6，至少有6台主机）的纠删码，可以通过如下命令查看：

```
ceph osd erasure-code-profile get my-ec-profile
k=4
m=2
packetsize=2048
plugin=jerasure
ruleset-failure-domain=host
ruleset-root=default
technique=liber8tion
w=8
```

基于该纠删码模板可以创建一个对应存储池：

```
ceph osd pool create my-ec-pool 128 erasure my-ec-profile
```

---

## 3.1.术语

**（1）块（chunk）**

纠删码的每次编码会产生若干大小相等的块（有序），Ceph通过数量相等的PG将其存到不同OSD中。序号相同的块总是由同一个PG负责存储。拼接所有块即可还原对象，通过校验块来修复损坏数据。

**（2）条带（stripe）**

对象过大时需要分多次编码，每次编码部分称为一个条带。同一对象的条带从0开始有序编号。

**（3）分片（shard）**

序号相同的块位于同一PG，它们组成对象的一个分片，分片的编号就是块序号。

三者关系如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241204144052.png)

**（4）其他**

* k：条带中数据块个数。
* m：条带中校验块个数。
* n：条带中块个数，n=k+m。
* rate：空间利用率，rate=k/n。

---

## 3.2.概述

创建存储池时，需要合理的指定PG数目，通常100个左右是推荐的选择。这一数目是指**逻辑数目**，实际中Ceph会用多个PG实例（PG实例组）来对应一个逻辑PG，负责将不同备份（多副本）或部分（纠删码）写入不同OSD。这一过程是受控的：

**1）每个逻辑PG被转化为多少个PG实例由数据保护策略决定**。多副本则等于副本个数（通常不用区分，但会设置一个主PG：Primary），纠删码则会转化为k+m个PG实例（需要通过Shard ID严格区分，且有一个首领PG：Primary Shard）。

**2）PG实例组通过CRUSH受控地分布在不同容灾域的OSD上**。如果用纠删码，则CRUSH返回的结果总是有序的（第一章介绍过，即便数量不足也要用NULL OSD填充）。Primary由CRUSH返回序列的第一个PG充当。

---

## 3.3.新写

首先介绍最简单情形的纠删码实现，即向存储池中写入一个全新的对象，我们假定一个k=3、m=2的存储池，将一个名为"NYAN"的全新对象写入该存储池：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20241204162647.png)

这是最简单的写操作，称为满条带写（Whole Stripe Write），即写入数据证号覆盖一个/多个条带。逻辑PG的切分和合并均由Primary Shard完成，其他PG不感知，即**每个PG都认为自己保存的是完整、独立的对象，逻辑地址从0起**。

---

## 3.4.读

读操作与满条带写类似——Primary Shard收到读请求，先对请求的逻辑地址范围进行条带对齐，分别读取再汇总后应答。此外，还有如下额外考虑因素：

**（1）是否进行条带对齐**

例如块大小为1KB，读取范围为[512,2560)，正常的读请求序列可能为：

```
PG(shard id = 0): [512, 1024)
PG(shard id = 1): [0, 1024)
PG(shard id = 2): [0, 512)
```

假定PG读取的最小粒度为扇区，那么**不进行条带对齐**会使首尾PG**有机会少读部分数据**。

但块大小不会很大（默认条带大小为4KB，也就是块的最大大小），经常会小于磁盘最小粒度，因此**不进行条带对齐有额外收益的情形有限**。并且PG处于降级状态或Recovery触发的读操作可能需要进行修复才能读取。因此**预先进行条带对齐是合适的**，以上面的读请求为例：

```
PG(shard id = 0): [0, 1024)
PG(shard id = 1): [0, 1024)
PG(shard id = 2): [0, 1024)
```

**（2）是否需要对校验块的位置进行调整**

校验块通常不用读取，所以可以动态调整其位置，避免浪费读带宽。但Ceph中PG实例通常不会在OSD之间迁移，因为PG的分布是随机的，每个OSD即存一些对象的数据块，也存一些对象的校验块，所以Ceph不必担心读带宽浪费。

**（3）是否仍然采用同步读**

多副本因为内容相同，且由Primary保证一致性，所以读操作直接同步在本地完成。但纠删码实现中，一个对象会分片存储在多个PG实例，从而分布在多个OSD中，导致单个读操作被分为多节点的多个读操作，所以**纠删码采用异步读**，有助于提升整体读带宽，但**纠删码的读、写过程响应时延仍然要比多副本要高，不适合时延敏感类**的应用。

**（4）是否自动进行数据修复**

纠删码最大能容忍m个块损坏，此时理论上可以在读的同时修复坏掉的块，称为**自动修复（auto-repair）**。实现上能否执行自动修复取决于是否存在**数据自校验机制**，FileStore不具备这一机制，因此默认直接将其踢出集群，BlueStore替代FileStore后，应当推出自动修复方案。

---

## 3.5.覆盖写

多副本处理新写和覆盖写没有区别，但纠删码将条带作为对象更新的最小单位，因此首尾没有进行条带对齐时，不足一个条带的部分只能通过”读取完整条带->修改数据->基于条带重新计算校验数据->写入（被修改部分和校验和）“进行，即RMW（Read Modify Write），其中R也被称为补齐读。提升覆盖写的性能通常有两种思路：减少RMW数量和减少补齐读的读取数据量。

写缓存是减少RMW的常见方法，假定块大小为1KB，条带大小为3KB：

```
write1:[0,4096)
write2:[4096,8192)
write3:[8192,12288)
...
```

不做任何处理时为2个满条带+4个RMW（浅灰1满+1RMW，中灰2RMW，深灰1RMW+1满）：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20250111171015.png)

而引入缓存时，通过合并可以减少RMW数目，如上述过程可以用4个stripe的一次满条带写完成：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/3/20250111171334.png)

纠删码的写缓存应该为对象级别，难点在于批量写入的时机。定时器驱动、阈值（水位）驱动或结合两种方法（任一条件满足则写入）是常见思路。写操作合并会提高写时延，增加数据丢失风险。

无法避免RMW时（如完全随机写），需要减少读操作，通常会预先计算需要补齐读的块，而不是每次都满条带读，例如在上图中，进行[0,2048)的覆盖写（块大小1K），对应条带[0,3072)仅需读取[2048,3072)，即第三个块即可。

---

## 3.6.日志

纠删码的写本质都是分布式写，需要用PG级别的日志来保证数据一致性，例如掉电时回滚（未发送完成应答）和前滚（已发送完成应答）日志。

多副本通过最新日志对应PG实例进行前滚即可修复损坏对象。而纠删码则要在覆盖写发生前将被覆盖内容进行备份。

---

## 3.7.Scrub

Scrub指数据扫描，通过重新计算校验和，再与原校验和进行比对，以检测静默数据错误。多副本由PG实例自行扫描，纠删码则有两种方案：

一种方案是Primary Shard生成和保存校验和，Scrub时Primary Shard读取数据并重新计算、比对。缺点是严重影响系统性能，少量修改也会触发全部读取和计算，且有大量跨节点的读流量。

另一种和多副本类似，每个PG实例保证分片的正确性。编写时暂无存储方案，因此该版本暂时禁止纠删码的Scrub。

---

# 4.总结与展望

纠删码带来了较高的存储空间收益，适用于时延不敏感的冷数据，如备份数据。

但相较于多副本，纠删码实现更复杂，且性能更差。



---

# 迁移之美——PG读写流程与状态迁移详解

PG的复杂源于定位复杂：首先，在架构上，PG位于RADOS层的中间——向上负责接收和处理客户端请求，向下负责将请求翻译到事务，是ceph核心中的核心。其次，PG是组成存储池的基本单位，很多特性依托于PG实现，例如多副本和纠删码。最后，容灾域备份策略对应的不同节点的同步、数据修复也依赖PG来完成。

PG最重要的特性是可以根据CRUSH结果在OSD之间自由迁移，这是自动数据恢复、自动数据平衡等高级特性的基础。

---

# 1.PG概述

对象大小一般是固定的，且不会设计的很大。PG是一些对象的集合——基于对象名称（和元数据）生成哈希值，对存储池的PG数目执行stable_mod后，得到对应的PGID。

以PG对对象进行二次组织有如下收益：

1）PG数目严格可控，基于PG精确控制单个OSD乃至节点的资源消耗成为可能。

2）PG数目远小于对象数目，且PG数目和PGID都相对固定，因此以PG为单位应用备份策略和数据同步、迁移等难度更小且灵活。

而PG的引入增大了数据管理粒度，保证数据强一致性极具挑战性。下表为PG常用术语和概念：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113110107.png)

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113110230.png)

---

# 2.读写流程

读写请求都以对象为基本单位。

**（1）head对象、克隆对象和snapdir对象**

**head对象即原始对象**。没有快照机制时，直接修改原始对象即可。引入快照机制之后，为支持快照回滚，可以考虑用COW预先克隆一份，但也要考虑**两个特殊场景——删除和重新创建原始对象**。

删除原始对象后（且仍被有效的快照引用），需要借助**snapdir对象来保存历史信息**，以便进行快照回滚。snapdir对象保存历史快照及克隆相关信息。

重新创建原始对象且存在snapdir对象时，需要清理snapdir对象，并同步转移其信息到原始对象。

对象的两个关键属性**OI（Object Info）和SS（Snap Set）**分别保存对象基本信息和快照信息。

**（2）object_info_t**

对象OI属性的磁盘结构，保存除快照外的对象元数据。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113152657.png)

**（3）ObjectState**

object_info_t的内存版本，增加了exists字段，表示对象是否逻辑上存在。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113153343.png)

**（4）SnapSet**

对象SS属性的磁盘结构，保存快照及克隆信息。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113153423.png)

**（5）SnapSetContext**

SnapSet的内存版本，主要增加了引用计数，便于共享。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113153512.png)

**（6）SnapContext**

前端应用自定义快照模式时（如RBD），下发的请求（op）会携带SnapContext；如果是存储池快照模式，那么SnapContext会只是当前存储池快照信息。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113154018.png)

**（7）ObjectContext**

保存了OI和SS属性，此外实现了一个属性缓存（主要缓存用户自定义属性对）和读写互斥锁机制，对客户端op保序。

op操作对象前要获取对象上下文，读写则需要获取对应锁。虽然op_shardedwq可以对同一个PG的op保序，但因为写是异步的（写堵塞会让出CPU），所以需要一套额外的读写互斥锁机制。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113155543.png)

**（8）Log**

用于解决数据一致性问题，使用PG元数据对象的omap保存。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113170823.png)

所有日志在PG中使用公共日志队列管理：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250113170947.png)

**（9）OpContext**

单个op可能操作多个对象，需要分别记录上下文和互斥锁；涉及修改操作时，会产生一条/多条日志；涉及异步操作时，需要注册一个/多个回调函数；并且需要收集读写次数、字节数等统计，后续上报到Monitor。由此引入OpContext

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250114111631.png)

**（10）RepGather**

包含修改操作的op需要执行副本间的分布式写，Primary完成封装后，由RepGather接管。

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250114111642.png)

有了上述预备知识，可以分析客户端读写流程大致分为四个阶段：

1）OSD收到读写请求，封装op并发送到PG。

2）PG检查执行条件后，执行op。

3）只包含读操作时，执行同步读（多副本）或异步读（纠删码），完成后应答客户端。

4）包含写操作时，由Primary生成操作原始对象的事务和日志，并提交到PGBackend，其按照备份策略为每个PG分发真正需要执行的本地事务，Primary收到所有副本的写入完成应答后，应答客户端。

---

## 2.1.消息接收与分发

OSD绑定的Public Messenger收到客户端读写op后，通过OSD注册的回调函数ms_fast_dispatch进行快速派发（OSD是Dispatcher的子类，后者负责派发Messenger的消息）：

	1.基于消息（message）创建op，追踪并记录消息的Epoch。
	2.查找关联的客户端会话上下文，将op加入其中的waiting_on_map队列，遍历waiting_on_map队列，如果op的Epoch大于OSD当前OSDMap的Epoch，则进一步派发到op_shardedwq队列。
	3.如果waiting_on_map不为空，则其中至少有一个op有更新的Epoch，将其加入全局session_waiting_for_map集合，汇集了需要等待更新OSDMap后才能处理的上下文。

op_shardedwq是OSD内部的op工作队列，内部实际存在多个队列，其最终关联osd_op_tp线程池，负责实际处理op。osd_op_num_shards控制op_shardedwq中工作队列数目s，osd_op_num_threads_per_shard控制工作队列中的服务线程数t，osd_op_tp总线程数为s*t。

op_shardedwq的工作队列和osd_op_tp的线程通过如下方式绑定：

	1.对线程编号：[0,s*t-1]。
	2.对工作队列编号：[0,s-1]。
	3.线程编号对s取模，得到[0,s-1]范围的结果，将对应[0,s*t-1]中的下标直接作为对应工作队列的索引。

接下来需要保证op在多个队列中的均匀分布，因为op最终由特定PG执行，PG在OSD间均匀分布，所以使用PGID对队列个数取模后得到op需要的索引，就能保证op的均匀分布，同时这样还能对同一PG的op保序。

op_shardedwq内部工作队列默认为WeightedPriorityQueue，基于权重进行调度。当op的优先级大于等于阈值osd_op_queue_cut_off时，队列工作在纯优先级模式（严格优先级模式）：

	队列内按优先级划分为优先级队列，优先级队列再根据客户端地址分为会话子队列。入队时，op进入对应的会话子队列队尾。出队时，会找到优先级最高的优先级队列，根据当前指针对应的会话子队列队头出队，同时指针移动到下一个会话子队列（防止op饿死）。

op的优先级小于阈值时，则队列工作在基于权重的Round-robin调度模式，其与纯优先级模式的区别在于：

	出队时根据权重随机选择队列，保证低优先级客户端op永远不会被饿死。

op成功加入工作队列后，服务线程被唤醒，找到PG后由其执行op。

---

## 2.2.do_request

do_request是PG处理op的第一步，主要完成全局（PG级别）的检查：

1）op的Epoch更新时，则需等待PG同步OSDMap后处理。

2）op能否被丢弃，例如：客户端断开、PG切换到更新的Interval、op在PG分裂前发送等。

3）PG不是Active状态时堵塞客户端op（PG内部op可以处理），等待Active状态。

具体处理逻辑如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250115195637.png)

因为op可能被各种原因推迟，所以PG内维护了多种op重试队列，均为FIFO队列且严格有序，如下表所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250115195805.png)

---

## 2.3.do_op

通过do_request校验确认是来自客户端的op，通过do_op进行处理，其主要进行对象级别的检查：

1）根据op的操作类型，初始化标志位，例如[CEPH_OSD_RMW_FLAG_]READ说明op携带读操作、WRITE说明op携带写操作、RWORDERED说明需要对op进行读写保序等。

2）合法性校验，出现以下情况的op不合法：

	1.PG不包含op携带的对象（即检查对象hash值对PG的bits取模后是否等于PGID）。
	2.携带可以并发执行标志CEPH_OSD_FLAG_PARALLELEXEC。
	3.携带可被Primary/Replica执行标志CEPH_OSD_FLAG_BALANCE[LOCALIZE]_READS，但当前PG既不是Primary也不是Replica（例如Stray）。
	4.客户端没有能力/权限执行对应op（例如客户端只有读权限但op中有写操作）。
	5.对象名称、key或命名空间长度超过最大限制（BlueStore无此限制）。
	6.客户端在黑名单中。
	7.op在集群被标记Full之前发送且没有CEPH_OSD_FLAG_FULL_FORCE标志。
	8.所在OSD空间不足。
	9.包含写操作且试图访问快照对象。
	10.包含写操作且一次写入数据量超过osd_max_write_size。

3）op携带的对象是否不可读/处于降级状态/正在被Scrub，是则加入对应队列。

4）检查op是否为重发（reqid是否在Log中出现过）。

5）获取对象上下文OpContext，通过execute_ctx真正执行op

do_op的具体执行流程如下图所示：

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/4/20250115200030.png)

<u>[TODO：书中提出了对**可用存储空间控制**和**对象上下文**的补充，较多暂不总结]</u>



