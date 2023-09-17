---
author:
  name: ruifeng
  link: https://blog.luckypeak.xyz
  email: me@luckypeak.xyz
  avatar: /safari-pinned-tab.svg
title: 【译】Presto - SQL on Everything
subtitle: ""
date: 2023-09-17
lastmod: 2023-09-17
categories:
  - 大数据
tags:
  - Presto
draft: false
---
Abstract—Presto is an open source distributed query engine that supports much of the SQL analytics workload at Facebook. Presto is designed to be adaptive, flexible, and extensible. It supports a wide variety of use cases with diverse characteristics. These range from user-facing reporting applications with subsecond latency requirements to multi-hour ETL jobs that aggregate or join terabytes of data. Presto’s Connector API allows plugins to provide a high performance I/O interface to dozens of data sources, including Hadoop data warehouses, RDBMSs, NoSQL systems, and stream processing systems. In this paper, we outline a selection of use cases that Presto supports at Facebook. We then describe its architecture and implementation, and call out features and performance optimizations that enable it to support these use cases. Finally, we present performance results that demonstrate the impact of our main design decisions.

Index Terms—SQL, query engine, big data, data warehouse

摘要 —— Presto 是一个开源的分布式查询引擎，支持 Facebook 大部分的 SQL 分析工作负载。Presto 设计得具有适应性、灵活性和可扩展性。它支持各种各样的用例，这些用例有不同的特点。这些用例范围从要求亚秒级延迟的面向用户的报告应用，到聚合或联接 TB 级数据的多小时 ETL 作业。Presto 的 Connector API 允许插件为数十种数据源提供高性能的 I/O 接口,包括 Hadoop 数据仓库、关系数据库、NoSQL 系统和流处理系统。在本文中，我们概述了 Presto 在 Facebook 上所支持的一些用例选择。然后，我们描述了它的体系结构和实现，并指出了支持这些用例的功能和性能优化的要点。最后，我们呈现了性能结果，展示了我们的主要设计决策的影响。

关键字 —— SQL，查询引擎，大数据，数据仓库

## I. Introduction

The ability to quickly and easily extract insights from large amounts of data is increasingly important to technologyenabled organizations. As it becomes cheaper to collect and store vast amounts of data, it is important that tools to query this data become faster, easier to use, and more flexible. Using a popular query language like SQL can make data analytics accessible to more people within an organization. However, ease-of-use is compromised when organizations are forced to deploy multiple incompatible SQL-like systems to solve different classes of analytics problems.

快速轻松地从大量数据中提取见解的能力对于技术驱动的组织日益重要。随着收集和存储海量数据变得更加便宜，查询这些数据的工具需要变得更快、更易于使用和更灵活。使用像 SQL 这样流行的查询语言可以使更多组织内的人进行数据分析。然而，当组织被迫部署多个不兼容的类 SQL 系统来解决不同类别的分析问题时，易用性会受到影响。

Presto is an open-source distributed SQL query engine that has run in production at Facebook since 2013 and is used today by several large companies, including Uber, Netflix, Airbnb, Bloomberg, and LinkedIn. Organizations such as Qubole, Treasure Data, and Starburst Data have commercial offerings based on Presto. The Amazon Athena1 interactive querying service is built on Presto. With over a hundred contributors on GitHub, Presto has a strong open source community.

Presto 是一个开源的分布式 SQL 查询引擎，自 2013 年以来在 Facebook 生产环境中运行，现在被包括 Uber、Netflix、Airbnb、Bloomberg 和 LinkedIn 在内的几家大公司使用。Qubole、Treasure Data 和 Starburst Data 等组织基于 Presto 提供商业产品。亚马逊 Athena 交互式查询服务也是建立在 Presto 之上。在 GitHub 上拥有超过一百名贡献者，Presto 有一个强大的开源社区

Presto is designed to be adaptive, flexible, and extensible. It provides an ANSI SQL interface to query data stored in Hadoop environments, open-source and proprietary RDBMSs, NoSQL systems, and stream processing systems such as Kafka. A ‘Generic RPC’2 connector makes adding a SQL interface to proprietary systems as easy as implementing a half dozen RPC endpoints. Presto exposes an open HTTP API, ships with JDBC support, and is compatible with several industry-standard business intelligence (BI) and query authoring tools. The built-in Hive connector can natively read from and write to distributed file systems such as HDFS and Amazon S3; and supports several popular open-source file formats including ORC, Parquet, and Avro.

Presto 的设计是适应性强、灵活且可扩展的。它为存储在 Hadoop 环境、开源和专有 RDBMS、NoSQL 系统以及 Kafka 等流处理系统中的数据提供 ANSI SQL 接口。一个「通用 RPC」连接器使得为专有系统添加 SQL 接口就像实现几个 RPC 端点一样简单。Presto 暴露一个开放的 HTTP API，内置 JDBC 支持，并与几种行业标准商业智能（BI）和查询创作工具兼容。内置的 Hive 连接器可以直接从分布式文件系统如 HDFS 和 Amazon S3 中读取和写入，并支持一些流行的开源文件格式，包括 ORC、Parquet 和 Avro。

As of late 2018, Presto is responsible for supporting much of the SQL analytic workload at Facebook, including interactive/BI queries and long-running batch extract-transform-load (ETL) jobs. In addition, Presto powers several end-user facing analytics tools, serves high performance dashboards, provides a SQL interface to multiple internal NoSQL systems, and supports Facebook’s A/B testing infrastructure. In aggregate, Presto processes hundreds of petabytes of data and quadrillions of rows per day at Facebook.

截至 2018 年底，Presto 支持 Facebook 大部分的 SQL 分析工作负载，包括交互式/BI 查询和长时间运行的批量提取-转换-加载（ETL）作业。此外，Presto 提供几个面向终端用户的分析工具，服务高性能仪表盘，为多个内部 NoSQL 系统提供 SQL 接口，并支持 Facebook 的 A/B 测试基础设施。总体而言，Presto 每天在 Facebook 处理数百 PB 的数据和数万亿行。

Presto has several notable characteristics:

- It is an adaptive multi-tenant system capable of concurrently running hundreds of memory, I/O, and CPU-intensive queries, and scaling to thousands of worker nodes while efficiently utilizing cluster resources.
- Its extensible, federated design allows administrators to set up clusters that can process data from many different data sources even within a single query. This reduces the complexity of integrating multiple systems.
- It is flexible, and can be configured to support a vast variety of use cases with very different constraints and performance characteristics.
- It is built for high performance, with several key related features and optimizations, including code-generation. Multiple running queries share a single long-lived Java Virtual Machine (JVM) process on worker nodes, which reduces response time, but requires integrated scheduling, resource management and isolation.

Presto有几个显著的特点:

- 它是一个适应性强的多租户系统，能够同时运行成百上千的内存、I/O 和 CPU 密集型查询，并扩展到数以千计的工作节点，同时高效利用集群资源。
- 其可扩展的联邦设计允许管理员设置能够处理来自许多不同数据源的数据的集群，甚至在单个查询中也是如此。这减少了集成多个系统的复杂性。
- 它非常灵活，可以配置以支持各种各样的用例，这些用例有非常不同的约束和性能特征。
- 它是为高性能而构建的，具有几个相关的关键功能和优化，包括代码生成。多个运行中的查询在工作节点上共享一个长期存在的 Java 虚拟机（JVM）进程，这减少了响应时间，但需要集成的调度、资源管理和隔离。

The primary contribution of this paper is to describe the design of the Presto engine, discussing the specific optimizations and trade-offs required to achieve the characteristics we described above. The secondary contributions are performance results for some key design decisions and optimizations, and a description of lessons learned while developing and maintaining Presto.

本文的主要贡献是描述 Presto 引擎的设计，讨论实现上述特性所需的具体优化和权衡。次要贡献是一些关键设计决策和优化的性能结果,以及开发和维护Presto过程中获得的经验教训。

Presto was originally developed to enable interactive querying over the Facebook data warehouse. It evolved over time to support several different use cases, a few of which we describe in Section II. Rather than studying this evolution, we describe both the engine and use cases as they exist today, and call out main features and functionality as they relate to these use cases. The rest of the paper is structured as follows. In Section III, we provide an architectural overview, and then dive into system design in Section IV. We then describe some important performance optimizations in Section V, present performance results in Section VI, and engineering lessons we learned while developing Presto in Section VII. Finally, we outline key related work in Section VIII, and conclude in Section IX. Presto is under active development, and significant new functionality is added frequently. In this paper, we describe Presto as of version 0.211, released in September 2018.

Presto 最初是为了支持 Facebook 数据仓库的交互式查询而开发的。随着时间的推移，它演变为支持几种不同的用例，我们在第 2 节中描述了其中的一些。我们没有研究这种演变，而是描述它们今天存在的引擎和用例，并在它们与这些用例的关系方面提出主要的功能。本文的其余部分结构如下:在第 3 节中，我们提供了架构概述，然后在第 4 节深入系统设计。接着，我们在第 5 节中描述了一些重要的性能优化，在第 6 节展示了性能结果，以及我们在开发 Presto 过程中获得的工程经验教训。最后，我们在第 8 节概述了关键的相关工作，并在第 9 节结束。Presto 正在积极开发中，重要的新功能经常添加。在本文中，我们描述了截止到 2018 年 9 月发布的 0.211 版本的 Presto。

## II. Use Cases

At Facebook, we operate numerous Presto clusters (with sizes up to ∼1000 nodes) and support several different use cases. In this section we select four diverse use cases with large deployments and describe their requirements.

在 Facebook，我们运营着许多 Presto 集群（最大规模达到约 1000 节点），并支持几种不同的用例。在本节中，我们选择了四种大规模部署的不同用例，并描述了它们的需求。

### A. Interactive Analytics

Facebook operates a massive multi-tenant data warehouse as an internal service, where several business functions and organizational units share a smaller set of managed clusters. Data is stored in a distributed filesystem and metadata is stored in a separate service. These systems have APIs similar to that of HDFS and the Hive metastore service, respectively. We refer to this as the ‘Facebook data warehouse’, and use a variant of the Presto ‘Hive’ connector to read from and write to it.

Facebook 运营着一个庞大的多租户数据仓库作为内部服务，这里几个业务功能部门和组织单元共享一小部分托管集群。数据存储在一个分布式文件系统中，元数据存储在一个单独的服务中。这些系统分别具有类似 HDFS 和 Hive 元存储服务的 API。我们将其称为「Facebook 数据仓库」，并使用Presto ’Hive‘ 连接器的变种从中读取和写入。

Facebook engineers and data scientists routinely examine small amounts of data (∼50GB-3TB compressed), test hypotheses, and build visualizations or dashboards. Users often rely on query authoring tools, BI tools, or Jupyter notebooks. Individual clusters are required to support 50-100 concurrent running queries with diverse query shapes, and return results within seconds or minutes. Users are highly sensitive to endto-end wall clock time, and may not have a good intuition of query resource requirements. While performing exploratory analysis, users may not require that the entire result set be returned. Queries are often canceled after initial results are returned, or use LIMIT clauses to restrict the amount of result data the system should produce.

Facebook 工程师和数据科学家们定期检查小规模的数据（压缩后约 50GB-3TB），测试假设，并构建可视化或仪表盘。用户们通常依赖查询创作工具、BI 工具或 Jupyter notebooks。每个集群需要支持 50-100 个并发运行的具有各种查询形状的查询，并在几秒钟或几分钟内返回结果。用户对端到端的真实时间非常敏感，并且可能没有很好地理解查询的资源需求。在执行探索性分析时，用户可能不需要返回全部结果集。查询通常在返回初始结果后就被取消，或者使用 LIMIT 子句来限制系统应该生成的结果数据量。

### B. Batch ETL

The data warehouse we described above is populated with fresh data at regular intervals using ETL queries. Queries are scheduled by a workflow management system that determines dependencies between tasks and schedules them accordingly. Presto supports users migrating from legacy batch processing systems, and ETL queries now make up a large fraction of the Presto workload at Facebook by CPU. These queries are typically written and optimized by data engineers. They tend to be much more resource intensive than queries in the Interactive Analytics use case, and often involve performing CPU-heavy transformations and memory-intensive (multiple TBs of distributed memory) aggregations or joins with other large tables. Query latency is somewhat less important than resource efficiency and overall cluster throughput.

我们上面描述的数据仓库是使用 ETL 查询定期加载新数据的。查询由一个工作流管理系统调度，该系统确定任务之间的依赖关系并相应地调度它们。Presto 支持用户从遗留的批处理系统迁移，现在 ETL 查询占了 Facebook 上 Presto 工作负载 CPU 使用的很大一部分。这些查询通常由数据工程师编写和优化。它们往往比交互式分析用例中的查询更耗费资源，并且通常涉及执行 CPU 密集的转换和内存密集的（数 TB 分布式内存）聚合或与其他大表的连接。查询延迟相对不那么重要，资源效率和整体集群吞吐量更重要。

