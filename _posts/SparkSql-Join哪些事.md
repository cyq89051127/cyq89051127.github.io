---
layout: post
title:  "Spark 配置"
date:   2019-04-20 11:57:12 +0800
tags:
      - Spark
---

本文基于Spark 2.4版本，对其中的sql Join简要梳理

支持Join类型：

SparkSql在2.4版本中，使用anltr4完成sql语句的解析，其所有支持的Sql语法课参看SqlBase.g4文件。
查看该文件中Join类型支持语法，如下：
    
    joinType
    : INNER?        # 支持INNER JOIN，INNER关键字可省略
    | CROSS         # 支持CROSS JOIN, CROSS 关键字比填
    | LEFT OUTER?   # 支持 LEFT OUTER JOIN， OUTER关键字可省略
    | LEFT SEMI     # 支持LEFT SEMI JOIN，不能省略LEFT,SEMI关键字
    | RIGHT OUTER?  # 支持 RIGHT OUTER JOIN， OUTER关键字可省略
    | FULL OUTER?   # 支持 FULL OUTER JOIN， OUTER 关键字可省略
    | LEFT? ANTI    # 支持 LEFT ANTI JOIN，LEFT关键字可省略
    ;

下面就来实操观察各个JOIN效果：
    
  
 *使用的文件是spark的examplse中的测试数据employees.json和people.json（为方便查看join效果，在people.json中添加了name是lily 年龄是21的记录，代码清单参考附录即可）*
 
 **SELECT * FROM employees**
 
    +-------+------+
    |   name|salary|
    +-------+------+
    |Michael|  3000|
    |   Andy|  4500|
    | Justin|  3500|
    |  Berta|  4000|
    +-------+------+

**SELECT * FROM people**

    +----+-------+
    | age|   name|
    +----+-------+
    |null|Michael|
    |  30|   Andy|
    |  19| Justin|
    |  21|   Lily|
    +----+-------+

左连接：以左表为基准，再右表中查看符合条件的值，查找不到时，输出左表记录，右表字段设置为null

 **SELECT * FROM  employees left outer join people on people.name = employees.name**
    
    +-------+------+----+-------+
    |   name|salary| age|   name|
    +-------+------+----+-------+
    |Michael|  3000|null|Michael|
    |   Andy|  4500|  30|   Andy|
    | Justin|  3500|  19| Justin|
    |  Berta|  4000|null|   null|
    +-------+------+----+-------+

左连接：以右表为基准，再左表中查看符合条件的值，查找不到时，输出右表记录，左表字段设置为null

**SELECT * FROM employees right outer join people on people.name = employees.name**
    
    +-------+------+----+-------+
    |   name|salary| age|   name|
    +-------+------+----+-------+
    |Michael|  3000|null|Michael|
    |   Andy|  4500|  30|   Andy|
    | Justin|  3500|  19| Justin|
    |   null|  null|  21|   Lily|
    +-------+------+----+-------+

全连接 ： 左连接和右连接的并集

**SELECT * FROM employees full outer join people on people.name = employees.name**
    
    +-------+------+----+-------+
    |   name|salary| age|   name|
    +-------+------+----+-------+
    |Michael|  3000|null|Michael|
    |   null|  null|  21|   Lily|
    |   Andy|  4500|  30|   Andy|
    |  Berta|  4000|null|   null|
    | Justin|  3500|  19| Justin|
    +-------+------+----+-------+
 
 左连接和右连接的交集
    
**SELECT * FROM employees inner join people on people.name = employees.name**

    +-------+------+----+-------+
    |   name|salary| age|   name|
    +-------+------+----+-------+
    |Michael|  3000|null|Michael|
    |   Andy|  4500|  30|   Andy|
    | Justin|  3500|  19| Justin|
    +-------+------+----+-------+
    
左表右表进行笛卡尔集，全量数据输出，***无需等值的条件***，生产中数据量较大时性能问题突出

**SELECT * FROM employees cross join people**

    +-------+------+----+-------+
    |   name|salary| age|   name|
    +-------+------+----+-------+
    |Michael|  3000|null|Michael|
    |Michael|  3000|  30|   Andy|
    |Michael|  3000|  19| Justin|
    |Michael|  3000|  21|   Lily|
    |   Andy|  4500|null|Michael|
    |   Andy|  4500|  30|   Andy|
    |   Andy|  4500|  19| Justin|
    |   Andy|  4500|  21|   Lily|
    | Justin|  3500|null|Michael|
    | Justin|  3500|  30|   Andy|
    | Justin|  3500|  19| Justin|
    | Justin|  3500|  21|   Lily|
    |  Berta|  4000|null|Michael|
    |  Berta|  4000|  30|   Andy|
    |  Berta|  4000|  19| Justin|
    |  Berta|  4000|  21|   Lily|
    +-------+------+----+-------+
 
 Semi Join的结果集是left join时，以左表为基准，当右表中***存在***可以匹配关联条件的记录时，输出左表记录 
**SELECT * FROM employees left semi join people on people.name = employees.name**  

    +-------+------+
    |   name|salary|
    +-------+------+
    |Michael|  3000|
    |   Andy|  4500|
    | Justin|  3500|
    +-------+------+
  
 Anti Join的结果集是left join时，以左表为基准，当右表中***不存在***可以匹配关联条件的记录时，输出左表记录 
**SELECT * FROM employees left anti join people on people.name = employees.name**
 
    +-----+------+
    | name|salary|
    +-----+------+
    |Berta|  4000|
    +-----+------+

代码附录
    
    val spark =  SparkSession
      .builder()
      .config("spark.master","local[2]")
      .getOrCreate()
    import spark.implicits._
    val df = spark.read.json("/path/to/spark/examples/src/main/resources/people.json")
    val df2 = spark.read.json("/path/to/source/spark/examples/src/main/resources/employees.json")
    df.createOrReplaceTempView("people")
    df2.createOrReplaceTempView("employees")
    spark.sql("SELECT * FROM employees").show()
    spark.sql("SELECT * FROM people").show()
    spark.sql("SELECT * FROM  employees left outer join people on people.name = employees.name").show()
    spark.sql("SELECT * FROM employees right outer join people on people.name = employees.name").show()
    spark.sql("SELECT * FROM employees full outer join people on people.name = employees.name").show()
    spark.sql("SELECT * FROM employees inner join people on people.name = employees.name").show()
    spark.sql("SELECT * FROM employees left semi join people on people.name = employees.name").show()
    spark.sql("SELECT * FROM employees left anti join people on people.name = employees.name").show()
