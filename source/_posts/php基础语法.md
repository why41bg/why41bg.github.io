---
title: php基础操作笔记
date: 2023-06-21 12:00:00
---



2023年6月份左右，有一个项目需要在本地建站，但是由于服务器配置不够，于是采用了轻量化的`php`来写网页。事实证明效果很不错，这篇笔记记录了一些关于`php`的基本操作。

# 1、PHP标记

开始标记：`<?php`

结束标记：`?>`



# 2、输出

`echo`：输出一个或多个字符串，逗号隔开

`print`：只允许输出一个字符串

`.`：连接符



# 3、变量

- 声明变量

```ph
<?php
	$a = 'hello php';
	echo $a;
?>
```



# 4、PHP和HTML混编

混编必须在`.php`文件中才行，需要执行php代码时，配合php标记即可



# 5、标量数据类型

`Boolean` `Integer` `Float` `String`



# 6、数组

- 创建空数组 & 输出数组元素

```php
<?php
	$arr = array(
		'ouyang' => '欧阳', //自定义索引为ouyang
		'wyy',
		'lwq'
	);
	echo $arr[0]; //输出wyy
	echo $arr['ouyang']; // 输出欧阳
	
	print_r($arr); //输出数组
	var_dump($arr); //输出数组
?>
```



- 数组循环

```php
<?php
	$arr = array(
		'ouyang' => '欧阳',
		'ximen' => '西门',
        'miejue' => '灭绝'
	);
	
	//值输出
	foreach($arr as $v){
		echo $v;
        echo '<hr>'; //换行
	}
	
	//键值对输出
	foreach($arr as $kkkk => $vvvv){
		echo $vvvv . $kkkk;
        echo '<hr>'; //换行
	}
?>
```



# 7、条件判断

`?:` 三目运算符

`if` `else`

```php
<?php
	if(){
        
    }elseif(){
        
    }else{
        
    }
?>
```



# 8、自定义函数

```php
function fun_name(//参数列表)
{
    //函数体
}
```



# 9、php操作Mysql

## PDO介绍

PDO即Php Data Object，是php数据对象。php对PDO的操作可以屏蔽不同数据库的差异。



## PDO连接

```php
$pdo = new PDO($dsn,$username,$password); //$dsn为数据源
//示例: $pdo = new PDO('mysql:host=localhost;dbname=php','root','123456');
```

```php
//准备一条sql语句并执行
$stmt = $pdo -> prepare('select * from test_db');
$stmt -> execute();
```

```php
//获取数据
$result = $stmt -> fetchAll();
print_r($result);
```



## 数据库连接

```php
<?php
	$con = mysqli_connect('host','user','password','db_name');
	// 返回一个代表到 MySQL 服务器的连接的对象
?>
```



# 10、built-in 函数说明

## 1.session_start()

**session_start()** 的作用是开启 \$\_SESION，需要在 \$\_SESION 使用之前调用。

PHP中 \$\_SESION 变量用于存储关于用户会话（session）的信息



## 2.isset()

**isset()** 函数用于检测变量是否已设置并且非 NULL，对应的释放变量为 **unset()**



## 3.mysqli_query()

参数为 `数据库连接对象`，`sql语句`

执行sql语句，并返回一个 **mysqli_result** 对象



# 11、超全局变量

## $_SESSION

**session** 存储在服务器中，用于保存用户的对话信息，创建一个 session 字段如下

```php
session_start(); //调用session之前必须start
$_SESSION['userid'] = $id; //将id变量赋给session中的userid字段，在其他的php文件中使用session_start()后也能使用这个字段
```