### C. A/B Testing

A/B testing is used at Facebook to evaluate the impact of product changes through statistical hypothesis testing. Much of the A/B test infrastructure at Facebook is built on Presto. Users expect test results be available in hours (rather than days) and that the data be complete and accurate. It is also important for users to be able to perform arbitrary slice and dice on their results at interactive latency (∼5-30s) to gain deeper insights. It is difficult to satisfy this requirement by pre-aggregating data, so results must be computed on the fly. Producing results requires joining multiple large data sets, which include user, device, test, and event attributes. Query shapes are restricted to a small set since queries are programmatically generated.

A/B 测试在 Facebook 被用来通过统计假设检验评估产品变更的影响。Facebook 大部分 A/B 测试基础设施是建立在 Presto 之上的。用户希望测试结果在几小时内（而不是几天）就可用，并且数据是完整和准确的。用户能够以交互延迟（~5-30秒）上的任意切片和切块分析他们的结果以获得更深入的洞察也很重要。通过预先聚合数据来满足这个要求是很困难的，因此结果必须即席计算。生成结果需要连接多个大数据集，这包括用户、设备、测试和事件属性。由于查询是以编程方式生成的，查询形状受到限制。

### D. Developer/Advertiser Analytics

Several custom reporting tools for external developers and advertisers are built on Presto. One example deployment of this use case is Facebook Analytics3 , which offers advanced analytics tools to developers that build applications which use the Facebook platform. These deployments typically expose a web interface that can generate a restricted set of query shapes. Data volumes are large in aggregate, but queries are highly selective, as users can only access data for their own applications or ads. Most query shapes contain joins, aggregations or window functions. Data ingestion latency is in the order of minutes. There are very strict query latency requirements (∼50ms-5s) as the tooling is meant to be interactive. Clusters must have 99.999% availability and support hundreds of concurrent queries given the volume of users.

一些面向外部开发者和广告商的定制报告工具是建立在 Presto 之上的。这个用例的一个示例部署是Facebook Analytics，它为使用 Facebook 平台构建应用的开发者提供高级分析工具。这些部署通常公开一个 Web 界面，可以生成一组受限的查询形状。从总体上看数据量很大，但是查询高度选择性很强，因为用户只能访问自己的应用或广告的数据。大多数查询形状包含连接、聚合或窗口函数。数据摄入延迟在几分钟的数量级。由于该工具意在进行交互，因此有非常严格的查询延迟要求（~50ms-5s）。鉴于用户量，集群必须有 99.999% 可用性并支持数百个并发查询。

## III. ARCHITECTURE OVERVIEW

A Presto cluster consists of a single coordinator node and one or more worker nodes. The coordinator is responsible for admitting, parsing, planning and optimizing queries as well as query orchestration. Worker nodes are responsible for query processing. Figure 1 shows a simplified view of Presto architecture.

一个 Presto 集群由一个协调器节点和一个或多个工作器节点组成。协调器负责接受、解析、规划和优化查询以及查询编排。工作器节点负责查询处理。图1 显示了 Presto 架构的简化视图。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/f64c36f0db75742ccfc68a05b1861a6b.png)
图一 Presto 架构

The client sends an HTTP request containing a SQL statement to the coordinator. The coordinator processes the request by evaluating queue policies, parsing and analyzing the SQL text, creating and optimizing distributed execution plan.

客户端向协调器发送包含 SQL 语句的 HTTP 请求。协调器通过评估队列策略、解析和分析 SQL 文本、创建和优化分布式执行计划来处理请求。

The coordinator distributes this plan to workers, starts execution of tasks and then begins to enumerate splits, which are opaque handles to an addressable chunk of data in an external storage system. Splits are assigned to the tasks responsible for reading this data.

协调器将该计划分发给工作器，启动任务执行，然后开始枚举分片，分片是外部存储系统中一段可寻址数据块的不透明句柄。将分片分配给负责读取此数据的任务。

Worker nodes running these tasks process these splits by fetching data from external systems, or process intermediate results produced by other workers. Workers use co-operative multi-tasking to process tasks from many queries concurrently. Execution is pipelined as much as possible, and data flows between tasks as it becomes available. For certain query shapes, Presto is capable of returning results before all the data is processed. Intermediate data and state is stored inmemory whenever possible. When shuffling data between nodes, buffering is tuned for minimal latency.

运行这些任务的工作器节点通过从外部系统获取数据或者处理其他工作器生成的中间结果来处理这些分片。工作器使用协作式多任务处理来同时处理许多查询的任务。尽可能采用流水线执行，并在数据可用时在任务之间流动。对于某些查询形状，Presto 可以在处理所有数据之前返回结果。尽可能将中间数据和状态存储在内存中。在节点之间洗牌数据时，缓冲区优化为最小延迟。

Presto is designed to be extensible; and provides a versatile plugin interface. Plugins can provide custom data types, functions, access control implementations, event consumers, queuing policies, and configuration properties. More importantly, plugins also provide connectors, which enable Presto to communicate with external data stores through the Connector API, which is composed of four parts: the Metadata API, Data Location API, Data Source API, and Data Sink API. These APIs are designed to allow performant implementations of connectors within the environment of a physically distributed execution engine. Developers have contributed over a dozen connectors to the main Presto repository, and we are aware of several proprietary connectors.

Presto 设计为可扩展的；并提供了通用的插件接口。插件可以提供自定义数据类型、函数、访问控制实现、事件消费者、队列策略和配置属性。更重要的是，插件还提供连接器，这使得 Presto 可以通过 Connector API 与外部数据存储进行通信，该 API 由四个部分组成：元数据 API、数据位置 API、数据源 API 和数据接收 API。这些 API 旨在允许在物理分布式执行引擎环境下高性能地实现连接器。开发者已经为主 Presto 存储库贡献了十几个连接器，我们还注意到了几个专有连接器。

## IV. SYSTEM DESIGN

In this section we describe some of the key design decisions and features of the Presto engine. We describe the SQL dialect that Presto supports, then follow the query lifecycle all the way from client to distributed execution. We also describe some of the resource management mechanisms that enable multitenancy in Presto. Finally, we briefly discuss fault tolerance.

在这个部分，我们将描述 Presto 引擎的一些关键设计决策和特点。我们将描述 Presto 支持的 SQL 方言，然后从客户端到分布式执行一路跟踪查询生命周期。我们还会描述一些资源管理机制，以实现 Presto 的多租户支持。最后，我们将简要讨论容错性。

### A. SQL Dialect

Presto closely follows the ANSI SQL specification [2]. While the engine does not implement every feature described, implemented features conform to the specification as far as possible. We have made a few carefully chosen extensions to the language to improve usability. For example, it is difficult to operate on complex data types, such as maps and arrays, in ANSI SQL. To simplify operating on these common data types, Presto syntax supports anonymous functions (lambda expressions) and built-in higher-order functions (e.g., transform, filter, reduce).

Presto 严格遵循 ANSI SQL 规范 [2]。虽然引擎没有实现规范中的每个功能，但已实现的功能尽可能符合规范。我们进行了一些精心选择的扩展来提高可用性。例如，在 ANSI SQL 中，对复杂数据类型（如映射和数组）进行操作比较困难。为了简化对这些常见数据类型的操作，Presto 语法支持匿名函数（Lambda 表达式）和内置的高阶函数（例如 transform、filter、reduce）。

### B. Client Interfaces, Parsing, and Planning

1. Client Interfaces: The Presto coordinator primarily exposes a RESTful HTTP interface to clients, and ships with a first-class command line interface. Presto also ships with a JDBC client, which enables compatibility with a wide variety of BI tools, including Tableau and Microstrategy.
    
2. Parsing: Presto uses an ANTLR-based parser to convert SQL statements into a syntax tree. The analyzer uses this tree to determine types and coercions, resolve functions and scopes, and extracts logical components, such as subqueries, aggregations, and window functions.
    
3. Logical Planning: The logical planner uses the syntax tree and analysis information to generate an intermediate representation (IR) encoded in the form of a tree of plan nodes. Each node represents a physical or logical operation, and the children of a plan node are its inputs. The planner produces nodes that are purely logical, i.e. they do not contain any information about how the plan should be executed. Consider a simple query:
    
    ```sql
    SELECT
    	orders.orderkey, SUM(tax)
    FROM orders
    LEFT JOIN lineitem
    	ON orders.orderkey = lineitem.orderkey
    WHERE discount = 0
    GROUP BY orders.orderkey
    ```
    
    The logical plan for this query is outlined in Figure 2.
    
	
    
4. 客户端接口：Presto 协调器主要向客户端提供了一个符合 RESTful HTTP 标准的接口，并附带了一流的命令行界面。Presto 还提供了一个 JDBC 客户端，使其与各种商业智能工具兼容，包括 Tableau 和 Microstrategy 等。
    
5. 解析：Presto 使用基于 ANTLR 的解析器将 SQL 语句转换为语法树。分析器使用此树来确定类型和强制转换，解析函数和范围，并提取逻辑组件，例如子查询、聚合和窗口函数。
    
6. 逻辑规划：逻辑规划器使用语法树和分析信息生成一个中间表示（IR），以树形式编码。每个节点代表一个物理或逻辑操作，计划节点的子节点是其输入。规划器生成的节点纯粹是逻辑的，即它们不包含有关如何执行计划的任何信息。考虑一个简单的查询：
    
    ```sql
    SELECT
    	orders.orderkey, SUM(tax)
    FROM orders
    LEFT JOIN lineitem
    	ON orders.orderkey = lineitem.orderkey
    WHERE discount = 0
    GROUP BY orders.orderkey
    ```
    
    该查询的逻辑计划如 图2 所示：
    
    ![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/364fe76c5386d113c92a06de42ffabc9.png)
    

### C. Query Optimization

The plan optimizer transforms the logical plan into a more physical structure that represents an efficient execution strategy for the query. The process works by evaluating a set of transformation rules greedily until a fixed point is reached. Each rule has a pattern that can match a sub-tree of the query plan and determines whether the transformation should be applied. The result is a logically equivalent sub-plan that replaces the target of the match. Presto contains several rules, including well-known optimizations such as predicate and limit pushdown, column pruning, and decorrelation.

计划优化器将逻辑计划转换为更具物理结构的形式，代表了查询的高效执行策略。该过程通过贪婪地评估一组转换规则，直到达到一个固定点来进行。每个规则都有一个模式，可以匹配查询计划的子树，并确定是否应该应用该转换。结果是一个在逻辑上等效的子计划，替代了匹配目标。Presto 包含多个规则，包括一些众所周知的优化，如谓词和限制下推、列剪枝和解关联等。

We are in the process of enhancing the optimizer to perform a more comprehensive exploration of the search space using a cost-based evaluation of plans based on the techniques introduced by the Cascades framework [13]. However, Presto already supports two cost-based optimizations that take table and column statistics into account - join strategy selection and join re-ordering. We will discuss only a few features of the optimizer; a detailed treatment is out of the scope of this paper.

我们正在增强优化器，以使用基于成本的计划评估技术，执行对搜索空间的更全面的探索，这些技术是由 Cascades 框架[13]引入的。然而，Presto 已经支持了两种基于成本的优化，考虑了表和列的统计信息 - 连接策略选择和连接重排序。我们只会讨论优化器的一些特性；详细的处理超出了本文的范围。

1. Data Layouts: The optimizer can take advantage of the physical layout of the data when it is provided by the connector Data Layout API. Connectors report locations and other data properties such as partitioning, sorting, grouping, and indices. Connectors can return multiple layouts for a single table, each with different properties, and the optimizer can select the most efficient layout for the query [15] [19]. This functionality is used by administrators operating clusters for the Developer/Advertiser Analytics use case; it enables them to optimize new query shapes simply by adding physical layouts. We will see some of the ways the engine can take advantage of these properties in the subsequent sections.
    
1. 数据布局：优化器可以在连接器数据布局 API 提供的情况下充分利用数据的物理布局。连接器报告位置和其他数据属性，如分区、排序、分组和索引。连接器可以为单个表返回多个不同属性的布局，而优化器可以为查询选择最高效的布局[15] [19]。这个功能被管理员用于操作用于开发者/广告商分析的集群；它使他们能够通过添加物理布局来简单地优化新的查询形状。我们将在后续部分看到引擎如何利用这些属性。
    
2. Predicate Pushdown: The optimizer can work with connectors to decide when pushing range and equality predicates down through the connector improves filtering efficiency.
    
	For example, the Developer/Advertiser Analytics use case leverages a proprietary connector built on top of sharded MySQL. The connector divides data into shards that are stored in individual MySQL instances, and can push range or point predicates all the way down to individual shards, ensuring that only matching data is ever read from MySQL. If multiple layouts are present, the engine selects a layout that is indexed on the predicate columns. Efficient index based filtering is very important for the highly selective filters used in the Developer/Advertiser Analytics tools. For the Interactive Analytics and Batch ETL use cases, Presto leverages the partition pruning and file-format features (Section V-C) in the Hive connector to improve performance in a similar fashion.

