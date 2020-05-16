# hive sql 基本操作

## 前言

hive依赖于hdfs存储数据，hive将hql转换成mapreduce执行，所以说hive是基于haddop的一个数据仓库工具，实质就是一款基于hdfs的mapreduce的计算框架，对存储在hdfs中的数据进行分析和管理。

## hive特点

优点：

1.可扩展性，横向扩展，hive可以自由的扩展集群的规模，一般不需要重启服务横向扩展。

2.延展性，支持自定义函数，用户可以根据自己的需求来实现自己的函数。

3.容错性：即使有节点出现问题，sql仍可完成执行。

缺点：

1.hive不支持记录级别的增删改操作，但是用户可以通过查询生成新表或者将查询结果导入到文件中。

2.hive 的查询延时严重，因为mapreduce job的启动过程消耗很长时间，所以不能用在交互查询系统中。

3.hive不支持事务（因为没有增删改，所以主要用来做olap（联机分析处理），而不是pltp（联机事务处理），这是数据处理的两大级别。

## hive架构

![image-20200516174507387](C:\Users\12466\AppData\Roaming\Typora\typora-user-images\image-20200516174507387.png)

元数据存储：

元数据通俗的说就是存储在hive中的数据的描述信息。hive中的元数据通常包括：表的名字，表的列和分区及其属性，表的属性（内部表和外部表），表的数据所在目录。

通常寸在我们自己创建的mysql库，hive和mysql之间通过metastore服务交互。

hivesql 通过命令行或者客户端提交，经过compiler编译器，运用metastore中的元数据进行类型检测和语法分析，生成一个逻辑方案，然后通过优化处理，产生一个mapreduce任务。

## hive数据存储

1.Hive 的存储结构包括数据库、表、视图、分区和表数据等。数据库，表，分区等等都对应 HDFS 上的一个目录。表数据对应 HDFS 对应目录下的文件。

2.Hive 中所有的数据都存储在 HDFS 中，没有专门的数据存储格式，因为 Hive 是读模式（Schema On Read），可支持 TextFile，SequenceFile，RCFile 或者自定义格式等

3.只需要在创建表的时候告诉 Hive 数据中的列分隔符和行分隔符，Hive 就可以解析数据    Hive 的默认列分隔符：控制符 Ctrl + A，\x01    Hive 的默认行分隔符：换行符 \n

4.Hive 中包含以下数据模型：    

database：在 HDFS 中表现为${hive.metastore.warehouse.dir}目录下一个文件夹    

table：在 HDFS 中表现所属 database 目录下一个文件夹    

external table：与 table 类似，不过其数据存放位置可以指定任意 HDFS 目录路径    

partition：在 HDFS 中表现为 table 目录下的子目录    

bucket：在 HDFS 中表现为同一个表目录或者分区目录下根据某个字段的值进行 hash 散列之后的多个文件    view：与传统数据库类似，只读，基于基本表创建

5.Hive 的元数据存储在 RDBMS 中，除元数据外的其它所有数据都基于 HDFS 存储。默认情况下，Hive 元数据保存在内嵌的 Derby 数据库中，只能允许一个会话连接，只适合简单的测试。实际生产环境中不适用，为了支持多用户会话，则需要一个独立的元数据库，使用MySQL 作为元数据库，Hive 内部对 MySQL 提供了很好的支持。

### hive中的表分为内部表，外部表，分区表和bucket表

内部表和外部表的区别：

删除内部表，删除表元数据和数据

删除外部表，删除元数据，不删除数据

hive其实仅仅是对存储在hdfs上的数据提供了一种新的抽象。而不是管理存储在hdfs上的数据。

#### 分区表和分通表的区别：

##### 分区表：

hive中的表对应为hdfs上的指定目录，在查询数据时候，默认会对全表进行扫描，这样时间和性能的消耗都非常大。

分区为hdfs上表目标的子目录，数据按照分区存储在子目录中。在查询的where字句中包含分区条件，则直接从该分区去查找，而不是扫描整个表目录，合理的分区设计可以极大提高查询速度和性能。

使用场景：

通常在管理大规模数据集的时候都需要进行分区，比如将日志文件按天进行分区，从而保证数据细粒度的划分，使得查询性能得到提升。

##### 分通表：

