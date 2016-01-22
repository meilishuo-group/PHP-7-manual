#从PHP5到PHP7

本文大部分内容参照“[Migrating from PHP 5.6.x to PHP 7.0.x](http://php.net/manual/en/migration70.php)”编写，主要是一些PHP7.x在实现上相较5.x的一些变化，给那些希望升级PHP 到7.x版本的童鞋参考。

##1. error 和 exception 的处理

###1.1 `fatal error` 转换成 `exception`

一些`fatal error`在`PHP7`被转换成`exception`。这些错误继承`Error`类并且实现了`Throwable`接口。（注：`Throwable`是一个新的接口，现在基础的`Exception`实现了这个接口）

也就是说，`set_exception_handler()`不一定只处理`Exception`，也可能是`Error`，以下是官网的一个🌰。

```php
<?php
// PHP 5 era code that will break.
function handler(Exception $e) { ... }
set_exception_handler('handler');

// PHP 5 and 7 compatible.
function handler($e) { ... }

// PHP 7 only.
function handler(Throwable $e) { ... }

```

另外关于`try catch`的写法也需要注意：
```php
try{
    ...
} catch(\Exception $e) {    //这里要注意是Exception or Error or Throwable
    ...
}
```

关于错误的详细描述：[Errors in PHP7](http://php.net/manual/en/language.errors.php7.php)

###1.2 所有的`E_STRICT`错误都被重新归类到其他的错误级别，`E_STRICT`依然被保留。

```php
<?php

error_reporting(E_ALL|E_STRICT);

class MyTest{
    function func(){
        echo "none static";
    }

}

//PHP 5.x : Strict Standards: Non-static method MyTest::func() ...
//PHP 7.X : Deprecated: Non-static method MyTest::func() ...
MyTest::func();
```

具体的分类参考下表：

|Situation|New level/behaviour
|-----------------------------------------------|---------------------------------
|Indexing by a resource                         |E_NOTICE
|Abstract static methods                        |Notice removed, triggers no error
|"Redefining" a constructor                     |Notice removed, triggers no error
|Signature mismatch during inheritance          |E_WARNING
|Same (compatible) property in two used traits  |Notice removed, triggers no error
|Accessing static property non-statically       |E_NOTICE
|Only variables should be assigned by reference |E_NOTICE
|Only variables should be passed by reference   |E_NOTICE
|Calling non-static methods statically          |E_DEPRECATED
 
##2. 变量的处理

PHP7编译源文件时使用虚拟语法树（abstract syntax tree）。它能带来很多方面的改进，这些改进都是由于之前受制于老版本的编译器而无法实现的。详细如下：

###2.1 非直接访问变量、属性、方法

非直接访问变量，现在会严格按照从左到右的顺序解析，参考下表：

|Expression|PHP 5 interpretation|PHP 7 interpretation
|-------------------|---------------------|---------------------
|$$foo['bar']['baz']|${$foo['bar']['baz']}|($$foo)['bar']['baz']
|$foo->$bar['baz']  |$foo->{$bar['baz']}  |($foo->$bar)['baz']
|$foo->$bar['baz']()|$foo->{$bar['baz']}()|($foo->$bar)['baz']()
|Foo::$bar['baz']() |Foo::{$bar['baz']}() |(Foo::$bar)['baz']()

###2.2 list()方法的变化

####2.2.1 list() 不在按倒序给变量赋值，而是按照正序赋值。

以下是官网的🌰：

```php
<?php
    list($a[], $a[], $a[]) = [1, 2, 3];
    var_dump($a);

```

PHP5输出：

```
array(3) {
    [0]=>
    int(3)
    [1]=>
    int(2)
    [2]=>
    int(1)
}
```

PHP7输出：

```
array(3) {
    [0]=>
    int(1)
    [1]=>
    int(2)
    [2]=>
    int(3)
}
```

当然，正常情况下不要依赖`list()`方法的赋值顺序，以防后续实现上再次发生变化。

####2.2.2 `list()`参数不可为空

以下情况均会产生错误：

```php
<?php
    list() = $a;
    list(,,) = $a;
    list($x, list(), $y) = $a;

```

####2.2.3 `list()`不能用作字符串的切割

如果有类似的需求，使用`str_split()`代替。

```php
<?php
    $str = 'abcde';
    list($a, $b) = $str;
    var_dump(array($a, $b));

```
PHP5输出：

```
array(2) {
    [0]=>
    string(1) "a"
    [1]=>
    string(1) "b"
}
```

PHP7输出：

```
array(2) {
    [0]=>
    NULL
    [1]=>
    NULL
}
```

###2.3 当元素被引用自动创建时的数组顺序发生改变的问题

以下是官网的🌰：

```php
<?php
    $array = [];
    $array["a"] =& $array["b"];
    $array["b"] = 1;
    var_dump($array);
```

PHP5输出：

```php
array(2) {
  ["b"]=>
  &int(1)
  ["a"]=>
  &int(1)
}
```

PHP7输出：

```php
array(2) {
  ["a"]=>
  &int(1)
  ["b"]=>
  &int(1)
}
```

###2.4 `global`声明基本变量赋值

动态变量不能通过`global`声明。有使用的情况，可以通过带花括号的方式模拟：

```php
<?php
function f() {
    // Valid in PHP 5 only.
    global $$foo->bar;

    // Valid in PHP 5 and 7.
    global ${$foo->bar};
}
```

要尽量避免，通过`global`去进行声明。

###2.5 通过`(`包裹`function`的参数会不起作用

当将一个方法作为引用参数时，在PHP7会抛出`Notice`错误，在PHP5中会抛出`Strict`错误。
当用`(`包裹`function`时，在PHP5中不会抛出任何错误，而在PHP7中同样还是会抛出`Notice`。
看🌰：

```php
<?php

error_reporting(E_ALL|E_STRICT);

function param() {
    return array(1, 2, 3);
}

function test(&$a) {
    var_export($a);
}

//PHP5 : Strict Standards: Only variables should be passed by reference in...
//PHP7 : Notice: Only variables should be passed by reference in ...
test(param());

//PHP5 : NO ERROR
//PHP7 : Notice: Only variables should be passed by reference in ...
test((param()));

```

##3. foreach

###3.1 `foreach`不在改变数组的内部指针

数组的内部指针将不会随着`foreach`的迭代发生变化，下面是官网的🌰：

```php
<?php
$array = [0, 1, 2];
foreach ($array as &$val) {
    var_dump(current($array));
}

```

PHP5的输出：

```
int(1)
int(2)
bool(false)
```

PHP7的输出

```
int(0)
int(0)
int(0)
```

###3.2 `foreach`的`by-value`和`by-reference`

遍历值的模式下，`foreach`的操作是遍历数组的备份而不是数组本身，所以在遍历的过程中对数组的修改不会生效。
引用模式下，`foreach`会在遍历的过程中记录数组的变化并会即可生效，看下面的🌰：

```php
<?php

$arr = array(
    'a' => 1,
);

foreach ($arr as $key => &$item) {
    echo $key."\n";
    $arr['b'] = 2;
}

unset($item);

```

PHP5输出：

```
a
```

PHP7输出：

```
a
b
```

##4. 整型

###4.1 非法8进制数字

PHP7中使用非法8进制数字，将会抛出`Parse error`，看下面🌰：

```php
<?php

//PHP5 : 非法8进制数字会进行转换，$a = 01
//PHP7 : Parse error: Invalid numeric literal in ...
$a = 0187;

```

###4.2 负数移位

PHP7中数字移位如果是负数，会抛出[ArithmeticError](http://php.net/manual/en/class.arithmeticerror.php)（注：`ArithmeticError`是PHP7中新加入的错误类型，表示计算型错误），下面是官网的一个🌰：

```php
<?php
//PHP5 : echo int(0)
//PHP7 : Fatal error: Uncaught ArithmeticError: Bit shift by negative number in...
var_dump(1 >> -1);
```

###4.3 移位超出整数范围的情况

在PHP7中，如果移位超出范围，则都返回0，而之前是要依赖于具体的架构的。

###4.4 除 0 问题

PHP7中，如果0作为除数，则会抛出`warning`，并返回`INF`或`NAN`；如果0作为余数会抛出`DivisionByZeroError`错误，下面的🌰：

```php
<?php

//PHP5 : Warning: Division by zero in ..., return false
//PHP7 : Warning: Division by zero in ..., return NAN
echo (0/0)."\n";

//PHP5 : Warning: Division by zero in ..., return false
//PHP7 : Warning: Division by zero in ..., return INF
echo (3/0)."\n";

//PHP5 : Warning: Division by zero in ..., return false
//PHP7 : Fatal error: Uncaught DivisionByZeroError: Modulo by zero in ...
echo (0%0)."\n";

```

##5. 字符串

###5.1 16进制字符串不再当做数字处理

下面是官网的🌰：

```php
<?php
    //PHP5 : bool(true)
    //PHP7 : bool(false)
    var_dump("0x123" == "291");
    //PHP5 : bool(true)
    //PHP7 : bool(false)
    var_dump(is_numeric("0x123"));
    //PHP5 : int(15)
    //PHP7 : int(0)
    var_dump("0xe" + "0x1");
    //PHP5 : string(2) "oo"
    //PHP7 : Notice: A non well formed numeric value encountered in ...
    var_dump(substr("foo", "0x1"));

```

###5.2 \u{ 引起的错误

PHP7引入的新特性[Unicode codepoint escape syntax](http://php.net/manual/en/migration70.new-features.php#migration70.new-features.unicode-codepoint-escape-syntax)，可以支持unicode字符编码的展示。这样带来一个问题，如果原来使用的字符包含`\u{`会引发PHP报错，看下面的🌰：

```php
<?php
echo "\u{9999}\n";
echo "\u{0009999}\n";
echo "\u{测试文本";

```

PHP5输出：

```
\u{9999}
\u{0009999}
\u{测试文本
```

PHP7:

```
Parse error: Invalid UTF-8 codepoint escape sequence in ...
```

如果要使用这种字符，需要将`\u{`前面的`\`转义，或者使用单引号。

##6 移除的方法

###6.1 call_user_method() 和 call_user_method_array()

|移除方法|替换方法
|------------------------|----------------------
|call_user_method()      |call_user_func()
|call_user_method_array()|call_user_func_array()

###5.2 mcrypt()

|移除方法|替换方法
|------------------------------------------------------|-----------------------
|mcrypt_generic_end()                                  |mcrypt_generic_deinit()
|mcrypt_ecb(), mcrypt_cbc(), mcrypt_cfb(), mcrypt_ofb()|mcrypt_decrypt() 

###5.3 intl()

|移除方法|替换方法
|-------------------------|----------------------------------
|datefmt_set_timezone_id()|IntlDateFormatter::setTimeZoneID() 
|datefmt_set_timezone()   |IntlDateFormatter::setTimeZone()


###6.4 set_magic_quotes_runtime()

移除set_magic_quotes_runtime(), magic_quotes_runtime()这两个方法，这两个方法之前主要是配合[魔法引号](http://php.net/manual/zh/security.magicquotes.php)使用的，在5.4版本中，已经将魔法引号废弃了。

###5.5 set_socket_blocking()

|移除方法|替换方法
|---------------------|---------------------
|set_socket_blocking()|stream_set_blocking() 

###6.6 dl()

dl()用来运行时载入PHP扩展，在5.3版本时一些SAPI已经将其移除，PHP7中 PHP-FPM 中将其移除。目前CLI和嵌入式SAPI依然保留此功能。

###6.7 GD()

GD扩展不在支持`PostScript Type1 font`，所以以下相关的方法被删除：

* imagepsbbox()
* imagepsencodefont()
* imagepsextendfont()
* imagepsfreefont()
* imagepsloadfont()
* imagepsslantfont()
* imagepstext()

建议使用新版的 `TrueType` 作为替换。

##7. ini 指令移除

* always_populate_raw_post_data
* asp_tags
* xsl.security_prefs

##8. 其他不兼容项

###8.1 不能将`new`的实例的引用给变量赋值，下面是🌰：

```php
<?php
class C {}

//PHP7 : Parse error: syntax error, unexpected 'new' (T_NEW) in
$c =& new C;

//It's OK
$a = new C;
$c =& $a;
```

###8.2 非法的`class`,`interface`,`trait`的命名

以下名字不能用作`class`,`interface`,`trait`的命名：

* bool
* int
* float
* string
* NULL
* TRUE
* FALSE

以下名字暂时不会引发错误，但不建议使用：

* resource
* object
* mixed
* numeric

###8.3 ASP 和 PHP脚本标签移除

|打开标签|关闭标签
|-----------------------|---------
|<%                     |%>
|<%=                    |%>
|<script language="php">|</script>

###8.4 上下文环境冲突

通过静态方式调用非静态方法，会抛出`Deprecated`，调用未定义的`$this`会抛出`Notice`，官网给出的🌰：

```php
<?php
class A {
    public function test() { var_dump($this); }
}

// Note: Does NOT extend A
class B {
    public function callNonStaticMethodOfA() { A::test(); }
}

(new B)->callNonStaticMethodOfA();
```

PHP5环境下：
```
Strict Standards: Non-static method A::test() should not be called statically, assuming $this from incompatible context in ...
```

PHP7环境下：
```
Deprecated: Non-static method A::test() should not be called statically in ...
Notice: Undefined variable: this in ...
```

###8.5 yield现在是一个右向运算符

`yield` 现在不需要使用`(`包裹，在PHP7中`yield`是一个右向运算符，优先级在`print`和`>=`之间，以下为官网的🌰：

```php
<?php
echo yield -1;
// Was previously interpreted as
echo (yield) - 1;
// And is now interpreted as
echo yield (-1);

yield $foo or die;
// Was previously interpreted as
yield ($foo or die);
// And is now interpreted as
(yield $foo) or die;

```

###8.6 方法参数不能同名

以下代码会引发`E_COMPILE_ERROR`：

```php
<?php
//PHP7 : Fatal error: Redefinition of parameter $b in ...
function test($a, $b, $b) {

}
```

###8.7 switch 语句不能有多个`default`代码块

以下代码会引发`E_COMPILE_ERROR`：

```php
<?php

//PHP7 : Fatal error: Switch statements may only contain one default clause in ...
switch (1) {
    case 1:
    break;
    default:
    break;
    default:
    break;
}
```

###8.8 移除$HTTP_RAW_POST_DATA

使用[php://input](http://php.net/manual/en/wrappers.php.php#wrappers.php.input)代替`$HTTP_RAW_POST_DATA`

###8.9 移除`#`符号在.ini文件中的注释

`.ini`不支持`#`表示注释，需要使用`;`代替，同时受影响的还有`parse_ini_file()`和`parse_ini_string()`的实现。

###8.8 JSON扩展用JSOND代替

这项改变，主要带来两个影响：
* 数字必须不能以`.`结尾
* 科学计数法标识`e`的前面不能是`.`，例如：`3.e3`为非法的，必须用`3.0e3`或`3e3`

###8.9 在数值溢出的时候，内部函数将会失败

如果浮点数过大，在转换时无法以整数表示，会抛出`warning`，并返回`null`。之前的处理方式是将整数截断。

###8.10 自定义会话处理器的返回值修复

之前自定义`session`处理器中，如果函数返回的不是 `false`, 也不是 `-1` 会引发 `fatal error`,现在如果函数返回值不是 `boolean`, `-1`, `0`，函数调用失败，引发 `warning`错误。

###8.11 func_get_arg() 和 func_get_args()返回当前参数值

在之前的版本中，func_get_arg() 和 func_get_args()返回最原始的方法参数，即使产生变化也会返回原始的数据，PHP7中返回当前最新数据，看下面🌰：

```php
<?php

error_reporting(E_ALL|E_STRICT);

function a($a, $b){
    $a = 11;
    var_export(func_get_args());
}

a(12,12);

```

PHP5 输出：
```
array (
  0 => 12,
  1 => 12,
)
```

PHP7 输出：
```
array (
  0 => 11,
  1 => 12,
)
```

##总结：
PHP7.x相较之前的版本的改变还是比较大的，同时到目前为止PHP7尚未发布一个稳定版本。个人看法是，如果是全新项目可以尝试，如果希望切换老版本需要格外谨慎，不建议这个马上迁移。

即便到目前为止都没有稳定版本放出，但是已经有很多web产品在生产环境中使用起来了，[点击查看](http://phpversions.info/php-7/)

##了解更多

* [PHP7的新特性](http://www.kkyfj.com/php/2016/01/21/PHP7-new-feature.html)