2. 谓词下推：优化器可以与连接器协作，决定何时通过连接器将范围和相等谓词下推，以提高过滤效率。

	例如，开发者/广告商分析用例利用了一个建立在分片的 MySQL 之上的专有连接器。该连接器将数据分成存储在单独的 MySQL 实例中的分片，并可以将范围或点谓词一直下推到单个分片，确保只有匹配的数据从 MySQL 中读取。如果存在多个布局，引擎会选择在谓词列上建立索引的布局。在开发者/广告商分析工具中使用的高度选择性过滤器中，高效的基于索引的过滤非常重要。对于交互式分析和批处理 ETL 用例，Presto 还利用了 Hive 连接器中的分区剪枝和文件格式特性，以类似的方式提高性能。

3. Inter-node Parallelism: Part of the optimization process involves identifying parts of the plan that can be executed in parallel across workers. These parts are known as ‘stages’, and every stage is distributed to one or more tasks, each of which execute the same computation on different sets of input data. The engine inserts buffered in-memory data transfers (shuffles) between stages to enable data exchange. Shuffles add latency, use up buffer memory, and have high CPU overhead. Therefore, the optimizer must reason carefully about the total number of shuffles introduced into the plan. Figure 3 shows how a naıve implementation would partition a plan into stages and connect them using shuffles.
    
3. 节点间并行性：优化过程的一部分涉及识别计划中可以在多个工作节点上并行执行的部分。这些部分被称为“阶段”，每个阶段分布到一个或多个任务，每个任务在不同的输入数据集上执行相同的计算。引擎在阶段之间插入了内存中的数据传输（洗牌）以启用数据交换。洗牌会增加延迟，使用缓冲内存，并具有高CPU开销。因此，优化器必须仔细考虑计划中引入的洗牌的总数。图3 显示了一个简单的实现将计划分成阶段并使用洗牌连接它们的方式。
    

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/39f6005feecc9784af508a3dcb36834a.png)
图3 - 图2 的分布式计划。连接器没有公开任何数据布局属性，并且没有应用洗牌减少优化。需要四次洗牌以执行查询。

Data Layout Properties: The physical data layout can be used by the optimizer to minimize the number of shuffles in the plan. This is very useful in the A/B Testing use case, where almost every query requires a large join to produce experiment details or population information. The engine takes advantage of the fact that both tables participating in the join are partitioned on the same column, and uses a co-located join strategy to eliminate a resource-intensive shuffle.

数据布局属性：优化器可以利用物理数据布局来最小化计划中的洗牌次数。这在 A/B 测试用例中非常有用，因为几乎每个查询都需要进行大型连接以生成实验详细信息或人口信息。引擎利用了参与连接的两个表都基于相同列进行分区的事实，并使用共同位置的连接策略来消除资源密集型的洗牌操作。

If connectors expose a data layout in which join columns are marked as indices, the optimizer is able to determine if using an index nested loop join would be an appropriate strategy. This can make it extremely efficient to operate on normalized data stored in a data warehouse by joining against production data stores (key-value or otherwise). This is a commonly used feature in the Interactive Analytics use case.

如果连接器公开了数据布局，其中连接列标记为索引，那么优化器可以确定是否使用索引嵌套循环连接是一种合适的策略。这可以使在数据仓库中存储的规范化数据与生产数据存储（键值或其他方式）进行连接变得极其高效。这是交互式分析用例中常用的功能。

Node Properties: Like connectors, nodes in the plan tree can express properties of their outputs (i.e. the partitioning, sorting, bucketing, and grouping characteristics of the data) [24]. These nodes have the ability to also express required and preferred properties, which are taken into account when introducing shuffles. Redundant shuffles are simply elided, but in other cases the properties of the shuffle can be changed to reduce the number of shuffles required. Presto greedily selects partitioning that will satisfy as many required properties as possible to reduce shuffles. This means that the optimizer may choose to partition on fewer columns, which in some cases can result in greater partition skew. As an example, this optimization applied to the plan in Figure 3 causes it to collapse to a single data processing stage.

节点属性：与连接器类似，计划树中的节点可以表达其输出的属性（即数据的分区、排序、分桶和分组特性）[24]。这些节点还可以表达所需和首选属性，这些属性在引入洗牌时会考虑在内。多余的洗牌会被简单地省略，但在其他情况下，可以更改洗牌的属性以减少所需的洗牌数量。Presto 会贪婪地选择分区，以满足尽可能多的所需属性，以减少洗牌。这意味着优化器可能选择在较少的列上进行分区，这在某些情况下可能会导致更大的分区偏斜。举个例子，应用于 图3 中的计划的这种优化会导致其坍缩为单个数据处理阶段。

4. Intra-node Parallelism: The optimizer uses a similar mechanism to identify sections within plan stages that can benefit from being parallelized across threads on a single node. Parallelizing within a node is much more efficient than inter-node parallelism, since there is little latency overhead, and state (e.g., hash-tables and dictionaries) can be efficiently shared between threads. Adding intra-node parallelism can lead to significant speedups, especially for query shapes where concurrency constrains throughput at downstream stages:

节点内并行性：优化器使用类似的机制来识别计划阶段内部的部分，可以从在单个节点上的线程之间进行并行化中受益。在节点内进行并行化比节点间并行化效率更高，因为几乎没有延迟开销，状态（例如哈希表和字典）可以在线程之间高效共享。添加节点内并行性可以显著提高性能，特别是对于那些并发约束下游阶段吞吐量的查询形状而言：

- The Interactive Analytics involves running many short oneoff queries, and users do not typically spend time trying to optimize these. As a result, partition skew is common, either due to inherent properties of the data, or as a result of common query patterns (e.g., grouping by user country while also filtering to a small set of countries). This typically manifests as a large volume of data being hash-partitioned on to a small number of nodes.
- Batch ETL jobs often transform large data sets with little or no filtering. In these scenarios, the smaller number of nodes involved in the higher levels of the tree may be insufficient to quickly process the volume of data generated by the leaf stage. Task scheduling is discussed in Section IV-D2.
- 交互式分析通常涉及运行许多短期一次性查询，用户通常不会花时间来优化这些查询。因此，分区偏斜很常见，要么是由于数据本身的固有属性，要么是由于常见的查询模式导致的（例如，按用户国家进行分组，同时筛选一小部分国家）。这通常表现为大量数据被哈希分区到少数节点上。
- 批处理 ETL 作业通常会转换大型数据集，很少或根本没有过滤。在这些情况下，参与树的较高级别的节点数量可能不足以快速处理由叶子阶段生成的大量数据。任务调度在第四部分中讨论。

In both of these scenarios, multiple threads per worker performing the computation can alleviate this concurrency bottleneck to some degree. The engine can run a single sequence of operators (or pipeline) in multiple threads. Figure 4 shows how the optimizer is able to parallelize one section of a join.

在这两种情况下，每个工作节点上执行计算的多个线程可以在一定程度上缓解并发瓶颈。引擎可以在多个线程中运行单个运算符序列（或管道）。图4 显示了优化器如何并行化连接的一部分。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/08d6061e7fbe112f87aac684b804f990.png)
图4 - 与 图3 对应的已物化和优化的计划，显示了任务、管道和运算符。管道1 和 2 在多个线程之间并行化，以加速哈希连接的构建侧。

### D. Scheduling

The coordinator distributes plan stages to workers in the form of executable tasks, which can be thought of as single processing units. Then, the coordinator links tasks in one stage to tasks in other stages, forming a tree of processors linked to one another by shuffles. Data streams from stage to stage as soon as it is available.

协调器将计划阶段以可执行任务的形式分发给工作节点，可以将其视为单个处理单元。然后，协调器将一个阶段中的任务链接到其他阶段中的任务，形成一个由洗牌链接在一起的处理器树。数据在可用时立即从一个阶段流向另一个阶段。

A task may have multiple pipelines within it. A pipeline consists of a chain of operators, each of which performs a single, well-defined computation on the data. For example, a task performing a hash-join must contain at least two pipelines; one to build the hash table (build pipeline), and one to stream data from the probe side and perform the join (probe pipeline). When the optimizer determines that part of a pipeline would benefit from increased local parallelism, it can split up the pipeline and parallelize that part independently. Figure 4 shows how the build pipeline has been split up into two pipelines, one to scan data, and the other to build partitions of the hash table. Pipelines are joined together by a local in-memory shuffle.

一个任务可能包含多个管道。一个管道由一系列运算符组成，每个运算符在数据上执行单一且明确定义的计算。例如，执行哈希连接的任务必须包含至少两个管道：一个用于构建哈希表（构建管道），另一个用于流式传输来自探测端的数据并执行连接操作（探测管道）。当优化器确定管道的一部分会受益于增加本地并行性时，它可以将管道拆分并独立地并行化该部分。图4 显示了构建管道已被拆分成两个管道，一个用于扫描数据，另一个用于构建哈希表的分区。管道通过本地内存洗牌连接在一起。

To execute a query, the engine makes two sets of scheduling decisions. The first determines the order in which stages are scheduled, and the second determines how many tasks should be scheduled, and which nodes they should be placed on.

为了执行查询，引擎需要做出两组调度决策。第一组决策确定了阶段的调度顺序，第二组决策确定了应该调度多少个任务以及它们应该放置在哪些节点上。

1. Stage Scheduling: Presto supports two scheduling policies for stages: all-at-once and phased. All-at-once minimizes wall clock time by scheduling all stages of execution concurrently; data is processed as soon as it is available. This scheduling strategy benefits latency-sensitive use cases such as Interactive Analytics, Developer/Advertiser Analytics, and A/B Testing. Phased execution identifies all the strongly connected components of the directed data flow graph that must be started at the same time to avoid deadlocks and executes those in topological order. For example, if a hash-join is executed in phased mode, the tasks to schedule streaming of the left side will not be scheduled until the hash table is built. This greatly improves memory efficiency for the Batch Analytics use case.

When the scheduler determines that a stage should be scheduled according to the policy, it begins to assign tasks for that stage to worker nodes.

1. 阶段调度：Presto 支持两种阶段调度策略：一次性和分阶段。一次性调度通过同时调度所有执行阶段来最小化墙上时间；数据在可用时立即处理。这种调度策略有利于对延迟敏感的用例，例如交互式分析、开发者/广告商分析和 A/B 测试。分阶段执行识别有向数据流图的所有强连接组件，这些组件必须同时启动，以避免死锁，并按拓扑顺序执行它们。例如，如果以分阶段模式执行哈希连接，那么只有在构建哈希表后，才会安排调度左侧流式传输的任务。这极大地提高了批量分析用例的内存效率。

当调度程序确定根据策略应该调度一个阶段时，它开始分配该阶段的任务给工作节点。

2. Task Scheduling: The task scheduler examines the plan tree and classifies stages into leaf and intermediate stages. Leaf stages read data from connectors; while intermediate stages only process intermediate results from other stages.
    
3. 任务调度：任务调度器检查计划树并将阶段分类为叶子阶段和中间阶段。叶子阶段从连接器读取数据，而中间阶段仅处理来自其他阶段的中间结果。
    

Leaf Stages: For leaf stages, the task scheduler takes into account the constraints imposed by the network and connectors when assigning tasks to worker nodes. For example, sharednothing deployments require that workers be co-located with storage nodes. The scheduler uses the Connector Data Layout API to decide task placement under these circumstances. The A/B Testing use case requires predictable high-throughput, low-latency data reads, which are satisfied by the Raptor connector. Raptor is a storage engine optimized for Presto with a shared-nothing architecture that stores ORC files on flash disks and metadata in MySQL.

叶子阶段：对于叶子阶段，任务调度器在将任务分配给工作节点时考虑了网络和连接器所施加的约束。例如，共享无关的部署要求工作节点与存储节点共同部署。调度程序在这些情况下使用连接器数据布局 API 来决定任务的放置位置。A/B 测试用例需要可预测的高吞吐量、低延迟的数据读取，这可以通过 Raptor 连接器来满足。Raptor 是一种专为 Presto 优化的存储引擎，具有共享无关的架构，将 ORC 文件存储在闪存磁盘上，并将元数据存储在 MySQL 中。

Profiling shows that a majority of CPU time across our production clusters is spent decompressing, decoding, filtering and applying transformations to data read from connectors. This work is highly parallelizable, and running these stages on as many nodes as possible usually yields the shortest wall time. Therefore, if there are no constraints, and the data can be divided up into enough splits, a leaf stage task is scheduled on every worker node in the cluster. For the Facebook data warehouse deployments that run in shared-storage mode (i.e. all data is remote), every node in a cluster is usually involved in processing the leaf stage. This execution strategy can be network intensive.

