# PDO场景下的SQL注入探究

## **前言**

PHP 数据对象（PDO）扩展为 PHP 访问数据库定义了一个轻量级的一致接口。PDO 提供了一个数据访问抽象层，这意味着，不管使用哪种数据库，都可以使用相同的函数（方法）来查询和获取数据。PDO 随 PHP 5.1 发行，在 PHP 5.0 的 PECL 扩展中也可以使用，无法运行于之前的 PHP 版本。今天我们讨论 PDO 多语句执行（堆叠查询）和 PDO 预处理下的 SQL 注入问题导致 SQL 注入的问题。如有不足，不吝赐教。

## **PDO多语句执行**

PHP 连接 MySQL 数据库有三种方式（MySQL、Mysqli、PDO），同时官方对三者也做了列表性比较：

|   | Mysqli  | PDO  | MySQL  |
| ------------ | ------------ | ------------ | ------------ |
| 引入的PHP版本  | 5.0  | 5.0  | 3.0之前  |
| PHP5.x是否包含 | 是  | 是  | 是  |
| 服务端prepare语句的支持情况 | 是  | 是  | 否  |
| 客户端prepare语句的支持情况 | 否  | 是  | 否  |
| 存储过程支持情况 | 是  | 是  | 否  |
| 多语句执行支持情况 | 是  | 大多数  | 否  |

可以看到Mysqli和PDO是都是支持多语句执行的，我们对比一下看看两者的区别。
1. Mysqli通过multi_query()函数来进行多语句执行。

```php
<?php
$host='192.168.27.61';
$dbName='test';
$user='root';
$pass='root';
$mysqli = mysqli_connect($host,$user,$pass,$dbName);
if(mysqli_connect_errno())
{
   echo mysqli_connect_error();
}
$sql = "select * from user where id=1;";
$sql .= "create table test2 like user";
$mysqli->multi_query($sql);
$data = $mysqli->store_result();
print_r($data->fetch_row());
mysqli_close($mysqli);
```

请求脚本后发现数据库中成功创建了 test2 表，说明多语句成功执行。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081038-7a56388a-1f6c-1.png)

我们通过 wireshark 分析一下，首先登录请求 Multiple statements 字段未设置。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081215-b4abe048-1f6c-1.png)

通过 multi_query() 函数可以看到在执行 Query 前向 Mysql 服务器发送了一次 Set Option 请求将 multi statements 设置打开。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081242-c4a2f0d6-1f6c-1.png)

而使用普通的 mysqli_query() 函数，在执行 Query 前不会向 Mysql 服务器发送 set option 请求。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081251-c9c184c4-1f6c-1.png)

2.使用 PDO 中的 query() 函数同数据库交互。

```php
<?php
$dbms='mysql';
$host='192.168.27.61';
$dbName='test';
$user='root';
$pass='root';
$dsn="$dbms:host=$host;dbname=$dbName";
try {
     $pdo = new PDO($dsn, $user, $pass);
} catch (PDOException $e) {
     echo $e;
}
$sql = "select * from user where id=1;";
$sql .= "create table test2 like user";
$stmt = $pdo->query($sql);
while($row=$stmt->fetch(PDO::FETCH_ASSOC))
{
    var_dump($row);
    echo "<br>";
}
```

请求脚本后发现数据库中成功创建了 test2 表，说明多语句成功执行。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081916-af9f10e2-1f6d-1.png)

通过 wireshark 分析，通过 PDO 方式同数据库交互时，在登录时会设置 Multiple statements 字段，然后通过 Query 方式直接发送多语句到 Mysql 服务器。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081931-b81b89f8-1f6d-1.png)

PDO 默认支持多语句查询，如果 php 版本小于 5.5.21 或者创建 PDO 实例时未设置`PDO::MYSQL_ATTR_MULTI_STATEMENTS`为 false 时可能会造成堆叠注入。

```php
<?php
$dbms='mysql';
$host='192.168.27.61';
$dbName='test';
$user='root';
$pass='root';
$dsn="$dbms:host=$host;dbname=$dbName";
try {
     $pdo = new PDO($dsn, $user, $pass);
} catch (PDOException $e) {
     echo $e;
}
$id = $_GET['id'];
$sql = "SELECT * from user where id =".$id;
$stmt = $pdo->query($sql);
while($row=$stmt->fetch(PDO::FETCH_ASSOC))
{
    var_dump($row);
    echo "<br>";
}
```

$id 变量可控，构造链接访问，成功创建 aaa 数据表。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124082057-ebc1ab70-1f6d-1.png)

如果想禁止多语句执行，可在创建 PDO 实例时将`PDO::MYSQL_ATTR_MULTI_STATEMENTS`设置为 false。
```php
new PDO($dsn, $user, $pass, array( PDO::MYSQL_ATTR_MULTI_STATEMENTS => false))
```

