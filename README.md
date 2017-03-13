# Mongo相关分享

@(mongodb扩展安装)[概念|使用方式|异常问题]


[TOC]

### MongoDB简介
> MongoDB 是由C++语言编写的，是一个基于分布式文件存储的开源数据库系统。

在高负载的情况下，添加更多的节点，可以保证服务器性能。
MongoDB 旨在为WEB应用提供可扩展的高性能数据存储解决方案。
MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。MongoDB 文档类似于 JSON 对象。字段值可以包含其他文档，数组及文档数组。

### MongoDB主要特点
- MongoDB的提供了一个面向文档存储，操作起来比较简单和容易。
- 你可以在MongoDB记录中设置任何属性的索引 (如：FirstName="Sameer",Address="8 Gandhi Road")来实现更快的排序。
- 你可以通过本地或者网络创建数据镜像，这使得MongoDB有更强的扩展性。
如果负载的增加（需要更多的存储空间和更强的处理能力） ，它可以分布在计算机网络中的其他节点上这就是所谓的分片。
- Mongo支持丰富的查询表达式。查询指令使用JSON形式的标记，可轻易查询文档中内嵌的对象及数组。
- MongoDB 使用update()命令可以实现替换完成的文档（数据）或者一些指定的数据字段 。
- MongoDB中的Map/reduce主要是用来对数据进行批量处理和聚合操作。
- Map和Reduce。Map函数调用emit(key,value)遍历集合中所有的记录，将key与value传给Reduce函数进行处理。
- Map函数和Reduce函数是使用Javascript编写的，并可以通过db.runCommand或mapreduce命令来执行MapReduce操作。
- GridFS是MongoDB中的一个内置功能，可以用于存放大量小文件。
- MongoDB允许在服务端执行脚本，可以用Javascript编写某个函数，直接在服务端执行，也可以把函数的定义存储在服务端，下次直接调用即可。
- MongoDB支持各种编程语言:RUBY，PYTHON，JAVA，C++，PHP，C#等多种语言。
- MongoDB安装简单。

### MongoDB 概念解析
>不管我们学习什么数据库都应该学习其中的基础概念，在mongodb中基本的概念是文档、集合、数据库，下面我们挨个介绍。

与MySQL对应的相关概念和术语：
| SQL术语/概念 |MongoDB术语/概念 |解释/说明  |
| :-------|  :-------|:-------|
| database | database  | 数据库   |
| table    | collection| 数据库表/集合  |
| row      | document  | 数据记录行/文档|
|column	|field	|数据字段/域|
|index	|index	|索引|
|table joins|	|表连接,MongoDB不支持
|primary key	|primary key	|主键,MongoDB自动将_id字段设置为主键|

### MongoDB 使用
MongoDB.php 中已经封装了MongoDB相关的用法
> $this->CollectHtmlBiz = new OneStore('mongodb');

- PHP7 连接 MongoDB 语法如下：
``` php
// 副本集： mongodb://rs1.example.com,rs2.example.com/?replicaSet=myReplicaSet
// 采用 副本集KEY：replSet  副本集值：application 正确 测试环境）：
$uri1 = "mongodb://ip:27017/?replSet=application";
//（正确 线上环境 applocation 幸亏看了看 运维在线上的副文本集名称写错  同时声明 replSet 而非 replicaSet）
$uri2 = "mongodb://ip:27017/?replSet=applocation";
$manager = new MongoDB\Driver\Manager($uri);
```
- 插入数据：
``` php
	/**
     * 存储抓取文件内容到MongoDb中
     * @param $date_time    年月
     * @param $rules_id     接口id
     * @param $clr_id       存入日志表中的主键id
     * @param $html         抓取的页面html内容
     */
    public function saveHtml($date_time, $rules_id, $clr_id, $html)
    {
        $collection = $this->collection_pre . $date_time;
        // 设置表
        $this->CollectHtmlBiz->table($collection);
        // 初始化数据
        $data = [
            'rules_id' => $rules_id,
            'clr_id'   => $clr_id,
            'html'     => $html
        ];
		// 插入数据操作
        $sava_data = $this->CollectHtmlBiz->data($data)->save();

        return $sava_data;
    }
```