性能分析显示，我们生产集群中大多数 CPU 时间都用于对从连接器读取的数据进行解压缩、解码、过滤和应用转换。这项工作高度可并行化，通常在尽可能多的节点上运行这些阶段会产生最短的墙上时间。因此，如果没有约束，并且数据可以分成足够多的分片，那么叶子阶段的任务会在集群中的每个工作节点上调度。对于以共享存储模式运行的 Facebook 数据仓库部署（即所有数据都是远程的），通常集群中的每个节点都会参与处理叶子阶段。这种执行策略可能会对网络产生较大压力。

The scheduler can also reason about network topology to optimize reads using a plugin-provided hierarchy. Network constrained deployments at Facebook can use this mechanism to express to the engine a preference for rack-local reads over rack-remote reads.

调度程序还可以根据网络拓扑来推理，以使用插件提供的层次结构来优化读取操作。在 Facebook 的受限网络部署中，可以使用这种机制来向引擎表达首选机架本地读取而不是机架远程读取的偏好。

Intermediate Stages: Tasks for intermediate stages can be placed on any worker node. However, the engine still needs to decide how many tasks should be scheduled for each stage. This decision is based on the connector configuration, the properties of the plan, the required data layout, and other deployment configuration. In some cases, the engine can dynamically change the number of tasks during execution. Section IV-E3 describes one such scenario.

中间阶段：中间阶段的任务可以放置在任何工作节点上。然而，引擎仍然需要决定每个阶段应该调度多少个任务。这个决策基于连接器配置、计划的属性、所需的数据布局以及其他部署配置。在某些情况下，引擎可以在执行过程中动态地改变任务的数量。第 IV-E3 节描述了一个这样的情景。

3. Split Scheduling: When a task in a leaf stage begins execution on a worker node, the node makes itself available to receive one or more splits (described in Section III). The information that a split contains varies by connector. When reading from a distributed file system, a split might consist of a file path and offsets to a region of the file. For the Redis key-value store, a split consists of table information, a key and value format, and a list of hosts to query, among other things.
    
4. 分片调度：当叶子阶段的任务在工作节点上开始执行时，该节点会使自己可用，以接收一个或多个分片（在第三部分描述）。分片包含的信息因连接器而异。当从分布式文件系统读取时，一个分片可能包括文件路径和文件区域的偏移量。对于 Redis 键值存储，一个分片包括表信息、键和值的格式，以及要查询的主机列表，等等。
    

Every task in a leaf stage must be assigned one or more splits to become eligible to run. Tasks in intermediate stages are always eligible to run, and finish only when they are aborted or all their upstream tasks are completed.

叶子阶段中的每个任务必须分配一个或多个分片，才能有资格运行。中间阶段中的任务始终有资格运行，只有在它们被中止或所有上游任务都已完成时才会结束。

Split Assignment: As tasks are set up on worker nodes, the coordinator starts to assign splits to these tasks. Presto asks connectors to enumerate small batches of splits, and assigns them to tasks lazily. This is a an important feature of Presto and provides several benefits:

分片分配：当任务在工作节点上设置时，协调器开始将分片分配给这些任务。Presto 会要求连接器枚举小批量的分片，并将它们惰性地分配给任务。这是 Presto 的一个重要功能，并提供了多个好处：

- Decouples query response time from the time it takes the connector to enumerate a large number of splits. For example, it can take minutes for the Hive connector to enumerate partitions and list files in each partition directory.
    
- Queries that can start producing results without processing all the data (e.g., simply selecting data with a filter) are frequently canceled quickly or complete early when a LIMIT clause is satisfied. In the Interactive Analytics use case, it is common for queries to finish before all the splits have even been enumerated.
    
- Workers maintain a queue of splits they are assigned to process. The coordinator simply assigns new splits to tasks with the shortest queue. Keeping these queues small allows the system to adapt to variance in CPU cost of processing different splits and performance differences among workers.
    
- Allows queries to execute without having to hold all their metadata in memory. This is important for the Hive connector, where queries may access millions of splits and can easily consume all available coordinator memory.
    
- 解耦了查询响应时间和连接器枚举大量分片所需的时间。例如,Hive连接器枚举分区和列出每个分区目录中的文件需要几分钟。
    
- 可以在不处理所有数据的情况下开始生成结果的查询（例如，仅使用过滤器选择数据）通常会在满足 LIMIT 子句时快速取消或提前完成。在交互式分析用例中，常见的情况是查询在所有分片甚至被枚举之前就完成了。
    
- 工作节点维护一个分配给它们处理的分片队列。协调器只是将新的分片分配给队列最短的任务。保持这些队列较小允许系统适应不同分片的处理 CPU 成本和工作节点性能差异的变化。
    
- 允许查询在不必将所有元数据保存在内存中的情况下执行。对于 Hive 连接器而言，这一点非常重要，因为查询可能会访问数百万个分片，并且很容易消耗掉所有可用的协调器内存。
    

These features are particularly useful for the Interactive Analytics and Batch ETL use cases, which run on the Facebook Hive-compatible data warehouse. It’s worth noting that lazy split enumeration can make it difficult to accurately estimate and report query progress.

这些功能特别适用于交互式分析和批量 ETL 用例，这些用例在 Facebook 的 Hive 兼容数据仓库上运行。值得注意的是，惰性的分片枚举可能会使准确估算和报告查询进度变得困难。

### E. Query Execution

1. Local Data Flow: Once a split is assigned to a thread, it is executed by the driver loop. The Presto driver loop is more complex than the popular Volcano (pull) model of recursive iterators [1], but provides important functionality. It is much more amenable to cooperative multi-tasking, since operators can be quickly brought to a known state before yielding the thread instead of blocking indefinitely. In addition, the driver can maximize work performed in every quanta by moving data between operators that can make progress without additional input (e.g., resuming computation of resource-intensive or explosive transformations). Every iteration of the loop moves data between all pairs of operators that can make progress.
    
2. 本地数据流：一旦分片分配给线程，它就会由驱动循环执行。Presto 的驱动循环比流行的 Volcano（pull）模型的递归迭代器更复杂，但提供了重要的功能。它更适合协作式多任务处理，因为运算符可以在将线程让出之前迅速达到已知状态，而不是无限期地阻塞。此外，驱动程序可以通过在可以在没有额外输入的情况下取得进展的运算符之间移动数据来最大化每个时间片内执行的工作（例如，恢复资源密集型或爆炸性转换的计算）。循环的每次迭代都在可以取得进展的所有运算符对之间移动数据。
    

The unit of data that the driver loop operates on is called a page, which is a columnar encoding of a sequence of rows. The Connector Data Source API returns pages when it is passed a split, and operators typically consume input pages, perform computation, and produce output pages. Figure5 shows the structure of a page in memory. The driver loop continuously moves pages between operators until the scheduling quanta is complete (discussed in Section IV-F1), or until operators cannot make progress.

驱动程序循环操作的数据单元称为页面，它是一系列行的列编码。当传递一个分片给 Connector 数据源 API 时，它会返回页面，而运算符通常会消耗输入页面，执行计算，并产生输出页面。图5 显示了内存中页面的结构。驱动循环在调度时间片完成（在第 IV-F1 节中讨论）或者运算符无法继续进行时，会不断地在运算符之间移动页面。


![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/ff33e7091365f51dab6cb2900e6ff133.png)
图5 - 页面中不同的块类型

2. Shuffles: Presto is designed to minimize end-to-end latency while maximizing resource utilization, and our internode data flow mechanism reflects this design choice. Presto uses in-memory buffered shuffles over HTTP to exchange intermediate results. Data produced by tasks is stored in buffers for consumption by other workers. Workers request intermediate results from other workers using HTTP long-polling. The server retains data until the client requests the next segment using a token sent in the previous response. This makes the acknowledgement implicit in the transfer protocol. The longpolling mechanism minimizes response time, especially when transferring small amounts of data. This mechanism offers much lower latency than other systems that persist shuffle data to disk [4], [21] and allows Presto to support latency-sensitive use cases such as Developer/Advertiser Analytics.
    
3. 洗牌（Shuffles）：Presto 的设计旨在最小化端到端的延迟，同时最大化资源利用率，我们的节点间数据流机制反映了这一设计选择。Presto 使用 HTTP 上的内存缓冲洗牌来交换中间结果。任务生成的数据存储在缓冲区中，供其他工作节点使用。工作节点使用 HTTP 长轮询请求来自其他工作节点的中间结果。服务器会保留数据，直到客户端使用前一次响应中发送的令牌请求下一个段。这使得确认在传输协议中是隐式的。长轮询机制可以最小化响应时间，特别是在传输少量数据时。这种机制提供了比将洗牌数据持久化到磁盘的其他系统低得多的延迟[4][21]，并允许 Presto 支持对延迟敏感的用例，如开发者/广告商分析。
    

The engine tunes parallelism to maintain target utilization rates for output and input buffers. Full output buffers cause split execution to stall and use up valuable memory, while underutilized input buffers add unnecessary processing overhead.

引擎会调整并行性以维持输出和输入缓冲区的目标利用率。完全填满的输出缓冲区会导致分片执行停滞，并消耗宝贵的内存，而未充分利用的输入缓冲区则会增加不必要的处理开销。

The engine continuously monitors the output buffer utilization. When utilization is consistently high, it lowers effective concurrency by reducing the number of splits eligible to be run. This has the effect of increasing fairness in sharing of network resources. It is also an important efficiency optimization when dealing with clients (either end-users or other workers) that are unable to consume data at the rate it is being produced. Without this functionality, slow clients running complex multistage queries could hold tens of gigabytes worth of buffer memory for long periods of time. This scenario is common even when a small amount of result data (∼10-50MB) is being downloaded by a BI or query authoring tool over slow connections in the Interactive Analytics use case.

引擎会持续监控输出缓冲区的利用率。当利用率持续较高时，它通过减少有资格运行的分片数量来降低有效并发性。这有助于提高共享网络资源的公平性。在处理无法以数据生成速率消耗数据的客户端（无论是终端用户还是其他工作节点）时，这也是一个重要的效率优化。如果没有这个功能，运行复杂多阶段查询的慢速客户端可能会长时间占用数十GB的缓冲内存。即使在交互式分析用例中，BI 或查询编写工具通过慢速连接下载少量结果数据（约 10-50 MB），这种情况也很常见。

On the receiver side, the engine monitors the moving average of data transferred per request to compute a target HTTP request concurrency that keeps the input buffers populated while not exceeding their capacity. This back pressure causes upstream tasks to slow down as their buffers fill up.

在接收端，引擎监控每个请求传输的数据的滑动平均值，以计算一个目标 HTTP 请求并发度，该并发度可保持输入缓冲区的填充，同时不超过其容量。这种背压会导致上游任务在其缓冲区填满时减速。

3. Writes: ETL jobs generally produce data that must be written to other tables. An important driver of write performance in a remote-storage environment is the concurrency with which the write is performed (i.e. the aggregate number of threads writing data through the Connector Data Sink API).
    
4. 写入：ETL 作业通常会生成必须写入其他表的数据。在远程存储环境中，写入性能的一个重要因素是执行写入操作的并发性（即通过 Connector 数据写入 API 写入数据的线程的总数）。
    

Consider the example of a Hive connector configured to use Amazon S3 for storage. Every concurrent write to S3 creates a new file, and hundreds of writes of a small aggregate amount of data are likely to create small files. Unless these small units of data can be later coalesced, they are likely to create unacceptably high overheads while reading (many slow metadata operations, and latency-bound read performance). However, using too little concurrency can decrease aggregate write throughput to unacceptable levels. Presto takes an adaptive approach again, dynamically increasing writer concurrency by adding tasks on more worker nodes when the engine determines that the stage producing data for the write exceeds a buffer utilization threshold (and a configurable perwriter data written threshold). This is an important efficiency optimization for the write-heavy Batch ETL use case.

考虑一个配置为使用 Amazon S3 存储的 Hive 连接器的示例。对 S3 的每个并发写入都会创建一个新文件，并且大量写入少量数据可能会创建小文件。除非这些小数据单元稍后可以合并，否则它们可能会在读取时创建不可接受的高开销（许多较慢的元数据操作和延迟约束的读取性能）。然而，使用太少的并发性可能会将聚合写入吞吐量降低到不可接受的水平。Presto 再次采取自适应方法，当引擎确定为写入生成数据的阶段超过了缓冲区利用率阈值（和可配置的每个写入器数据写入阈值）时，通过在更多的工作节点上添加任务来动态增加写入并发性。这对于写入密集型的批量 ETL 用例是一个重要的效率优化。

### F. Resource Management

