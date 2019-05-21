---
layout: post
title: "HBase shell操作及JAVA编程"
author: "Zreal"
catalog: true
header-img: "img/roman-post.jpg"
tags:
  - Hadoop
  - JAVA
---



# HBase shell命令行操作及JAVA API编程

>Author: Zreal 曾令泽
>
>Student_ID:1120162062
>
>Platform: Mac OS



## Install HBase on Mac OS

>参考：
>
>https://www.jianshu.com/p/510e1d599123
>
>https://hbase.apache.org/1.2/book.html

**Homebrew 安装HBase**

```shell
$ brew install hbase
# 安装在/usr/local/Cellar/hbase/1.2.9
```

**配置JAVA路径**

在`conf/hbase-env.sh`下配置JAVA_HOME

```shell
$ cd /usr/local/Cellar/hbase/1.2.9/conf
$ vim hbase-env.sh
```

注：JAVA的路径可以在命令行中入`/usr/libexec/java_home`可以得到。

将JAVA路径export入hbase-env.sh

```shell
export JAVA_HOME={LOCAL_JAVA_HOME}
```



**单机版Hbase核心配置**

在`conf/hbase-site.sh`下设置核心配置

```sh
<configuration>
  <property>
    <name>hbase.rootdir</name>
    //这里设置让HBase存储文件的地方
    <value>file:///Users/andrew_liu/Downloads/hbase</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    //这里设置让HBase存储内建zookeeper文件的地方
    <value>/Users/andrew_liu/Downloads/zookeeper</value>
  </property>
</configuration>
```

>注：这里也可以不设置，这两个property
>
>如果不设置，Hbase会默认在创建一个/tmp的文件夹来存储这些文件，但是一旦重启reboot，这些文件会被服务器删除掉



**伪分布式版本配置**

>修改配置文件时应先将hbase关闭
>
>### 键入stop-abase.sh

在`conf/hbase-site.sh`下设置核心配置

```sh
<configuration>
 <property>
  <name>hbase.cluster.distributed</name>
  <value>true</value>
</property>
<property>
  <name>hbase.rootdir</name>
  <value>hdfs://localhost:8020/hbase</value>
</property>
</configuration>
```

>第一个property 打开hbase的分布式集群
>
>第二个property 
>
>！！！`hbase.rootdir`路径一定要跟hadoop中`core-site.xml`中fs.default.name相同

>如果两处不同会报错：
>
>ERROR: Can't get master address from ZooKeeper; znode data == null

**启动hbase shell**

在`bin/`下可以启动和停止hbase，键入

```shell
$ start-hbase.sh
$ hbase shell
```

进入hbase shell命令行

![img](/img/assets/image-20190517001100149.png)

![img](/img/assets/image-20190517000545672.png)



## HBase shell 简单命令

<u>帮助</u>

```shell
> help
# 查看文档
```

![img](/img/assets/image-20190517000839918.png)

<u>创建表</u>

```shell
> create "Teacher","Teacher_name","Teacher_ID","Teacher_major","Teacher_info"
#create tablename,colFamily1,colFamily2,colFamily3
```

<u>获得表的描述</u>

```shell
> describe "Teacher"
# describe "TableName"
```

![img](/img/assets/image-20190517091854155.png)

<u>查看所有表</u>

```shell
> list
```

![img](/img/assets/image-20190517092104663.png)

<u>增加记录</u>

```shell
> put "Teacher","001","Teacher_name:FirstName","zenglingze"
# put TableName, Keyrow, colFamily:Qualifier, value
```

![img](/img/assets/image-20190517092654742.png)

<u>增加列族</u>

```shell
> alter "Teacher","Teacher_info"
# alter TableName, add_colFamily
```

![img](/img/assets/image-20190517092915756.png)

<u>删除列族</u>

```shell
> alter "Teacher" ,{NAME=> "Teacher_info",METHOD=>"delete"}
# alter tablename ,{NAME=> colFamily, METHOD=>"delete"}
```

![img](/img/assets/image-20190517093308497.png)



<u>清空表</u>

```shell
> truncate "Teacher"
# truncate tabelname
```

![img](/img/assets/image-20190517093903949.png)



<u>删除表</u>

```shell
> disable "Teacher"
> drop "Teacher"
# 现将表禁用，再删除
# disable tablename
# drop tablename
```

![img](/img/assets/image-20190517094112879.png)



<u>统计表行数</u>

```shell
> count "student"
# count tablename
```

![img](/img/assets/image-20190517094219927.png)



<u>浏览表</u>

```shell
> scan "student"
#扫描整个表
> scan "student",{COLUMN=>"Student_id"}
#扫描整个列族
> scan "student",{COLUMN=>"Student_info:sex"}
#扫描列
```

![img](/img/assets/scan.png)

<u>获取数据</u>

