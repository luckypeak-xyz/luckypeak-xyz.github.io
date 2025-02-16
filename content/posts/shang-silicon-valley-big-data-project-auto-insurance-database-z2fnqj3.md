---
title: 尚硅谷大数据项目之车险数仓
slug: shang-silicon-valley-big-data-project-auto-insurance-database-z2fnqj3
url: >-
  /post/shang-silicon-valley-big-data-project-auto-insurance-database-z2fnqj3.html
date: '2025-02-15 08:22:10+08:00'
lastmod: '2025-02-15 22:05:15+08:00'
toc: true
isCJKLanguage: true
---



## 数据仓库概念

数据仓库（data warehouse）是为企业提供大数据支持，以协助企业制定决策、改进业务流程和提高产品质量等方面的工具。

​![image](http://127.0.0.1:50577/assets/image-20250215084608-x2v1xig.png)​

## 项目需求及架构设计

### 项目需求分析

* 采集平台
* 维度建模
* 分析，交易，理赔 等车险核心主题，统计上百个报表指标
* 即席查询工具
* 性能监控、异常告警
* 元数据管理

  * 数据血缘（数据流转间表与表之间的依赖关系）
* 质量监控
* 权限管理

‍

思考

1. 技术选型
2. 版本选型
3. 云主机 or 物理机
4. 集群规模

### 项目框架

#### 技术选型

主要考虑因素：数据量大小、业务需求、行业内经验、技术成熟度、开发维护成本、总成本预算

* 数据采集：Flume、Kafka、DataX、Maxwell、Sqoop、Logstash

  * 日志数据

    * Flume
    * Logstash 通常不会单独使用，而是使用 ELK 整个技术栈
  * 业务数据

    * DataX 和 Sqoop 适合 全量数据采集
    * Maxwell 适合 增量数据采集
  * 数据通道

    * Kafka
* 数据存储：MySQL、HDFS、HBase、Redis、MongoDB

  * 业务数据

    * MySQL
  * 离线数据

    * HDFS
  * 实时数据

    * HBase、Redis、MongoDB
* 数据计算：Hive（Map Reduce）、Spark、Tez、Flink、Storm

  * 离线计算：Hive、Spark、Tez
  * 实时计算：Flink、Spark Streaming、Storm
* 数据（即席）查询：Presto、Kylin、Impala、Druid、ClickHose、Doris

  * Impala 内存
  * Kylin，Druid 预计算
* 数据可视化：Superset、Echarts、QuickBI、DataV

  * Superset 开源
  * Echarts 要写代码
  * QuickBI、DataV 收费
* 调度工具：DolphinScheduler、Azkaban、Oozie、Airflow
* 集群监控：Zabbix、Prometheus
* 元数据管理：Atlas
* 权限管理：Ranger、Sentry

#### 数据流程

​![image](http://127.0.0.1:50577/assets/image-20250215103528-96cayyd.png)​

#### 框架版本选型

如何选择 Apache、CDH、HDP 版本？

* Apache（推荐）

  * 优点：开源免费
  * 缺点：组件兼容需要自己调研
* CHD

  * 优点：无需考虑兼容
  * 缺点：收费
* HDP（不推荐）

  * 优点：开源
  * 缺点：国内使用较少

具体（Apache）版本（选择最新版本往前半年的稳定版本）

|组件|旧版本|新版本|
| ------------------| --------| ------------------------------|
|Hadoop|3.1.3|3.3.5|
|Zookeeper|3.5.7|3.7.1|
|MySQL|5.7.16|8.0.31|
|Hive|3.1.2|3.1.3（修改源码）|
|Flume|1.9.0|1.10.1|
|Kafka|3.0.0|3.3.1|
|Spark|3.0.0|3.3.1|
|DataX|3.0|3.0（master 分支，修改源码）|
|Superset|1.5.3|2.0.0|
|DolphinScheduler|1.3.9|2.0.5|
|Maxwell|1.29.2|1.29.2（修改源码）|
|Flink|1.13.0|1.17.1|
|Redis||6.0.8|
|Hbase||2.4.11|
|Doris||1.2.4.1|

#### 服务器选型

EMR

阿里云 MaxCompute DataWorks

云主机 or 物理机？

* 物理机

  * 缺点：成本高（机器+人力+网络+机房+电力）
  * 优点：数据安全
* 云主机

  * 缺点：数据不安全
  * 优点：成本低

#### 集群规模

用户量

数据量

副本数

数据保留时长

资源预留（30%）

数据压缩 ⬇️

数据分层 ⬆️ 2～3 倍

#### 集群资源规划设计

测试+生产集群

##### 生产集群

|类型|规格|数量|
| --------| ----------------------------------| ------|
|Master|8C16G<br />系统盘：50G<br />数据盘：200G|2|
|Core|4C8G<br />系统盘：50G<br />数据盘：200G|3|
|Task|4C8G<br />系统盘：50G<br />数据盘：200G|0|
|Common|2C4G<br />系统盘：50G<br />数据盘：200G|3|

* Master 节点：管理节点，保证集群的正常运行；主要部署 NameNode、ResourceManager、HMaster 等进程；非 HA 模式数量为 1，HA 数量为 2；
* Core 节点：计算及存储节点，HDFS 中的数据全部存储于 Core 节点；主要部署 DataNode、NodeManager、RegionServer 等进程；非 HA 模式数量 >= 2，HA 数量 >= 3；
* Common 节点：为 HA 集群 Master 节点提供数据共享及高可用容错服务；主要部署 ZooKeeper、JournalNode 等协调组件；非 HA 模式数量 0，HA 数量 >= 3

1. 消耗内存的分开部署
2. 数据传输紧密的放在一起
3. 客户端放在一起
4. 有依赖关系的放在一起

|Master|Master|Core|Core|Core|Common|Common|Common|
| -----------------| -----------------| ------------| ------------| ------------| -------------| -------------| -------------|
|NameNode|NameNode|DataNode|DataNode|DataNode|JournalNode|JournalNode|JournalNode|
|ResourceManager|ResourceManager|NodeManaer|NodeManaer|NodeManaer||||
||||||ZooKeeper|ZooKeeper|ZooKeeper|
|||Hive|Hive|Hive||||
|||Kafka|Kafka|Kafka||||
|||Spark|Spark|Spark||||
|||DataX|DataX|DataX||||
|ds-master|ds-master|ds-worker|ds-worker|ds-worker||||
|Maxwell||||||||
|Superset||||||||
|MySQL||||||||
||Flume|||||||
|Flink||||||||

##### 测试集群

|服务名称|子服务|服务器 hadoop10|服务器 hadoop11|服务器 hadoop12|
| ------------------| ----------------------| -----------------| -----------------| -----------------|
|HDFS|NameNode|✅|||
||DataNode|✅|✅|✅|
||SecondaryNameNode|||✅|
|Yarn|NodeManager|✅|✅|✅|
||ResourceManager||✅||
|Zookeeper|Zookeeper Server|✅|✅|✅|
|Kafka|Kafka|✅|✅|✅|
|Flume||✅|||
|Hive||✅|✅|✅|
|MySQL||✅|||
|DataX||✅|✅|✅|
|Spark||✅|✅|✅|
|DolphinScheduler|ApiApplicationServer|✅|||
||AlertServer|✅|||
||MasterServer|✅|||
||Worker|✅|✅|✅|

## 车险业务数据说明

### 业务流程说明

车险分为 下单 和 理赔 两条业务线

​![image](http://127.0.0.1:50577/assets/image-20250215122829-vaha0sf.png)​

​![image](http://127.0.0.1:50577/assets/image-20250215122859-4agqzqr.png)​

### 原始表结构说明

#### 行政区划表 administrative_region

```sql
create table `t_administrative_region` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `level` tinyint not null default 0 comment '行政区划级别',
  `name` varchar(255) not null default '' comment '名称',
  `superior_region` not null default '' comment '上级区划',
  `zip_code` varchar(6) not null default '' comment '邮政编码',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='行政区划表';
```

#### 产品表 product

```sql
create table `t_product` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `base_premium` bigint not null default 0 comment '基本保费',
  `coverage_amount` bigint not null default 0 comment '保额',
  `name` varchar(255) not null default '' comment '产品名称',
  `premium_rate` bigint not null default 0 comment '费率',
  `premium_type` tinyint(4) not null default 0 comment '保费计算基数类型：1-购置价格；2-当前价值',
  `type` int not null default 0 comment '保险类型：1-强制险；2-车损险；3-三者险',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='产品表';
```

#### 投保人表 policyholder

```sql
create table `t_policyholder` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `birthday` datetime not null default current_timestamp comment '出身日期',
  `gender` tinyint(4) not null default 0 comment '性别',
  `identification_number` varchar(255) not null default '' comment '证件号码',
  `identification_type` tinyint(4) not null default 0 comment '证件类型',
  `name` varchar(255) not null default '' comment '产品名称',
  `telephone` varchar(255) not null default '' cmment '电话',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='投保人表';
```

#### 车辆表 vehicle

```sql
create table `t_vehicle` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `brand` varchar(255) not null default '' comment '品牌',
  `purchase_date` datetime not null default current_timestamp comment '购入时间',
  `purchase_price` bigint not null default 0 comment '购入价格',
  `real_value` bigint not null default 0 comment '现价',
  `vehicle_model` varchar(255) not null default '' comment '车型',
  `vehicle_number` varchar(255) not null default '' cmment '车牌号',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='车辆表';
```

#### 保险经理人表 insurance_agent

```sql
create table `t_insurance_agent` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `birthday` datetime not null default current_timestamp comment '出身日期',
  `gender` tinyint(4) not null default 0 comment '性别',
  `name` varchar(255) not null default '' comment '产品名称',
  `telephone` varchar(255) not null default '' cmment '电话',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='保险经理人表';
```

#### 订单表 insurance_order

```sql
create table `t_insurance_order` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `total_amount` bigint not null default 0 comment '总金额',
  `administrative_region_id` bigint unsigned not null comment '地区',
  `insurance_agent_id` bigint unsigned not null comment '保险经纪人',
  `policyholder_id` bigint unsigned not null comment '投保人',
  `renew_from_id` bigint unsigned not null comment '续保单号',
  `vehicle_id` bigint unsigned not null comment '投保车辆',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='订单表';
```

#### 保单表 insurance_policy

```sql
create table `t_insurance_policy` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `amount` bigint not null default 0 comment '保险费用',
  `coverage_amount` bigint not null default 0 comment '保额',
  `end_time` datetime not null default current_timestamp comment '保险到期时间',
  `begin_time` datetime not null default current_timestamp comment '保险起始时间',
  `insurance_order_id` bigint unsigned not null comment '关联订单',
  `policyholder_id` bigint unsigned not null comment '投保人',
  `product_id` bigint unsigned not null comment '保险产品',
  `vehicle_id` bigint unsigned not null comment '投保车辆',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='保单表';
```

#### 分期表 installment

```sql
create table `t_installment` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `amount` bigint not null default 0 comment '应付费用',
  `due_time` datetime not null default current_timestamp comment '应付时间',
  `payment_time` datetime not null default current_timestamp comment '支付时间',
  `insurance_order_id` bigint unsigned not null comment '关联订单',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='分期表';
```

#### 支付表 payment

```sql
create table `t_payment` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `amount` bigint not null default 0 comment '支付金额',
  `payment_status` tinyint(4) not null default 0 comment '支付结果',
  `installment_id` bigint unsigned not null comment '关联分期记录',
  `policyholder_id` bigint unsigned not null comment '支付用户 ID',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='支付表';
```

#### 报案表 insurance_claim_reporting

```sql
create table `t_insurance_claim_reporting` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `address` varchar(255) not null default '' comment '详细地址',
  `reporting_telephone` varchar(255) not null default '' comment '报案人电话',  
  `status` tinyint(4) not null default 0 comment '案件状态：1-已报案；2-已出险；3-已关闭',
  `administrative_region_id` bigint unsigned not null comment '报案地区',
  `vehicle_id` bigint unsigned not null comment '关联车辆',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='报案表';
```

#### 出险表 insurance_claim

```sql
create table `t_insurance_claim` (
  `id` bigint unsigned not null auto_increment comment '主键 ID',
  `create_time` datetime not null default current_timestamp comment '创建时间',
  `update_time` datetime not null default current_timestamp on update current_timestamp comment '修改时间',
  `amount` bigint not null default 0 comment '出险费用',
  `insurance_agent_id` bigint unsigned not null comment '经受经纪人',
  `insurance_claim_reporting_id` bigint unsigned not null comment '关联案件',
  `insurance_policy_id` bigint unsigned not null comment '关联保险单',
  `vehicle_id` bigint unsigned not null comment '关联车辆',
  primary key (`id`)
) engine=innodb default charset=utf8mb4 comment='出险表';
```

## 环境准备

### 服务器和 JDK 准备

#### 服务器准备

修改主机名

```shell
# 查看主机名
hostname

# 修改主机名
hostnamectl --static set-hostname hadoop1000
```

设置静态 IP 地址

```shell
cd /etc/NetworkManager/system-connections/
vi ens160.nmconnection

[connection]
id=ens160
uuid=4060b5e3-b96f-3c58-b4da-54d13184466b
type=ethernet
autoconnect-priority=-999
interface-name=ens160
timestamp=1712000800
[ethernet]
[ipv4]
method=manual
# ip 地址,网关地址
address1=192.168.8.10/24,192.168.8.1
dns=192.168.8.1;114.114.114.114
may-fail=false
[ipv6]
addr-gen-mode=eui64
method=auto
[proxy]


nmcli connection reload ens160.nmconnection
nmcli connection up ens160
```

启用 root ssh

```shell
vi /etc/ssh/sshd_config

# 找到 PermitRootLogin，修改为
PermitRootLogin yes

systemctl restart sshd
```

关闭防火墙

```shell
➜  ~ systemctl stop firewalld
➜  ~ systemctl disable firewalld
```

禁用 SELINUX：

```shell
vi /etc/selinux/config

# 找到并更改
SELINUX=disabled
```

查看 SELinux 是否已被禁用：

```sql
sestatus

SELinux status:                 disabled
```

配置 IP 和 主机名 的映射

```shell
➜  ~ cat /etc/hosts
192.168.8.10 hadoop10
192.168.8.11 hadoop11
192.168.8.12 hadoop12
```

新建非 root  用户

```shell
useradd username
passwd username
```

配置 sudo 权限

```shell
vi /etc/sudoers

# 最好配在最后面，避免被覆盖
username ALL=(ALL) NOPASSWD:ALL
```

创建 /opt/module /opt/software 两个目录，并修改所属用户

```shell
mkdir /opt/module /opt/software

chown username:usergroup /opt/module
chown username:usergroup /opt/software
```

复制出 11 和 12

配置 root 免密登录

```shell
ssh-keygen
# 一路回车

# warp terminal 可以使用 Synchronized Inputs 功能
# https://docs.warp.dev/features/entry/synchronized-inputs
# 将公钥发给其他服务器
ssh-copy-id root@hadoop10
ssh-copy-id root@hadoop11
ssh-copy-id root@hadoop12
```

#### 编写集群分发脚本 xsync

安装 rsync，与 scp 的区别是 scp 是全量复制，rsync 是增量复制。

```shell
yum install -y rsync

```

xsync

```shell
#!/bin/bash

# 判断参数是否传入
if [ $# -lt 1 ]
then
	echo "必须至少传入一个文件或目录"
	exit
fi

# 遍历待分发的文件和目录
for f in $@
do
	# 判断待分发的文件、目录是否存在
	if [ -e $f ]
	then
		# 获取待分发文件、目录父目录
		pdir=$(cd $(dirname $f);pwd)
		# 获取待分发文件、目录的名称
		fname=$(basename $f)
		# 遍历主机准备分发
		for host in hadoop10 hadoop11 hadoop12 
		do
			echo "========== $host =========="
			if [ "$(hostname)" == "$host" ]; then
				echo "跳过当前主机：$host"
			else
				# 创建目录
				ssh $host "mkdir -p $pdir"
				# 分发
				# -av 打印进度
				rsync -av $pdir/$fname $host:$pdir
			fi
		done
	fi
done
```

将上述脚本保存在 ~/bin 下，并授予执行权限：

```shell
chmod +x xsync
```

执行 `xsync xsync`​ 将自己分发到其他主机

#### JDK 准备

卸载现有的 JDK

```shell
rpm -qa | grep -i java | xargs -n1 rpm -e --nodeps
```

下载 jdk

https://adoptium.net/zh-CN/temurin/releases/

```shell
cd /opt/software
```

解压到 /opt/module

```shell
tar zxvf openjdk8u212-b03.tar.xz -C /opt/module
```

配置环境变量

```shell
vi /etc/profile.d/my_env.sh
```

```shell
export JAVA_HOME=/opt/module/java
export PATH=$PATH:$JAVA_HOME/bin
```

```shell
source /etc/profile
```

#### 环境变量配置说明

bash 的运行模式可分为 login shell 和 non-login shell

例如，通过账号、密码登录系统就是 login shell，通过 ssh hadoop10 command 在 hadoop10 执行 command 就是在一个 non-login shell

​![image](http://127.0.0.1:50577/assets/image-20250215220117-ygw09po.png)​

可以看到 /etc/profile 和 ~/.bash_profile 中配置的环境变量在 non-login shell 下无法生效。

### 集群所有进程查看脚本

‍

```shell
#!/bin/bash

if [ $# -lt 1 ]
then
	echo "必须至少传入一个命令"
	exit
fi

for host in hadoop10 hadoop11 hadoop12
do
	echo "==========$host=========="
	if [ "$(hostname)" == "$host" ]; then
		"$@"
	else
		ssh $host "$@"
	fi
done
```

### Zookeeper 安装

```shell
cd /opt/software

wget https://archive.apache.org/dist/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz

tar zxvf apache-zookeeper-3.7.1-bin.tar.gz -C /opt/module

cd /opt/module/apache-zookeeper-3.7.1-bin

mkdir zkData

echo '10' >> zkData/myid

cd conf

cp zoo_sample.cfg zoo.cfg

vi zoo.cfg
dataDir=/opt/module/apache-zookeeper-3.7.1-bin/zkData
server.10=hadoop10:2888:3888
server.11=hadoop11:2888:3888
server.12=hadoop12:2888:3888

cd /opt/module

xsync apache-zookeeper-3.7.1-bin

# 修改其他 server 的 myid 为对应值

/opt/module/apache-zookeeper-3.7.1-bin/bin/zkServer.sh --help

# 以下命令在所有 server 同时执行
/opt/module/apache-zookeeper-3.7.1-bin/bin/zkServer.sh start

/opt/module/apache-zookeeper-3.7.1-bin/bin/zkServer.sh status
```

集群操作脚本，xzk

```shell
#!/bin/bash
case $1 in
"start")
	xcall /opt/module/apache-zookeeper-3.7.1-bin/bin/zkServer.sh start
;;
"stop")
	xcall /opt/module/apache-zookeeper-3.7.1-bin/bin/zkServer.sh stop
;;
"status")
	xcall /opt/module/apache-zookeeper-3.7.1-bin/bin/zkServer.sh status
;;
esac
```

常见命令

```shell
cd /opt/module/apache-zookeeper-3.7.1-bin/bin

./zkCli.sh

help

ls /

create /test

set /test hello,world

get /test

delete /test

create /test

# 必须先有父节点，才能新建子节点
create /test/test

create /test/test/test

# delete 只能删除子节点，deleteall 可以删除所有节点
deleteall /test
```

### Hadoop 安装

|服务名称|子服务|服务器 hadoop10|服务器 hadoop11|服务器 hadoop12|
| ----------| -------------------| -----------------| -----------------| -----------------|
|HDFS|NameNode|✅|||
||DataNode|✅|✅|✅|
||SecondaryNameNode|||✅|
|Yarn|NodeManager|✅|✅|✅|
||ResourceManager||✅||

```shell
cd /opt/software

wget wget https://mirrors.aliyun.com/apache/hadoop/common/hadoop-3.3.5/hadoop-3.3.5.tar.gz

tar zxvf hadoop-3.3.5.tar.gz -C /opt/module

cd /opt/module/hadoop-3.3.5

cd etc/hadoop/

vi core-site.xml
	<property>
		<name>hadoop.tmp.dir</name>
		<value>/opt/module/hadoop-3.3.5/data</value>
		<description>Where Hadoop will place all of its working files</description>
	</property>
	<property>
		<name>fs.defaultFS</name>
		<value>hdfs://hadoop10:8020</value>
		<description>Where HDFS NameNode can be found on the network</description>
	</property>

	<property>
		<name>hadoop.http.staticuser.user</name>
		<value>root</value>
		<description>配置HDFS网页登录使用的静态用户为root</description>
	</property>

	<property>
		<name>hadoop.proxyuser.root.groups</name>
		<value>*</value>
		<description>
			What user groups are allow to connect to the HDFS proxy.
			* for all.</description>
	</property>
	<property>
		<name>hadoop.proxyuser.root.hosts</name>
		<value>*</value>
		<description>
			What user hosts are allow to connect to the HDFS proxy.
			* for all.
		</description>
	</property>
	<property>
		<name>hadoop.proxyuser.root.users</name>
		<value>*</value>
		<description>
			配置允许通过代理访问的用户
		</description>
	</property>

vi hdfs-site.xml
	<property>
		<name>dfs.namenode.http-address</name>
		<value>hadoop10:9870</value>
		<description>
			namenode web 端访问地址
		</description>
	</property>
	<property>
		<name>dfs.namenode.secondary.http-address</name>
		<value>hadoop12:9868</value>
		<description>
			secondary namenode web 端访问地址
		</description>
	</property>
	<property>
		<name>dfs.replication</name>
		<value>3</value>
		<description>
			指定 HDFS 副本数量
		</description>
	</property>

vi yarn-site.xml
	<property>
		<name>yarn.nodemanager.aux-services</name>
		<value>mapreduce_shuffle</value>
	</property>

	<property>
		<name>yarn.resourcemanager.hostname</name>
		<value>hadoop11</value>
	</property>
	
	<property>
		<name>yarn.nodemanagermanager.env-whitelist</name>
		<value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPERD_HOME</value>
		<description>环境变量的继承</description>
	</property>

	<property>
		<name>yarn.scheduler.minimum-allocation-mb</name>
		<value>512</value>
	</property>
	<property>
		<name>yarn.scheduler.maximum-allocation-mb</name>
		<value>4096</value>
	</property>
	<property>
		<name>yarn.nodemanager.resource.memory-mb</name>
		<value>4096</value>
	</property>

	<property>
		<name>yarn.nodemanager.pmem-check-enabled</name>
		<value>false</value>
		<description>Whether physics memory limits will be enforced for containers</description>
	</property>
	<property>
		<name>yarn.nodemanager.vmem-check-enabled</name>
		<value>false</value>
		<description>Whether virtual memory limits will be enforced for containers</description>
	</property>

vi workers
hadoop10
hadoop11
hadoop12

vi mapred-site.xml
	<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
	</property>


```

‍

配置历史服务器

```shell
vi mapred-site.xml
	<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
	</property>

	<property>
		<name>mapreduce.jobhistory.address</name>
		<value>hadoop10:10020</value>
	</property>

	<property>
		<name>mapreduce.jobhistory.webapp.address</name>
		<value>hadoop10:19888</value>
	</property>

vi yarn-site.xml
	<!-- 追加 -->
	<property>
		<name>yarn.log-aggregation-enable</name>
		<value>true</value>
		<description>开启日志聚集功能</description>
	</property>
	<property>
		<name>yarn.log.server.url</name>
		<value>http://hadoop10:19888/jobhistory/logs</value>
	</property>
	<property>
		<name>yarn.log-aggregation.retain-seconds</name>
		<value>604800</value>
	</property>
```

配置完成后，将 hadoop 分发到其他 server：

```shell
cd /opt/module
xsync hadoop-3.3.5
```

在 hadoop10 上进行 namenode format：

```shell
vi /etc/profile.d/my_env.sh
# 新增
export HADOOP_HOME=/opt/module/hadoop-3.3.5
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin

export HDFS_NAMENODE_USER=root
export HDFS_DATANODE_USER=root
export HDFS_SECONDARYNAMENODE_USER=root
export YARN_RESOURCEMANAGER_USER=root
export YARN_NODEMANAGER_USER=root

source /etc/profile

# 分发给其他 server，并执行 source /etc/profile

hadoop namenode -format
...
2025-02-16 12:34:13,451 INFO common.Storage: Storage directory /opt/module/hadoop-3.3.5/data/dfs/name has been successfully formatted.
...

start-dfs.sh

# 在 sourcemanager server 也就是 hadoop11 启动 yarn
start-yarn.sh

xcall jps
==========hadoop10==========
32263 NameNode
33994 Jps
10043 QuorumPeerMain
32460 DataNode
33660 NodeManager
==========hadoop11==========
15665 ResourceManager
14962 DataNode
5428 QuorumPeerMain
15813 NodeManager
16287 Jps
==========hadoop12==========
5474 QuorumPeerMain
14970 DataNode
15098 SecondaryNameNode
15630 Jps
15487 NodeManager
```

打开 web 页面进一步验证：

hdfs 192.168.8.10:9870

yarn 192.168.8.11:8088

‍

#### 启动脚本

xhdp

```shell
#!/bin/bash
if [ $# -lt 1 ]; then
	echo "必须传入 start、stop ..."
	exit
fi
case $1 in
"start")
	echo "==========启动 hdfs=========="
	ssh hadoop10 "start-dfs.sh"
	echo "==========启动 yarn=========="
	ssh hadoop11 "start-yarn.sh"
;;
"stop")
	echo "==========停止 yarn=========="
	ssh hadoop11 "stop-yarn.sh"
	echo "==========停止 hdfs=========="
	ssh hadoop10 "stop-dfs.sh"
;;
esac
```

授予启动权限，并同步至其他机器：

```shell
cd ~/bin

vi xhdp

chmod +x xhdp

xsync xhdp
```

#### HDFS 存储多目录

```shell
vi hdfs-site.xml
	<property>
		<name>dfs.datanode.data.dir</name>
		<value>file:///dfs/data1,file:///hd2/dfs/data2,file:///hd3/dfs/data3,file:///hd4/dfs/data4</value>
	</property>
```

#### 集群数据均衡

节点数据均衡

```shell
start-balancer.sh -threshold 10
```

* -threshold 10 各节点间磁盘利用率不超过 10%

```shell
stop-balancer.sh
```

磁盘数据均衡

```shell
# 生成均衡计划
hdfs diskbalancer -plan hadoop11
# 执行
hdfs diskbalancer -execute hadoop11.plan.json
# 查询
hdfs diskbalancer -query hadoop11
# 取消
hdfs diskbalancer -cancel hadoop11.plan.json
```

#### Hadoop 参数调优

大集群或大量客户端时，需增大 dfs.namenode.handler.count 为 $20 \times log_e^{clusterSize}$：

```shell
vi hdfs-site.xml
	<property>
		<name>dfs.namenode.handler.count</name>
		<value>10</value>
	</property>
```

数据量大时，需增大 yarn.nodemanager.resource.memory-mb。

### Kafka 安装

#### 安装步骤

```shell
cd /opt/software

wget https://archive.apache.org/dist/kafka/3.3.1/kafka_2.12-3.3.1.tgz

tar zxvf kafka_2.12-3.3.1.tgz -C /opt/module

cd /opt/module/kafka_2.12-3.3.1/config

vi server.properties
broker.id=10 # 10 11 12
log.dirs=/opt/module/kafka_2.12-3.3.1/data
zookeeper.connect=hadoop10:2181,hadoop11:2181,hadoop12:2181/kafka_2.12-3.3.1

cd /opt/module

xsync kafka_2.12-3.3.1

# 修改 broker.id 为对应值

# 启动（需先启动 zk）
/opt/module/kafka_2.12-3.3.1/bin/kafka-server-start.sh -daemon /opt/module/kafka_2.12-3.3.1/config/server.properties

# 在其他 server 启动
```

#### 启动脚本

xkafka

```shell
#!/bin/bash
if [ $# -lt 1 ]; then
	echo "必须传入 start、stop ..."
	exit
fi
case $1 in
"start")
	echo "==========启动 kafka=========="
	xcall /opt/module/kafka_2.12-3.3.1/bin/kafka-server-start.sh -daemon /opt/module/kafka_2.12-3.3.1/config/server.properties
;;
"stop")
	echo "==========停止 kafka=========="
	xcall /opt/module/kafka_2.12-3.3.1/bin/kafka-server-stop.sh
;;
esac
```

#### 常用命令操作

topic：

```shell
cd /opt/module/kafka_2.12-3.3.1/bin

./kafka-topics.sh --create --bootstrap-server hadoop10:9092 --topic test --partitions 2 --replication-factor 2

./kafka-topics.sh --list --bootstrap-server hadoop10:9092

./kafka-topics.sh --describe --topic test --bootstrap-server hadoop10:9092

./kafka-topics.sh --alter --topic test --bootstrap-server hadoop10:9092 --partitions 3

./kafka-topics.sh --delete --topic test --bootstrap-server hadoop10:9092
```

producer 和 consumer：

```shell
cd /opt/module/kafka_2.12-3.3.1/bin

./kafka-console-producer.sh --topic test --bootstrap-server hadoop10:9092
k1 v1
k2 v2
k3 v3

./kafka-console-consumer.sh --topic test --bootstrap-server hadoop10:9092 --from-beginning
```

### Flume 安装

#### 安装步骤

```shell
cd /opt/software

wget https://mirrors.aliyun.com/apache/flume/1.11.0/apache-flume-1.11.0-bin.tar.gz

tar zxvf apache-flume-1.11.0-bin.tar.gz -C /opt/module

cd /opt/module/apache-flume-1.11.0-bin/conf

vi log4j2.xml 
...
<Configuration status="ERROR">
  <Properties>
    <Property name="LOG_DIR">/opt/module/apache-flume-1.11.0-bin/logs</Property>
  </Properties>
  ...

  <Loggers>
    ...
    <Root level="INFO">
      <AppenderRef ref="LogFile" />
      <AppenderRef ref="Console" />
    </Root>
  </Loggers>
</Configuration>

# 修改配置
vi flume-env.sh.template
export JAVA_OPTS="-Xms2048m -Xmx2048m -Dcom.sun.management.jmxremote"
```

### MySQL 安装

## 数据模拟