分区提供了一个隔离数据和优化查询的可行方案，但是并非所有的数据集都可以形成合理的分区，分区的数量也不是越多越好，过多的分区条件可能会导致很多分区上没有数据。同时hive会限制动态分区可以创建的最大分区数，用来避免过多分区文件对文件系统产生负担。所以hive提供了一种更加细粒度的数据拆分方案：分桶

分桶表会将指定列的值进行哈希散列，并对bucket取余，然后存储到对应的bucket中。

## hive使用

### 基本使用

```sql
创建库
create database if not exists mydb;

查看库
show databases;

切换库：
use mydb;

创建表
create table if not exists t_user(id string,name string);
create table t_user(id string,name string) row format delemited fields terminated by ',';

查看表
show tables;

插入数据
insert into table t_user values('1','huangbo'),('2','xuzheng'),('3','wangbaoqiang');

查询数据
select * from t_user;

导入数据
(a)导入hdfs数据：
load data inpath '/user.txt' into table t_user;
(b)导入本地数据：
load data local inpath '/home/hadoop/user.txt' into table t_user;


tips:
1.进入到用户的主目录，使用命令cat /home/hadoop/.hivehistory 可以查看到hive执行的历史命令
2.执行查询时若想显示表头信息时:  set hive.cli.print.header=true;
3.hive的执行日志的存储目录在${java.io.tmpdir}/${user.name}/hive.log中，假如使用hadoop用户操作的 hive，那么日志文件的存储路径为：/temp/hadoop/hive.log
```

### DDl

#### 库操作

```sql
创建库：

创建普通库    
create database dbname;

创建库的时候检查存与否    
create databse if not exists dbname;

创建库的时候带注释    
create database if not exists dbname comment 'create my db named dbname';

创建带属性的库    
create database if not exists dbname with dbproperties ('a'='aaa','b'='bbb');    
create database if not exists myhive with dbproperties ('a'='aaa','b'='bbb');
```

```sql
查看库：

查看有哪些数据库    
show databases;

显示数据库的详细属性信息
desc database [extended] dbname;    
示例：desc database extended myhive;

查看正在使用哪个库    
select current_database();

查看创建库的详细语句    
show create database mydb;
```

```sql
删除库：

删除库操作
drop database dbname;    
drop database if exists dbname; 

默认情况下，hive 不允许删除包含表的数据库，有两种解决办法：

1.手动删除库下所有表，然后删除库

2.使用 cascade 关键字    
drop database if exists dbname cascade;    

默认情况下就是 restrict    
drop database if exists myhive ==== drop database if exists myhive restrict
```

#### 表操作

