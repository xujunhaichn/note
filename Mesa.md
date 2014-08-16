# Mesa: 全球备份, 近乎实时, 可拓展数据仓库

> 翻译自 [Mesa: Geo-Replicated, Near Real-Time, Scalable Data Warehousing](http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/42851.pdf)

<div align="center">
Ashish Gupta, Fan Yang, Jason Govig, Adam Kirsch, Kelvin Chan, Kevin Lai, Shuo Wu, Sandeep Govind Dhoot, Abhilash Rajesh Kumar, Ankur Agiwal, Sanjay Bhansali, Mingsheng Hong, Jamie Cameron, Masood Siddiqi, David Jones, Jeff Shute, Andrey Gubarev, Shivakumar Venkataraman, Divyakant Agrawal
</div>

<div align="center">
Google, Inc.
</div>


## 1. 介绍

Google在多个地域运行着一个可拓展的广告平台，它每天为全球的用户提供数十亿广告服务。与每个广告相关的详细信息，例如目标的标准、给人留下印象和点击的数量等等，都会实时记录并处理。这些数据广泛地用在Google不同的用例上，包括报告、内部审计，分析，计费和预测。广告要在推广效果上获得良好的效果，必须通过与复杂的前端服务交互来解决对底层数据的在线和按需查询。Google的内部广告服务平台使用实时数据来确定预算和之前已经上线的广告性能，用以提高现在和未来广告服务的相关性。作为Google广告平台就需要继续拓展，而作为内部和外部用户则要求对他们广告行为更大的可视化，对更多细节和细粒度信息的需求导致了原始数据大小的急速增长。可拓展性和数据的重要性导致了针对处理、存储和查询的技术和可操作性的独特挑战。对这种数据存储的需求如下：

**原子性更新。**一个单独的用户行为可能导致对相关数据的多次更新，影响着数千个一致性的视图，这些视图由一些指标集组成（如点击和花费）而且跨越一系列维度（例如广告商和国家）。它绝不可能在部分更新完成时就查询到系统的某个状态。

**一致性和正确性。**由于商业和法律的原因，这个系统必须返回一直而且正确的数据。我们要求强一致性并且是可重复的，即使这个请求跨越了多个数据中心。
 
**可用性。**这个系统必须没有单点故障。在预计和没有预计到的维护和失败下也没有停机时间，这包括整个数据中心或者整个地域停运的影响。

**近乎实时的更新吞吐量。**这个系统必须支持持续的更新，不管是新的行数据还是对现有行的增量更新，在一秒内数百万行数据的顺序上实现更新。这个更新操作在几分钟内对跨越不同视图和数据中心的一致性请求都必须是有效的。


**可拓展性。**这个系统必须能够随着数据量和请求数的增长而拓展。例如，它必须能够支持万亿行和PT级的数据。更新和查询的效率必须在这些数据增长得很大时维持住。

**在线数据和元数据转换。**为了支持新的特性和对现有数据粒度的改变，客户端通常要求改变数据Schema或者修改现有数据的值。这些改变必须不会影响到普通的查询和更新操作。

Mesa是Google对于这些技术性和可操作性难题的解决方案。即使这些需求的子集已经被现有的数据仓库解决了，Mesa是唯一同时为商业关键数据解决所有这些难题的。

Mesa是一个分布式、可备份并且高可用的结构化数据处理、存储和查询系统。Mesa处理由上层服务生成的数据，聚合然后持久化这些数据到内部，并且通过用户的查询提供数据的服务。即使这篇论文大部分都在讨论将Mesa应用在广告系统中，Mesa其实是一个通用的数据仓库解决方案，用以满足上述的所有需求。

Mesa综合了Google的基础架构和服务，例如Colossus（Google的下一代分布式文件系统）[22, 23]、BigTable[12]和MapReduce[19]。为了实现存储的可拓展性和可用性，数据进行水平分区和复制。更新操作将应用在单个表或者多个表的粒度上。为了实现在更新时一致而且可重复的查询，底层存储的数据都是有多个版本的。为了实现可拓展的更新，数据更新都是批量处理、同时分配一个新的版本号和周期性（例如每几分钟）合并到Mesa中。为了实现在多个数据中心之间更新的一致性，Mesa使用基于Paxos[35]的分布式同步协议。