## **MySQL预处理**

MySQL 数据库支持预处理，预处理或者说是可传参的语句用来高效的执行重复的语句。

MySQL 官方将 prepare、execute、deallocate 统称为 PREPARE STATEMENT。

预制语句的 SQL 语法基于三个 SQL 语句：

`prepare stmt_name from  preparable_stmt;
execute stmt_name [using @var_name [, @var_name] ...];
{deallocate | drop} prepare stmt_name;`

## **PDO预处理**

PDO分为模拟预处理和非模拟预处理。

模拟预处理是防止某些数据库不支持预处理而设置的，在初始化 PDO 驱动时，可以设置一项参数，`PDO::ATTR_EMULATE_PREPARES`，作用是打开模拟预处理(true)或者关闭(false),默认为 true。PDO 内部会模拟参数绑定的过程，SQL 语句是在最后 execute() 的时候才发送给数据库执行。

非模拟预处理则是通过数据库服务器来进行预处理动作，主要分为两步：第一步是 prepare 阶段，发送 SQL 语句模板到数据库服务器；第二步通过 execute() 函数发送占位符参数给数据库服务器进行执行。

首先我们通过wireshark抓包方式对比一下模拟预处理和非模拟预处理。
模拟预处理代码：

```php
<?php
$dbms='mysql';
$host='192.168.27.61';
$dbName='test';
$user='root';
$pass='root';
$dsn="$dbms:host=$host;dbname=$dbName";
try {
     $pdo = new PDO($dsn, $user, $pass, array( PDO::MYSQL_ATTR_MULTI_STATEMENTS => false));
} catch (PDOException $e) {
     echo $e;
}
$username = $_GET['username'];
$sql = "select * from user where username = ?";
$stmt = $pdo->prepare($sql);
$stmt->bindParam(1,$username);
$stmt->execute();
while($row=$stmt->fetch(PDO::FETCH_ASSOC))
{
     var_dump($row);
     echo "<br>";
}

```

PDO 在模拟预处理通过 wireshark 抓包可以看到是将处理完的 SQL 语句发送给 MySQL 服务器

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124082953-2b4a1ea2-1f6f-1.png)

非模拟预处理代码，在`$username = $_GET['username'];`代码前增加`$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);`
这里就是上面提到的，首先给 MySQL 服务器发送 SQL 语句模板，然后通过 EXECUTE 发送占位符参数给服务器

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083011-35eda5ae-1f6f-1.png)

## **预处理下的安全问题**

模拟预处理下

```php
<?php
$dbms='mysql';
$host='192.168.27.61';
$dbName='test';
$user='root';
$pass='root';
$dsn="$dbms:host=$host;dbname=$dbName";
try {
    $pdo = new PDO($dsn, $user, $pass);
} catch (PDOException $e) {
    echo $e;
}
//$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
$username = $_GET['username'];
$sql = "select id,".$_GET['field']." from user where username = ?";
$stmt = $pdo->prepare($sql);
$stmt->bindParam(1,$username);
$stmt->execute();
while($row=$stmt->fetch(PDO::FETCH_ASSOC))
{
    var_dump($row);
    echo "<br>";
}
```

可以看到 sql 语句 field 字段可控，这样我们构造 field，达到多语句执行的效果。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083244-91096d92-1f6f-1.png)

当设置`$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);`时，也可以达到报错注入效果。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083257-98b0789c-1f6f-1.png)

将上面模拟预处理代码`$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);`的注释关闭来进行非模拟预处理。

同样的 field 字段可控，这时多语句不可执行，但是当设置`$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);`时，也可进行报错注入。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083316-a40635e2-1f6f-1.png)

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083337-b073d71c-1f6f-1.png)

这里可进行报错注入是因为 MySQL 服务端 prepare 时报错，然后通过设置`PDO::ATTR_ERRMODE`将 MySQL 错误信息打印。
在 MySQL 中执行 prepare 语句

```php
prepare statm from "select id,updatexml(0x7e,concat(0x7e,user(),0x7e),0x7e) from user where username=?";
```

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083403-bfd009b0-1f6f-1.png)

## **总结**

1. 使用 PDO 时尽量使用非模拟预处理。
2. 创建 PDO 实例时将`PDO::MYSQL_ATTR_MULTI_STATEMENTS`设置为 false，禁止多语句查询。
3. SQL 语句模板不使用变量动态拼接生成。

## **参考**

https://dev.mysql.com/doc/apis-php/en/apis-php-mysqli.quickstart.multiple-statement.html
https://secure.php.net/manual/en/ref.pdo-mysql.php#pdo.constants.mysql-attr-multi-statements
https://www.leavesongs.com/PENETRATION/thinkphp5-in-sqlinjection.html