```sql
创建表

概念：
CREATE TABLE：创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXISTS 选项来忽略这个异常。

EXTERNAL：关键字可以让用户创建一个外部表，在建表的同时指定一个指向实际数据的路径（LOCATION），如果不存在，则会自动创建该目录。Hive 创建内部表时，会将数据移动到数据仓库指向的路径；若创建外部表，仅记录数据所在的路径，不对数据的位置做任何改变。

PARTITIONED BY：在 Hive Select 查询中一般会扫描整个表内容，会消耗很多时间做没必要的工作。有时候只需要扫描表中关心的一部分数据，因此建表时引入 partition 概念。个表可以拥有一个或者多个分区，每个分区以文件夹的形式单独存在表文件夹的目录下，分区是以字段的形式在表结构中存在，通过 desc table 命令可以查看到字段存在，但是该字段不存放实际的数据内容，仅仅是分区的表示。    
分区建表分为 2 种：        
一种是单分区，也就是说在表文件夹目录下只有一级文件夹目录        
一种是多分区，表文件夹下出现多文件夹嵌套模式LIKE：允许用户复制现有的表结构，但是不复制数据。    
示例：
create table tableA like tableB（创建一张 tableA 空表复制 tableB 的结构）

COMMENT：可以为表与字段增加描述ROW FORMAT：用户在建表的时候可以自定义 SerDe 或者使用自带的 SerDe。如果没有指定ROW FORMAT或者 ROW FORMAT DELIMITED，将会使用自带的 SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的 SerDe，Hive 通过 SerDe 确定表的具体的列的数据。

STORED AS：如果文件数据是纯文本，可以使用 STORED AS TEXTFILE，默认也是 textFile 格式，可以通过执行命令 set hive.default.fileformat，进行查看，如果数据需要压缩，使用 STORED AS SEQUENCEFILE。    
A、默认格式 TextFile，数据不做压缩，磁盘开销大，数据解析开销大。可结合 gzip、bzip2使用(系统自动检查，执行查询时自动解压)，但使用这种方式，hive 不会对数据进行切分，从而无法对数据进行并行操作    
B、SequenceFile 是 Hadoop API 提供的一种二进制文件支持，文件内容是以序列化的 kv对象来组织的，其具有使用方便、可分割、可压缩的特点。 SequenceFile 支持三种压缩选择：NONE，RECORD，BLOCK。Record 压缩率低，一般建议使用 BLOCK 压缩    
C、RCFILE 是一种行列存储相结合的存储方式。首先，其将数据按行分块，保证同一个record 在一个块上，避免读一个记录需要读取多个 block。其次，块数据列式存储，有利于数据压缩和快速的列存取。相比 TEXTFILE 和SEQUENCEFILE，RCFILE 由于列式存储方式，数据加载时性能消耗较大，但是具有较好的压缩比和查询响应。数据仓库的特点是一次写入、多次读取，因此，整体来看，RCFILE 相比其余两种格式具有较明显的优势

CLUSTERED BY：对于每一个表（table）或者分区，Hive 可以进一步组织成桶，也就是说桶是更为细粒度的数据范围划分。Hive 也是针对某一列进行桶的组织。Hive 采用对列值哈希，然后除以桶的个数求余的方式决定该条记录存放在哪个桶当中。    
把表（或者分区）组织成桶（Bucket）有两个理由：        
（1）获得更高的查询处理效率。桶为表加上了额外的结构，Hive 在处理有些查询时能利用这个结构。具体而言，连接两个在（包含连接列的）相同列上划分了桶的表，可以使用 Map 端连接（Map Join）高效的实现。比如 JOIN 操作。对于 JOIN 操作两个表有一个相同的列，如果对这两个表都进行了桶操作。那么将保存相同列值的桶进行 JOIN操作就可以，可以大大较少 JOIN 的数据量。        
（2）使取样（Sampling）更高效。在处理大规模数据集时，在开发和修改查询的阶段，如果能在数据集的一小部分数据上试运行查询，会带来很多方便。LOCATION：指定数据文件存放的 HDFS 目录，不管内部表还是外表，都可以指定。不指定就在默认的仓库路径。

最佳实践：        
如果创建内部表请不要指定 location        
如果创建表时要指定 location，请创建外部表。   

1.创建内部表    
create table mytable (id int, name string)row format delimited fields terminated by ',' stored as textfile;

2.创建外部表    
create external table mytable (id int, name string) row format delimited fields terminated by ',' location '/user/hive/warehouse/mytable2';

3.创建分区表    
create table mytable (id int, name string) partitioned by(sex string) row format delimited fields terminated by ',' stored as textfile;    
插入数据        
插入男分区数据：
load data local inpath '/root/hivedata/mingxing.txt' overwrite into table mytable partition(sex='boy');       
插入女分区数据：
load data local inpath '/root/hivedata/mingxing.txt' overwrite into table mytable partition(sex='girl');        
查询表分区：
show partitions mytable

4.创建分桶表    
create table stu_buck(Sno int,Sname string,Sex string,Sage int,Sdept string) clustered by(Sno) sorted by(Sno DESC) into 4 buckets row format delimited fields terminated by ',';

5.使用like关键字拷贝表
// 不管老表是内部表还是外部表，new_table 都是内部表
create table new_table like mytable;
// 不管老表是内部表还是外部表，如果加 external 关键字，new_table 都是外部表
create external table if not exists new_table like mytable;
```

```sql
修改表

重命名表
alter table stu rename to student;

修改表属性
不支持修改表名，和表的数据存储目录
alter table table_name set TBLPROPERTIES('comment'='my new students table');

修改列分隔符
alter table student set SERDEPROPERTIES ('field.delim'='-');

增加/删除/改变/替换列
alter table student add columns(course string);
alter table student change column id ids int;
alter table student replace columns(id int,name string,address string);

增加/删除分区
添加分区示例：
// 不指定分区路径，默认路径
ALTER TABLE student ADD partition(part='a') partition(part='b'); 
// 指定分区路径
ALTER TABLE student ADD IF NOT EXISTS partition(part='bb') location '/myhive_bb' partition(part='cc') location '/myhive_cc'; 
// 修改分区路径示例
ALTER TABLE student partition (part='bb') SET location '/myhive_bbbbb'; 
// 删除分区示例
ALTER TABLE student DROP if exists partition(part='aa');
ALTER TABLE student DROP if exists partition(part='aa') if exists partition(part='bb'); 

最后补充：
1.防止分区被删除：
alter table student_p partition (part='aa') enable no_drop;
2.防止分区被查询：alter table student_p partition (part='aa') enable offline;
enable 和 disable 是反向操作
```

