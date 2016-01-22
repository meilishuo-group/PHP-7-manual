#PHP7 带来的新特性

本文大部分内容参照“[Backward incompatible changes - New features](http://php.net/manual/en/migration70.new-features.php)”编写，主要介绍PHP7相较PHP5带来的一些新特性。

##1. 参数、返回值类型声明

类型参数扩充了以下类型：`string`, `int`, `float`, `bool`。
支持返回值的类型声明，类型与参数类型一致。

```php
<?php

/**
 * [test description]
 * @param  int    $a 
 * @param  int    $b 
 * @return int
 */
function test(int $a, int $b): int 
{
    return $a + $b;
}

//echo 3
echo test(1, '2'), PHP_EOL;

```

##2. null合并运算符

经常会有使用三元运算符判断`一个值如果存在即返回这个值，否则返回另一个值的情况`，PHP7专门提供了这样一个运算符`??`来解决这个问题，当前一个值存在并且不为空，则返回，否则返回后一个值，请看下面的🌰，注意`??`和`?:`的区别。

```php
<?php

$arr = array();

//PHP5&PHP7 : Notice: Undefined index: a in ..., 同时echo 10
echo ($arr['a'] ?: 10), PHP_EOL;

//PHP7 : echo 10
echo ($arr['a'] ?: 10), PHP_EOL;

```

##3. 宇宙飞船运算符

这个符号`<=>`是不是很像宇宙飞船，实现的功能是比较两个值，大于: `1`, 小于: `-1`, 等于: `0`，看个🌰：

```php
<?php

echo 1 <=> 1; // 0
echo 1 <=> 2; // -1
echo 2 <=> 1; // 1

```

##4. define()可以支持数组

下面是官网的一个🌰：

```php
<?php
define('ANIMALS', [
    'dog',
    'cat',
    'bird'
]);
```

##4. 匿名类

可以通过`new`来实例化一个匿名类，看官网的🌰：

```php

<?php
interface Logger {
    public function log(string $msg);
}

class Application {
    private $logger;

    public function getLogger(): Logger {
         return $this->logger;
    }

    public function setLogger(Logger $logger) {
         $this->logger = $logger;
    }
}

$app = new Application;
//此处注意匿名类
$app->setLogger(new class implements Logger {
    public function log(string $msg) {
        echo $msg;
    }
});

var_dump($app->getLogger());

```

##5. Unicode码转译

使用双引号包裹的以`\u{`开始的字符串将会进行“Unicode码转译”，`\u{`可以跟着`0`，在转译时前面的0将会被忽略。注意如果是从PHP5升级到PHP7的并且有使用过`\u{`字符的要将反斜杠转义，否则会异常，具体的可以看[从PHP5到PHP7](http://www.kkyfj.com/php/2016/01/02/from-PHP5-to-PHP7.html)，下面是官网提供的🌰：

```php
<?php

echo "\u{aa}";          //ª
echo "\u{0000aa}";      //ª
echo "\u{9999}";        //香

```

##6. 新增Closure::call()方法

闭包与对象临时绑定并调用，变得更加简单，看下官网提供的🌰：

```php
<?php
class A {private $x = 1;}

// Pre PHP 7 code
$getXCB = function() {return $this->x;};
$getX = $getXCB->bindTo(new A, 'A'); // intermediate closure
echo $getX();

// PHP 7+ code
$getX = function() {return $this->x;};
echo $getX->call(new A);

```

更多细节查看[Closure::call](http://php.net/manual/en/closure.call.php)

##7. unserialize() 过滤功能

`unserialize()`支持设置白名单功能，使得当反序列化不可靠的数据时更加安全，看🌰：

```php
<?php

// converts all objects into __PHP_Incomplete_Class object
$data = unserialize($foo, ["allowed_classes" => false]);

// converts all objects into __PHP_Incomplete_Class object except those of MyClass and MyClass2
$data = unserialize($foo, ["allowed_classes" => ["MyClass", "MyClass2"]]);

// default behaviour (same as omitting the second argument) that accepts all classes
$data = unserialize($foo, ["allowed_classes" => true]);
```

更多细节请查看[unserialize()](http://php.net/manual/en/function.unserialize.php)。

##8. 新增IntlChar

暴露了很多ICU库的方法，以获取Unicode字符的相关信息，更多信息请查看[IntlChar](http://php.net/manual/en/class.intlchar.php)。

##9. 预期（assert的增强）

`预期`是对`assert()`功能的一个增强，它降低了`assert()`的使用成本，并且能够支持断言失败是抛出异常。

出于兼容之前`assert()`的目的，将`assert.exception`设置为`disable`时，`assert`将按照前的方式执行。

```php
<?php
ini_set('assert.exception', 1);

class CustomError extends AssertionError {}

//echo Fatal error: Uncaught CustomError: Some error message...
assert(false, new CustomError('Some error message'));

```

更多内容，请查看：[expectations section](http://php.net/manual/en/function.assert.php#function.assert.expectations)

##10. use 组声明

可以通过一个`use`导入多个 类、方法和常量，看下官网给出的🌰：

```php
<?php
// Pre PHP 7 code
use some\namespace\ClassA;
use some\namespace\ClassB;
use some\namespace\ClassC as C;

use function some\namespace\fn_a;
use function some\namespace\fn_b;
use function some\namespace\fn_c;

use const some\namespace\ConstA;
use const some\namespace\ConstB;
use const some\namespace\ConstC;

// PHP 7+ code
use some\namespace\{ClassA, ClassB, ClassC as C};
use function some\namespace\{fn_a, fn_b, fn_c};
use const some\namespace\{ConstA, ConstB, ConstC};

```

##11. Generator 支持 return 表达式

`Generator`中支持`return`，`return`值可以通过`getReturn()`方法获得，但是必须要在`Generator`迭代结束以后，否则会抛出`Fatal error`，看下下面使用的🌰：

```php
<?php

$gen = (function() {
    yield 1;
    yield 2;

    return 3;
})();

foreach ($gen as $val) {
    echo $val, PHP_EOL;
}

echo $gen->getReturn(), PHP_EOL;

```

##12. Generator 代理

`Generator`可以代理到其他的`Generator` 和 一切实现了`Traversable`的对象 或 数组，使用`yield from`实现`Generator`代理功能，看下面的🌰：

```php
<?php
function generator()
{
    yield 1;
    yield 2;
    yield from generator_delegate_test();   //代理 Generator
    yield from array('a', 'b', 'c');        //代理 array
}

function generator_delegate_test()
{
    yield 3;
    yield 4;
}

foreach (generator() as $val)
{
    echo $val, PHP_EOL;
}

```

##12. intdiv() 整数除法

个人理解，`intdiv($a, $b) ≈ intval($a / $b) ≈ $a / $b >> 0`。

注意，当`0`作为被除数时，会抛出`DivisionByZeroError`；如果被除数为`PHP_INT_MIN`并且除数为`-1`时，会抛出`ArithmeticError`，原因是正常计算的情况下`PHP_INT_MIN`除以`-1`得到的数字超出了PHP整数的最大范围，看下🌰：

```php
<?php

//echo 3
echo intdiv(10, 3). PHP_EOL;

//Fatal error: Uncaught DivisionByZeroError: Division by zero in ...
echo intdiv(1, 0). PHP_EOL;

//Fatal error: Uncaught ArithmeticError: Division of PHP_INT_MIN by -1 is not an integer in ...
echo intdiv(PHP_INT_MIN, -1). PHP_EOL;

//echo PHP_INI_MAX
echo intdiv(PHP_INT_MIN + 1, -1). PHP_EOL;


```

##Session 选项

`session_start()`方法，现在支持一个`array()`参数，数组内容会覆盖掉`php.ini`中的配置，看🌰：

```php
<?php
session_start([
    'cache_limiter' => 'private',
    'read_and_close' => true,
]);

```

##preg_replace_callback_array()

`preg_replace_callback`可支持`function`数组，看下面的🌰：

```php
<?php
$str = 'Aaaaa bbbb cccc';

$result = preg_replace_callback_array(
    [
        '/a/i' => function ($match) {
            return 1;
        },
        '/b/i' => function ($match) {
            return 2;
        }
    ],
    $str
);

//echo 11111 2222 cccc
echo $result, PHP_EOL;

```

更多细节，请看：[preg_replace_callback_array](http://php.net/manual/en/function.preg-replace-callback-array.php)

##CSPRNG 方法

加入了两个新方法`random_bytes()` 和 `random_int()`，🌰：

```php
<?php

//生成一个50个字符长度的随机串
echo random_bytes(50), PHP_EOL;
//生成一个1 ~ 50之间的字符串
echo random_int(1, 50), PHP_EOL;

```

## list() 能够正确处理`ArrayAccess`

之前版本`list()`不能保证正确处理所有`ArrayAccess`，在PHP7中解决了这个问题。

了解更多：[从PHP5到PHP7](http://www.kkyfj.com/php/2016/01/02/from-PHP5-to-PHP7.html)
