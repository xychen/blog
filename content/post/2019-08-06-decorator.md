---
title: "装饰者模式（PHP实现）"
date: 2019-06-06 17:00:00
---

## 背景
在Laravel框架中，解析请求生成响应之前或之后需要经过中间件的处理，包括验证、Cookie加密、开启会话、CSRF保护等。这些处理有些是在生成响应之前，有些是在生成响应之后，在实际开发过程中可能还需要添加新的处理功能，如果在不修改原有类的基础上动态地添加或减少处理功能，将使框架可扩展性大大增强，而这种需求正好可以用装饰者模式解决。  

## array_slice的用法
在说装饰者模式之前，我们先了解一下array_slice的用法。  

- 函数原型：  
```
array_reduce ( array $array , callable $callback [, mixed $initial = NULL ] ) : mixed
```

array_reduce() 将回调函数 callback 迭代地作用到 array数组中的每一个单元中，从而将数组简化为单一的值。  

- 例1：  

```
function sum($carry, $item)
{
    $carry += $item;
    return $carry;
}

$a = array(1, 2, 3, 4);

var_dump(array_reduce($a, "sum")); // int(10)
var_dump(array_reduce($a, "sum", 10)); // int(20)
```

把sum函数迭代到$a数组中，注意：array_reduce($a, "sum")中，由于没有设置第3个参数，所以$carry的值第一次是null，一定要做好妥善处理，这个调用的结果就是1+2+3+4 = 10 。而array_reduce($a, "sum", 10)有一个初始值10，所以结果是10+1+2+3+4 = 20。  

- 例2： 对上边的例子做一下等价换算  

```
array_reduce([1, 2, 3, 4], "sum");
array_reduce([2, 3, 4],    "sum",           sum(null, 1)                );
array_reduce([3, 4],       "sum",       sum(sum(null, 1), 2)            );
array_reduce([4],          "sum",   sum(sum(sum(null, 1), 2), 3)        );
array_reduce([],           "sum", sum(sum(sum(sum(null, 1), 2), 3), 4)  );
最终的函数调用：
sum(sum(sum(sum(null, 1), 2), 3), 4);
```

以上就是等价的调用过程，其中null会被转成0处理。

## 中间件的实现  

首先，我们选用《Laravel框架关键技术解析》中的列子：  
```
interface Middleware
{
  public static function handle(Closure $next);
}

class VerifyCsrfToken implements Middleware
{
  public static function handle(Closure $next)
  {
    echo "验证csrf-token\n";
    $next();
  }
}

class ShareErrorsFromSession implements Middleware
{
  public static function handle(Closure $next)
  {
    echo "如果session中有‘errors’变量，则共享它\n";
    $next();
  }
}

class StartSession implements Middleware
{
  public static function handle(Closure $next)
  {
    echo "开启session, 获取数据.\n";
    $next();
    echo "保存数据，关闭session\n";
  }
}

class AddQueuedCookiesToResponse implements Middleware
{
  public static function handle(Closure $next)
  {
    $next();
    echo "添加下一次请求需要的cookie\n";
  }
}

class EncryptCookies implements Middleware
{
  public static function handle(Closure $next)
  {
    echo "对输入请求的cookie进行解密\n";
    $next();
    echo "对输出响应的cookie进行加密\n";
  }
}

class CheckForMaintenanceMode implements Middleware
{
  public static function handle(Closure $next)
  {
    echo "确定当前程序是否处于维护状态\n";
    $next();
  }
}

function getSlice()
{
  return function($stack, $pipe)
  {
    return function() use ($stack, $pipe) 
    {
      return $pipe::handle($stack);
    };
  };
}

function then()
{
  $pipes = [
    "CheckForMaintenanceMode",
    "EncryptCookies",
    "AddQueuedCookiesToResponse",
    "StartSession",
    "ShareErrorsFromSession",
    "VerifyCsrfToken"
  ];

  $firstSlice = function() {
    echo "请求向路由器传递，返回响应.\n";
  };

  $pipes = array_reverse($pipes);

  call_user_func(array_reduce($pipes, getSlice(), $firstSlice));
}

then();
```

