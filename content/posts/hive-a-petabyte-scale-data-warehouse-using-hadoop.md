---
author:
  name: ruifeng
  link: https://blog.luckypeak.xyz
  email: me@luckypeak.xyz
  avatar: /safari-pinned-tab.svg
title: 【译】Hive – A Petabyte Scale Data Warehouse Using Hadoop
subtitle: ""
date: 2023-09-23
lastmod: 2023-09-23
categories:
  - 大数据
tags:
  - Hive
draft: false
---
## Abstract

The size of data sets being collected and analyzed in the industry for business intelligence is growing rapidly, making traditional warehousing solutions prohibitively expensive. Hadoop [^1] is a popular open-source map-reduce implementation which is being used in companies like Yahoo, Facebook etc. to store and process extremely large data sets on commodity hardware. However, the map-reduce programming model is very low level and requires developers to write custom programs which are hard to maintain and reuse. In this paper, we present _Hive_, an open-source data warehousing solution built on top of Hadoop. Hive supports queries expressed in a SQL-like declarative language - _HiveQL_, which are compiled into map- reduce jobs that are executed using Hadoop. In addition, HiveQL enables users to plug in custom map-reduce scripts into queries. The language includes a type system with support for tables containing primitive types, collections like arrays and maps, and nested compositions of the same. The underlying IO libraries can be extended to query data in custom formats. Hive also includes a system catalog _- Metastore_ – that contains schemas and statistics, which are useful in data exploration, query optimization and query compilation. In Facebook, the Hive warehouse contains tens of thousands of tables and stores over 700TB of data and is being used extensively for both reporting and ad-hoc analyses by more than 200 users per month.

行业中为商业智能（business intelligence）收集和分析的数据集的规模正在迅速增长，使传统的数据仓库解决方案变得昂贵难负担。Hadoop [^1] 是一个流行的开源 MapReduce 实现，被雅虎、Facebook 等公司广泛使用，以在通用硬件上存储和处理极大规模的数据集（extremely large data sets）。然而，MapReduce 编程模型过于低级，需要开发人员编写自定义程序，这些程序不易维护和重用。本文介绍 _Hive_，这是一个建立在 Hadoop 之上的开源数据仓库解决方案。Hive 支持使用类 SQL 的声明式语言 _HiveQL_ 进行查询，这些查询会被编译成 MapReduce 作业在 Hadoop 上执行。此外，HiveQL 还使得用户可以在查询中嵌入自定义的 MapReduce 脚本。该语言包含一个类型系统,支持包含基本类型、数组和映射等集合类型，以及同种类型的嵌套组合的表。底层的 IO 库可以被扩展以查询自定义格式的数据。Hive 还包括一个系统目录 _Metastore_，包含了对数据探索、查询优化和编译有用的模式和统计信息。在 Facebook，Hive 数据仓库包含数万张表，存储了超过 700TB 的数据，并被超过 200 名用户每月广泛用于报告和临时分析。

## I. Introduction

Scalable analysis on large data sets has been core to the functions of a number of teams at Facebook - both engineering and non-engineering. Apart from ad hoc analysis and business intelligence applications used by analysts across the company, a number of Facebook products are also based on analytics. These products range from simple reporting applications like Insights for the Facebook Ad Network, to more advanced kind such as Facebook's Lexicon product [^2]. As a result a flexible infrastructure that caters to the needs of these diverse applications and users and that also scales up in a cost effective manner with the ever increasing amounts of data being generated on Facebook, is critical. Hive and Hadoop are the technologies that we have used to address these requirements at Facebook.

Facebook 的许多团队，包括工程和非工程团队，核心职能之一是对大数据集的可扩展分析。除了全公司分析师使用的 adhoc 分析和商业智能应用之外，许多 Facebook 产品也基于分析。这些产品范围从简单的报告应用程序，如 Facebook 广告网络的 Insights，到更高级的产品，如 Facebook 的 Lexicon 产品[^2]。因此，一个灵活的基础架构，可以满足这些不同应用程序和用户的需求，并且随着 Facebook 上生成的数据量不断增加而以合算的方式扩展,对我们来说至关重要。Hive 和 Hadoop 是我们在 Facebook 使用的技术，以解决这些需求。

The entire data processing infrastructure in Facebook prior to 2008 was built around a data warehouse built using a commercial RDBMS. The data that we were generating was growing very fast - as an example we grew from a 15TB data set in 2007 to a 700TB data set today. The infrastructure at that time was so inadequate that some daily data processing jobs were taking more than a day to process and the situation was just getting worse with every passing day. We had an urgent need for infrastructure that could scale along with our data. As a result we started exploring Hadoop as a technology to address our scaling needs. The fact that Hadoop was already an open source project that was being used at petabyte scale and provided scalability using commodity hardware was a very compelling proposition for us. The same jobs that had taken more than a day to complete could now be completed within a few hours using Hadoop.

2008 年之前，Facebook 的整个数据处理基础架构全部围绕使用商业 RDBMS 构建的数据仓库。我们生成的数据增长非常快，例如，我们的数据集从 2007 年的 15TB 增长到了今天的700TB。当时的基础架构非常不足，一些日常的数据处理作业需要超过一天才能完成，情况每天都在变得更糟。我们迫切需要一个能够随数据扩展的基础架构。因此，我们开始探索 Hadoop 作为解决扩展需求的技术。Hadoop 已经是一个开源项目，可以在 PB 级别使用，并通过通用硬件提供可扩展性，这对我们来说是一个非常吸引人的主张。相同的作业如果使用 Hadoop，就可以在几个小时内完成，而之前需要超过一天的时间。

However, using Hadoop was not easy for end users, especially for those users who were not familiar with map- reduce. End users had to write map-reduce programs for simple tasks like getting raw counts or averages. Hadoop lacked the expressiveness of popular query languages like SQL and as a result users ended up spending hours (if not days) to write programs for even simple analysis. It was very clear to us that in order to really empower the company to analyze this data more productively, we had to improve the query capabilities of Hadoop. Bringing this data closer to users is what inspired us to build Hive in January 2007. Our vision was to bring the familiar concepts of tables, columns, partitions and a subset of SQL to the unstructured world of Hadoop, while still maintaining the extensibility and flexibility that Hadoop enjoyed. Hive was open sourced in August 2008 and since then has been used and explored by a number of Hadoop users for their data processing needs.

然而，对最终用户来说，使用 Hadoop 并不容易，特别是对于那些不熟悉 MapReduce 的用户。最终用户需要编写 MapReduce 程序来完成一些简单的任务，如获取原始计数或平均值。Hadoop 缺乏像 SQL 等流行查询语言的表达能力，结果是用户花费数小时（甚至数天）才能为即使是一个简单的分析编写程序。对我们来说非常明确，为了真正让公司能够更高效地分析这些数据，我们必须改进 Hadoop 的查询功能。将这些数据更接近用户才启发我们在 2007 年 1 月构建 Hive。我们的愿景是将表、列、分区等熟悉的概念以及 SQL 的一个子集引入 Hadoop 的非结构化世界，同时仍然保持 Hadoop 所享有的可扩展性和灵活性。Hive 于 2008 年 8 月开源，自那时以来被许多 Hadoop 用户用作满足其数据处理需求。

Right from the start, Hive was very popular with all users within Facebook. Today, we regularly run thousands of jobs on the Hadoop/Hive cluster with hundreds of users for a wide variety of applications starting from simple summarization jobs to business intelligence, machine learning applications and to also support Facebook product features.

从一开始，Hive 就在 Facebook 的所有用户中非常受欢迎。如今，我们定期在 Hadoop/Hive 集群上运行成千上万的作业，涉及数百位用户，支持各种各样的应用程序，从简单的汇总作业到商业智能、机器学习应用程序，以及支持 Facebook 的产品功能。

In the following sections, we provide more details about Hive architecture and capabilities. Section II describes the data model, the type systems and the HiveQL. Section III details how data in Hive tables is stored in the underlying distributed file system – HDFS(Hadoop file system). Section IV describes the system architecture and various components of Hive . In Section V we highlight the usage statistics of Hive at Facebook and provide related work in Section VI. We conclude with future work in Section VII.

