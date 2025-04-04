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