One of the key features that makes Presto a good fit for multi tenant deployments is that it contains a fully-integrated fine grained resource management system. A single cluster can execute hundreds of queries concurrently, and maximize the use of CPU, IO, and memory resources.

使 Presto 适用于多租户部署的关键特性之一是它包含了一个完全集成的细粒度资源管理系统。单个集群可以同时执行数百个查询，并最大化使用 CPU、IO 和内存资源。

1. CPU Scheduling: Presto primarily optimizes for overall cluster throughput, i.e. aggregate CPU utilized for processing data. The local (node-level) scheduler additionally optimizes for low turnaround time for computationally inexpensive queries, and the fair sharing of CPU resources amongst queries with similar CPU requirements. A task’s resource usage is the aggregate thread CPU time given to each of its splits. To minimize coordination overhead, Presto tracks CPU resource usage at the task level and makes scheduling decisions locally.
    
2. CPU 调度：Presto 主要针对整个集群的吞吐量进行优化，即用于处理数据的 CPU 总利用率。本地（节点级）调度程序另外还优化计算成本较低的查询的低周转时间，以及在具有类似 CPU 需求的查询之间公平共享 CPU 资源。任务的资源使用是分配给其每个分片的线程 CPU 时间的总和。为了最小化协调开销，Presto 在任务级别跟踪 CPU 资源使用情况，并在本地做出调度决策。
    

Presto schedules many concurrent tasks on every worker node to achieve multi-tenancy and uses a cooperative multitasking model. Any given split is only allowed to run on a thread for a maximum quanta of one second, after which it must relinquish the thread and return to the queue. When output buffers are full (downstream stages cannot consume data fast enough), input buffers are empty (upstream stages cannot produce data fast enough), or the system is out of memory, the local scheduler simply switches to processing another task even before the quanta is complete. This frees up threads for runnable splits, helps Presto maximize CPU usage, and is highly adaptive to different query shapes. All of our use cases benefit from this granular resource efficiency.

Presto 在每个工作节点上调度许多并发任务以实现多租户，并使用协作多任务模型。每个给定的分片只允许在一个线程上运行最多一秒钟，之后必须释放线程并返回队列。当输出缓冲区已满（下游阶段无法快速消耗数据）、输入缓冲区为空（上游阶段无法快速产生数据）或系统内存不足时，本地调度程序甚至在时间片完成之前就切换到处理另一个任务。这释放了线程以运行可运行的分片，帮助 Presto 最大化 CPU 使用率，并对不同的查询形状高度自适应。我们所有的用例都受益于这种细粒度的资源效率。

When a split relinquishes a thread, the engine needs to decide which task (associated with one or more splits) to run next. Rather than predict the resources required to complete a new query ahead of time, Presto simply uses a task’s aggregate CPU time to classify it into the five levels of a multi-level feedback queue [8]. As tasks accumulate more CPU time, they move to higher levels. Each level is assigned a configurable fraction of the available CPU time. In practice, it is challenging to accomplish fair cooperative multi-tasking with arbitrary workloads. The I/O and CPU characteristics for splits vary wildly (sometimes even within the same task), and complex functions (e.g., regular expressions) can consume excessive amounts of thread time relative to other splits. Some connectors do not provide asynchronous APIs, and worker threads can be held for several minutes.

当一个分片释放一个线程时，引擎需要决定下一个要运行的任务（与一个或多个分片相关联）。Presto 并不是提前预测完成新查询所需的资源，而是简单地使用任务的累积 CPU 时间将其分类到多级反馈队列的五个级别中[8]。随着任务累积更多的 CPU 时间，它们会移动到更高的级别。每个级别都被分配了可配置的可用 CPU 时间的一部分。实际上，要在任意工作负载下实现公平的协作多任务处理是具有挑战性的。分片的 I/O 和 CPU 特性变化很大（有时甚至在同一个任务内也是如此），复杂的函数（例如正则表达式）相对于其他分片可能会消耗过多的线程时间。某些连接器不提供异步 API，工作线程可能会被占用数分钟。

The scheduler must be adaptive when dealing with these constraints. The system provides a low-cost yield signal, so that long running computations can be stopped within an operator. If an operator exceeds the quanta, the scheduler ‘charges’ actual thread time to the task, and temporarily reduces future execution occurrences. This adaptive behavior allows us to handle the diversity of query shapes in the Interactive Analytics and Batch ETL use cases, where Presto gives higher priority to queries with lowest resource consumption. This choice reflects the understanding that users expect inexpensive queries to complete quickly, and are less concerned about the turnaround time of larger, computationally-expensive jobs. Running more queries concurrently, even at the expense of more context-switching, results in lower aggregate queue time, since shorter queries exit the system quickly.

在处理这些约束时，调度程序必须具有适应性。系统提供了一个低代价的让出信号，以便可以在运算符内部停止长时间运行的计算。如果一个运算符超额，调度程序会向任务“收费”实际的线程时间，并暂时减少未来的执行次数。这种自适应行为使我们能够处理交互式分析和批处理 ETL 用例中的查询多样性，其中 Presto 给资源消耗最小的查询以更高优先级。这种选择反映了这样的理解：用户希望廉价的查询快速完成，而对大型计算密集型作业的周转时间不那么在意。即使以更多的上下文切换为代价，并发运行更多查询也可以导致较小的总队列时间，因为短查询可以快速退出系统。

2. Memory Management: Memory poses one of the main resource management challenges in a multi-tenant system like Presto. In this section we describe the mechanism by which the engine controls memory allocations across the cluster.
    
3. 内存管理：在像 Presto 这样的多租户系统中，内存是主要的资源管理挑战之一。在本节中，我们描述了引擎如何在整个集群中控制内存分配的机制。
    

Memory Pools: All non-trivial memory allocations in Presto must be classified as user or system memory, and reserve memory in the corresponding memory pool. User memory is memory usage that is possible for users to reason about given only basic knowledge of the system or input data (e.g., the memory usage of an aggregation is proportional to its cardinality). On the other hand, system memory is memory usage that is largely a byproduct of implementation decisions (e.g., shuffle buffers) and may be uncorrelated with query shape and input data volume.

内存池：在 Presto 中，所有非微不足道的内存分配都必须被归类为用户内存或系统内存，并在相应的内存池中保留内存。用户内存是用户可以根据对系统或输入数据的基本了解来推断的内存使用情况（例如，聚合的内存使用量与其基数成比例）。另一方面，系统内存主要是实现决策的副产品（例如，洗牌缓冲区），可能与查询形状和输入数据量不相关的内存使用情况。

The per-node and global user memory limits on queries are usually distinct; this enables a maximum level of permissible usage skew. Consider a 500 node cluster with 100GB of query memory available per node and a requirement that individual queries can use up to 5TB globally. In this case, 10 queries can concurrently allocate up to that amount of total memory. However, if we want to allow for a 2:1 skew (i.e. one partition of the query consumes 2x the median memory), the per-node query memory limit would have to be set to 20GB. This means that only 5 queries are guaranteed to be able to run without exhausting the available node memory.

查询的每个节点和全局用户内存限制通常是不同的；这允许最大级别的可允许使用偏差。考虑一个有 500 个节点的集群，每个节点有 100GB 的查询内存可用，并要求单个查询可以全局使用高达 5TB。在这种情况下，最多可以同时分配 10 个查询的总内存。然而，如果我们想允许 2:1 的偏差（即查询的一个分区消耗的内存是中位数内存的2倍），则每个节点的查询内存限制必须设置为 20 GB。这意味着只有 5 个查询能够保证能够运行而不会耗尽可用的节点内存。

It is important that we be able to run more than 5 queries concurrently on a 500-node Interactive Analytics or Batch ETL cluster. Given that queries in these clusters vary wildly in their memory characteristics (skew, allocation rate, and allocation temporal locality), it is unlikely that all five queries allocate up to their limit on the same worker node at any given point in time. Therefore, it is generally safe to overcommit the memory of the cluster as long as mechanisms exist to keep the cluster healthy when nodes are low on memory. There are two such mechanisms in Presto – spilling, and reserved pools.

对于一个有 500 个节点的交互式分析或批量 ETL 集群，能够同时运行超过 5 个查询非常重要。考虑到这些集群中的查询在内存特性（偏差、分配速率和分配时间局部性）方面变化很大，不太可能所有五个查询在任何给定时间点都在同一工作节点上分配到它们的限制。因此，通常情况下可以安全地超配集群的内存，只要存在机制来确保在节点内存不足时保持集群的健康状态。在 Presto 中有两种这样的机制 - 溢出和保留池。

Spilling: When a node runs out of memory, the engine invokes the memory revocation procedure on eligible tasks in ascending order of their execution time, and stops when enough memory is available to satisfy the last request. Revocation is processed by spilling state to disk. Presto supports spilling for hash joins and aggregations. However, we do not configure any of the Facebook deployments to spill. Cluster sizes are typically large enough to support several TBs of distributed memory, users appreciate the predictable latency of fully in memory execution, and local disks would increase hardware costs (especially in Facebook’s shared-storage deployments).

溢出：当一个节点的内存用尽时，引擎会按照它们的执行时间升序调用合格任务的内存吊销过程，并在满足最后一个请求所需的足够内存时停止。吊销是通过将状态溢出到磁盘来处理的。Presto 支持对哈希连接和聚合进行溢出。然而，我们没有配置 Facebook 的任何部署进行溢出。集群的规模通常足够大，可以支持数 TB 的分布式内存，用户喜欢完全基于内存执行的可预测的延迟，而本地磁盘会增加硬件成本（特别是在 Facebook 的共享存储部署中）。

Reserved Pool: If a node runs out of memory and the cluster is not configured to spill, or there is no revocable memory remaining, the reserved memory mechanism is used to unblock the cluster. The query memory pool on every node is further sub divided into two pools: general and reserved. When the general pool is exhausted on a worker node, the query using the most memory on that worker gets ‘promoted’ to the reserved pool on all worker nodes. In this state, the memory allocated to that query is counted towards the reserved pool rather than the general pool. To prevent deadlock (where different workers stall different queries) only a single query can enter the reserved pool across the entire cluster. If the general pool on a node is exhausted while the reserved pool is occupied, all memory requests from other tasks on that node are stalled. The query runs in the reserved pool until it completes, at which point the cluster unblocks all outstanding requests for memory. This is somewhat wasteful, as the reserved pool on every node must be sized to fit queries running up against the local memory limits. Clusters can be configured to instead kill the query that unblocks most nodes.

保留池：如果一个节点的内存用尽，而集群没有配置溢出，或者没有可吊销的内存剩余，那么将使用保留内存机制来解锁集群。每个节点上的查询内存池进一步分为两个池：通用池和保留池。当一个工作节点上的通用池用尽时，使用该节点上内存最多的查询会在所有工作节点上被“提升”到保留池。在这种状态下，分配给该查询的内存会计入保留池，而不是通用池。为了防止死锁（不同的工作节点阻塞了不同的查询），整个集群只能有一个查询进入保留池。如果一个节点上的通用池用尽，而保留池已经被占用，那么该节点上的其他任务的所有内存请求都会被阻塞。查询在保留池中运行，直到完成，此时集群解锁了所有未完成的内存请求。这有点浪费，因为每个节点上的保留池必须被调整大小以适应接近本地内存限制的查询。集群可以配置为杀死解锁大多数节点的查询。

### G. Fault Tolerance

Presto is able to recover from many transient errors using low-level retries. However, as of late 2018, Presto does not have any meaningful built-in fault tolerance for coordinator or worker node crash failures. Coordinator failures cause the cluster to become unavailable, and a worker node crash failure causes all queries running on that node to fail. Presto relies on clients to automatically retry failed queries.

截止到 2018 年底，Presto 对于协调器或工作节点崩溃故障没有任何有意义的内置容错性。协调器故障会导致集群不可用，而工作节点崩溃故障会导致在该节点上运行的所有查询失败。Presto 依赖客户端自动重试失败的查询来处理这些故障。

In production at Facebook, we use external orchestration mechanisms to run clusters in different availability modes depending on the use case. The Interactive Analytics and Batch ETL use cases run standby coordinators, while A/B Testing and Developer/Advertiser Analytics run multiple active clusters. External monitoring systems identify nodes that cause an unusual number of failures and remove them from clusters, and nodes that are remediated automatically re-join the cluster. Each of these mechanisms reduce the duration of unavailability to varying degrees, but cannot hide failures entirely.

在 Facebook 的生产环境中，我们使用外部编排机制来根据用例运行不同的可用性模式的集群。交互式分析和批量 ETL 用例运行备用协调器，而 A/B 测试和开发者/广告分析运行多个活动集群。外部监控系统会识别导致异常故障的节点并将其从集群中移除，而自动修复的节点会自动重新加入集群。这些机制各自降低了不可用的持续时间，但不能完全隐藏故障。