在下面的章节中，我们将提供有关 Hive 架构和功能的更多细节。第 2 节描述了数据模型、类型系统和 HiveQL。第 3 节详细介绍了 Hive 表中的数据如何存储在底层的分布式文件系统 HDFS（Hadoop 文件系统）中。第 4 节描述了 Hive 的系统架构和各种组件。在第 5 节中，我们重点介绍了 Hive 在 Facebook 的使用统计信息，并在第 6 节给出了相关工作。我们将在第 7 节总结未来的工作。

## II. DATA MODEL, TYPE SYSTEM AND QUERY LANGUAGE

Hive structures data into the well-understood database concepts like tables, columns, rows, and partitions. It supports all the major primitive types – integers, floats, doubles and strings – as well as complex types such as maps, lists and structs. The latter can be nested arbitrarily to construct more complex types. In addition, Hive allows users to extend the system with their own types and functions. The query language is very similar to SQL and therefore can be easily understood by anyone familiar with SQL. There are some nuances in the data model, type system and HiveQL that are different from traditional databases and that have been motivated by the experiences gained at Facebook. We will highlight these and other details in this section.

Hive 将数据结构化为众所周知的数据库概念，如表、列、行和分区。它支持所有主要的基本类型，包括整数、浮点数、双精度和字符串，以及复杂类型，如映射、列表和结构体。后者可以任意嵌套以构建更复杂的类型。此外，Hive 允许用户通过其自定义类型和函数来扩展系统。查询语言非常类似于 SQL，因此任何熟悉 SQL 的人都可以轻松理解。在数据模型、类型系统和 HiveQL 中，与传统数据库不同的一些细微差别受到了 Facebook 的经验启发。我们将在本节中突出显示这些和其他细节。

### A. Data Model and Type System

Similar to traditional databases, Hive stores data in tables, where each table consists of a number of rows, and each row consists of a specified number of columns. Each column has an associated type. The type is either a primitive type or a complex type. Currently, the following primitive types are supported:
- Integers – bigint(8 bytes), int(4 bytes), smallint(2 bytes), tinyint(1 byte). All integer types are signed.
- Floating point numbers – float(single precision), double(double precision)
- String

与传统数据库类似，Hive 将数据存储在表中，每个表包含多行数据，每行数据由指定数量的列组成。每列都有一个关联的数据类型。数据类型可以是基本类型或复杂类型。目前，支持以下基本类型：

- 整数 - bigint（8字节），int（4字节），smallint（2字节），tinyint（1字节）。所有整数类型都是有符号的。
- 浮点数 - float（单精度），double（双精度）
- 字符串

Hive also natively supports the following complex types:
- Associative arrays – `map<key-type, value-type>`
- Lists – `list<element-type>`
- Structs – `struct<file-name: field-type, ... >`
此外，Hive 还本地支持以下复杂类型：
- 关联数组 - `map<键类型, 值类型>`
- 列表 - `list<元素类型>`
- 结构体 - `struct<文件名: 字段类型, ... >`

These complex types are templated and can be composed to generate types of arbitrary complexity. For example, list<map<string, struct<p1:int, p2:int>> represents a list of associative arrays that map strings to structs that in turn contain two integer fields named p1 and p2. These can all be put together in a create table statement to create tables with the desired schema. For example, the following statement creates a table t1 with a complex schema.

这些复杂类型是可模板化的，可以组合生成任意复杂度的类型。例如，`list<map<string, struct<p1:int, p2:int>>>` 表示一个列表，其中包含将字符串映射到包含两个整数字段（命名为 `p1` 和 `p2`）的结构体的关联数组。所有这些可以在一个 "create table" 语句中组合在一起，以创建具有所需模式的表。例如，以下语句创建了一个具有复杂模式的表 `t1`。

```sql
CREATE TABLE t1(st string, fl float, li list<map<string, struct<p1:int, p2:int>>);
```

Query expressions can access fields within the structs using a '.' operator. Values in the associative arrays and lists can be accessed using '[]' operator. In the previous example, `t1.li[0]` gives the first element of the list and `t1.li[0]['key']` gives the struct associated with 'key' in that associative array. Finally the p2 field of this struct can be accessed by `t1.li[0]['key'].p2`. With these constructs Hive is able to support structures of arbitrary complexity.

查询表达式可以使用'.'运算符访问结构体内的字段。可以使用'[]'运算符访问关联数组和列表中的值。在先前的示例中，`t1.li[0]` 给出了列表的第一个元素，`t1.li[0]['key']` 给出了与关联数组中的 'key' 相关联的结构体。最后，可以通过 `t1.li[0]['key'].p2` 访问这个结构体的 `p2` 字段。有了这些构造，Hive 能够支持任意复杂度的数据结构。

The tables created in the manner describe above are serialized and deserialized using default serializers and deserializers already present in Hive. However, there are instances where the data for a table is prepared by some other programs or may even be legacy data. Hive provides the flexibility to incorporate that data into a table without having to transform the data, which can save substantial amount of time for large data sets. As we will describe in the later sections, this can be achieved by providing a jar that implements the SerDe java interface to Hive. In such situations the type information can also be provided by that jar by providing a corresponding implementation of the ObjectInspector java interface and exposing that implementation through the getObjectInspector method present in the SerDe interface. More details on these interfaces can be found on the Hive wiki [^3], but the basic takeaway here is that any arbitrary data format and types encoded therein can be plugged into Hive by providing a jar that contains the implementations for the SerDe and ObjectInspector interfaces. All the native SerDes and complex types supported in Hive are also implementations of these interfaces. As a result once the proper associations have been made between the table and the jar, the query layer treats these on par with the native types and formats. As an example, the following statement adds a jar containing the SerDe and ObjectInspector interfaces to the distributed cache([^4]) so that it is available to Hadoop and then proceeds to create the table with the custom serde.

以上述描述的方式创建的表会使用 Hive 中已经存在的默认序列化器和反序列化器进行序列化和反序列化。然而，有些情况下，表的数据是由其他程序准备的，甚至可能是遗留数据。Hive 提供了灵活性，允许将这些数据合并到表中，而无需转换数据，这可以节省大数据集的大量时间。正如我们将在后面的部分中描述的那样，可以通过提供一个实现 SerDe Java 接口的 JAR 文件来实现这一点。在这种情况下，类型信息也可以通过该 JAR 文件提供，通过提供 ObjectInspector Java 接口的相应实现，并通过 SerDe 接口中的 getObjectInspector 方法公开该实现。关于这些接口的更多细节可以在 Hive 的 Wiki[^3] 中找到，但在这里的基本要点是，任何任意的数据格式和其中编码的类型都可以通过提供包含 SerDe 和 ObjectInspector 接口实现的 JAR 文件插入到 Hive 中。Hive 支持的所有原生 SerDe 和复杂类型也都是这些接口的实现。因此，一旦在表和 JAR 之间建立了正确的关联，查询层将这些类型和格式与本地类型和格式平等对待。例如，以下语句将包含 SerDe 和 ObjectInspector 接口的 JAR 文件添加到分布式缓存([^4])，以便 Hadoop 可以使用，并随后使用自定义 SerDe 创建表。

```sql
add jar /jars/myformat.jar;  
CREATE TABLE t2  
ROW FORMAT SERDE 'com.myformat.MySerDe';
```

Note that, if possible, the table schema could also be provided by composing the complex and primitive types.

请注意，如果可能的话，表的模式也可以通过组合复杂和基本类型来提供。

### B. QueryLanguage

The Hive query language(HiveQL) comprises of a subset of SQL and some extensions that we have found useful in our environment. Traditional SQL features like from clause sub- queries, various types of joins – inner, left outer, right outer and outer joins, cartesian products, group bys and aggregations, union all, create table as select and many useful functions on primitive and complex types make the language very SQL like. In fact for many of the constructs mentioned before it is exactly like SQL. This enables anyone familiar with SQL to start a hive cli(command line interface) and begin querying the system right away. Useful metadata browsing capabilities like show tables and describe are also present and so are explain plan capabilities to inspect query plans (though the plans look very different from what you would see in a traditional RDBMS). There are some limitations e.g. only equality predicates are supported in a join predicate and the joins have to be specified using the ANSI join syntax such as