大部分基于关系型技术和数据魔方[25]的商业数据仓库都不支持每隔几分钟连续的数据集成和聚合，同时不能对用户的查询提供几乎实时的响应。通常，这些解决方案与传统的企业氛围相关，他们不太频繁地将数据聚合到数据仓库，例如一天或者一周一次。相似的，没有Google处理大数据内部技术可以应用于此，例如BigTable[12]、Megastore[11]、Spanner[18]和F1[41]。BigTable不支持Mesa应用的原子性需求。同时Megastore、Spanner和F1（这三个都是针对在线的事务处理）能够在全球同步的数据中提供强一致性，但他们不支持Mesa客户端要求峰值更新吞吐量。然而，Mesa借鉴了基于Spanner的BigTable和Paxos技术用于存储和管理元信息。

最近的研究结果也要求数据分析和数据存储要能够动态拓展。Wong[49]已经开发了一个系统可以在云中提供大批量并行分析服务。然而，这个系统是设计与多租户的环境，它有大量用户和相对较小的数据痕迹。Xin[51]已经实现了Shark利用分布式共享内存来支持拓展数据分析。然而Shark只专注于内存中的处理和分析性查询。Athanassoulis[10]已经提出了MaSM（物化排序合并）算法，它可用于结合闪存存储设备来支持数据仓库的在线更新。

这篇论文的关键贡献在于：

* 我们展示了我们如何创建一个PT级数据仓库系统，它支持需要事务处理的ACID语义，并且能够拓展到好大的吞吐量来处理Google的广告指标。

* 我们介绍了一个版本管理系统，它通过批处理更新来实现更新操作的低延时和高吞吐量，查询操作也达到同样低延时高吞吐量的性能。

* 我们描述了一个高度可拓展的分布式架构，它在一个数据中心对于机器和网络失败可以弹性容忍。我们同样展示了解决数据仓库错误的全球同步架构。与我们的设计有所区别的是应用数据是通过在独立而且冗余的多个数据中心异步备份，然而只有关键的元数据是通过复制状态同步到所有副本。这种技术最小化了在多个数据中心管理副本的开销，同时提供了非常高的更新吞吐量。

* 我们展示了如果动态而有效地改变大量表的Schema，同时不影响已存在应用的正确性和性能。

* 我们描述了用于承受软硬件故障导致的问题和数据损坏的关键技术。

* 我们 描绘了维护一个可拓展系统的一些可操作性挑战，在正确性、一致性、性能强和新研究能做贡献的领域来提高最先进的技术。

论文其他部分的组织如下。第二部分描述Mesa得存储子系统。第三部分展示Mesa系统的架构和它的跨数据中心部署。第四部分展示Mesa的一些高级功能和特性。第五部分报告Mesa开发的经验，而第六部分报告Mesa生成环境下部署的指标。第七部分回顾了相关同坐，而第八部分总结这篇论文。

## 2. Mesa存储子系统

Mesa中的数据是持续生成的，它是Google最大而且最有价值的数据之一。对这些数据的分析查询有简单的查询，如“在特定一天某个特别广告商的广告有多少点击率？”，或者是更多查询的场景，如“在十月第一个的8点到11点显示在google.com针对美国地区使用移动设备的广告商有多少广告点击率？”在Mesa中数据本质是多维的，它包括在Google广告平台多维度的细微的数据。这些数据主要由两种属性组成：多维属性（我们成为键）和度量属性（我们成为值）。细微很多维度属性是分层的（并且甚至有多个层次等等，数据维度能以日/月/年或者财政的周/季/年来组织），一个单个数据局域这些维度层次可以聚合到多个物化视图中，它支持数据的向下聚合和向上聚合。一个稳定的数据仓库设计要求单个属性的存在是一致的，不管经过任何可能的方式进行物化和聚合。