运行结果：

```
确定当前程序是否处于维护状态
对输入请求的cookie进行解密
开启session, 获取数据.
如果session中有‘errors’变量，则共享它
验证csrf-token
请求向路由器传递，返回响应.
保存数据，关闭session
添加下一次请求需要的cookie
对输出响应的cookie进行加密
```

### then函数拆解

- 首先，对then函数中的代码做一个简单的精简，可以看到array_reduce的第二个参数是一个函数调用getSlice()，那么我们可以直接转成一个函数f  
```
function f($stack, $pipe)
{
  return function() use ($stack, $pipe)
  {
    return $pipe::handle($stack);
  };
}
```
那么对于的调用就变成了：  
```
call_user_func(array_reduce($pipes, "f", $firstSlice));
```

- 然后把函数进一步拆解：
```
$res = array_reduce($pipes, "f", $firstSlice);
call_user_func($res);
```

好了，我们来重点分析一下array_reduce的调用关系。

#### 核心迭代函数的拆解

- 我们把$pipes精简成3个，另外暂时不考虑顺序（即不考虑array_reverse）  

```
array_reduce(["EncryptCookies","StartSession","VerifyCsrfToken"], "f", $firstSlice);

array_reduce(["StartSession","VerifyCsrfToken"], "f", f($firstSlice, "EncryptCookies"));

array_reduce(["VerifyCsrfToken"], "f", f(f($firstSlice, "EncryptCookies"),"StartSession"));

array_reduce([], "f", f(f(f($firstSlice, "EncryptCookies"),"StartSession"), "VerifyCsrfToken"));

f(f(f($firstSlice, "EncryptCookies"),"StartSession"), "VerifyCsrfToken");
```

- 再来拆解一下f($firstSlice, "EncryptCookies")，返回值是一个函数:  
```
function()
{
    return EncryptCookies::handle($firstSlice);
}

handle函数中的调用：
function handle(Closure $next)
{
    echo "对输入请求的cookie进行解密\n";
    $next();    //即调用firstSlice()
    echo "对输出响应的cookie进行加密\n";
}

```

- 拆解f(f($firstSlice, "EncryptCookies"),"StartSession")
```
第1步转换：
f(function(){
    return EncryptCookies::handle($firstSlice);
}, "StartSession");

第2步转换：
function() {
    StartSession::handle(function(){
        return EncryptCookies::handle($firstSlice);
    });
}

对应的handle函数：
function handle(Closure $next)
{
    echo "开启session, 获取数据.\n";
    $next();        //即调用function(){return EncryptCookies::handle($firstSlice);}
    echo "保存数据，关闭session\n";
}
```

- 拆解f(f(f($firstSlice, "EncryptCookies"),"StartSession"), "VerifyCsrfToken")  
```
第1步转换：
f(function() {
    StartSession::handle(function(){
        return EncryptCookies::handle($firstSlice);
    });
}, "VerifyCsrfToken");
第2步转换：
VerifyCsrfToken::handle(function() {
    StartSession::handle(function(){
        return EncryptCookies::handle($firstSlice);
    });
});
对应的handle函数：
function handle(Closure $next)
{
    echo "验证csrf-token\n";
    $next();   
}
$next对应的函数调用为：
VerifyCsrfToken::handle(function() {
    StartSession::handle(function(){
        return EncryptCookies::handle($firstSlice);
    });
});
```

## 总结

用PHP用装饰者模式实现中间件的关键点：  

- 借助array_reduce实现迭代调用  
- array_reduce中的第二个参数(即callback函数f)的返回值是匿名函数，即Closure类型  
- 有对应的接口或者抽象类定义同名的handle函数，对应的参数是Closure类型，即可调用的匿名函数  
- array_reduce的第3个参数一定要传，避免handle函数调用时报错（为null时）  

## 参考列表

- [array_reduce官方手册](https://www.php.net/manual/zh/function.array-reduce.php)  
- [《Laravel框架关键技术解析》](https://item.jd.com/10538397403.html)