Hive 查询语言（HiveQL）包括 SQL 的一个子集以及在我们的环境中发现的一些扩展。传统 SQL 功能，如 from 子查询、各种类型的连接 - 内连接、左外连接、右外连接和外连接、笛卡尔积、group by 和聚合、union all、create table as select 以及对基本和复杂类型的许多有用函数，使该语言非常类似于 SQL。实际上，对于之前提到的许多构造，它与 SQL 完全一样。这使得任何熟悉 SQL 的人都可以启动 Hive CLI（命令行界面）并立即开始查询系统。还有一些有用的元数据浏览功能，如 show tables 和 describe，以及解释查询计划的功能（尽管查询计划与传统的关系数据库管理系统中看到的计划非常不同）。但也有一些限制，例如只支持等值谓词作为连接谓词，并且连接必须使用 ANSI 连接语法指定，如：

```sql
SELECT t1.a1 as c1, t2.b1 as c2 FROM t1 JOIN t2 ON (t1.a2 = t2.b2);
```

instead of the more traditional

而不是更传统的方式

```sql
SELECT t1.a1 as c1, t2.b1 as c2 FROM t1, t2  
WHERE t1.a2 = t2.b2;
```

Another limitation is in how inserts are done. Hive currently does not support inserting into an existing table or data partition and all inserts overwrite the existing data. Accordingly, we make this explicit in our syntax as follows:

另一个限制是关于插入操作的方式。目前，Hive 不支持向现有表或数据分区插入数据，所有插入操作都会覆盖现有数据。因此，我们在语法中明确说明如下：

```sql
INSERT OVERWRITE TABLE t1
SELECT * FROM t2;
```

In reality these restrictions have not been a problem. We have rarely seen a case where the query cannot be expressed as an equi-join and since most of the data is loaded into our warehouse daily or hourly, we simply load the data into a new partition of the table for that day or hour. However, we do realize that with more frequent loads the number of partitions can become very large and that may require us to implement INSERT INTO semantics. The lack of INSERT INTO, UPDATE and DELETE in Hive on the other hand do allow us to use very simple mechanisms to deal with reader and writer concurrency without implementing complex locking protocols.

实际上，这些限制并没有成为问题。我们很少见到查询无法表示为等值连接的情况，而且由于大多数数据每天或每小时都会加载到我们的数据仓库中，我们只需将数据加载到该天或该小时的表分区中即可。但是，我们意识到随着加载频率的增加，分区的数量可能会变得非常大，这可能需要我们实现 INSERT INTO 语义。另一方面，Hive 不支持 INSERT INTO、UPDATE 和 DELETE 的特性使我们能够使用非常简单的机制来处理读写并发，而无需实现复杂的锁协议。

Apart from these restrictions, HiveQL has extensions to support analysis expressed as map-reduce programs by users and in the programming language of their choice. This enables advanced users to express complex logic in terms of map- reduce programs that are plugged into HiveQL queries seamlessly. Some times this may be the only reasonable approach e.g. in the case where there are libraries in python or php or any other language that the user wants to use for data transformation. The canonical word count example on a table of documents can, for example, be expressed using map- reduce in the following manner:

除了这些限制之外，HiveQL 还有扩展支持用户以其选择的编程语言表达为 map-reduce 程序的分析。这使高级用户能够以 map-reduce 程序的形式表达复杂逻辑，无缝嵌入到 HiveQL 查询中。有时这可能是唯一合理的方法，例如，当用户想要使用 Python、PHP 或任何其他语言中的库进行数据转换时。例如，对文档表进行标准的单词计数示例可以通过以下方式使用 map-reduce 来表达：

```sql
FROM (
	MAP doctext USING 'python wc_mapper.py' AS (word, cnt) FROM docs
	CLUSTER BY word
)a
REDUCE word, cnt USING 'python wc_reduce.py';
```