Standard checkpointing or partial-recovery techniques are computationally expensive, and difficult to implement in a system designed to stream results back to clients as soon as they are available. Replication-based fault tolerance mechanisms [6] also consume significant resources. Given the cost, the expected value of such techniques is unclear, especially when taking into account the node mean-time-to-failure, cluster sizes of ∼1000 nodes and telemetry data showing that most queries complete within a few hours, including Batch ETL. Other researchers have come to similar conclusions [17].

标准的检查点或部分恢复技术在计算上是昂贵的，并且难以在设计为在结果可用时立即向客户端流式传输的系统中实现。基于复制的容错机制[6]也会消耗大量资源。考虑到成本，这些技术的预期价值不清楚，特别是考虑到节点平均故障时间、集群规模约为 1000 个节点以及遥测数据显示大多数查询在几小时内完成，包括批量 ETL。其他研究人员也得出了类似的结论[17]。

However, we are actively working on improved fault tolerance for long running queries. We are evaluating adding optional checkpointing and limiting restarts to sub-trees of a plan that cannot be run in a pipelined fashion.

然而，我们正在积极研究针对长时间运行的查询的改进容错性。我们正在评估添加可选的检查点和将重启限制为无法以流水线方式运行的执行计划的子树。

## V. QUERY PROCESSING OPTIMIZATIONS

In this section, we describe a few important query processing optimizations that benefit most use cases.

在本节中，我们描述了一些对大多数用例都有益的重要查询处理优化。

### A. Working with the JVM

Presto is implemented in Java and runs on the Hotspot Java Virtual Machine (JVM). Extracting the best possible performance out of the implementation requires playing to the strengths and limitations of the underlying platform. Performance-sensitive code such as data compression or checksum algorithms can benefit from specific optimizations or CPU instructions. While there is no application-level mechanism to control how the JVM Just-In-Time (JIT) compiler generates machine code, it is possible to structure the code so that it can take advantage of optimizations provided by the JIT compiler, such as method inlining, loop unrolling, and intrinsics. We are exploring the use of Graal [22] in scenarios where the JVM is unable to generate optimal machine code, such as 128-bit math operations.

Presto 是用 Java 实现的，并运行在 Hotspot Java 虚拟机（JVM）上。从实现中提取出最佳性能需要充分利用底层平台的优势和限制。性能敏感的代码，如数据压缩或校验和算法，可以从特定的优化或 CPU 指令中受益。虽然没有应用级别的机制来控制 JVM 即时编译（JIT）编译器生成的机器代码，但可以结构化代码以便利用 JIT 编译器提供的优化，如方法内联、循环展开和内置函数。我们正在探索在 JVM 无法生成最佳机器代码的情况下使用 Graal [22]的可能性，比如 128 位的数学运算。

The choice of garbage collection (GC) algorithm can have dramatic effects on application performance and can even influence application implementation choices. Presto uses the G1 collector, which deals poorly with objects larger than a certain size. To limit the number of these objects, Presto avoids allocating objects or buffers bigger than the ‘humongous’ threshold and uses segmented arrays if necessary. Large and highly linked object graphs can also be problematic due to maintenance of remembered set structures in G1 [10]. Data structures in the critical path of query execution are implemented over flat memory arrays to reduce reference and object counts and make the job of the GC easier. For example, the HISTOGRAM aggregation stores the bucket keys and counts for all groups in a set of flat arrays and hash tables instead of maintaining independent objects for each histogram.

垃圾回收（GC）算法的选择可以对应用程序性能产生重大影响，甚至可能影响应用程序的实现选择。Presto 使用 G1 收集器，该收集器对大于某个大小的对象处理效果不佳。为了限制这些对象的数量，Presto 避免分配大于“巨大”阈值的对象或缓冲区，并在必要时使用分段数组。大型和高度连接的对象图也可能因 G1 中记忆集结构的维护而成为问题 [10]。在查询执行的关键路径中的数据结构是基于平面内存数组实现的，以减少引用和对象计数，并使 GC 的工作更容易。例如，HISTOGRAM 聚合将所有组的桶键和计数存储在一组平面数组和哈希表中，而不是维护每个直方图的独立对象。

### B. Code Generation

One of the main performance features of the engine is code generation, which targets JVM bytecode. This takes two forms:

引擎的主要性能特性之一是代码生成，它针对 JVM 字节码。代码生成有两种形式：

1. Expression Evaluation: The performance of a query engine is determined in part by the speed at which it can evaluate complex expressions. Presto contains an expression interpreter that can evaluate arbitrarily complex expressions that we use for tests, but is much too slow for production use evaluating billions of rows. To speed this up, Presto generates bytecode to natively deal with constants, function calls, references to variables, and lazy or short-circuiting operations.
    
2. 表达式评估：查询引擎的性能在一定程度上取决于它评估复杂表达式的速度。Presto 包含一个表达式解释器，可以评估任意复杂的表达式，我们在测试中使用它，但对于在生产环境中评估数十亿行的数据来说，速度太慢了。为了加速这一过程，Presto 生成字节码，以本地方式处理常量、函数调用、对变量的引用以及惰性或短路操作。
    
3. Targeting JIT Optimizer Heuristics: Presto generates bytecode for several key operators and operator combinations. The generator takes advantage of the engine’s superior knowledge of the semantics of the computation to produce bytecode that is more amenable to JIT optimization than that of a generic processing loop. There are three main behaviors that the generator targets:
    
4. 针对 JIT 优化器启发式规则：Presto 生成了一些关键操作符和操作符组合的字节码。生成器利用引擎对计算语义的卓越了解，生成的字节码更容易进行 JIT 优化，而不是使用通用处理循环的字节码。生成器针对三种主要行为进行了优化：
    

- Since the engine switches between different splits from distinct task pipelines every quanta (Section IV-F1), the JIT would fail to optimize a common loop based implementation since the collected profiling information for the tight processing loop would be polluted by other tasks or queries.
- Even within the processing loop for a single task pipeline, the engine is aware of the types involved in each computation and can generate unrolled loops over columns. Eliminating target type variance in the loop body causes the profiler to conclude that call sites are monomorphic, allowing it to inline virtual methods.
- As the bytecode generated for every task is compiled into a separate Java class, each can be profiled independently by the JIT optimizer. In effect, the JIT optimizer further adapts a custom program generated for the query to the data actually processed. This profiling happens independently at each task, which improves performance in environments where each task processes a different partition of the data. Furthermore, the performance profile can change over the lifetime of the task as the data changes (e.g., time-series data or logs), causing the generated code to be updated.
- 由于引擎在每个 quanta（Section IV-F1）切换不同任务流程中的不同分片，JIT 将无法优化常见的基于循环的实现，因为紧密处理循环的收集的性能分析信息将被其他任务或查询污染。
- 即使在单个任务流程的处理循环内，引擎也了解每个计算中涉及的类型，并且可以生成针对列的展开循环。在循环体内消除目标类型差异会导致分析器得出调用站点是单态的结论，从而使其能够内联虚拟方法。
- 由于为每个任务生成的字节码编译为单独的 Java 类，因此可以由 JIT 优化器独立对每个进行分析。实际上，JIT 优化器进一步使为查询生成的自定义程序适应实际处理的数据。此分析在每个任务中都是独立进行的，这在每个任务处理数据的不同分区的环境中提高了性能。此外，性能概况可能会随着任务的生命周期而发生变化，因为数据发生变化（例如，时间序列数据或日志），导致生成的代码得以更新。

Generated bytecode also benefits from the second order effects of inlining. The JVM is able to broaden the scope of optimizations, auto-vectorize larger parts of the computation, and can take advantage of frequency-based basic block layout to minimize branches. CPU branch prediction also becomes far more effective [7]. Bytecode generation improves the engine’s ability to store intermediate results in registers or caches rather than in memory [16].

生成的字节码还受益于内联的二阶效应。JVM 能够扩大优化的范围，自动向量化更大部分的计算，并可以利用基于频率的基本块布局来最小化分支。CPU 分支预测也变得更加有效。字节码生成提高了引擎将中间结果存储在寄存器或高速缓存中而不是内存中的能力。

### C. File Format Features

Scan operators invoke the Connector API with leaf split information and receive columnar data in the form of Pages. A page consists of a list of Blocks, each of which is a column with a flat in-memory representation. Using flat memory data structures is important for performance, especially for complex types. Pointer chasing, unboxing, and virtual method calls add significant overhead to tight loops.

扫描操作符使用叶子分片信息调用连接器 API，并以 Page 的形式接收列式数据。一个 Page 包含一个 Block 列表，每个 Block 是一个具有平面内存表示的列。使用平面内存数据结构对性能非常重要，特别是对于复杂类型。指针遍历、解引用和虚方法调用会在紧密循环中增加显著的开销。

Connectors such Hive and Raptor take advantage of specific file format features where possible [20]. Presto ships with custom readers for file formats that can efficiently skip data sections by using statistics in file headers/footers (e.g., minmax range headers and Bloom filters). The readers can convert certain forms of compressed data directly into blocks, which can be efficiently operated upon by the engine (Section V-E).

像 Hive 和 Raptor 这样的连接器会尽可能地利用特定的文件格式特性[20]。Presto 内置了对那些可以通过使用文件头/尾中的统计信息（例如 minmax 范围头和 Bloom 过滤器）来高效跳过数据部分的文件格式的自定义读取器。这些读取器可以将某些形式的压缩数据直接转换成块，引擎可以高效地操作这些块（参见 V-E）。

Figure 5 shows the layout of a page with compressed encoding schemes for each column. Dictionary-encoded blocks are very effective at compressing low-cardinality sections of data and run-length encoded (RLE) blocks compress repeated data. Several pages may share a dictionary, which greatly improves memory efficiency. A column in an ORC file can use a single dictionary for an entire ‘stripe’ (up to millions of rows).

图5 展示了一个页面的布局，其中包含了每列的压缩编码方案。字典编码块非常有效地压缩数据的低基数部分，而行程编码（RLE）块则压缩重复的数据。多个页面可以共享一个字典，这大大提高了内存利用率。ORC 文件中的一列可以使用一个字典来表示整个“条带”（最多可达数百万行）的数据。

### D. Lazy Data Loading

Presto supports lazy materialization of data. This functionality can leverage the columnar, compressed nature of file formats such as ORC, Parquet, and RCFile. Connectors can generate lazy blocks, which read, decompress, and decode data only when cells are actually accessed. Given that a large fraction of CPU time is spent decompressing and decoding and that it is common for filters to be highly selective, this optimization is highly effective when columns are infrequently accessed. Tests on a sample of production workload from the Batch ETL use case show that lazy loading reduces data fetched by 78%, cells loaded by 22% and total CPU time by 14%.

Presto 支持数据的惰性物化。这个功能可以利用文件格式（如 ORC、Parquet 和 RCFile）的列式、压缩的特性。连接器可以生成惰性加载的块，仅在实际访问单元格时才读取、解压缩和解码数据。考虑到大部分 CPU 时间都用于解压缩和解码，并且过滤器通常具有很高的选择性，当很少访问列时，这种优化非常有效。对批量 ETL 用例的生产工作负载样本进行的测试显示，延迟加载可以减少 78% 的数据获取、22% 的单元格加载以及 14% 的总 CPU 时间。

### E. Operating on Compressed Data

Presto operates on compressed data (i.e. dictionary and runlength-encoded blocks) sourced from the connector wherever possible. Figure 5 shows how these blocks are structured within a page. When a page processor evaluating a transformation or filter encounters a dictionary block, it processes all of the values in the dictionary (or the single value in a runlength-encoded block). This allows the engine to process the entire dictionary in a fast unconditional loop. In some cases, there are more values present in the dictionary than rows in the block. In this scenario the page processor speculates that the un-referenced values will be used in subsequent blocks. The page processor keeps track of the number of real rows produced and the size of the dictionary, which helps measure the effectiveness of processing the dictionary as compared to processing all the indices. If the number of rows is larger than the size of the dictionary it is likely more efficient to process the dictionary instead. When the page processor encounters a new dictionary in the sequence of blocks, it uses this heuristic to determine whether to continue speculating.

Presto 在尽可能的情况下对来自连接器的压缩数据（即字典和游程编码块）进行操作。图5 展示了这些块在页面内的结构。当评估转换或过滤器的页面处理器遇到字典块时，它会处理字典中的所有值（或游程编码块中的单个值）。这使得引擎可以在快速无条件的循环中处理整个字典。在某些情况下，字典中的值可能比块中的行数多。在这种情况下，页面处理器假设未引用的值将在后续块中使用。页面处理器跟踪生成的实际行数和字典的大小，这有助于衡量处理字典与处理所有索引相比的效率。如果行数大于字典的大小，那么处理字典可能更有效。当页面处理器在块序列中遇到新的字典时，它会使用这个启发式方法来确定是否继续进行假设。

