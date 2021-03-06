<!--
 * @Github: https://github.com/Certseeds/CS302_OS
 * @Organization: SUSTech
 * @Author: nanoseeds
 * @Date: 2020-05-25 16:53:29
 * @LastEditors: nanoseeds
 * @LastEditTime: 2020-05-26 10:59:31
 * @License: CC-BY-NC-SA_V4_0 or any later version 
 -->
[TOC]

## Spark SQL: Relational Data Processing in Spark

### Part_I: introduction
1. 大数据非常的复杂,很多新系统提供了声明式查询API,关系型系统提供的操作比较少.用户需要对多种的数据集的读写编写额外的代码,并且关系型运算和过程形两者不能同时选择.

2. 所以引入了SparkSQL,基于Shark(不是Spark),并提供了两个特性.
  + 提供了DataFrame,一个可以同时处理外部数据源和内部数据结构RDD的API,有惰性求值的特性.方便进行关系型优化.
  + 引入了设计精妙的优化器:`Catalyst`.

3. DataFrame提供丰富的操作,同时与Spark RDD兼容,既可以转换成RDD,也可以被当作RDD输出,并且比RDD的API操作更简单,性能更高.比如DataFrame可以用一个SQL完成多个聚合操作(multiple aggregates),而RDD-API实现起来就很复杂.并且DataFrame存储时还有优化,使用列式存储(Columnar format),占用空间更小,DataFrame还会使用Catalyst来实现对后端的优化.

4. Catalyst: 为了支持多种数据源,分析工作负荷(analytics workloads),用scala撰写了`Catalyst`,充分利用了已有规则,比如使用模式识别来实现规则的组合.还提供了一个框架,可以实现  
 + 添加新数据源,比如json和smart数据存储(HBASE).
 + 用户自定义函数(UDF,下略).
 + 以及用户自定义的域类型.

5. 最后是效果总结,SparkSQL被应用的非常广泛,2014年-2015年就是Spark最活跃的模块之一了.并且效率相比于传统的RDD提高了10倍以上.

6. 然后,我们将在Part_II介绍Spark背景与SparkSQL的目标,Part_III介绍DataframeAPI,Part_IV:Catalyst优化器,Part_V:Catalyst上的高级特性,Part_VI:使用评估SparkSql,Part_VII:catalyst的外部研究,最后,Part_VIII,cover相关工作.

### Part_II: Background and goals.

#### what is Spark?
一个通用的集群计算引擎,API横跨多个语言,提供函数式API,主要操纵RDD:分布式的object集合,RDD具有容错性,并且是惰性求值,虽然会利用惰性求值进行一些优化,但是优化有限,因为不理解RDD中的数据结构,也不会将UDF纳入优化范围.

#### Spark已经有的关系型系统
最初的尝试是Shark,使得Spark兼容了Apache Hive,同时实现了一些传统优化.
但是Shark有三个问题:
  + Shark对Spark中的数据没有帮助,比如在RDD上进行查询.
  + Shark只能通过SQL的方式被调用.
  + Hive的优化是针对MapReduce的,对Spark来说没有作用.

#### 所以想要完成的目表
+ API更简单,同时支持RDD,关系处理,外部的数据.
+ 利用现有的DBMS技术,提高效率.
+ 对新的数据源支持更好.
+ 对高级分析的算法更好.

### Part_III: Programming Interface
SparkSQL基于spark,是Spark的模块,同时支持JDBC/ODBC,命令行,
#### DataFrame API
DataFrame,等价于传统关系型数据库中的表,--每一列都有名称和类型,而不像RDD一样对内部的元素一无所知,