As shown in this example the MAP clause indicates how the input columns (doctext in this case) can be transformed using a user program (in this case ‘python wc_mapper.py') into output columns (word and cnt). The CLUSTER BY clause in the sub-query specifies the output columns that are hashed on to distributed the data to the reducers and finally the REDUCE clause specifies the user program to invoke (python wc_reduce.py in this case) on the output columns of the sub- query. Sometimes, the distribution criteria between the mappers and the reducers needs to provide data to the reducers such that it is sorted on a set of columns that are different from the ones that are used to do the distribution. An example could be the case where all the actions in a session need to be ordered by time. Hive provides the DISTRIBUTE BY and SORT BY clauses to accomplish this as shown in the following example:

正如这个示例所示，MAP 子句指示如何使用用户程序（在本例中是 `'python wc_mapper. py'`）将输入列（在本例中为 doctext）转换为输出列（word 和 cnt）。子查询中的 CLUSTER BY 子句指定要在其上进行哈希分布以将数据发送到 Reducer 的输出列，最后，REDUCE 子句指定要在子查询的输出列上调用的用户程序（在本例中为 `'python wc_reduce.py'`）。有时，在 Mapper 和 Reducer 之间的分布条件需要提供数据给 Reducer，以便它按一组与用于分布的列不同的列排序。例如，可能需要对会话中的所有操作按时间排序的情况。Hive 提供了 DISTRIBUTE BY 和 SORT BY 子句来实现这一点，如下面的示例所示：

```sql
FROM (  
	FROM session_table  
	SELECT sessionid, tstamp, data DISTRIBUTE BY sessionid SORT BY tstamp
)a
REDUCE sessionid, tstamp, data USING 'session_reducer.sh';
```

Note, in the example above there is no map clause which indicates that the input columns are not transformed. Similarly, it is possible to have a MAP clause without a REDUCE clause in case the reduce phase does not do any transformation of data. Also in the examples shown above, the FROM clause appears before the SELECT clause which is another deviation from standard SQL syntax. Hive allows users to interchange the order of the FROM and SELECT/MAP/REDUCE clauses within a given sub-query. This becomes particularly useful and intuitive when dealing with multi inserts. HiveQL supports inserting different transformation results into different tables, partitions, hdfs or local directories as part of the same query. This ability helps in reducing the number of scans done on the input data as shown in the following example:

注意，上面的示例中没有 MAP 子句，这表示输入列没有被转换。类似地，如果 REDUCE 阶段不对数据进行任何转换，也可以有 MAP 子句而没有 REDUCE 子句。此外，在上面的示例中，FROM 子句出现在 SELECT 子句之前，这是与标准 SQL 语法的另一种偏差。Hive 允许用户在给定的子查询中交换 FROM 和 SELECT/MAP/REDUCE 子句的顺序。这在处理多个插入时特别有用和直观。HiveQL 支持将不同的转换结果作为同一查询的一部分插入不同的表、分区、HDFS 或本地目录。这种能力有助于减少对输入数据的扫描次数，如下面的示例所示：

```sql
FROM t1  
	INSERT OVERWRITE TABLE t2 SELECT t3.c2, count(1)  
	FROM t3  
	WHERE t3.c1 <= 20  
	GROUP BY t3.c2
	
	INSERT OVERWRITE DIRECTORY '/output_dir' SELECT t3.c2, avg(t3.c1)  
	FROM t3  
	WHERE t3.c1 > 20 AND t3.c1 <= 30
	GROUP BY t3.c2
	
	INSERT OVERWRITE LOCAL DIRECTORY '/home/dir' SELECT t3.c2, sum(t3.c1)  
	FROM t3  
	WHERE t3.c1 > 30
	GROUP BY t3.c2;
```

In this example different portions of table t1 are aggregated and used to generate a table t2, an hdfs directory(/output_dir) and a local directory(/home/dir on the user’s machine).

在这个示例中，表 `t1` 的不同部分被汇总并用于生成表 `t2`、HDFS 目录（/output_dir）和用户机器上的本地目录（/home/dir）。

## III. DATA STORAGE, SERDE AND FILE FORMATS

### A. DataStorage

While the tables are logical data units in Hive, table metadata associates the data in a table to hdfs directories. The primary data units and their mappings in the hdfs name space are as follows:
- Tables – A table is stored in a directory in hdfs.  
- Partitions – A partition of the table is stored in a sub-directory within a table's directory.  
- Buckets – A bucket is stored in a file within the partition's or table's directory depending on whether the table is a partitioned table or not.  

虽然在 Hive 中，表是逻辑数据单元，但表的元数据将表中的数据关联到 HDFS 目录。主要的数据单元及其在 HDFS 命名空间中的映射如下：
- 表 - 表存储在 HDFS 的一个目录中。
- 分区 - 表的分区存储在表目录内的子目录中。
- 存储桶 - 存储桶存储在分区或表的目录中，具体取决于表是否为分区表。

As an example a table test_table gets mapped to `<warehouse_root_directory>/test_table` in hdfs. The warehouse_root_directory is specified by the `hive.metastore.warehouse.dir` configuration parameter in hive-site.xml. By default this parameter's value is set to /user/hive/warehouse.

例如，表 test_table 映射到 HDFS 中的 `<warehouse_root_directory>/test_table`。`warehouse_root_directory` 是由 `hive-site.xml` 中的 `hive.metastore.warehouse.dir` 配置参数指定的。默认情况下，此参数的值设置为 `/user/hive/warehouse`。

A table may be partitioned or non-partitioned. A partitioned table can be created by specifying the PARTITIONED BY clause in the CREATE TABLE statement as shown below.

表可以分为分区表和非分区表。可以通过在 CREATE TABLE 语句中指定 PARTITIONED BY 子句来创建分区表，如下所示：

```sql
CREATE TABLE test_part(c1 string, c2 int) PARTITIONED BY (ds string, hr int);
```

In the example shown above the table partitions will be stored in `/user/hive/warehouse/test_part` directory in hdfs. A partition exists for every distinct value of ds and hr specified by the user. Note that the partitioning columns are not part of the table data and the partition column values are encoded in the directory path of that partition (they are also stored in the table metadata). A new partition can be created through an INSERT statement or through an ALTER statement that adds a partition to the table. Both the following statements

在上面示例中，表的分区将存储在 HDFS 中的 `/user/hive/warehouse/test_part` 目录中。对于用户指定的每个 ds 和 hr 的不同值，都会存在一个分区。**请注意，分区列不是表数据的一部分，分区列的值被编码在该分区的目录路径中（它们也存储在表的元数据中）**。可以通过 INSERT 语句或通过 ALTER 语句向表添加分区来创建新的分区。以下两个语句都可以用来创建新分区：

```sql
INSERT OVERWRITE TABLE  
test_part PARTITION(ds='2009-01-01', hr=12)
SELECT * FROM t;

ALTER TABLE test_part  
ADD PARTITION(ds='2009-02-02', hr=11);
```

add a new partition to the table test_part. The INSERT statement also populates the partition with data from table t, where as the alter table creates an empty partition. Both these statements end up creating the corresponding directories - `/user/hive/warehouse/test_part/ds=2009-01-01/hr=12` and `/user/hive/warehouse/test_part/ds=2009-02-02/hr=11` – in the table’s hdfs directory. This approach does create some complications in case the partition value contains characters such as / or : that are used by hdfs to denote directory structure, but proper escaping of those characters does take care of a producing an hdfs compatible directory name.

向表 test_part 添加一个新的分区。INSERT 语句还将表 t 中的数据填充到分区中，而 ALTER TABLE 语句创建一个空分区。这两个语句都会在表的 HDFS 目录中创建相应的目录 - `/user/hive/warehouse/test_part/ds=2009-01-01/hr=12` 和 `/user/hive/warehouse/test_part/ds=2009-02-02/hr=11`。这种方法在分区值包含 HDFS 用于表示目录结构的字符（例如 `/` 或 `:`）时可能会引发一些复杂情况，但适当的字符转义可以解决生成兼容 HDFS 的目录名称的问题。

The Hive compiler is able to use this information to prune the directories that need to be scanned for data in order to evaluate a query. In case of the test_part table, the query

Hive 编译器能够利用这些信息来剪裁需要扫描以评估查询的目录。对于 test_part 表，查询如下：

```sql
SELECT * FROM test_part WHERE ds='2009-01-01';
```

will only scan all the files within the `/user/hive/warehouse/test_part/ds=2009-01-01` directory and the query

将仅扫描 `/user/hive/warehouse/test_part/ds=2009-01-01` 目录中的所有文件，而查询如下：

```sql
SELECT * FROM test_part  
WHERE ds='2009-02-02' AND hr=11;
```

will only scan all the files within the `/user/hive/warehouse/test_part/ds=2009-01-01/hr=12` directory. Pruning the data has a significant impact on the time it takes to process the query. In many respects this partitioning scheme is similar to what has been referred to as list partitioning by many database vendors ([^6]), but there are differences in that the values of the partition keys are stored with the metadata instead of the data.

将仅扫描 `/user/hive/warehouse/test_part/ds=2009-01-01/hr=12` 目录中的所有文件。剪裁数据对处理查询所需的时间有重大影响。在许多方面，这种分区方案类似于许多数据库供应商所称的列表分区 ([^6])，但不同之处在于分区键的值与元数据一起存储，而不是与数据一起存储。

The final storage unit concept that Hive uses is the concept of Buckets. A bucket is a file within the leaf level directory of a table or a partition. At the time the table is created, the user can specify the number of buckets needed and the column on which to bucket the data. In the current implementation this information is used to prune the data in case the user runs the query on a sample of data e.g. a table that is bucketed into 32 buckets can quickly generate a 1/32 sample by choosing to look at the first bucket of data. Similarly, the statement

Hive 使用的最后一个存储单元概念是存储桶（Buckets）的概念。一个存储桶是表或分区的叶级目录中的一个文件。在创建表时，用户可以指定所需的存储桶数量和用于存储桶数据的列。在当前的实现中，如果用户对数据的样本运行查询，这些信息将用于剪裁数据，例如，将表分成 32 个存储桶的表可以通过选择查看数据的第一个存储桶，快速生成 1/32 的样本。同样，以下语句：

```sql
SELECT * FROM t TABLESAMPLE(2 OUT OF 32);
```

would scan the data present in the second bucket. Note that the onus of ensuring that the bucket files are properly created and named are a responsibility of the application and HiveQL DDL statements do not currently try to bucket the data in a way that it becomes compatible to the table properties. Consequently, the bucketing information should be used with caution.

将扫描位于第二个存储桶中的数据。请注意，确保存储桶文件被正确创建和命名是应用程序的责任，HiveQL DDL 语句目前不会尝试以使其与表属性兼容的方式进行存储桶数据。因此，应谨慎使用存储桶信息。

Though the data corresponding to a table always resides in the `<warehouse_root_directory>/test_table` location in hdfs, Hive also enables users to query data stored in other locations in hdfs. This can be achieved through the EXTERNAL TABLE clause as shown in the following example.

虽然与表对应的数据始终驻留在 HDFS 中的 `<warehouse_root_directory>/test_table` 位置，但 Hive 还允许用户查询存储在 HDFS 中其他位置的数据。可以通过 EXTERNAL TABLE 子句来实现，如下面的示例所示：

```sql
CREATE EXTERNAL TABLE test_extern(c1 string, c2 int) LOCATION '/user/mytables/mydata';
```

With this statement, the user is able to specify that test_extern is an external table with each row comprising of two columns – c1 and c2. In addition the data files are stored in the location `/user/mytables/mydata` in hdfs. Note that as no custom SerDe has been defined it is assumed that the data is in Hive’s internal format. An external table differs from a normal table in only that a drop table command on an external table only drops the table metadata and does not delete any data. A drop on a normal table on the other hand drops the data associated with the table as well.

通过这个语句，用户可以指定 test_extern 是一个外部表，每行包括两列 - `c1` 和 `c2`。此外，数据文件存储在 HDFS 中的位置 `/user/mytables/mydata`。请注意，由于未定义自定义 SerDe，因此假定数据采用 Hive 的内部格式。外部表与普通表唯一的区别在于，对外部表执行的 drop table 命令仅删除表的元数据，不会删除任何数据。而对普通表执行的 drop 命令则删除与表关联的数据。

### B. Serialization/Deserialization(SerDe)

As mentioned previously Hive can take an implementation of the SerDe java interface provided by the user and associate it to a table or partition. As a result custom data formats can easily be interpreted and queried from. The default SerDe implementation in Hive is called the LazySerDe – it deserializes rows into internal objects lazily so that the cost of deserialization of a column is incurred only if the column of the row is needed in some query expression. The LazySerDe assumes that the data is stored in the file such that the rows are delimited by a newline (ascii code 13) and the columns within a row are delimited by ctrl-A (ascii code 1). This SerDe can also be used to read data that uses any other delimiter character between columns.

如前所述，Hive 可以接受用户提供的 SerDe Java 接口实现，并将其关联到表或分区。因此，用户可以轻松解释和查询自定义数据格式。Hive 中的默认 SerDe 实现称为 LazySerDe，它以惰性方式将行反序列化为内部对象，以便仅在查询表达式中需要行的某一列时才会发生反序列化列的成本。LazySerDe 假设数据以一种这样的方式存储在文件中，即行由换行符（ASCII 码 13）分隔，行内的列由 ctrl-A（ASCII 码 1）分隔。此 SerDe 还可以用于读取使用任何其他列分隔符的数据。

```sql
CREATE TABLE test_delimited(c1 string, c2 int)
	ROW FORMAT DELIMITED
		FIELDS TERMINATED BY '\002' 
		LINES TERMINATED BY '\012';
```

specifies that the data for table test_delimited uses ctrl-B (ascii code 2) as a column delimiter and uses ctrl-L(ascii code 12) as a row delimiter. In addition, delimiters can be specified to delimit the serialized keys and values of maps and different delimiters can also be specified to delimit the various elements of a list (collection). This is illustrated by the following statement.

指定表 test_delimited 的数据使用 ctrl-B（ASCII 码 2）作为列分隔符，并使用 ctrl-L（ASCII 码 12）作为行分隔符。此外，还可以指定分隔符来分隔映射的序列化键和值，也可以指定不同的分隔符来分隔列表（集合）的各个元素。以下语句进行了说明：

```sql
CREATE TABLE test_delimited2(c1 string, c2 list<map<string, int>>)
ROW FORMAT DELIMITED  
	FIELDS TERMINATED BY '\002'
	COLLECTION ITEMS TERMINATED BY '\003'
	MAP KEYS TERMINATED BY '\004';
```

Apart from LazySerDe, some other interesting SerDes are present in the hive_contrib.jar that is provided with the distribution. A particularly useful one is RegexSerDe which enables the user to specify a regular expression to parse various columns out from a row. The following statement can be used for example, to interpret apache logs.

除了 LazySerDe 之外，Hive 的分发版中还提供了一些其他有趣的 SerDes，这些 SerDes 包含在 `hive_contrib.jar` 中。其中一个特别有用的 SerDe 是 RegexSerDe，它允许用户指定正则表达式来从行中解析出各种列。例如，以下语句可以用于解释 Apache 日志：

```sql
add jar 'hive_contrib.jar';
CREATE TABLE apachelog(
	host string,
	identity string,
	user string,
	time string,
	request string,
	status string,
	size string,
	referer string,
	agent string)
ROW FORMAT SERDE 'org.apache.hadoop.hive.contrib.serde2.RegexSerDe'
WITH SERDEPROPERTIES(  
	'input.regex' = '([^ ]*) ([^ ]*) ([^ ]*) (-|\\[[^\\]]*\\]) ([^\"]*|\"[^\"]*\") (-|[0-9]*) (-|[0-9]*)(?: ([^ \"]*|\"[^\"]*\") ([^ \"]*|\"[^\"]*\"))?',
	'output.format.string' = '%1$s %2$s %3$s %4$s %5$s %6$s %7$s %8$s %9$s');
```

The input.regex property is the regular expression applied on each record and the output.format.string indicates how the column fields can be constructed from the group matches in the regular expression. This example also illustrates how arbitrary key value pairs can be passed to a serde using the WITH SERDEPROPERTIES clause, a capability that can be very useful in order to pass arbitrary parameters to a custom SerDe.

`Input.regex` 属性是应用于每个记录的正则表达式，`output.ormat.string` 指示如何从正则表达式中的组匹配构造列字段。这个示例还说明了如何使用 WITH SERDEPROPERTIES 子句传递任意键值对给 SerDe，这个功能在向自定义 SerDe 传递任意参数时非常有用。

### C. File Formats

Hadoop files can be stored in different formats. A file format in Hadoop specifies how records are stored in a file. Text files for example are stored in the TextInputFormat and binary files can be stored as SequenceFileInputFormat. Users can also implement their own file formats. Hive does not impose an restrictions on the type of file input formats, that the data is stored in. The format can be specified when the table is created. Apart from the two formats mentioned above, Hive also provides an RCFileInputFormat which stores the data in a column oriented manner. Such an organization can give important performance improvements specially for queries that do not access all the columns of the table. Users can add their own file formats and associate them to a table as shown in the following statement.

Hadoop 文件可以以不同的格式存储。在 Hadoop 中，文件格式指定了记录在文件中的存储方式。例如，文本文件存储为 TextInputFormat，而二进制文件可以存储为 SequenceFileInputFormat。用户还可以实现自己的文件格式。Hive 不会对数据存储的文件输入格式施加任何限制。在创建表时可以指定格式。除了上面提到的两种格式外，Hive 还提供了 RCFileInputFormat，以列为导向的方式存储数据。这种组织方式可以在特别是对不访问表的所有列的查询中提供重要的性能改进。用户可以添加自己的文件格式，并将它们关联到表，如下面的语句所示：

```sql
CREATE TABLE dest1(key INT, value STRING) STORED AS
	INPUTFORMA T 'org.apache.hadoop.mapred.SequenceFileInputFormat'
	OUTPUTFORMA T 'org.apache.hadoop.mapred.SequenceFileOutputFormat'
```

The STORED AS clause specifies the classes to be used to determine the input and output formats of the files in the table’s or partition’s directory. This can be any class that implements the FileInputFormat and FileOutputFormat java interfaces. The classes can be provded to Hadoop in a jar in ways similar to those shown in the examples on adding custom SerDes.

STORED AS 子句指定用于确定表或分区目录中文件的输入和输出格式的类。这可以是实现了 FileInputFormat 和 FileOutputFormat Java 接口的任何类。这些类可以以与添加自定义 SerDes 类似的方式以 jar 的形式提供给 Hadoop。

## IV. SYSTEM ARCHITECTURE AND COMPONENTS

![image.png](https://s2.loli.net/2023/09/23/14vrVukHtMUODJT.png)

The following components are the main building blocks in Hive:
- Metastore – The component that stores the system catalog and metadata about tables, columns, partitions etc.
- Driver – The component that manages the lifecycle of a HiveQL statement as it moves through Hive. The driver also maintains a session handle and any session statistics.
- Query Compiler – The component that compiles HiveQL into a directed acyclic graph of map/reduce tasks.
- Execution Engine – The component that executes the tasks produced by the compiler in proper dependency order. The execution engine interacts with the underlying Hadoop instance.
- HiveServer – The component that provides a thrift interface and a JDBC/ODBC server and provides a way of integrating Hive with other applications.
- Clients components like the Command Line Interface (CLI), the web UI and JDBC/ODBC driver.
- Extensibility Interfaces which include the SerDe and ObjectInspector interfaces already described previously as well as the UDF (User Defined Function) and UDAF (User Defined Aggregate Function) interfaces that enable users to define their own custom functions.

以下组件是 Hive 的主要构建块：
- Metastore（元数据存储） - 该组件存储系统目录和关于表、列、分区等的元数据。
- Driver（驱动程序） - 该组件管理 HiveQL 语句在 Hive 中的生命周期，维护会话句柄和任何会话统计信息。
- 查询编译器 - 该组件将 HiveQL 编译成一组 map/reduce 任务的有向无环图。
- 执行引擎 - 该组件按正确的依赖顺序执行编译器生成的任务。执行引擎与底层的 Hadoop 实例进行交互。
- HiveServer（Hive 服务器） - 该组件提供 Thrift 接口和 JDBC/ODBC 服务器，为将 Hive 与其他应用程序集成提供了一种方法。
- 客户端组件，如命令行界面（CLI）、Web 界面以及 JDBC/ODBC 驱动程序。
- 可扩展性接口，其中包括之前已描述的 SerDe 和 ObjectInspector 接口，以及允许用户定义自己的自定义函数的 UDF（用户定义函数）和 UDAF（用户定义聚合函数）接口。

A HiveQL statement is submitted via the CLI, the web UI or an external client using the thrift, odbc or jdbc interfaces. The driver first passes the query to the compiler where it goes through the typical parse, type check and semantic analysis phases, using the metadata stored in the Metastore. The compiler generates a logical plan that is then optimized through a simple rule based optimizer. Finally an optimized plan in the form of a DAG of map-reduce tasks and hdfs tasks is generated. The execution engine then executes these tasks in the order of their dependencies, using Hadoop.

HiveQL 语句可以通过 CLI、Web 界面或使用 Thrift、ODBC 或 JDBC 接口的外部客户端提交。驱动程序首先将查询传递给编译器，其中查询经历了典型的解析、类型检查和语义分析阶段，使用存储在 Metastore 中的元数据。编译器生成一个逻辑计划，然后通过基于规则的简单优化器进行优化。最后，生成了一个优化后的计划，以 map/reduce 任务和 HDFS 任务的 DAG 形式存在。然后，执行引擎按其依赖关系的顺序执行这些任务，使用 Hadoop 进行执行。
 
In this section we provide more details on the Metastore, the Query Compiler and the Execution Engine.

在本节中，我们提供有关 Metastore、查询编译器和执行引擎的更多详细信息。

### A. Metastore

The Metastore acts as the system catalog for Hive. It stores all the information about the tables, their partitions, the schemas, the columns and their types, the table locations etc. This information can be queried or modified using a thrift ([^7]) interface and as a result it can be called from clients in different programming languages. As this information needs to be served fast to the compiler, we have chosen to store this information on a traditional RDBMS. The Metastore thus becomes an application that runs on an RDBMS and uses an open source ORM layer called DataNucleus ([^8]), to convert object representations into a relational schema and vice versa. We chose this approach as opposed to storing this information in hdfs as we need the Metastore to be very low latency. The DataNucleus layer allows us to plugin many different RDBMS technologies. In our deployment at Facebook, we use mysql to store this information. 

Metastore 充当 Hive 的系统目录。它存储关于表、它们的分区、模式、列及其类型、表位置等所有信息。可以使用 Thrift 接口查询或修改此信息，因此可以从不同编程语言的客户端调用。由于需要快速为编译器提供此信息，我们选择将此信息存储在传统的 RDBMS 上。因此，Metastore 成为在 RDBMS 上运行的应用程序，并使用名为 DataNucleus 的开源 ORM 层，将对象表示转换为关系模式，反之亦然。与将此信息存储在 HDFS 中相比，我们选择了这种方法，因为需要使 Metastore 具有非常低的延迟。DataNucleus 层允许我们插入许多不同的 RDBMS 技术。在 Facebook 的部署中，我们使用 mysql 来存储此信息。

Metastore is very critical for Hive. Without the system catalog it is not possible to impose a structure on hadoop files. As a result it is important that the information stored in the Metastore is backed up regularly. Ideally a replicated server should also be deployed in order to provide the availability that many production environments need. It is also important to ensure that this server is able to scale with the number of queries submitted by the users. Hive addresses that by ensuring that no Metastore calls are made from the mappers or the reducers of a job. Any metadata that is needed by the mapper or the reducer is passed through xml plan files that are generated by the compiler and that contain any information that is needed at the run time. 

Metastore 对于 Hive 非常关键。没有系统目录，无法对 Hadoop 文件强制执行结构。因此，重要的是定期备份存储在 Metastore 中的信息。理想情况下，还应部署复制服务器，以提供许多生产环境所需的可用性。还必须确保此服务器能够根据用户提交的查询数量进行扩展。Hive 通过确保不从作业的 mapper 或 reducer 中进行 Metastore 调用来解决这个问题。mapper 或 reducer 需要的任何元数据都通过编译器生成的包含运行时所需任何信息的 XML 计划文件传递。

The ORM logic in the Metastore can be deployed in client libraries such that it runs on the client side and issues direct calls to an RDBMS. This deployment is easy to get started with and ideal if the only clients that interact with Hive are the CLI or the web UI. However, as soon as Hive metadata needs to get manipulated and queried by programs in languages like python, php etc., i.e. by clients not written in Java, a separate Metastore server has to be deployed.

Metastore 中的 ORM 逻辑可以部署在客户端库中，以便在客户端端运行并直接调用 RDBMS。这种部署易于开始使用，并且在仅与 Hive 进行交互的客户端是 CLI 或 Web UI 的情况下是理想的。但是，一旦需要使用像 Python、PHP 等语言编写的程序（即非 Java 编写的客户端）来操作和查询 Hive 元数据，必须部署单独的 Metastore 服务器。

### B. QueryCompiler

The metadata stored in the Metastore is used by the query compiler to generate the execution plan. Similar to compilers in traditional databases, the Hive compiler processes HiveQL statements in the following steps:
- Parse – Hive uses Antlr to generate the abstract syntax tree (AST) for the query.
- Type checking and Semantic Analysis – During this phase, the compiler fetches the information of all the input and output tables from the Metastore and uses that information to build a logical plan. It checks type compatibilities in expressions and flags any compile time semantic errors at this stage. The transformation of an AST to an operator DAG goes through an intermediate representation that is called the query block (QB) tree. The compiler converts nested queries into parent child relationships in a QB tree. At the same time, the QB tree representation also helps in organizing the relevant parts of the AST tree in a form that is more amenable to be transformed into an operator DAG than the vanilla AST.
- Optimization – The optimization logic consists of a chain of transformations such that the operator DAG resulting from one transformation is passed as input to the next transformation. Anyone wishing to change the compiler or wishing to add new optimization logic can easily do that by implementing the transformation as an extension of the Transform interface and adding it to the chain of transformations in the optimizer.
The transformation logic typically comprises of a walk on the operator DAG such that certain processing actions are taken on the operator DAG when relevant conditions or rules are satisfied. The five primary interfaces that are involved in a transformation are Node, GraphWalker, Dispatcher, Rule and Processor. The nodes in the operator DAG implement the Node interface. This enables the operator DAG to be manipulated using the other interfaces mentioned above. A typical transformation involves walking the DAG and for every Node visited, checking if a Rule is satisfied and then invoking the corresponding Processor for that Rule in case the later is satisfied. The Dispatcher maintains the mappings from Rules to Processors and does the Rule matching. It is passed to the GraphWalker so that the appropriate Processor can be dispatched while a Node is being visited in the walk. The flowchart in Fig. 2 shows how a typical transformation is structured.

Metastore 中存储的元数据被查询编译器用于生成执行计划。与传统数据库中的编译器类似，Hive 编译器按以下步骤处理 HiveQL 语句：
- 解析（Parse）：Hive 使用 Antlr 生成查询的抽象语法树（AST）。
- 类型检查和语义分析（Type checking and Semantic Analysis）：在此阶段，编译器从 Metastore 中提取所有输入和输出表的信息，并使用该信息构建逻辑计划。它在表达式中检查类型兼容性，并在此阶段标记任何编译时语义错误。AST 到操作符 DAG 的转换通过一个称为查询块（QB）树的中间表示进行。编译器将嵌套查询转换为 QB 树中的父子关系。与此同时，QB 树表示还有助于以更易于转换为操作符 DAG 的形式组织 AST 树的相关部分，而不是纯粹的 AST。
- 优化（Optimization）：优化逻辑包括一系列转换，其中一个转换生成的操作符 DAG 作为下一个转换的输入传递。任何希望更改编译器或添加新优化逻辑的人都可以轻松通过将转换实现为 Transform 接口的扩展并将其添加到优化器中的转换链来实现。
转换逻辑通常包括在操作符 DAG 上执行步骤，以便在满足相关条件或规则时对操作符 DAG 采取某些处理操作。涉及转换的五个主要接口是 Node、GraphWalker、Dispatcher、Rule 和 Processor。操作符 DAG 中的节点实现 Node 接口。这使得可以使用上面提到的其他接口来操作操作符 DAG。典型的转换包括遍历 DAG，并对每个访问的节点进行检查，看是否满足某个规则，然后在后者满足的情况下调用该规则的相应处理器。Dispatcher 维护了从规则到处理器的映射，并执行规则匹配。它传递给 GraphWalker，以便在遍历期间访问节点时可以分派适当的处理器。图2 中的流程图显示了典型的转换结构。

![image.png](https://s2.loli.net/2023/09/23/t6xqMKUXQzeC1Wd.png)

The following transformations are done currently in Hive as part of the optimization stage:
1. Column pruning – This optimization step ensures that only the columns that are needed in the query processing are actually projected out of the row.
2. Predicate pushdown – Predicates are pushed down to the scan if possible so that rows can be filter early in the processing.
3. Partition pruning – Predicates on partitioned columns are used to prune out files of partitions that do not satisfy the predicate.
4. Map side joins – In the cases where some of the tables in a join are very small, the small tables are replicated in all the mappers and joined with other tables. This behavior is triggered by a hint in the query of the form:

```sql
SELECT /*+ MAPJOIN (t 2) */ t 1. C 1, t 2. C 1 FROM t 1 JOIN t 2 ON (t 1. C 2 = t 2. C 2);
```

5. A number of parameters control the amount of memory that is used on the mapper to hold the contents of the replicated table. These are `hive.mapjoin.size.key` and `hive.mapjoin.cache.numrows` that control the number of rows of the table that are kept in memory at any time and also provide the system the size of the join key.
6. Join reordering – The larger tables are streamed and not materialized in memory in the reducer while the smaller tables are kept in memory. This ensures that the join operation does not exceed memory limits on the reducer side.

目前在 Hive 中，作为优化阶段的一部分，执行以下转换：
1. 列修剪（Column pruning） - 此优化步骤确保只有在查询处理中实际需要的列才会从行中投影出来。
2. 谓词下推（Predicate pushdown） - 如果可能的话，将谓词下推到扫描操作，以便在处理早期筛选行。
3. 分区修剪（Partition pruning） - 分区列上的谓词用于修剪不满足谓词的分区文件。
4. 映射端连接（Map side joins） - 在连接中的某些表非常小的情况下，小表会在所有映射器中复制，并与其他表连接。此行为由查询中的形式为的提示触发：
```sql
SELECT /*+ MAPJOIN(t2) */ t1.c1, t2.c1 FROM t1 JOIN t2 ON(t1.c2 = t2.c2);
```
5. 一些参数控制了在映射器上用于保存复制表内容的内存量。这些参数包括 `hive. mapjoin.size.key` 和 `hive.mapjoin.cache.numrows`，它们控制在任何时间内保留在内存中的表的行数，并提供连接键的大小。 
6. 连接重排序（Join reordering） - 较大的表会被流式传输，而不会在 Reducer 中存储在内存中，而较小的表会保留在内存中。这确保连接操作不会超出 Reducer 端的内存限制。

In addition to the MAPJOIN hint, the user can also provide hints or set parameters to do the following:

除了 MAPJOIN 提示外，用户还可以提供提示或设置参数来执行以下操作：

Repartitioning of data to handle skews in GROUP BY processing – Many real world datasets have a power law distribution on columns used in the GROUP BY clause of common queries. In such situations the usual plan of distributing the data on the group by columns and then aggregating in the reducer does not work well as most of the data gets sent to a very few reducers. A better plan in such situations is to use two map/reduce stages to compute the aggregation. In the first stage the data is randomly distributed (or distributed on the DISTINCT column in case of distinct aggregations) to the reducers and the partial aggregations are computed. These partial aggregates are then distributed on the GROUP BY columns to the reducers in the second map/reduce stage. Since the number of the partial aggregation tuples is much smaller than the base data set, this approach typically leads to better performance. In Hive this behavior can be triggered by setting a parameter in the following manner:

重新分区数据以处理 GROUP BY 处理中的偏斜 - 许多实际世界的数据集在常见查询的 GROUP BY 子句中使用的列上都具有幂律分布。在这种情况下，通常的计划是将数据分布到 GROUP BY 列上，然后在 Reducer 中进行聚合，但由于大部分数据都发送到非常少量的 Reducer，这种计划效果不佳。在这种情况下，更好的计划是使用两个 map/reduce 阶段来计算聚合。在第一阶段，数据会被随机分布（或者在进行不同聚合时分布到 DISTINCT 列），然后计算部分聚合。然后，在第二个 map/reduce 阶段，将这些部分聚合分布到 GROUP BY 列上的 Reducer 中。由于部分聚合元组的数量远小于基础数据集，因此这种方法通常会导致更好的性能。在 Hive 中，可以通过以下方式设置参数来触发这种行为：

```sql
set hive.groupby.skewindata=true; SELECT t1.c1, sum(t1.c2)  
FROM t1  
GROUP BY t1;
```

基于哈希的部分聚合在映射器中可以减少映射器发送给 Reducer 的数据量。这反过来减少了在对此数据进行排序和合并时花费的时间。因此，使用这种策略可以实现很多性能提升。Hive 允许用户控制在映射器上用于保存哈希表中行的内存量，以进行此优化。参数 `hive. map.aggr.hash.percentmemory` 指定可以用于保存哈希表的映射器内存的分数，例如，0.5 将确保一旦哈希表大小超过映射器的最大内存的一半，其中存储的部分聚合将被发送到 Reducer。参数 `hive.map.aggr.hash.min.reduction` 也用于控制映射器中使用的内存量。

Generation of the physical plan - The logical plan generated at the end of the optimization phase is then split into multiple map/reduce and hdfs tasks. As an example a group by on skewed data can generate two map/reduce tasks followed by a final hdfs task which moves the results to the correct location in hdfs. At the end of this stage the physical plan looks like a DAG of tasks with each task encapsulating a part of the plan.

生成物理计划 - 在优化阶段结束时生成的逻辑计划然后被拆分成多个 map/reduce 和 hdfs 任务。例如，在偏斜数据上进行 GROUP BY 可能会生成两个 map/reduce 任务，然后是最终的 hdfs 任务，将结果移动到 hdfs 中的正确位置。在此阶段结束时，物理计划看起来像是一个包含计划的任务 DAG，每个任务都封装了计划的一部分。

```sql
FROM (SELECT a.status, b.school, b.gender
		FROM status_updates a JOIN profiles b
			ON (a.userid = b.userid  
				AND a.ds='2009-03-20' )) subq1

INSERT OVERWRITE TABLE gender_summary PARTITION(ds='2009-03-20')

SELECT subq1.gender, COUNT(1) GROUP BY subq1.gender

INSERT OVERWRITE TABLE school_summary PARTITION(ds='2009-03-20')

SELECT subq1.school, COUNT(1) GROUP BY subq1.school
```

This query has a single join followed by two different aggregations. By writing the query as a multi-table-insert, we make sure that the join is performed only once. The plan for the query is shown in Fig 3 below.

这个查询包含一个连接操作，然后进行两个不同的聚合操作。通过将查询编写为多表插入（multi-table-insert），我们确保连接只执行一次。查询的计划如下图所示（图 3）。

The nodes in the plan are physical operators and the edges represent the flow of data between operators. The last line in each node represents the output schema of that operator. For lack of space, we do not describe the parameters specified within each operator node. The plan has three map-reduce jobs.

计划中的节点是物理运算符，边表示运算符之间的数据流。每个节点中的最后一行表示该运算符的输出模式。由于空间有限，我们不描述每个运算符节点中指定的参数。计划涉及三个 map-reduce 作业。

![image.png](https://s2.loli.net/2023/09/23/XEenzv7swKBYLMy.png)

Within the same map-reduce job, the portion of the operator tree below the repartition operator (ReduceSinkOperator) is executed by the mapper and the portion above by the reducer. The repartitioning itself is performed by the execution engine.

在同一个 map-reduce 作业中，重分区运算符（ReduceSinkOperator）下方的运算符树部分由映射器执行，而上方的部分由 Reducer 执行。重分区本身由执行引擎执行。

Notice that the first map-reduce job writes to two temporary files to HDFS, tmp1 and tmp2, which are consumed by the second and third map-reduce jobs respectively. Thus, the second and third map-reduce jobs wait for the first map-reduce job to finish.

请注意，第一个 map-reduce 作业将数据写入到 HDFS 的两个临时文件 tmp 1 和 tmp 2，这两个文件分别由第二和第三个 map-reduce 作业使用。因此，第二和第三个 map-reduce 作业需要等待第一个 map-reduce 作业完成。

### C. Execution Engine

最后，任务按照它们的依赖关系的顺序执行。只有在执行了所有先决条件后，才会执行每个依赖任务。一个 map/reduce 任务首先将其计划的一部分序列化为一个 plan.xml 文件。然后，将此文件添加到任务的作业缓存中，并使用 Hadoop 生成 ExecMapper 和 ExecReducers 的实例。这些类中的每一个都会反序列化 plan.xml 并执行运算符 DAG 的相关部分。最终的结果存储在临时位置。在整个查询结束时，对于 DML 操作，最终的数据会移动到所需的位置。对于查询，数据将直接从临时位置提供。

## V . HIVE USAGE IN FACEBOOK

Hive and Hadoop are used extensively in Facebook for different kinds of data processing. Currently our warehouse has 700TB of data(which comes to 2.1PB of raw space on Hadoop after accounting for the 3 way replication). We add 5TB(15TB after replication) of compressed data daily. Typical compression ratio is 1:7 and sometime more than that. On any particular day more than 7500 jobs are submitted to the cluster and more than 75TB of compressed data is processed every day. With the continuous growth in the Facebook network we see continuous growth in data. At the same time as the company scales, the cluster also has to scale with the growing users.

Facebook 广泛使用 Hive 和 Hadoop 进行不同类型的数据处理。目前，我们的数据仓库中有 700 TB 的数据（考虑到 3 倍复制后，在 Hadoop 上的原始空间为 2.1 PB）。我们每天添加 5 TB（经过复制后为 15 TB）的压缩数据。典型的压缩比例为1:7，有时甚至更高。在任何特定的一天，有超过 7500 个作业提交到集群，每天处理超过 75 TB 的压缩数据。随着 Facebook 网络的持续增长，数据也在持续增长。与此同时，随着公司规模的扩大，集群也必须随着不断增长的用户规模而扩展。

More than half the workload is on adhoc queries where as the rest is for reporting dashboards. Hive has enabled this kind of workload on the Hadoop cluster in Facebook because of the simplicity with which adhoc analysis can be done. However, sharing the same resources by the adhoc users and reporting users presents significant operational challenges because of the unpredictability of adhoc jobs. Many times these jobs are not properly tuned and therefore consume valuable cluster resources. This can in turn lead to degraded performance of the reporting queries, many of which are time critical. Resource scheduling has been somewhat weak in Hadoop and the only viable solution at present seems to be maintaining separate clusters for adhoc queries and reporting queries.

超过一半的工作负载用于自由查询，其余用于报告仪表板。由于自由查询作业的不可预测性，共享自由查询用户和报告用户相同的资源存在重大操作挑战。许多时候，这些作业没有经过适当的调整，因此消耗了宝贵的集群资源。这反过来可能导致报告查询的性能下降，其中许多查询是时间关键的。目前，Hadoop 中的资源调度相对较弱，目前唯一可行的解决方案似乎是为自由查询和报告查询维护单独的集群。

There is also a wide variety in the Hive jobs that are run daily. They range from simple summarization jobs generating different kinds of rollups and cubes to more advanced machine learning algorithms. The system is used by novice users as well as advanced users with new users being able to use the system immediately or after an hour long beginners training.

每天运行的 Hive 作业种类繁多。它们从生成不同类型的摘要和多维数据立方体的简单汇总作业，到更高级的机器学习算法。该系统被初学者和高级用户使用，新用户可以立即使用系统，或在一个小时的初学者培训后使用系统。

A result of heavy usage has also lead to a lot of tables generated in the warehouse and this has in turn tremendously increased the need for data discovery tools, especially for new users. In general the system has enabled us to provide data processing services to engineers and analysts at a fraction of the cost of a more traditional warehousing infrastructure.Added to that the ability of Hadoop to scale to thousands of commodity nodes gives us the confidence that we will be able to scale this infrastructure going forward as well.

大量使用的结果也导致了仓库中生成了大量的表，这反过来极大地增加了对数据发现工具的需求，特别是对于新用户。总的来说，该系统使我们能够以传统数据仓库基础设施的一小部分成本为工程师和分析师提供数据处理服务。此外，Hadoop 能够扩展到数千个廉价节点的能力使我们有信心能够在未来扩展这个基础设施。

## VII. CONCLUSIONS AND FUTURE WORK

Hive is a work in progress. It is an open-source project, and is being actively worked on by Facebook as well as several external contributors.

Hive 是一个不断发展的项目。它是一个开源项目，由 Facebook 以及多个外部贡献者积极开发。

HiveQL currently accepts only a subset of SQL as valid queries. We are working towards making HiveQL subsume SQL syntax. Hive currently has a naive rule-based optimizer with a small number of simple rules. We plan to build a cost- based optimizer and adaptive optimization techniques to come up with more efficient plans. We are exploring columnar storage and more intelligent data placement to improve scan performance. We are running performance benchmarks based on [^9] to measure our progress as well as compare against other systems. In our preliminary experiments, we have been able to improve the performance of Hadoop itself by ~20% compared to [^9]. The improvements involved using faster Hadoop data structures to process the data, for example, using Text instead of String. The same queries expressed easily in HiveQL had ~20% overhead compared to our optimized Hadoop implementation, i.e., Hive's performance is on par with the Hadoop code from [^9]. We have also run the industry standard decision support benchmark – TPC-H [^11]. Based on these experiments, we have identified several areas for performance improvement and have begun working on them. More details are available in [^10] and [^12]. We are enhancing the JDBC and ODBC drivers for Hive for integration with commercial BI tools that only work with traditional relational warehouses. We are exploring methods for multi-query optimization techniques and performing generic n-way joins in a single map-reduce job.

目前，HiveQL 只接受 SQL 的一个子集作为有效查询。我们正在努力使 HiveQL 包含更多 SQL 语法。Hive 目前具有一个简单的基于规则的优化器，拥有少量简单的规则。我们计划构建一个基于成本的优化器和自适应优化技术，以生成更有效的执行计划。我们正在研究列存储和更智能的数据放置，以提高扫描性能。我们正在运行基于[^9]的性能基准来衡量我们的进展，并与其他系统进行比较。在初步实验中，与[^9]相比，我们已经能够将 Hadoop 本身的性能提高了约20%。改进涉及使用更快的 Hadoop 数据结构来处理数据，例如使用 Text 而不是 String。与我们优化后的 Hadoop 实现相比，同样的查询在 HiveQL 中的执行性能有约20%的开销，即 Hive 的性能与[^9]中的 Hadoop 代码相当。我们还运行了行业标准的决策支持基准测试 TPC-H [^11]。根据这些实验，我们已经确定了性能改进的几个方面，并已开始着手解决这些问题。有关更多详细信息，请参阅[^10]和[^12]。我们正在增强 Hive 的 JDBC 和 ODBC 驱动程序，以与只与传统关系型数据仓库一起使用的商业 BI 工具进行集成。我们正在探索多查询优化技术的方法，并在单个 map-reduce 作业中执行通用的 n-way 连接。

## REFERENCES

[^1]: Apache Hadoop. Available at http://wiki.apache.org/hadoop.
[^2]: Facebook Lexicon at http://www.facebook.com/lexicon.
[^3]: Hive wiki at http://www.apache.org/hadoop/hive.
[^4]: Hadoop Map-Reduce Tutorial at http://hadoop.apache.org/common/docs/current/mapred_tutorial.html.
[^5]: Hadoop HDFS User Guide at http://hadoop.apache.org/common/docs/current/hdfs_user_guide.html.
[^6]: Mysql list partitioning at http://dev.mysql.com/doc/refman/5.1/en/partitioning-list.html.
[^7]: Apache Thrift. Available at http://incubator.apache.org/thrift.
[^8]: DataNucleus .Available at http://www.datanucleus.org.
[^9]: A. Pavlo et. Al. A Comparison of Approaches to Large-Scale Data Analysis. In Proc. Of ACM SIGMOD, 2009.
[^10]: Hive Performance Benchmark. Available at http://issues.apache.org/jira/browse/HIVE-396
[^11]: TPC-H Benchmark. Available at http://www.tpc.org/tpch
[^12]: Running TPC-H queries on Hive. Available at http://issues.apache.org/jira/browse/HIVE-600
[^13]: Hadoop Pig. Available at http://hadoop.apache.org/pig
[^14]: R. Chaiken, et. Al. Scope: Easy and Efficient Parallel Processing of Massive Data Sets. In Proc. Of VLDB, 2008.
[^15]: HadoopDB Project. Available at http://db.cs.yale.edu/hadoopdb/hadoopdb.html
[^16]: MicroStrategy. Available at http://www.microstrategy.com