Presto also leverages dictionary block structure when building hash tables (e.g., joins or aggregations). As the indices are processed, the operator records hash table locations for every dictionary entry in an array. If the entry is repeated for a subsequent index, it simply re-uses the location rather than re-computing it. When successive blocks share the same dictionary, the page processor retains the array to further reduce the necessary computation.

Presto 还在构建哈希表时（例如，连接或聚合操作）利用字典块结构。当处理索引时，运算符会记录每个字典条目的哈希表位置，并将其存储在一个数组中。如果在后续索引中出现相同的条目，它会直接重用位置，而不是重新计算。当连续的块共享相同的字典时，页面处理器会保留该数组，以进一步减少所需的计算量。

Presto also produces intermediate compressed results during execution. The join processor, for example, produces dictionary or run-length-encoded blocks when it is more efficient to do so. For a hash join, when the probe side of the join looks up keys in the hash table, it records value indices into an array rather than copying the actual data. The operator simply produces a dictionary block where the index list is that array, and the dictionary is a reference to the block in the hash table.

Presto 在执行过程中还会生成中间压缩结果。例如，连接处理器在更高效的情况下会生成字典或游程编码块。对于哈希连接，当连接的探测侧在哈希表中查找键时，它会记录值的索引到一个数组中，而不是复制实际的数据。运算符只需生成一个字典块，其中索引列表是该数组，而字典是对哈希表中块的引用。

## VI. PERFORMANCE

In this section, we present performance results that demonstrate the impact of some of the main design decisions described in this paper.

在本节中，我们将展示性能结果，以演示本文中描述的一些主要设计决策的影响。

### A. Adaptivity

Within Facebook, we run several different connectors in production to allow users to process data stored in various internal systems. Table 1 outlines the connectors and deployments that are used to support the use cases outlined in Section II.

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/65d6890b6abc3343a020928ee2d71e30.png)

在 Facebook 内部，我们运行了几种不同的连接器，以允许用户处理存储在各种内部系统中的数据。表1 概述了用于支持第 2 节所述用例的连接器和部署。

To demonstrate how Presto adapts to connector characteristics, we compare runtimes for queries from the TPC-DS benchmark at scale factor 30TB. Presto is capable of running all TPC-DS queries, but for this experiment we select a lowmemory subset that does not require spilling.

为了演示 Presto 如何适应连接器的特性，我们比较了在 30TB 规模尺度下运行 TPC-DS 基准的查询运行时间。Presto 能够运行所有 TPC-DS 查询，但对于这个实验，我们选择了一个不需要溢出的低内存子集。

We use Presto version 0.211 with internal variants of the Hive/HDFS and Raptor connectors. Raptor is a sharednothing storage engine designed for Presto. It uses MySQL for metadata and stores data on local flash disks in ORC format. Raptor supports complex data organization (sorting, bucketing, and temporal columns), but for this experiment our data is randomly partitioned. The Hive connector uses an internal service similar to the Hive Metastore and accesses files encoded in an ORC-like format on a remote distributed filesystem that is functionally similar to HDFS (i.e., a shared-storage architecture). Performance characteristics of these connector variants are similar to deployments on public cloud providers.

我们使用 Presto 0.211 版本以及 Hive/HDFS 和 Raptor 连接器的内部变体。Raptor 是专为 Presto 设计的一个共享无存储引擎。它使用 MySQL 来存储元数据，并将数据存储在 ORC 格式的本地闪存磁盘上。Raptor 支持复杂的数据组织（排序、分桶和时间列），但在这个实验中，我们的数据是随机分区的。Hive 连接器使用类似于 Hive Metastore 的内部服务，并访问编码为 ORC 类似格式的文件，这些文件存储在功能上类似于 HDFS 的远程分布式文件系统中（即共享存储架构）。这些连接器变种的性能特性与公共云提供商的部署相似。

Every query is run with three settings on a 100-node test cluster: (1) Data stored in Raptor with table shards randomly distributed between nodes. (2) Data stored in Hive/HDFS with no statistics. (3) Data stored in Hive/HDFS along with table and column statistics. Presto’s optimizer can make costbased decisions about join order and join strategy when these statistics are available. Every node is configured with a 28- core IntelTM XeonTM E5-2680 v4 CPU running at 2.40GHz, 1.6TB of flash storage and 256GB of DDR4 RAM.

每个查询在一个由 100 个节点组成的测试集群上以三种设置运行：（1）数据存储在 Raptor 中，表分片随机分布在节点之间。 （2）数据存储在 Hive/HDFS 中，没有统计信息。 （3）数据存储在 Hive/HDFS 中，并包括表和列统计信息。只要这些统计信息可用，Presto 的优化器就可以基于成本做出关于连接顺序和连接策略的决策。每个节点配置了 28 核的 IntelTM XeonTM E5-2680 v4 CPU，运行速度为 2.40GHz，1.6TB 的闪存存储和 256GB 的 DDR4 RAM 内存。

Figure 6 shows that Presto query runtime is greatly impacted by the characteristics of connectors. With no change to the query or cluster configuration, Presto is able to adapt to the connector by taking advantage of its characteristics, including throughput, latency, and the availability of statistics. It also demonstrates how a single Presto cluster can serve both as a traditional enterprise data warehouse (that data must be ingested into) and also a query engine over a Hadoop data warehouse. Data engineers at Facebook frequently use Presto to perform exploratory analysis over the Hadoop warehouse and then load aggregate results or frequently-accessed data into Raptor for faster analysis and low-latency dashboards.

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/1eda013a299e129ad0be6dee152aa09f.png)

图6 显示，Presto 查询运行时间受连接器特性的影响很大。在没有更改查询或集群配置的情况下，Presto 能够根据连接器的特性，包括吞吐量、延迟和统计信息的可用性来进行适应。它还演示了单个 Presto 集群可以同时作为传统企业数据仓库（需要将数据导入其中）和基于 Hadoop 数据仓库的查询引擎。Facebook 的数据工程师经常使用 Presto 在 Hadoop 数据仓库上执行探索性分析，然后将汇总结果或经常访问的数据加载到 Raptor 中，以便进行更快的分析和低延迟的仪表板查询。

### B. Flexibility

Presto’s flexibility is in large part due to its low-latency data shuffle mechanism in conjunction with a Connector API that supports performant processing of large volumes of data. Figure 7 shows a distribution of query runtimes from production deployments of the selected use cases. We include only queries that are successful and actually read data from storage. The results demonstrate that Presto can be configured to effectively serve web use cases with strict latency requirements (20- 100ms) as well as programmatically scheduled ETL jobs that run for several hours.

Presto 的灵活性在很大程度上归功于其低延迟的数据传输机制，以及支持高性能处理大量数据的Connector API。图7 显示了选定用例的生产部署中查询运行时间的分布。我们只包括成功并实际从存储中读取数据的查询。结果表明，Presto 可以配置为有效地满足对延迟要求严格的 Web 用例（20-100 毫秒），以及运行数小时的程序化调度的 ETL 作业。

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/dc4ec25dbfcb00b71a5a71dda5fdb7f5.png)

### C. Resource Management

Presto’s integrated fine-grained resource management system allows it to quickly move CPU and memory resources between queries to maximize resource efficiency in multi-tenant clusters. Figure 8 shows a four hour trace of CPU and concurrency metrics from one of our Interactive Analytics clusters. Even as demand drops from a peak of 44 queries to a low of 8 queries, Presto continues to utilize an average of ∼90% CPU across worker nodes. It is also worth noting that the scheduler prioritizes new and inexpensive workloads as they arrive to maintain responsiveness (Section IV-F1). It does this by allocating large fractions of cluster-wide CPU to new queries within milliseconds of them being admitted.

