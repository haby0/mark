# **前言**

PHP数据对象（PDO）扩展为PHP访问数据库定义了一个轻量级的一致接口。PDO提供了一个数据访问抽象层，这意味着，不管使用哪种数据库，都可以使用相同的函数（方法）来查询和获取数据。PDO随PHP 5.1发行，在PHP 5.0的PECL扩展中也可以使用，无法运行于之前的PHP版本。今天我们讨论PDO多语句执行（堆叠查询）和PDO预处理下的SQL注入问题导致SQL注入的问题。如有不足，不吝赐教。

# **PDO多语句执行**

PHP连接MySQL数据库有三种方式（MySQL、Mysqli、PDO），同时官方对三者也做了列表性比较：

|   | Mysqli  | PDO  | MySQL  |
| ------------ | ------------ | ------------ | ------------ |
| 引入的PHP版本  | 5.0  | 5.0  | 3.0之前  |
| PHP5.x是否包含 | 是  | 是  | 是  |
| 服务端prepare语句的支持情况 | 是  | 是  | 否  |
| 客户端prepare语句的支持情况 | 否  | 是  | 否  |
| 存储过程支持情况 | 是  | 是  | 否  |
| 多语句执行支持情况 | 是  | 大多数  | 否  |

可以看到Mysqli和PDO是都是支持多语句执行的，我们对比一下看看两者的区别
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
请求脚本后发现数据库中成功创建了test2表，说明多语句成功执行

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081038-7a56388a-1f6c-1.png)

我们通过wireshark分析一下，首先登录请求Multiple statements字段未设置

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081215-b4abe048-1f6c-1.png)

通过multi_query()函数可以看到在执行Query前向Mysql服务器发送了一次Set Option请求将multi statements设置打开

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081242-c4a2f0d6-1f6c-1.png)

而使用普通的mysqli_query()函数，在执行Query前不会向Mysql服务器发送set option请求

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081251-c9c184c4-1f6c-1.png)

2.使用PDO中的query()函数同数据库交互
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
请求脚本后发现数据库中成功创建了test2表，说明多语句成功执行

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081916-af9f10e2-1f6d-1.png)

通过wireshark分析，通过PDO方式同数据库交互时，在登录时会设置Multiple statements字段，然后通过Query方式直接发送多语句到Mysql服务器

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124081931-b81b89f8-1f6d-1.png)

PDO默认支持多语句查询，如果php版本小于5.5.21或者创建PDO实例时未设置PDO::MYSQL_ATTR_MULTI_STATEMENTS为false时可能会造成堆叠注入
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

$id变量可控，构造链接访问，成功创建aaa数据表

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124082057-ebc1ab70-1f6d-1.png)

如果想禁止多语句执行，可在创建PDO实例时将PDO::MYSQL_ATTR_MULTI_STATEMENTS设置为false
```php
new PDO($dsn, $user, $pass, array( PDO::MYSQL_ATTR_MULTI_STATEMENTS => false))
```

# **MySQL预处理**

MySQL数据库支持预处理，预处理或者说是可传参的语句用来高效的执行重复的语句。

MySQL官方将prepare、execute、deallocate统称为PREPARE STATEMENT

预制语句的SQL语法基于三个SQL语句：
prepare stmt_name from  preparable_stmt;
execute stmt_name [using @var_name [, @var_name] ...];
{deallocate | drop} prepare stmt_name;

# **PDO预处理**

PDO分为模拟预处理和非模拟预处理。

模拟预处理是防止某些数据库不支持预处理而设置的，在初始化PDO驱动时，可以设置一项参数，PDO::ATTR_EMULATE_PREPARES，作用是打开模拟预处理(true)或者关闭(false),默认为true。PDO内部会模拟参数绑定的过程，SQL语句是在最后execute()的时候才发送给数据库执行。

非模拟预处理则是通过数据库服务器来进行预处理动作，主要分为两步：第一步是prepare阶段，发送SQL语句模板到数据库服务器；第二步通过execute()函数发送占位符参数给数据库服务器进行执行。

首先我们通过wireshark抓包方式对比一下模拟预处理和非模拟预处理
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

PDO在模拟预处理通过wireshark抓包可以看到是将处理完的SQL语句发送给MySQL服务器

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124082953-2b4a1ea2-1f6f-1.png)

非模拟预处理代码，在$username = $_GET['username'];代码前增加$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
这里就是上面提到的，首先给MySQL服务器发送SQL语句模板，然后通过EXECUTE发送占位符参数给服务器

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083011-35eda5ae-1f6f-1.png)

# **预处理下的安全问题**

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

可以看到sql语句field字段可控，这样我们构造field，达到多语句执行的效果。

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083244-91096d92-1f6f-1.png)

当设置$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);时，也可以达到报错注入效果

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083257-98b0789c-1f6f-1.png)

将上面模拟预处理代码$pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);的注释关闭来进行非模拟预处理

同样的field字段可控，这时多语句不可执行，但是当设置$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);时，也可进行报错注入

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083316-a40635e2-1f6f-1.png)

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083337-b073d71c-1f6f-1.png)

这里可进行报错注入是因为MySQL服务端prepare时报错，然后通过设置PDO::ATTR_ERRMODE将MySQL错误信息打印
在MySQL中执行prepare语句
```php
prepare statm from "select id,updatexml(0x7e,concat(0x7e,user(),0x7e),0x7e) from user where username=?";
```

![image.png](https://xzfile.aliyuncs.com/media/upload/picture/20190124083403-bfd009b0-1f6f-1.png)

# **总结**

1. 使用PDO时尽量使用非模拟预处理。
2. 创建PDO实例时将PDO::MYSQL_ATTR_MULTI_STATEMENTS设置为false，禁止多语句查询。
3. SQL语句模板不使用变量动态拼接生成

**参考**

https://dev.mysql.com/doc/apis-php/en/apis-php-mysqli.quickstart.multiple-statement.html
https://secure.php.net/manual/en/ref.pdo-mysql.php#pdo.constants.mysql-attr-multi-statements
https://www.leavesongs.com/PENETRATION/thinkphp5-in-sqlinjection.html