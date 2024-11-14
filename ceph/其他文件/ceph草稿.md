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

![](https://raw.githubusercontent.com/HentaiYang/Pics/main/NoteBooks/ceph/1/20241111163437.png)

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



