# 网鼎杯第三场WEB部分wp

<sub>*记录</sub>

源码：

```php
<?php
ini_set("display_errors",0);
$uri = $_SERVER['REQUEST_URI'];
if(stripos($uri,".")){
        die("Unkonw URI.");
}
if(!parse_url($uri,PHP_URL_HOST)){
        $uri = "http://".$_SERVER['REMOTE_ADDR'].$_SERVER['REQUEST_URI'];
}
$host = parse_url($uri,PHP_URL_HOST);
if($host === "c7f.zhuque.com"){
        setcookie("AuthFlag","flag{*******");
}
?>
```

梳理一下执行过程：
首先会从`$_SERVER[]`变量中获取`REQUEST_URI`的值赋值给变量`$uri`，然后判断`$uri`第一位是否是.(点)，如果不是点，则`die()`，如果是点，则继续执行。

然后获取`$uri`中的`host`值，如果为`null`，则执行`if`中操作，重新给`$uri`赋值。
接下来继续获取`$uri`中的`host`值，并做判断，是否等于`c7f.zhuque.com`，等于则获得`flag`。

1.绕过stripos

![image.png](Net-Ding-Cup-third-game-WEB-part-wp/网鼎杯第三场WEB部分wp-1.png)

使用`./..//demo2.php`可绕过
`./`理解成一个目录，然后`../`就是回退到上一级目录

![image.png](Net-Ding-Cup-third-game-WEB-part-wp/网鼎杯第三场WEB部分wp-2.png)

成功绕过了stripos。

2.下面绕过第二个`parse_url`这里可利用以下请求方式绕过。
```javascript
http://username:password@hostname/path?arg=value
```
这样，就需要进入第一个`parse_url`，给`$uri`赋值。

![image.png](Net-Ding-Cup-third-game-WEB-part-wp/网鼎杯第三场WEB部分wp-3.png)

请求

```javascript
http://127.0.0.1.@c7f.zhuque.com/..//demo2.php
```

类似于上面的

```javascript
http://username:password@hostname/path?arg=value
```

成功获取flag

![image.png](Net-Ding-Cup-third-game-WEB-part-wp/网鼎杯第三场WEB部分wp-4.png)



