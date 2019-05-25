---
layout: post
title: "MongoDB Introduction"
author: "Zreal"
catalog: true
header-img: "img/roman-post.jpg"
tags:
  - Data Base
---



# MongoDB Introduction

>Author: Zreal 	曾令泽
>
>Student_ID: 1120162062
>
>Platform: Mac OS



## Install MongoDB

可参考[官方文档](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-os-x/)

但是在参考官方文档的过程中出现了问题，在键入

```shell
$ mongod --config /usr/local/etc/mongod.conf
```

长时间没有反应，最后不得已参考网上的博客操作成功



**Step one**

用homebrew安装

```shell
$ brew tap mongodb/brew
$ brew install mongodb
```

此时会有提示：配置文件，日志文件，数据文件的默认位置

- the [configuration file](https://docs.mongodb.com/manual/reference/configuration-options/) (`/usr/local/etc/mongod.conf`)
- the [`log directory path`](https://docs.mongodb.com/manual/reference/configuration-options/#systemLog.path) (`/usr/local/var/log/mongodb`)
- the [`data directory path`](https://docs.mongodb.com/manual/reference/configuration-options/#storage.dbPath) (`/usr/local/var/mongodb`)



>按照官方文档应该运行mongodb
>
>```shell
>$ mongod --config /usr/local/etc/mongod.conf
>```
>
>但是进行输入后电脑一只无反应



**Step two** 

重新配置日志文件及数据文件的路径

配置`/usr/local/etc/mongod.conf`文件如下

```js
systemLog:
  destination: file
  path: /data/db/mongod.log
  logAppend: true
storage:
  dbPath: /data/db
net:
  bindIp: 127.0.0.1
```

创建 `/data/db` 文件夹及`/data/db/mongod.log`文件

给/data/db 文件夹赋予权限

```shell
$ sudo chown id -u /data/db
```

启动mongod 服务器,因为自定义了储存路径 要使用 —dbpath

```shell
$ mongod --dbpath /data/db
```



**Step three**

开启mongo shell(加入环境变量后)

```shell
$ mongo
```



## Introduction to Mongo shell command

 **创建数据库**

``` powershell
> use newDB
switched to db newDB
> db
newDB
```

**查看当前数据库**

```powershell
> db.stats()
# 查看当前数据库状态
> db.getName()
# 查看当前数据库名字
> db.version
# 查看mongodb 版本
```

![img](/img/assets/checktheDatabase.png)

**查看所有数据库**

```powershell
> show dbs
```

> 刚刚创建的数据库，如果该数据库里面没有内容，show dbs 不会显示该数据库

![img](/img/assets/CreateDataBase.png)



**创建集合**

```powershell
> db.createColletion(name,options)
```

option 又如下参数

| 字段        | 类型 | 描述                                                         |
| :---------- | :--- | :----------------------------------------------------------- |
| capped      | 布尔 | （可选）如果为 true，则创建固定集合。固定集合是指有着固定大小的集合，当达到最大值时，它会自动覆盖最早的文档。 **当该值为 true 时，必须指定 size 参数。** |
| autoIndexId | 布尔 | （可选）如为 true，自动在 _id 字段创建索引。默认为 false。   |
| size        | 数值 | （可选）为固定集合指定一个最大值（以字节计）。 **如果 capped 为 true，也需要指定该字段。** |
| max         | 数值 | （可选）指定固定集合中包含文档的最大数量。                   |

或者直接向表中插入数据，数据库会自动呢新建一个Collection

```powershell
> db.newCollection.insertOne(document)
```

![img](/img/assets/createCollection.png)



**插入文档**

```powershell
> db.collection.insertOne(document)
#插入一条document
> db.collection.insertMany([document1,document2,document3])
#插入多条document
```

 

**查询文档**

```powershell
> db.collection.find(<query>)
#<query>查询语句
> db.collection.find()
#全表查询
```

![img](/img/assets/checkthecollection.png)



**修改更新文档**

```powershell
> db.collection.update( <query> , <update>, {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>
   })
```

- query : update的查询条件，类似sql update查询内where后面的。
- update : update的对象和一些更新的操作符（如$,$inc...）等，也可以理解为sql update查询内set后面的
- upsert : 可选，这个参数的意思是，如果不存在update的记录，是否插入objNew,true为插入，默认是false，不插入。
- multi : 可选，mongodb 默认是false,只更新找到的第一条记录，如果这个参数为true,就把按条件查出来多条记录全部更新。
- writeConcern :可选，抛出异常的级别。

```powershell
> db.collection.update({item:"ame"},{$set:{name:zlz}})
```

![img](/img/assets/update.png)



**删除数据库**

```powershell
> db.dropDatabase()
# 删除当前数据库
```

![img](/img/assets/DropDataBase.png)



**删除集合**

```powershell
> db.Onecollection.drop
#删除Onecollection 集合
```

![img](/img/assets/dropCollection.png)



**删除文档**

```powershell
> db.collection.deleteOne(<query>)
#删除一条文档
> db.collection.deleteMany(<query>)
#删除所有符合条件的文档
```

![img](/img/assets/DeleteCollection.png)



## Mongon Programming in Node.js

```js
MongoClient.connect(url, { useNewUrlParser: true }, function(err, db){
	var names=[
	'Yolanda',
		'Iska',
		'Malone',
		'Frank',
		'Foxton',
		'Pirate',
		'Poppelhoffen',
		'Elbow',
		'Fluffy',
		'Paphat'
	]
	var randName=function(){
	var n=names.length;
		return [
		names[Math.floor(Math.random()*n)],
		names[Math.floor(Math.random()*n)]
		].join(' ');
	}
	var randAge=function(n){
		return Math.floor(Math.random()*n);
	}
	var dbase=db.db("test");
	for(var i=0;i<100;i++){
		var person={
			name:randName(),
			age:randAge(100)
		}
		if(Math.random()>0.8){
		person.cat={
			name:randName(),
			age:randAge(18)
		}}
		dbase.collection("people").insertOne(person);
	};
	db.close();
});
```



**Requirement**

1) 简要描述以上代码功能
2) 只查看 1 条文档
3) 只查看 1 条文档，且在显示的结果中隐去_id 字段
4) 查看上述代码创建的文档数
5) 查看所有年龄大于 90 的人，并按年龄倒序输出
6) 查看所有年龄大于 12 小于 18 的人
7) 查看所有养猫且猫的年龄大于 10 岁的人
8) 统计所有养描的人的个数



1)

randName 随机函数随机得到 一个有 姓和名的 名字

randAge 随机函数随机得到1-n的一个年龄

然后插入100条数据条数据



2)

```powershell
> db.people.findOne()
```

![img](/img/assets/image-20190524210602907.png)



3)

```powershell
> db.people.findOne({},{_id:0})
```

![img](/img/assets/image-20190524210651305.png)



4)

```powershell
> db.people.count()
```

 

5)

```powershell
> db.people.find({age:{$gr:90}}).sort(age:-1)
```

![img](/img/assets/image-20190524211122260.png)



6)

```powershell
> db.people.find({age:{$gt:12,$gt:18}})
```

![img](/img/assets/image-20190524211342827.png)



7)

```powershell
> db.people.find({"cat.age":{$gt:10}})
```

![img](/img/assets/image-20190524213702182.png)



8)

```powershell
> db.people.find({"cat":{$exists:true}}).count()
```

![img](/img/assets/image-20190524213831930.png)