- 查询数据：
>**注意：**查询条件字段的类型要与Collection中的该字段的类型相同 如果Collection中的类型是int类型，查询条件是string类型将无法查询到正确结果
``` php
	/**
     * 获取MongoDB中的数据 类型还要一致
     * @param $date_time
     * @param $clr_id
     * @return array
     */
    public function getHtml($date_time, $clr_id)
    {
        $collection = $this->collection_pre . $date_time;
        // 设置表
        $this->CollectHtmlBiz->table($collection);
        $filter = [
            'clr_id' => intval($clr_id)
        ];
        $data = $this->CollectHtmlBiz->where($filter)->find();

        return $data;
    }
```

- 添加索引
>**注意：**创建索引的方法是通过Command方法相关的操作创建的 找了比较多的资料如何用程序去创建， 答案在这里，目前已经封装好createIndex的通用方法https://github.com/mongodb/mongo-php-driver/issues/170
``` php
	/**
     * MongoDB 创建索引
     * @param $index_name 索引名字
     * @param array $index 索引内容["title":1,"description":-1] 复合索引 title 升序 description 降序
     * @return array
     */
    public function createIndex($index_name, array $index)
    {
        $command = new \MongoDB\Driver\Command([
            "createIndexes" => $this->getCollection(),
            "indexes"       => [
                [
                    "name" => $index_name,
                    "key"  => $index,
                    "ns"   => $this->_name . '.' . $this->getCollection(),
                ]
            ],
        ]);
        $result = $this->_manager->executeCommand($this->_name, $command);

        return $result->toArray();
    }

	/**
     * 创建MongoDB索引 ?集合
     * @param $date_time
     * @return array 下面是已创建索引放回的结果
     * ex: Array ( [0] => stdClass Object ( [createdCollectionAutomatically] => [numIndexesBefore] => 2 [numIndexesAfter] => 2 [note] => all indexes already exist [ok] => 1 ) )
     */
    public function actCreateIndex($date_time)
    {
        // clr_id建立索引 因为每月一表clr_id唯一 升序建立
        $index_name = 'clr_id_1';
        $index = ['clr_id' => 1];
        $collection = $this->collection_pre . $date_time;
        $this->CollectHtmlBiz->table($collection);
        $result = $this->CollectHtmlBiz->createIndex($index_name, $index);

        return $result;
    }
```

### MongoDB异常 均为Save时的报错
| Exception       |   Subscribe |  Time     | 是否影响使用|
| :----------- |  :------- |:-------|:-------|
| Mongodb waiting for replication timed out |等待复制超时  |   2017-02-23 14:57:35 ~ 2017-02-23 14:57:38   |否 primary插入 复制时出错|
| Failed to send "insert" command with database "db": socket error or timeout     | 对数据库”db”发送“插入”命令时失败：套接字错误和超时|   2017-02-23 14:53:30 ~ 2017-02-23 14:57:20  |是 数据丢失|
| No suitable servers found (`serverSelectionTryOnce` set): [Server closed connection. calling ismaster on 'ip:27017'] [connection closed calling ismaster on 'ip:27017'] | 没有发现合适的服务器（` serverselectiontryonce `集）：[Server closed connection. calling ismaster on 'ip:27017'] [关闭连接的ip:27017 ]|  2017-02-23 22:48:19 ~ 2017-02-23 22:54:52|是 数据丢失|

#### 出错时PHP报错信息
``` php
#0 MongoDB.php(127): MongoDB\Driver\Manager->executeBulkWrite('db.c...', Object(MongoDB\Driver\BulkWrite), Object(MongoDB\Driver\WriteConcern))
#1 OneStore.php(245): MongoDB->save(Array)
#2 CollectHtmlBiz.php(42): OneStore->save()
#3 SearchBiz.php(364): CollectHtmlBiz->saveHtml('201702', 207, 147219, '
```

### PHP7安装Mongodb扩展 
``` vim
cd /root/soft/
wget http://pecl.php.net/get/mongodb-1.2.5.tgz
tar zxf mongodb-1.2.5.tgz 
cd mongodb-1.2.5
/usr/local/php7/bin/phpize
./configure --with-php-config=/usr/local/php7/bin/php-config
make && make install


vim /usr/local/php7/etc/php.ini
(根据php.ini配置存放的地方也可能是 vim /usr/local/php7/lib/php.ini) 
加这行   
extension=mongodb.so
```

### 副本集

http://www.lanceyan.com/tech/mongodb/mongodb_repset1.html


