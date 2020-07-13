---
title: "深入理解 Python 装饰器"
date: 2015-12-07 14:12:24 +0800
comments: true
weight: 5
draft: false
categories: ["Code", "Python"]
tags: ["Python", "Syntactic sugar"]
lightgallery: true
---
最近在写 Python+Django 的时候发现，有时候封装 API 的时候经常会遗失一些重复的装饰信息，但是直接封装到方法里是比较差劲的写法，因为有多个模块可能同时需要这些装饰信息，所以我希望使用一种可以迭代的装饰器。于是我在 [Stack Overflow](http://stackoverflow.com/questions/739654/how-can-i-make-a-chain-of-function-decorators-in-python/1594484#1594484) 上找到了相应的解答。下面以这篇解答为引写下我理解 Python decorator 的思路过程。
<!--more-->

## 装饰器是做什么用的？
装饰器实现对一个已有的模块做一些“修饰工作”，所谓修饰工作就是想给现有的模块加上一些小装饰（一些小功能，这些小功能可能好多模块都会用到），但又不让这个小装饰（小功能）侵入到原有的模块中的代码里去。

## 装饰器的定义
首先，你需要知道 [Python 的闭包](http://www.liaoxuefeng.com/wiki/001434446689867b27157e896e74d51a89c25cc8b43bdb3000/00143449934543461c9d5dfeeb848f5b72bd012e1113d15000)，接着发现3点 Python 的特性在装饰器中运用：

1. 函数可以赋值给一个变量。
2. 函数可以定义在另一个函数内部。
3. 函数名可以作为函数返回值。
   辣么，先来看一段代码:
``` python
def getTalk(type="shout"):

    # 定义函数
    def shout(word="yes"):
        return word.capitalize()+"!"

    def whisper(word="yes") :
        return word.lower()+"...";

    # 返回函数
    if type == "shout":
        # 没有使用"()", 并不是要调用函数，而是要返回函数对象
        return shout
    else:
        return whisper

# 如何使用？

# 将函数返回值赋值给一个变量
talk = getTalk()

# 我们可以打印下这个函数对象
print talk
#outputs : <function shout at 0xb7ea817c>

# 这个对象是函数的返回值
print talk()
#outputs : Yes!

# 不仅如此，你还可以直接使用之
print getTalk("whisper")()
#outputs : yes...
```

既然函数可以作为返回值，是不是函数也可以作为参数传递呢
``` python
def doSomethingBefore(func):
    print "I do something before then I call the function you gave me"
    print func()

doSomethingBefore(scream)
#outputs:
#I do something before then I call the function you gave me
#Yes!
```

所以看过这两段代码，你一定明白了，装饰器的定义。

装饰器就是封装器，可以让你在被装饰函数之前或之后执行代码，而不必修改函数本身代码。

## 怎么写封装器：
首先，我们来手写一个封装器：
``` python
def new_decorator(func):
    def wrapper():
        print("before the function runs")
        func()
        print("after the function runs")
    return wrapper

def along_func():
    print("I am a alone function")

decorated_along_func = new_decorator(along_func)
decorated_along_func()

#outputs:
#before the function runs
#I am a alone function
#after the function runs
```
这里每次调用 decorated_along_func 函数时，都会将 along_func 函数传入到装饰函数 new_decorator 中，完成封装。

## 怎么写装饰器：
那将上例代码稍微进行修改：
``` python
def new_decorator(func):
    def wrapper():
        print("before the function runs")
        func()
        print("after the function runs")
    return wrapper

@new_decorator
def along_func():
    print("I am a alone function")

along_func()
```
就会发现会得到相同的结果，这就是装饰器！

那么回到我最初的问题，装饰器能否迭代呢？

可以！

``` python
def decorator1(func):
    def wrapper():
        print("before the function runs")
        func()
        print("after the function runs")
    return wrapper

def decorator2(func):
    def wrapper():
        print("before the decorator1 runs")
        func()
        print("after the decorator1 runs")
    return wrapper

@decorator2
@decorator1
def along_func():
    print("I am a alone function")

along_func()

#outpus:
#before the decorator1 runs
#before the function runs
#I am a alone function
#after the function runs
#after the decorator1 runs
```
这种特性十分的便捷，但是必须注意装饰器的顺序。

如果上例代码写成：
``` python
@decorator1
@decorator2
def along_func():
    print("I am a alone function")
```
那么结果将变为
``` python
before the function runs
before the decorator1 runs
I am a alone function
after the decorator1 runs
after the function runs
```

## 一些迭代装饰器的用法
``` python
# bold装饰器
def makebold(fn):
    def wrapper():
        # 在前后加入标签
        return "<b>" + fn() + "</b>"
    return wrapper

# italic装饰器
def makeitalic(fn):
    def wrapper():
        # 加入标签
        return "<i>" + fn() + "</i>"
    return wrapper

@makebold
@makeitalic
def say():
    return "hello"

print say()
#outputs: <b><i>hello</i></b>

# 等价的代码
def say():
    return "hello"
say = makebold(makeitalic(say))

print say()
#outputs: <b><i>hello</i></b>
```
是不是灰常炫酷。

## 高级用法
关于更多装饰器的高级用法，你可以戳以下链接：

[戳我](https://wiki.python.org/moin/PythonDecoratorLibrary)

关于 Python Decroator 的各种提案，可以参看：

[Python Decorator Proposals](https://wiki.python.org/moin/PythonDecoratorProposals)
