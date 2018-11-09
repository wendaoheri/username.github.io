---
layout: post
title:  "HBase数据模型"
categories: hbase
---
## 1. Hbase 模型概念
HBase数据模型自上而下由以下几个概念组成

***Namespace***

类似RDBMS系统中DB的概念，由一堆表组成，主要用于控制权限、资源配额以及绑定RegionServer保证粗粒度的隔离

***Table***

HBase的数据是存在Table中的，类似RDBMS系统中的Table的概念，也有行和列。但是在实际使用中，为了更好的理解将其想成一个多维Map更合适。

***Column Family***

逻辑上一个Column Family由一个或多个Column组成，物理上这些Column在文件系统中是存储在一起的，也有着相同的存储参数，如是否在内存中cache、数据如何压缩、rowkey的编码方法等。Column Family在schema定义的时候就需要确定，而Column则可以在任何时候添加。

***Column***

HBase中的Column更像mongodb或者es等非关系型数据库中的field，不需要预先定义，随时可以添加，数据中有了这个column就会自动添加上这个column。一般而言，column的名字以column family的名字加冒号作为前缀，形如"cf_name:col_name"。

***Row***

HBase中一行由一个RowKey和一个或多个Column组成。一张表中的每一行逻辑上会包含该张表的所有Column Family，但是对于没有任何Column的Column Family并不会存储，这就是Hbase的稀疏存储。

***Cell***

Cell是HBase的最小存储单位，连接了HBase中的行、列族、列、版本这几个概念，所以一个cell的坐标由这几个元素确定{row, column_family:column,version}

***Timestamp***

HBase使用时间戳来标记每一个值的版本，在插入或者修改数据的时候默认会使用RegionServer的时间来作为版本，但是也可以自行指定。可以通过Column Family参数上的"VERSION"和"MIN_VERSION"来控制保留的最大、最小版本数。



## 2. 概念视图

|Row Key	|Timestamp	|ColumnFamily contents|	ColumnFamily anchor|ColumnFamily people|
|-----------|:---------:|:-------------------:|:------------------:|:-----------------:|
|"com.cnn.www"|t9||anchor:cnnsi.com = "CNN"||
|"com.cnn.www"|t8||anchor:my.look.ca = "CNN.com"||
|"com.cnn.www"|t6|contents:html = "<html>…​"|||
|"com.cnn.www"|t5|contents:html = "<html>…​"|||
|"com.cnn.www"|t3|contents:html = "<html>…​"|||
|"com.example.www"|t5|contents:html = "<html>…​"||people:author = "John Doe"|

借用HBase官网上的例子对上述概念进行解释。

* 这张表由三个Column Family组成: `contents`、`anchor`、`people`。`contents`列族只有一列`contens:html`，等号后面的是具体的值，而`anchor`有两个列`anchor:cnnsi.com`和`anchor:my.look.ca`。

* 在表格中显式有很多行，但是实际上在HBase中只有两行，分别以`com.cnn.www`和`com.example.www`作为RowKey。一个张表中，一个唯一的RowKey代表了一行。`com.cnn.www`有三列`contens:html`、`anchor:cnnsi.com`和`anchor:my.look.ca`，而`com.example.www`有两列`contens:html`和`people:author`。

* 表中的`Timestamp`列代表每一个值的版本。`com.cnn.www`有五个版本而`com.example.www`只有一个。

为了更好地理解概念视图的数据结构，同样抄袭官网的说法，上面的表格使用如下的json也基本上表达了同样的数据结构，可能看起来更加会产生更少的误解
```json
{
  "com.cnn.www": {
    "contents": {
      "t6": "contents:html: <html>..."
      "t5": "contents:html: <html>..."
      "t3": "contents:html: <html>..."
    }
    "anchor": {
      "t9": "anchor:cnnsi.com = CNN"
      "t8": "anchor:my.look.ca = CNN.com"
    }
    "people": {}
  }
  "com.example.www": {
    "contents": {
      "t5": "contents:html: <html>..."
    }
    "anchor": {}
    "people": {
      "t5": "people:author: John Doe"
    }
  }
}
```

## 3. 物理视图

上面说过一张表的一个`Column Family`是物理上存在一起的，已`anchor`为例，看一下物理上是如何存储的。

|Row Key|Timestamp|Column Family anchor|
|-------|---------|--------------------|
|"com.cnn.www"|t9|anchor:cnnsi.com = "CNN"|
|"com.cnn.www"|t8|anchor:my.look.ca = "CNN.com"|

HBase只会存储非空的值，在概念视图中无论是行值为空还是版本值为空都不会存储，这就是所谓的稀疏存储。