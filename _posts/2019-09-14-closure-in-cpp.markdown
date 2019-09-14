---
layout: post
title:  "C++ 中的闭包"
date:   2019-09-14 04:56:27 +0000
categories: programming c++
---

## 啥是闭包

先贴 Wiki 的链接： https://zh.wikipedia.org/wiki/%E9%97%AD%E5%8C%85_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6)

要点：
* 闭包是一类函数
* 闭包函数内，引用到了上层作用域的变量
* 闭包一般指的是词法闭包，动态作用域的语言，（如 Bash、C 的宏）中不存在本文所说的闭包

## Javascript 中的闭包

假设我们需要一个函数 `add1`，它的行为满足下面的条件：

```javascript
add1(1) == 2
add1(2) == 3
add1(3) == 4
```

这样的一个函数可以很容易地写出来：

```javascript
function add1(n) {
    return n + 1;
}
```

但是我们如果需要 `add2`、`add3`，该怎么办呢？
也许我们可以写一个函数去生成它：

```javascript
function generate_addx(x) {
    function addx(n) {
        return x + n;
    }
    return addx;
}
```

然后这样去使用：
```javascript
const add1 = generate_addx(1);
const add2 = generate_addx(2);
```

这样，`add1` 就成为了一个闭包，它引用了上层变量 `x`，而 `x` 是 `generate_addx` 的实参。
同时，`add1` 和 `add2` 虽然都是 `function addx(n)` 处定义的，但是它们不是同一个函数，执行的效果也不同，`add1` 返回值是参数+1，而 `add2` 的返回值是参数+2。
所以说闭包是有状态的，闭包的状态就是上层变量的值。

## C 语言中的闭包

C 语言不支持函数的嵌套定义，所以不存在闭包的概念，不借助第三方库的情况下只能手动传递状态来达到相同的功能。

使用下文提到的 libffi，也可以实现闭包的效果。

## C++ 闭包 1.0

C++ 有了函数对象，可以用成员变量来实现：

```c++
class addx
{
private:
    int x;

public:
    addx(int x): x(x)
    {
    }

    int operator(int n) const
    {
        return x + n;
    }
};
```

然后这么使用：
```c++
addx add1(1);
int add1_1 = add1(1);
```

## C++ 闭包 2.0

闭包 1.0 原理简单，但是和 Javascript 版本的闭包相差还很远，有没有办法改进呢？

C++11 带来了几个新特性，比如 `std::bind`。
它的功能是运行时绑定函数参数，也就是所谓的柯里化。

```c++
int sum(int a, int b)
{
    return a + b;
}
```

然后这么使用：

```c++
auto add2 = std::bind(sum, _1, 2);
int add2_1 = add2(1);
```

你可能注意到了我用了 `auto` 来接受 `std::bind` 的返回值，这是因为经过 `std::bind` 的魔法操作之后，`add2` 的类型变得难以确定。
`std::bind` 实际上是一个模板函数，这里存在一个模板类型自动推导过程。
`add2` 本身是一个函数对象，它是某个模板类的一个实例，而哪个类的模板类型取决于 `std::bind` 的模板类型。

那咋整呢？
好在 C++11 给我们提供了另一个魔法：`std::function`，我们用 `std::function` 去存储这个闭包就行了。

```c++
std::function<int(int)> add2 = std::bind(sum, _1, 2);
```

`std::function` 的类型是固定的，所以可以作为回调函数传来传去了。
同时，你也可以把你自己定义的函数对象或者是普通函数、函数指针放进去，只要参数返回值对上就行了。

## C++ 闭包 3.0

C++11 还给我们带来了一个重量级特性，甚至可以完全代替 `std::bind`，那就是 lambda。

lambda 支持闭包，但是 C++ 本身没有 gc 加持，使用上总有些不方便的地方，但是至少能用了。

```c++
auto generate_addx(int x)
{
    return [x](int n)
    {
        return x + n;
    };
}
```

此处 `generate_addx` 的返回值还是 `auto`，因为 lambda 的类型不确定，但是我们还是可以用 `std::function`：

```c++
auto add1 = generate_addx(1);
std::function<int(int)> add2 = generate_addx(2);
```

lambda 实现闭包的原理是把捕获到的变量存储到生成的函数对象中。
你尝试捕获不同类型、不同数量的变量，然后用 `sizeof` 查看 lambda 函数对象的大小（你会发现结果是不同的）。
而 `std::function` 总是固定大小的。

## 零开销抽象：有时候你只需要模板

```c++
template<int x>
int addx(int n)
{
    return x + n;
}
```

也许你也不需要 `std::function`：

```c++
template<typename F, typename T>
void apply(F change, T &value)
{
    value = change(value);
}

int n = 1;
auto add1 = addx<1>;
apply(add1, n);
// n == 2
```

## 重量级解决方案：libffi

与 `std::bind` 用法类似，但是动态生成代码，得到的是一个 C 函数指针。
适合需要给 C 库提供回调函数的情况。

http://www.chiark.greenend.org.uk/doc/libffi-dev/html/Closure-Example.html