```shell
> get "student","Ame"
# get tablename, rowkey
> get "student","Ame","Student_id"
# get tablename, rowkey, colFamily
> get "student", "Ame", "Student_info:sex"
# get tablename, rowkey, colFamily:qualifier
```



## HBase JAVA API

>源码地址：https://github.com/Zrealshadow/hadoop_practice/tree/master/LAB2_HbaseExample
>
>源码中实现了除下列列出的其他功能



IDEA 里项目构建里导入jar包 `usr/local/Cellar/hbase/1.2.9/lib`



<u>建立连接/关闭连接</u>

```java
   /**
     * @Description: connect 
     * @Param: []
     * @return void
     * @Author: Zreal
     * @Date:2019/5/14
     * @Time:4:48 PM
    */
    public static void init(){
        BasicConfigurator.configure();
        configuration=HBaseConfiguration.create();
        configuration.set("hbase.rootdir","hdfs://localhost:9000/hbase");
        try{
            connection=ConnectionFactory.createConnection(configuration);
            admin=connection.getAdmin();
        }catch (Exception e){
            e.printStackTrace();
        }
    }

    /**
     * @Description: close
     * @Param: []
     * @return void
     * @Author: Zreal
     * @Date:2019/5/14
     * @Time:4:48 PM
    */
    public static void close(){
        try{
            if(admin!=null){
                admin.close();
            }if(connection!=null){
                connection.close();
            }
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```



<u>建表</u>

```java
    /**
     * @Description: createTable
     * @Param: [myTableName, colFamily]
     * @return void
     * @Author: Zreal
     * @Date:2019/5/14
     * @Time:4:48 PM
    */
    public static void createTable(String myTableName, String[] colFamily)throws Exception{
        init();
        TableName tableName=TableName.valueOf(myTableName);
        if(admin.tableExists(tableName)){
            System.out.println("this table is exist");
        }else{
            HTableDescriptor hTableDescriptor=new HTableDescriptor(tableName);
            for(String str:colFamily){
                HColumnDescriptor hColumnDescriptor=new HColumnDescriptor(str);
                hTableDescriptor.addFamily(hColumnDescriptor);
            }
            admin.createTable(hTableDescriptor);
            System.out.println("create table success!");
        }
        close();
    }

    public void Test_CreateTable(){
        String tablename="student";
        String[] columns=new String[]{"Student_ID","Student_Major","Student_Name","Student_Info"};
//        String tablename="members";
//        String[]columns=new String[]{"Member_ID","Member_Major","Member_Name","Member_Info"};
        try{
            HbaseTheExample.createTable(tablename,columns);
            HbaseTheExample a=new HbaseTheExample();
            a.init();
            boolean ans=a.admin.tableExists(TableName.valueOf(tablename));
            System.out.print(ans);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```



<u>展示所有表</u>

```java
    /**
     * @Description: listAllTable
     * @Param: []
     * @return void
     * @Author: Zreal
     * @Date:2019/5/14
     * @Time:8:35 PM
    */
    public static void listAllTable()throws Exception{
        init();
        HTableDescriptor[] hTableDescriptors=admin.listTables();
        // iterator 列表
        for(HTableDescriptor hTableDescriptor:hTableDescriptors) {
            TableName tn = hTableDescriptor.getTableName();
            System.out.println("Table Name:" + tn.toString());
            HColumnDescriptor[] hColumnDescriptors = hTableDescriptor.getColumnFamilies();
            System.out.println("the Column:\n{");
            //iterator 列族
            for (HColumnDescriptor hColumnDescriptor : hColumnDescriptors) {
                String column = hColumnDescriptor.getNameAsString();
                System.out.println(column);
            }
            System.out.println("}");
        }
        close();
    }

    public void Test_listAllTables() throws Exception{
        try{
            HbaseTheExample.listAllTable();
        }
        catch (Exception e){
            e.printStackTrace();
        }
    }
```

Test_result

![img](/img/assets/list_JAVA.png)



<u>打印出表的所有记录</u>