![picture_01,come from https://www.zhihu.com/question/48684460](./paper_01.png)

也因此可以执行各种关系型操作,并且可以进行抽象语法树的简化与惰性求值.并且可以大大有选择地读取,速度提升了.

#### Data Model.
而DataFrame的每一列支持的元素式样很多,int,boolean,double,decimal,string,data,timestamp还有各种各样的复杂组合,对DataFrame对各个语言的支持都很精确.
``` python
users.where(users("age") < 21)
       .registerTempTable("young")
ctx.sql("select count(*) , avg("age") from young")
```
#### DataFrame 的操作.
DataFrame支持常见的关系型操作,查询,过滤,join以及聚合,最终形成一个巨大的抽象语法树,交给`Catalyst`优化,并且还支持临时表(这种形式也支持惰性求值.).

#### DataFrame的独到之处
DartFrame因为内部存有元数据,所以可以被编译器实时检测到,并且给出提醒,节省了大量的空间(但是并不会像C++那样有编译期求值,还是运算期求值.).

#### DataFrame与数据集的交互
DataFrame既然能够无缝替换RDD,可以直接从RDD中提取元信息(Java使用反射,Python使用动态获取),并且还可以将RDD的数据和外部的数据进行合并.

#### Caching and UDF.
SparkSQL提供了缓存机制,可以直接调用cache()进行缓存.
用户定义的函数最终会转化成底层的SparkAPI,可以被优化,但是最上层的是被Python等编写的,也可以被调试.

### Part_IV: Catalyst
Catalyst是立足于取代Hive的优化而产生,基于抽象语法树的优化器,完全基于Scala.

#### Tree.
这里介绍一下语法结构,比如表达式x+(1+2)就可以表达为
``` scala
Add(Attribute(x), Add(Literal(1)), Literal(2))
```

#### Rules,转化规则
既然要进行优化,那自然要针对抽象语法树做简化,而在Catalyst中,简化的方式遵循的就是Rule,规则,使用模式匹配的方式来简化
``` scala
tree.transform{
    case Add(Literal(c1), Literal(c2) => Literal(c1+c2))
}
```
case之间都是独立的,所以增加或者修改一些功能也比较方便.
不断地递归使用规则,最终简化到无法再简化.

#### Spark中的Catalyst
1. 通过分析逻辑计划来替换引用；
2. 优化逻辑计划；
3. 物理计划；
4. 代码生成（编译部分查询代码到 Java 二进制）；

![picture_02](./paper_02.png)

阶段一会从代码中抽取出抽象语法树,阶段二应用各种规则进行简化,阶段三在多个方案中进行评估,最后一步生成最终的代码

#### 优秀之处
1. 规则方便拓展
2. 对数据源提供多种优化等级.
3. 对用户自定义的类型提供良好的支持.

### Part_V: Advanced Analytics Features 
In this section, i describe three features we added to Spark SQL
specifically to handle challenges in “big data" environments. 

#### the first Advanced Analytics Features is Schema Inference for Semistructured Data

```json
{
"text": "This is a tweet about #Spark",
"tags": ["#Spark"],
"loc": {"lat": 45.1, "long": 90}
} 
{
"text": "This is another tweet",
"tags": [],
"loc": {"lat": 39, "long": 88.5}
} 
{
"text": "A #tweet without #location",
"tags": ["#tweet", "#location"]
}
```



1.  In  big data are, semistructured data is common in large-scale environments because it is easy to produce and to add fields to over time.

2. JSON is cumbersome to work with in a procedural environment like Spark or MapReduce: 

   most users resorted to ORM-like libraries framework(e.g.,mybatis,Jackson ) to map JSON structures to
   Java objects, or some tried parsing each input record directly with
   lower-level libraries.

3. **In Spark SQL, we added a data source and  automatically
   infers a schema from a set of records just like a table**, 

   so we can query it with
   syntax that accesses fields by their path, such as:

   ```mysql
   SELECT loc.lat, loc.long FROM tweets
   WHERE text LIKE ’%Spark%’ AND tags IS NOT NULL
   ```

   

4. another feature is **Auto types detection**: the algorithm finds the most specific
   Spark SQL data type that matches observed instances of the field.

5. We implement this algorithm using a single
   reduce operation over the data, which starts with schemata  from each individual record and merges them using an
   associative “most specific supertype" function that generalizes the
   types of each field. 

6. For example,note how in Figures 5 and 6, the algorithm
   generalized the types of loc.lat and loc.long. Each field appears
   as an integer in one record and a floating-point number in another,
   so the algorithm returns FLOAT. 

####  the second Advanced Analytics Features is Integration with Spark’s Machine Learning Library

1. firt we need to talk what is pipelines: a graph
   of transformations on data, such as feature extraction, normalization, dimensionality reduction, and model training, each of which
   will exchange datasets. 
2. In pipeline process we**need a format that was compact (as datasets can be large) yet
   flexible, allowing multiple types of fields to be stored for each
   record --> dataset --> DataFrame**.
3. New API use DataFrames 
   where each column represents a feature of the data. All algorithms
   that can be called in pipelines take a name for the input column(s)
   and output column(s), and can thus be called on any subset of the
   fields and produce new ones. This makes it easy for developers to
   build complex pipelines while retaining the original data for each
   record.
4. For example, In TF-IDF, here ...
5. TF-IDF: short for term frequency–inverse document frequency, is a numerical statistic that is intended to reflect how important a word is to a [document](https://en.wikipedia.org/wiki/Document) in a collection ; it increases [proportionally](https://en.wikipedia.org/wiki/Proportionality_(mathematics)) to the number of times a word appears in the document and is decrease by the number of documents  that contain the word, which helps to adjust for the fact that some words appear more frequently in general.
6. * **dataframe made it much easier to expose MLlib’s new
     API in all of Spark’s supported programming languages.**
   * Previously,
     each algorithm in MLlib took objects for domain-specific concepts and each of these classes had to be implemented
     in the various languages (e.g., copied from Scala to Python). Using
     DataFrames made it much simpler to expose all algorithms in all languages, as we only need data conversions in Spark
     SQL, where they already exist. This is especially important as Spark
     adds bindings for new programming languages.

7. add new row and didnt keep the previous rows

![image-20200526150335741](paper.assets/image-20200526150335741.png)



```python
data = <DataFrame of (text , label) records >
tokenizer = Tokenizer()
.setInputCol("text").setOutputCol("words")
tf = HashingTF()
.setInputCol("words").setOutputCol("features")
lr = LogisticRegression()
.setInputCol("features")
pipeline = Pipeline().setStages([tokenizer , tf, lr])
model = pipeline.fit(data)
```



#### 外部数据库的优化 Query Federation to External Databases
Data pipelines often combine data from heterogeneous sources. such as different mechanes, databases

sparksql can do it effientily

``` SQL
CREATE TEMPORARY TABLE users USING jdbc
OPTIONS(driver "mysql" url "jdbc:mysql://userDB/users")

CREATE TEMPORARY TABLE logs
USING json OPTIONS (path "logs.json")

SELECT users.id, users.name, logs.message
FROM users JOIN logs WHERE users.id = logs.userId AND users.registrationDate > "2015-01-01"
```
* for example, the following uses the JDBC data source and the
  JSON data source . with sparksql, both data sources
  can automatically infer the schema without users having to define it.

* The JDBC data source will also push the filter predicate down into
  MySQL to reduce the amount of data transferred.

  Under the hood, the JDBC data source uses the Filterer interface , which gives it both the names of the
  columns requested and simple predicates on these columns. In this case, the JDBC data source
  will run the following query on MySQL:

``` SQL
SELECT users.id, users.name FROM users WHERE users.registrationDate > "2015-01-01"
```

### Part_VI Evaluation 

#### SQL Performance

* We see that in all queries, Spark SQL is substantially faster than
  Shark and generally competitive with Impala. The main reason
  for the difference with Shark is code generation in Catalyst which reduces CPU overhead. This feature also makes Spark
  SQL competitive with the C++ and LLVM based Impala engine in
  many of these queries. 

![image-20200526153434757](paper.assets/image-20200526153434757.png)

#### DataFrames vs. Native Spark Code

* Catalyst can perform optimizations on 

  DataFrame operations that are hard to do with hand written code,
  such as predicate pushdown.whats more, the DataFrame API can
  result in more efficient execution use code generation. 

* Figure9, shows that the DataFrame version of the code outperforms the hand written Python version by 12×, 

  This is because in the DataFrame API, only the
  logical plan is constructed in Python, and all physical execution is
  compiled down into native Spark code as JVM bytecode, resulting
  in more efficient execution. In fact, the DataFrame version also
  outperforms a Scala version of the Spark code above by 2×. This
  is mainly due to code generation: the code in the DataFrame version avoids expensive allocation of key-value pairs that occurs in
  hand-written Scala code.

![image-20200526153614766](paper.assets/image-20200526153614766.png)

#### Pipeline Performance

* first method: using a separate SQL queryfollowed by a Scala-based Spark job
* second method: using theDataFrame API,using DataFrame’s relational operators to per-form the filter, and using the RDD API to perform a word counton the result. 
* the second pipelineavoids the cost of saving the whole result of the SQL query to anHDFS file as an intermediate dataset before passing it into the Sparkjob, 

* Figure 10 compares the
  runtime performance of the two approaches. the DataFrame-based pipeline 
  improves performance by 2×.

![image-20200526154448198](paper.assets/image-20200526154448198.png)

### Part_VII Research Applications and Related Work

#### 1 Generalized Online Aggregation

divided into serval batch, allows users to view the progress of executing queries by seeing results computed over a fraction of the total data.

#### 2 Computational Genomics

```mysql
SELECT * FROM a JOIN b 
WHERE a.start < a.end 
    AND b.start < b.end
    AND a.start < b.start 
    AND b.start < a.end
```





源自

> https://blog.csdn.net/fansy1990/article/details/89065547