![](image/Mesa1.png)

### 2.1 数据模型

在Mesa中，数据是使用表来管理的。每个表有一个用于指定其结构的Schema。特别的是，一个表Schema指定了表的键空间K和相关的值空间V，而K和V都是集合。表的Schema也指定函数F : V x V-> V，它用户聚合相同键的所有值。这个集合函数一定要是相关的（例如对于所有值F且v0 , v1 , v2 ∈ V，（F(v0,v1),v2) = F(v0,F(v1,v2)）实际上，它通常也是交换的（例如F(v0,v1) = F(v1,v0)），尽管Mesa确实有不可交换集合函数的表（例如F(v0,v1) = v1来替换一个值）。这个Schema也用于指定表的一个或多个索引，这些索引都是在K中的全排序。

键空间K和值空间V代表着每个列的多个元组，每个元组有一个固定的类型（例如int32、int64、字符串等等）。Schema为每个独立的值列指定了一个相关的聚合函数，而F是隐式定义为智能聚合坐标的值列，例如：

F((x1,...,xk),(y1,...,yk)) = (f1(x1,y1),...,fk(xk,yk))，其中(x1,...,xk),(y1,...,yk) ∈ V是列值的任意两个元组，而且f1到fk是Schema为每个值列显示定义的。

举个例子，图1显示了三个Mesa表。这三个表都包含了广告点击和成本指标（值列），而且通过各种属性细分了，例如点击的数据、广告商、展示这个广告的网站和国家（键列）。这个用于所有值列的聚合函数就是SUM累加。所有指标在这三个表中是一致的，假设相同的底层事件在所有的这些表更新了数据。图1是Mesa表Schema的简化视图。在生产环境中，Mesa包含超过一千个表，大部分的表拥有数百个列，而且使用各种聚合函数。

![](image/Mesa2.png)

### 2.2 更新和查询

为了实现很高的更新吞吐量，Mesa批量地调用更新。这些更新的批量包是在Mesa外部的上游应用产生的，典型的是按几分钟的频率产生（更小更频繁的批量包能实现更低的更新延时，但消耗更多的资源）。通常，对Mesa的一个更新操作指定一个版本号n（从0开始按顺序分配），和一个表单包含的行集合（表名、键和值）。每个更新操作对每个列包含最多一个聚合值。

对Mesa的一个查询操作包含一个版本号n和在键空间的一个谓词P。而响应包含对于每个匹配P的键的一行，它出现在版本在0到n的更新中。响应中一个键的值是聚合了在那些更新中的所有值。Mesa实际上支持比这个更复杂的请求功能，但这些全部都能看做是这个原语的前置操作和后置操作。

举个例子，图2显示了两个针对图1定义的表的更新操作，当聚合在表A、B和C。为了维护表的一致性（在2.1部分讨论到），每个更新操作包含两个表A和B的一致行行。Mesa自动对表B处理这些更新操作，因为它们能直接从表B的更新中衍生出来。理论上，单个更新操作包括广告ID和出版商ID属性能够用于更新三个表，但那样的操作太重了，尤其是在更通用的例子中各个表都有很多属性（例如是一个交叉的产品）。

版本化数据在Mesa的更新和查询中都是非常重要的一环。然而，它也面临着多种挑战。第一，根据广告数据的聚合本质，独立地存储每个版本从存储的角度是非常浪费的。聚合的数据在传统中能够做得更小。第二，在查询的时候访问各个版本并且聚合它们是很重的而且增加了访问延时。第三，原生的在每次更新时对所有版本的预聚合可能变成非常的重。

为了处理这些挑战，Mesa预聚合某些版本化数据然后存储它，数据存储时包含一个行集合（没有重复的键）和特定版本（或者更简单的一个版本），这是通过[V1,V2]来表示的，V1和V2是更新的版本号，而且V1 ≤ V2。我们更倾向于某个版本当它的含义是清晰的。在[V1,V2]中的行对应了键的集合，这些键出现在更新操作而且它们的版本号在V1和V2之间（包括前后）。每个键的值是它们更新中值的聚合。更新操作在Mesa中会合并成单体（或者更简单的单体）。对于单体这个版本[V1 , V2]对应了一个版本号n的更新，而且通过设V1 = V2 = n。

一个版本[V1, V2]和另一个版本[V2 + 1, V3]能够聚合到版本[V1,V3]，通过简单地合并行键和因而聚合值。（已经在2.4部分讨论过，行是通过键和其他两个版本能够在线性时间中合并。）这个计算的正确性跟随在相关的聚合函数F后。值得注意的是，正确性不依赖于F的可交换性，而Mesa对给定的键聚合两个值中，这个版本总是在表单[V1 , V2] and [V2 + 1, V3]中，而且聚合是执行在版本的递增顺序中。


(particularly queries) that are able to use those deltas instead of singletons.







Mesa是使用Google通用的基础架构和服务构建的，包括BigTable[12]和Colossus[22,23]。Mesa运行在多个数据中心里，每个数据中心运行一个Mesa实体。我们从描述实体的设计来开始。然后我们会讨论这些实体是怎样集成构建一个完整的跨数据中心Mesa部署系统。




![](image/Mesa5.png)

![](image/Mesa6.png)


server updates (e.g., binary releases) without unduly im- pacting clients, who can automatically failover to another set in the same (or even a different) Mesa instance. Within a set, each query server is in principle capable of handling a query for any table. However, for performance reasons, Mesa prefers to direct queries over similar data (e.g., all queries over the same table) to a subset of the query servers. This technique allows Mesa to provide strong latency guarantees by allowing for effective query server in-memory pre-fetching and caching of data stored in Colossus, while also allowing for excellent overall throughput by balancing load across the query servers. On startup, each query server registers the list of tables it actively caches with a global locator service, which is then used by clients to discover query servers.















Figure 9 illustrates the overhead of query processing and the effectiveness of the scan-to-seek optimization discussed in Section 4.1 over the same 7 day period. The rows returned are only about 30%-50% of rows read due to delta merging and filtering specified by the queries. The scan-to-seek op- timization avoids decompressing/reading 60% to 70% of the delta rows that we would otherwise need to process.





Google其他内部的数据解决方案[11, 12, 18, 41]都不能支持Google广告业务要求作为数据仓库的数据大小和更新效率。Mesa通过批处理更新搞作实现这个拓展性。每个更新操作会花费几分钟来提交并且每次批处理的元数据都会使用Paxos来提交以确保像Megastore、Spanner和F1所提供强一致性。因此Mesa在应用数据在多个数据仓库中是冗余的（而且独立的）也能保持唯一，同时元数据是使用同步备份来管理的。这种方法在数据损坏的情况下提供了额外的健壮性保证。



在这篇论文，我们展示了一个叫Mesa的系统从头到尾的设计和实现，它是全球同步备份、近乎实时并且可拓展的数据仓库。Mesa的工程设计权衡了在数据库和分布式系统的基础调研意见。特别的是，Mesa在提供强一致和事务正确性保证的同时支持在线的查询和更新。它通过使用基于批处理的接口，通过介绍瞬间的数据版本而消除在查询和更新事务基于锁的同步来实现这些原子性操作等特性。Mesa能跨数据中心实现全球同步备份来提高错误容忍性。最后，在每个数据中心里，Mesa的分层架构允许它分配工作并且根据计算的需求动态拓展，通过大量服务器来实现大规模拓展。



[29] H. V. Jagadish, L. V. S. Lakshmanan, and D. Srivastava. Snakes and Sandwiches: Optimal Clustering Strategies for a Data Warehouse. In SIGMOD, pages 37–48, 1999.  
[47] A. Thusoo, Z. Shao, et al. Data Warehousing and Analytics Infrastructure at Facebook. In SIGMOD, pages 1013–1020, 2010.  
[49] P. Wong, Z. He, et al. Parallel Analytics as a Service. In SIGMOD, pages 25–36, 2013.  