![](https://raw.githubusercontent.com/luckypeak-xyz/images/main/d82194f82ee5f290c0ac7d8100f06adc.png)

Presto 的集成细粒度资源管理系统允许它在多租户集群中快速在查询之间移动 CPU 和内存资源，以最大限度地提高资源效率。图8 显示了我们的一个交互式分析集群中四小时的 CPU 和并发指标追踪。即使需求从峰值 44 个查询下降到最低 8 个查询，Presto 仍继续在工作节点上利用平均约 90%的 CPU。同样值得注意的是，调度程序在新的廉价工作负载到达时将其优先级提高以维持响应能力(第 IV-F1 节)。它通过在查询被允许的几毫秒内将集群范围内的大部分 CPU 分配给新查询来实现这一点。

## VII. ENGINEERING LESSONS

Presto has been developed and operated as a service by a small team at Facebook since 2013. We observed that some engineering philosophies had an outsize impact on Presto’s design through feedback loops in a rapidly evolving environment:

Presto 自 2013 年以来一直由 Facebook 的一个小团队开发和运营。我们观察到一些工程哲学通过在不断发展的环境中的反馈循环对 Presto 的设计产生了巨大影响：

Adaptiveness over configurability: As a complex multitenant query engine that executes arbitrary user defined computation, Presto must be adaptive not only to different query characteristics, but also combinations of characteristics. For example, until Presto had end-to-end adaptive backpressure (Section IV-E2), large amounts of memory and CPU was utilized by a small number of jobs with slow clients, which adversely affected latency-sensitive jobs that were running concurrently. Without adaptiveness, it would be necessary to narrowly partition workloads and tune configuration for each workload independently. That approach would not scale to the wide variety of query shapes that we see in production.

自适应性优于可配置性：作为一个复杂的多租户查询引擎，执行任意用户定义的计算，Presto 必须适应不同的查询特性，以及这些特性的组合。例如，直到 Presto 具备了端到端的自适应背压功能（见第 IV-E2 节），大量的内存和 CPU 资源被少数几个与慢客户端有关的作业所利用，这对同时运行的对延迟要求敏感的作业产生了不利影响。如果没有自适应性，就需要狭窄地分割工作负载，并为每个工作负载独立地调整配置。这种方法不适用于我们在生产中看到的各种查询形状。

Effortless instrumentation: Presto exposes fine-grained performance statistics at the query and node level. We maintain our own libraries for efficient statistics collection which use flat-memory for approximate data structures. It is important to encourage observable system design and allow engineers to instrument and understand the performance of their code. Our libraries make adding statistics as easy as annotating a method. As a consequence, the median Presto worker node exports ∼10,000 real-time performance counters, and we collect and store operator level statistics (and merge up to task and stage level) for every query. Our investment in telemetry tooling allows us to be data-driven when optimizing the system.

无缝的插桩化：Presto 在查询和节点级别提供了细粒度的性能统计信息。我们维护了用于高效统计信息收集的自己的库，这些库使用扁平内存来进行近似数据结构。鼓励可观察的系统设计并允许工程师插桩化和了解其代码的性能是很重要的。我们的库使添加统计信息变得像注释方法一样容易。因此，中位数的 Presto 工作节点导出了大约 10,000 个实时性能计数器，我们为每个查询收集和存储操作员级别的统计信息（并向上合并到任务和阶段级别）。我们在遥测工具方面的投资使我们能够在优化系统时依赖数据。

Static configuration: Operational issues in a complex system like Presto are difficult to root cause and mitigate quickly. Configuration properties can affect system performance in ways that are hard to reason about, and we prioritize being able to understand the state of the cluster over the ability to change configuration quickly. Unlike several other systems at Facebook, Presto uses static rather than dynamic configuration wherever possible. We developed our own configuration library, which is designed to fail ‘loudly’ by crashing at startup if there are any warnings; this includes unused, duplicated, or conflicting properties. This model poses its own set of challenges. However, with a large number of clusters and configuration sets, it is more efficient to shift complexity from operational investigations to the deployment process/tooling.

静态配置：在像 Presto 这样的复杂系统中，操作问题很难迅速找出原因并进行快速缓解。配置属性可能以难以理解的方式影响系统性能，我们优先考虑能够理解集群状态而不是快速更改配置的能力。与 Facebook 的其他一些系统不同，Presto 在尽可能的地方使用静态而不是动态配置。我们开发了自己的配置库，该库的设计是在启动时如果存在任何警告就会“大声”崩溃；这包括未使用、重复或冲突的属性。这种模型带来了自己一套挑战。然而，对于大量的集群和配置集，将复杂性从业务调查转移到部署过程/工具更加高效。

Vertical integration: Like other engineering teams, we design custom libraries for components where performance and efficiency are important. For example, custom file-format readers allow us to use Presto-native data structures end-to-end and avoid conversion overhead. However, we observed that the ability to easily debug and control library behaviors is equally important when operating a highly multi-threaded system that performs arbitrary computation in a long-lived process.

垂直整合：与其他工程团队一样，我们为性能和效率很重要的组件设计了自定义库。例如，自定义文件格式读取器允许我们端到端使用 Presto 本机数据结构，避免了转换开销。然而，我们观察到在操作高度多线程的、在长时间运行的进程中执行任意计算的系统时，轻松调试和控制库行为的能力同样重要。

Consider an example of a recent production issue. Presto uses the Java built-in gzip library. While debugging a sequence of process crashes, we found that interactions between glibc and the gzip library (which invokes native code) caused memory fragmentation. For specific workload combinations, this caused large native memory leaks. To address this, we changed the way we use the library to influence the right cache flushing behavior, but in other cases we have gone as far as writing our own libraries for compression formats.

考虑一个最近的生产问题的例子。Presto 使用了 Java 内置的 gzip 库。在调试一系列进程崩溃的情况下，我们发现 glibc 与 gzip 库（调用本地代码）之间的交互导致了内存碎片化。对于特定的工作负载组合，这会导致大量的本机内存泄漏。为了解决这个问题，我们改变了使用库的方式，以影响正确的缓存刷新行为，但在其他情况下，我们甚至写了自己的压缩格式库。

Custom libraries can also improve developer efficiency – reducing the surface area for bugs by only implementing necessary features, unifying configuration management, and supporting detailed instrumentation to match our use case.

自定义库还可以提高开发效率，通过仅实现必要的功能减少了错误的表面积，统一配置管理，并支持详细的插桩化以匹配我们的用例。

## VIII. RELATED WORK

Systems that run SQL against large data sets have become popular over the past decade. Each of these systems present a unique set of tradeoffs. A comprehensive examination of the space is outside the scope of this paper. Instead, we focus on some of the more notable work in the area.

在过去的十年中，运行 SQL 查询的大数据系统变得越来越流行。每个这些系统都提供了一组独特的权衡方案。本文不打算对该领域进行全面的探讨。相反，我们将重点关注该领域一些更显著的工作。

Apache Hive [21] was originally developed at Facebook to provide a SQL-like interface over data stored in HDFS, and executes queries by compiling them into MapReduce [9] or Tez [18] jobs. Spark SQL [4] is a more modern system built on the popular Spark engine [23], which addresses many of the limitations of MapReduce. Spark SQL can run large queries over multiple distributed data stores, and can operate on intermediate results in memory. However, these systems do not support end-to-end pipelining, and usually persist data to a filesystem during inter-stage shuffles. Although this improves fault tolerance, the additional latency causes such systems to be a poor fit for interactive or low-latency use cases.

Apache Hive [21] 最初由 Facebook 开发，用于在 HDFS 中存储的数据上提供类似 SQL 的接口，并通过将查询编译成 MapReduce [9]或 Tez [18]作业来执行查询。Spark SQL [4] 是一个构建在流行的 Spark 引擎 [23]上的更现代的系统，它解决了 MapReduce 的许多局限性。Spark SQL 可以在多个分布式数据存储上运行大型查询，并且可以在内存中处理中间结果。然而，这些系统不支持端到端的流水线处理，并且通常会在跨阶段洗牌期间将数据持久化到文件系统中。尽管这提高了容错性，但额外的延迟使这些系统不适合交互式或低延迟的用例。

Products like Vertica [15], Teradata, Redshift, and Oracle Exadata can read external data to varying degrees. However, they are built around an internal data store and achieve peak performance when operating on data loaded into the system. Some systems take the hybrid approach of integrating RDBMS-style and MapReduce execution, such as Microsoft SQL Server Polybase [11] (for unstructured data) and Hadapt [5] (for performance). Apache Impala [14] can provide interactive latency, but operates within the Hadoop ecosystem. In contrast, Presto is data source agnostic. Administrators can deploy Presto with a vertically-integrated data store like Raptor, but can also configure Presto to query data from a variety of systems (including relational/NoSQL databases, proprietary internal services and stream processing systems) with low overhead, even within a single Presto cluster.

诸如 Vertica [15]、Teradata、Redshift 和 Oracle Exadata 等产品可以在不同程度上读取外部数据。然而，它们都是围绕内部数据存储构建的，当操作加载到系统中的数据时，可以实现最佳性能。一些系统采用了集成 RDBMS 风格和 MapReduce 执行的混合方法，比如 Microsoft SQL Server Polybase [11]（用于非结构化数据）和 Hadapt [5]（用于性能）。Apache Impala [14] 可以提供交互式的延迟，但在 Hadoop 生态系统内运行。相比之下，Presto 不受数据源的限制。管理员可以部署 Presto 与 Raptor 等垂直集成的数据存储一起使用，但也可以配置 Presto 以从各种系统（包括关系型/NoSQL 数据库、专有内部服务和流处理系统）查询数据，而且在单个 Presto 集群内开销很低。

Presto builds on a rich history of innovative techniques developed by the systems and database community. It uses techniques similar to those described by Neumann [16] and Diaconu et al. [12] on compiling query plans to significantly speed up query processing. It operates on compressed data where possible, using techniques from Abadi et al. [3], and generates compressed intermediate results. It can select the most optimal layout from multiple projections a la Vertica and ` C-Store [19] and uses strategies similar to Zhou et al. [24] to minimize shuffles by reasoning about plan properties.

Presto 基于系统和数据库社区开发的创新技术丰富的历史。它使用了类似于 Neumann [16]和Diaconu 等人 [12]在编译查询计划以显著加速查询处理方面所描述的技术。它在可能的情况下操作压缩数据，使用了 Abadi 等人 [3]的技术，并生成了压缩的中间结果。它可以从多个投影中选择最优的布局，类似于 Vertica 和 C-Store [19]，并使用了与 Zhou 等人 [24]类似的策略，通过推断计划属性来最小化洗牌。

## IX. CONCLUSION

In this paper, we presented Presto, an open-source MPP SQL query engine developed at Facebook to quickly process large data sets. Presto is designed to be flexible; it can be configured for high-performance SQL processing in a variety of use cases. Its rich plugin interface and Connector API make it extensible, allowing it to integrate with various data sources and be effective in many environments. The engine is also designed to be adaptive; it can take advantage of connector features to speed up execution, and can automatically tune read and write parallelism, network I/O, operator heuristics, and scheduling to the characteristics of the queries running in the system. Presto’s architecture enables it to service workloads that require very low latency and also process expensive, longrunning queries efficiently.

Presto allows organizations like Facebook to deploy a single SQL system to deal with multiple common analytic use cases and easily query multiple storage systems while also scaling up to ∼1000 nodes. Its architecture and design have found a niche within the crowded SQL-on-Big-Data space. Adoption at Facebook and in the industry is growing quickly, and our open-source community continues to remain engaged.

在本文中，我们介绍了 Presto，这是一个由 Facebook 开发的开源 MPP SQL 查询引擎，用于快速处理大数据集。Presto 的设计灵活，可以在各种用例中配置为高性能的 SQL 处理工具。其丰富的插件接口和 Connector API 使其具有可扩展性，可以集成各种数据源，并在许多环境中发挥作用。该引擎还被设计为自适应的，它可以利用连接器功能加速执行，并可以自动调整读写并行性、网络 I/O、操作器启发式和调度，以适应系统中运行的查询的特性。Presto 的架构使其能够处理需要极低延迟以及高效处理昂贵的长时间运行查询的工作负载。

Presto 允许像 Facebook 这样的组织部署单个 SQL 系统来处理多个常见的分析用例，并轻松查询多个存储系统，同时还可以扩展到∼1000 个节点。其架构和设计在拥挤的 SQL-on-Big-Data 领域找到了一席之地。在 Facebook 和整个行业中，对 Presto 的采用正在迅速增长，我们的开源社区也继续保持活跃。

## REFERENCES

[1] Volcano - An Extensible and Parallel Query Evaluation System. IEEE Transactions on Knowledge and Data Engineering, 6(1):120–135, 1994.

[2] SQL – Part 1: Framework (SQL/Framework). ISO/IEC 9075-1:2016, International Organization for Standardization, 2016.

[3] D. Abadi, S. Madden, and M. Ferreira. Integrating Compression and Execution in Column-Oriented Database Systems. In SIGMOD, 2006.

[4] M. Armbrust, R. S. Xin, C. Lian, Y. Huai, D. Liu, J. K. Bradley, X. Meng, T. Kaftan, M. J. Franklin, A. Ghodsi, and M. Zaharia. Spark SQL: Relational Data Processing in Spark. In SIGMOD, 2015.

[5] K. Bajda-Pawlikowski, D. J. Abadi, A. Silberschatz, and E. Paulson. Efficient Processing of Data Warehousing Queries in a Split Execution Environment. In SIGMOD, 2011.

[6] M. Balazinska, H. Balakrishnan, S. Madden, and M. Stonebraker. Faulttolerance in the borealis distributed stream processing system. In SIGMOD, 2005.

[7] B. Chattopadhyay, L. Lin, W. Liu, S. Mittal, P. Aragonda, V. Lychagina, Y. Kwon, and M. Wong. Tenzing A SQL Implementation On The MapReduce Framework. In PVLDB, volume 4, pages 1318–1327, 2011.

[8] F. J. Corbato, M. Merwin-Daggett, and R. C. Daley. An Experimental ´ Time-Sharing System. In Proceedings of the Spring Joint Computer Conference, 1962.

[9] J. Dean and S. Ghemawat. MapReduce: Simplified Data Processing on Large Clusters. In OSDI, 2004.

[10] D. Detlefs, C. Flood, S. Heller, and T. Printezis. Garbage-first Garbage Collection. In ISMM, 2004.

[11] D. J. DeWitt, A. Halverson, R. V. Nehme, S. Shankar, J. Aguilar-Saborit, A. Avanes, M. Flasza, and J. Gramling. Split Query Processing in Polybase. In SIGMOD, 2013.

[12] C. Diaconu, C. Freedman, E. Ismert, P.-A. Larson, P. Mittal, R. Stonecipher, N. Verma, and M. Zwilling. Hekaton: SQL Server’s MemoryOptimized OLTP Engine. In SIGMOD, 2013.

[13] G. Graefe. The Cascades Framework for Query Optimization. IEEE Data Engineering Bulletin, 18(3):19–29, 1995.

[14] M. Kornacker, A. Behm, V. Bittorf, T. Bobrovytsky, C. Ching, A. Choi, J. Erickson, M. Grund, D. Hecht, M. Jacobs, I. Joshi, L. Kuff, D. Kumar, A. Leblang, N. Li, I. Pandis, H. Robinson, D. Rorke, S. Rus, J. Russell, D. Tsirogiannis, S. Wanderman-Milne, and M. Yoder. Impala: A Modern, Open-Source SQL Engine for Hadoop. In CIDR, 2015.

[15] A. Lamb, M. Fuller, R. Varadarajan, N. Tran, B. Vandiver, L. Doshi, and C. Bear. The Vertica Analytic Database: C-Store 7 Years Later. PVLDB, 5(12):1790–1801, 2012.

[16] T. Neumann. Efficiently Compiling Efficient Query Plans for Modern Hardware. PVLDB, 4(9):539–550, 2011.

[17] A. Rasmussen, M. Conley, G. Porter, R. Kapoor, A. Vahdat, et al. Themis: An I/O-Efficient MapReduce. In SoCC, 2012.

[18] B. Saha, H. Shah, S. Seth, G. Vijayaraghavan, A. Murthy, and C. Curino. Apache Tez: A Unifying Framework for Modeling and Building Data Processing Applications. In SIGMOD, 2015.

[19] M. Stonebraker, D. J. Abadi, A. Batkin, X. Chen, M. Cherniack, M. Ferreira, E. Lau, A. Lin, S. Madden, E. O’Neil, et al. C-Store: A Column-oriented DBMS. In VLDB, 2005.

[20] D. Sundstrom. Even faster: Data at the speed of Presto ORC, 2015. [https://code.facebook.com/posts/370832626374903/even-faster-data-at-the-speed-of-presto-orc/](https://code.facebook.com/posts/370832626374903/even-faster-data-at-the-speed-of-presto-orc/).

[21] A. Thusoo, J. S. Sarma, N. Jain, Z. Shao, P. Chakka, N. Zhang, S. Anthony, H. Liu, and R. Murthy. Hive A Petabyte Scale Data Warehouse Using Hadoop. In ICDE, 2010.

[22] T. Wurthinger, C. Wimmer, A. W ¨ oß, L. Stadler, G. Duboscq, C. Humer, ¨ G. Richards, D. Simon, and M. Wolczko. One VM to Rule Them All. In ACM Onward! 2013, pages 187–204. ACM, 2013.

[23] M. Zaharia, M. Chowdhury, T. Das, A. Dave, J. Ma, M. McCauly, M. J. Franklin, S. Shenker, and I. Stoica. Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing. In NSDI, 2012.

[24] J. Zhou, P. A. Larson, and R. Chaiken. Incorporating partitioning and parallel plans into the scope optimizer. In ICDE, 2010.