```sql
删除表
drop table if exists mytable;

清空表
truncate table student;
truncate table student partition(city='beijing');
```

```sql
辅助命令

查看数据库列表
show databases;
show databases like 'my*';

查看数据表
show tables;
show tables in myhive;

查看数据表的建表语句
show create table student;

查看hive函数列表
show functions;

查看hive表分区
show partitions student;
show partitions student partition(city='shanghai');

查看表的详细信息(元数据信息)
desc student;
desc extended student;
desc formatted student;

查看数据库的详细属性
desc database student;
desc database extended student;

清空数据表
truncate table student;
```

### DML

```sql
load加载数据

1.LOAD 操作只是单纯的复制或者移动操作，将数据文件移动到 Hive 表对应的位置。

2.LOCAL 关键字    
如果指定了 LOCAL， LOAD 命令会去查找本地文件系统中的 filepath。    
如果没有指定 LOCAL 关键字，则根据 inpath 中的 uri 查找文件    
注意：uri 是指 hdfs 上的路径，分简单模式和完整模式两种，
例如：    
简单模式：/user/hive/project/data1    
完整模式：hdfs://namenode_host:9000/user/hive/project/data1

3.filepath：    
相对路径，例如：project/data1     
绝对路径，例如：/user/home/project/data1     
包含模式的完整 URI，列如：hdfs://namenode_host:9000/user/home/project/data1    
注意：inpath 子句中的文件路径下，不能再有文件夹。

4.overwrite 关键字    
如果使用了 OVERWRITE 关键字，则目标表（或者分区）中的内容会被删除，然后再将 filepath 指向的文件/目录中的内容添加到表/分区中。    
如果目标表（分区）已经有一个文件，并且文件名和 filepath 中的文件名冲突，那么现有的文件会被新文件所替代。

1.加载本地相对路径数据    
load data local inpath '/student.txt' into table student;

2.加载绝对路径数据    
load data local inpath '/hadoop/data/student.txt' into table student;

3.加载绝对路径数据    
load data local inpath 'hdfs://192.168.10.201:9000/hadoop/data/student.txt' into table student;

4.overwrite 关键字使用    
load data local inpath '/student.txt' overwrite into table student;
```