```java
  /**
     * @Description: showDetailsOfTable
     * @Param: [tableName]
     * @return void
     * @Author: Zreal
     * @Date:2019/5/14
     * @Time:7:40 PM
    */
    public static void showDetailsOfTable(String tableName)throws Exception{
        init();
        TableName tablename=TableName.valueOf(tableName);
        Table table=connection.getTable(tablename);
        Scan scan=new Scan();
        ResultScanner scanner=table.getScanner(scan);
        System.out.println("**************Table:"+tableName+"**************");
        for(Result result=scanner.next();result!=null;result=scanner.next())
            displayCell(result);
        System.out.println("***********************************************");
        scanner.close();
        table.close();
        close();
    }
   
		private static void displayCell(Result result){
        Cell[] cells = result.rawCells();
        for(Cell cell:cells){
            System.out.print("RowName:"+new String(CellUtil.cloneRow(cell))+"\t");
            System.out.print("Timetamp:"+cell.getTimestamp()+" ");
            System.out.print("column Family:"+new String(CellUtil.cloneFamily(cell))+" ");
            System.out.print("column:"+new String(CellUtil.cloneQualifier(cell))+" ");
            System.out.println("value:"+new String(CellUtil.cloneValue(cell))+" ");
        }
    }

   public void Test_printDetailsOfTable()throws Exception{
        String tablename="student";
        try{
            HbaseTheExample.showDetailsOfTable(tablename);
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

Test_result

![img](/img/assets/Details_JAVA.png)



<u>向指定表中增加列族和列，删除列族或列</u>

```java
    public static void alterColumn(String tablename,String colFamily)throws Exception{
        init();
        TableName tn=TableName.valueOf(tablename);
        if(admin.tableExists(tn)){
            try{
                admin.disableTable(tn);
                HTableDescriptor hTableDescriptor=admin.getTableDescriptor(tn);
                HColumnDescriptor hColumnDescriptor=new HColumnDescriptor(colFamily);
                hTableDescriptor.addFamily(hColumnDescriptor);
                admin.addColumn(tn,hColumnDescriptor);
//        admin.modifyTable(tn,hTableDescriptor);
                admin.enableTable(tn);
                System.out.println("the Alter is finished!");
            }catch (Exception e){
                e.printStackTrace();
            }
        }
        close();
    }

    /**
     * @Description: deleteColumn
     * @Param: [tablename, colFamily]
     * @return void
     * @Author: Zreal
     * @Date:2019/5/16
     * @Time:10:37 AM
    */

    public static void deleteColumn(String tablename,String colFamily)throws Exception{
        init();
        TableName tn=TableName.valueOf(tablename);
        if(admin.tableExists(tn)){
            try{
                admin.disableTable(tn);
                admin.deleteColumn(tn,colFamily.getBytes());
                admin.enableTable(tn);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
        close();
    }

/*
删除列稍稍麻烦
首先根据colFamily，qualifier 筛选得到 ResultScanner
然后便利 ResultScanner 每个 Result里面 的到rowkey
按行为单位，addColumn加入colFamily和qualifier，
最后table.delete一起删除
*/
    public static void deleteQualifier(String tablename, String colFamily,String qualifier)throws  Exception{
        init();
        TableName tn=TableName.valueOf(tablename);
        Table table=connection.getTable(tn);
        Scan scan=new Scan();
        scan.addColumn(colFamily.getBytes(),qualifier.getBytes());
        ResultScanner results=table.getScanner(scan);
        List<Delete> Deletes=new ArrayList<>();
        for(Result result:results){
            Delete delete=new Delete(result.getRow());
            delete.addColumn(colFamily.getBytes(),qualifier.getBytes());
            Deletes.add(delete);
        }
        table.delete(Deletes);
        table.close();
        System.out.println("delete is finished");
        close();
    }

```

Test_result:

Alter_colfamily

![img](/img/assets/Alter_col_JAVA-8063011.png)



Delete_qualifier

![img](/img/assets/delete_qualifier-8063032.png)



<u>清空表的所有数据</u>

```java
    /**
     * @Description: clearTable
     * @Param: [tableName]
     * @return void
     * @Author: Zreal
     * @Date:2019/5/14
     * @Time:7:20 PM
    */
    public static void clearTable(String tableName)throws Exception{
        init();
        TableName tn=TableName.valueOf(tableName);
        if(!admin.tableExists(tn)){
            System.out.println("the table is not exist");
        }
        else{
            HTableDescriptor it=admin.getTableDescriptor(tn);
            admin.disableTable(tn);;
            admin.deleteTable(tn);
            admin.createTable(it);
            System.out.println(tableName+" is being format");
        }
        close();
    }
```



<u>统计表的行数</u>

```java
/**
 * @Description: countRowOfTable
 * @Param: [tableName]
 * @return void
 * @Author: Zreal
 * @Date:2019/5/14
 * @Time:8:35 PM
*/

public static void countRowOfTable(String tableName)throws Exception{
    init();
    TableName tablename=TableName.valueOf(tableName);
    Table table=connection.getTable(tablename);
    Scan scan=new Scan();
    ResultScanner scanner=table.getScanner(scan);
    int sum=0;
    for(Result result=scanner.next();result!=null;result=scanner.next())
        sum+=1;
    System.out.println("the number of rows in table "+tableName+":"+sum);
    table.close();
    close();
}
```

![img](/img/assets/counting_row_JAVA.png)



## Problems

>ERROR: Can't get master address from ZooKeeper; znode data == null

Solution:

参考：

https://stackoverflow.com/questions/22663484/get-error-cant-get-master-address-from-zookeeper-znode-data-null-when-us