```sql
insert插入数据

1.插入一条数据：    
INSERT INTO TABLE table_name VALUES(XX,YY,ZZ);

2.利用查询语句将结果导入新表：    
INSERT OVERWRITE [INTO] TABLE table_name [PARTITION (partcol1=val1, partcol2=val2 ...)]    select_statement1 FROM from_statement

3.多重插入    
FROM from_statement INSERT OVERWRITE TABLE table_name1 [PARTITION (partcol1=val1,partcol2=val2 ...)] select_statement1 INSERT OVERWRITE TABLE table_name2 [PARTITION (partcol1=val1, partcol2=val2 ...)] select_statement2] ...

4.分区插入    
分区插入有两种，一种是静态分区，另一种是动态分区。如果混合使用静态分区和动态分区，则静态分区必须出现在动态分区之前。现分别介绍这两种分区插入。

静态分区：    
A)创建静态分区表    
B)从查询结果中导入数据    
C)查看插入结果动态分区：静态分区需要创建非常多的分区，那么用户就需要写非常多的 SQL！

Hive 提供了一个动态分区功能，其可以基于查询参数推断出需要创建的分区名称。    
A)创建分区表，和创建静态分区表是一样的    
B)参数设置        
set hive.exec.dynamic.partition=true;        
set hive.exec.dynamic.partition.mode=nonstrict;（注意：动态分区默认情况下是开启的。但是却以默认是”strict”模式执行的，在这种模式下要求至少有一列分区字段是静态的。这有助于阻止因设计错误导致查询产生大量的分区。但是此处我们不需要静态分区字段，估将其设为 nonstrict。）对应还有一些参数可设置：    
set hive.exec.max.dynamic.partitions.pernode=100; //每个节点生成动态分区最大个数    
set hive.exec.max.dynamic.partitions=1000; //生成动态分区最大个数，如果自动分区数大于这个参数，将会报错    
set hive.exec.max.created.files=100000; //一个任务最多可以创建的文件数目   
set dfs.datanode.max.xcievers=4096; //限定一次最多打开的文件数    
set hive.error.on.empty.partition=false; //表示当有空分区产生时，是否抛出异常    （小技能补充：如果以上参数被更改过，想还原，请使用 reset 命令执行一次即可）

5.CTAS（create table … as select …） CREATE TABLE mytest AS SELECT name, age FROM test;    注意：CTAS 操作是原子的，因此如果 select 查询由于某种原因而失败，新表是不会创建的！   


示例：
from mingxing insert into table mingxing2 select id,name,sex,ageinsert into table mingxing select id,name,sex ,age,department ;

从 mingxing 表中，按不同的字段进行查询得的结果分别插入不同的 hive 表 
from student insert into table ptn_student partition(city='MA') select id,name,sex,age,department where department='MA'insert into table ptn_student partition(city='IS') select id,name,sex,age,department where department='IS';

insert into table ptn_student partition(city='CS') select id,name,sex,age,department where department='CS'; 
动态数据插入：
// 一个分区字段：insert into table test2 partition (age) select name,address,school,age from students;
// 多个分区字段：insert into table student_ptn2 partition(city='sa',zipcode) select id, name, sex, age,department, department as zipcode from studentss;
```

```sql
select查询

1. select_ condition 查询字段

2.table_name 表名

3.order by(字段) 全局排序，因此只有一个 reducer，只有一个 reduce task 的结果，比如文件名是 
000000_0，会导致当输入规模较大时，需要较长的计算时间。

4.sort by(字段) 局部排序，不是全局排序，其在数据进入 reducer 前完成排序。因此，如果用 sort by 进行排序，并且设置 mapred.reduce.tasks>1，则 sort by 只保证每个 reducer的输出有序，不保证全局有序。

5.distribute by(字段) 根据指定的字段将数据分到不同的 reducer，且分发算法是 hash 散列。

6.cluster by(字段) 除了具有 Distribute by 的功能外，还会对该字段进行排序。因此，如果分桶和 sort 字段是同一个时，此时，cluster by = distribute by + sort by,如果我们要分桶的字段和要排序的字段不一样，那么我们就不能使用 clustered by ,分桶表的作用：最大的作用是用来提高 join 操作的效率. 

示例

1.获取年龄大的三个学生    
select id, age,name from student where stat_date= '20140101' order by age desc limit 3;

2.查询学生年龄按降序排序    
set mapred.reduce.tasks=4;    
select id, age, name from student sort by age desc;    
select id, age, name from student order by age desc;    
select id, age, name from student distribute by age;  

分桶和排序的组合操作，对 id 进行分桶，对 age，id 进行降序排序
insert overwrite directory '/root/outputdata6' select * from mingxing2 distribute by id sort by age desc, id desc;     

分桶操作，按照 id 分桶，但是不进行排序    
insert overwrite directory '/root/outputdata4' select * from mingxing2 distribute by id sort by age;     

分桶操作，按照 id 分桶，并且按照 id 排序    
insert overwrite directory '/root/outputdata3' select * from mingxing2 cluster by id;     

分桶查询：    
指定开启分桶：    
set hive.enforce.bucketing = true; 
// 在旧版本中需要开启分桶查询的开关    指定 reducetask 数量，也就是指定桶的数量    
set mapreduce.job.reduces=4;    
insert overwrite directory '/root/outputdata3' select * from mingxing2 cluster by id;

3.按学生名称汇总学生年龄    
select name, sum(age) from student group by name;
```

```sql
一、解释三个执行参数    
In order to change the average load for a reducer (in bytes):        
set hive.exec.reducers.bytes.per.reducer=<number>    

In order to limit the maximum number of reducers:        
set hive.exec.reducers.max=<number>    

In order to set a constant number of reducers:        
set mapreduce.job.reduces=<number>    

1.直接使用不带设置值得时候是可以查看到这个参数的默认值：        
set hive.exec.reducers.bytes.per.reducer        
hive.exec.reducers.bytes.per.reducer：一个 hive，就相当于一次 hive 查询中，每一个reduce 任务它处理的平均数据量        
如果要改变值，我们使用这种方式：        
set hive.exec.reducers.bytes.per.reducer=51200000    

2.查看设置的最大 reducetask 数量        
set hive.exec.reducers.max        
hive.exec.reducers.max：一次 hive 查询中，最多使用的 reduce task 的数量我们可以这样使用去改变这个值:set hive.exec.reducers.max = 20    

3.查看设置的一个 reducetask 常量数量        
set mapreduce.job.reduces        
mapreduce.job.reduces：我们设置的 reducetask 数量



二、HQL 是否被转换成 MR 的问题    
前面说过，HQL 语句会被转换成 MapReduce 程序执行，但是上面的例子可以看出部分HQL 语句并不会转换成 MapReduce，那么什么情况下可以避免转换呢？   
1.select * from student; // 简单读取表中文件数据时不会    
2.where 过滤条件中只是分区字段时不会转换成 MapReduce    
3.set hive.exec.mode.local.auto=true; // hive 会尝试使用本地模式执行    否则，其他情况都会被转换成 MapReduce 程序执行
```

```sql
hive join查询

Hive 支持等值连接（equality join）、外连接（outer join）和（left/right join）。
Hive 不支持非等值的连接，因为非等值连接非常难转化到 map/reduce 任务。
另外，Hive 支持多于 2 个表的连接。

写查询时要注意以下几点：

1. 只支持等值链接，支持 and，不支持 or
例如：
SELECT a.* FROM a JOIN b ON (a.id = b.id)
SELECT a.* FROM a JOIN b ON (a.id = b.id AND a.department = b.department)是正确的；
然而：
SELECT a.* FROM a JOIN b ON (a.id>b.id)是错误的。

2.可以 join 多于 2 个表
例如：
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)
如果 join 中多个表的 join key 是同一个，则 join 会被转化为单个 map/reduce 任务，
例如:
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)被转化为单个 map/reduce 任务，因为 join 中只使用了 b.key1 作为 join key。
例如：
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2),而这一 join 被转化为 2 个 map/reduce 任务。因为 b.key1 用于第一次 join 条件，而b.key2 用于第二次 join。

3.Join 时，每次 map/reduce 任务的逻辑reducer 会缓存 join 序列中除了最后一个表的所有表的记录，再通过最后一个表将结果序列化到文件系统。这一实现有助于在 reduce 端减少内存的使用量。实践中，应该把最大的那个表写在最后（否则会因为缓存浪费大量内存）。
例如：
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key1)所有表都使用同一个 join key（使用 1 次 map/reduce 任务计算）。
Reduce 端会缓存 a 表和 b 表的记录，然后每次取得一个 c 表的记录就计算一次 join 结果，类似的还有：
SELECT a.val, b.val, c.val FROM a JOIN b ON (a.key = b.key1) JOIN c ON (c.key = b.key2)这里用了 2 次 map/reduce 任务：
第一次缓存 a 表，用 b 表序列化；
第二次缓存第一次 map/reduce 任务的结果，然后用 c 表序列化。

4.HiveJoin 分三种：inner join, outer join, semi join其中：outer join 包括 left join，right join 和 full outer join,主要用来处理 join 中空记录的情况



a)inner join（内连接）（把符合两边连接条件的数据查询出来）    
select * from tablea a inner join tableb b on a.id=b.id;

b)left join（左连接，等同于 left outer join）    
1.以左表数据为匹配标准，左大右小    2.匹配不上的就是 null    3.返回的数据条数与左表相同   
HQL 语句：
select * from tablea a left join tableb b on a.id=b.id; 

c)right join（右连接，等同于 right outer join）    
1.以右表数据为匹配标准，左小右大    2.匹配不上的就是 null    3.返回的数据条数与右表相同    
HQL 语句：
select * from tablea a right join tableb b on a.id=b.id; 

d)left semi join（左半连接）（因为 hive 不支持 in/exists 操作（1.2.1 版本的 hive 支持in 的操作），所以用该操作实现，并且是 in/exists 的高效实现）    
select * from tablea a left semi join tableb b on a.id=b.id; 

e)full outer join（完全外链接）    
select * from tablea a full outer join tableb b on a.id=b.id;
```

