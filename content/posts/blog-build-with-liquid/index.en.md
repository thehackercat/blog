---
weight: 5
title: "在 Octopress 中生成包含 liquid 语句的代码"
date: 2015-11-20 16:32:12 +0800
comments: true
draft: false
categories: ["Code"]
tags: ["Octopress"]
lightgallery: true
---

## 背景

由于之前写 [Django -- Templates](http://thehackercat.me/blog/2015/11/16/django-learning2/) 篇时要用到包含 Liquid 语法的示例代码，而 Octopress (Jekyll) 在后端使用 Liquid 来处理生成 Web Pages ，对于文章内部插入的原本用来作示例的 Liquid 代码会被解析成 Web Pages 生成语句而不是原本的内容。故苦恼了我一会儿 Q.Q

~~不过这都不是事儿~~

## 解决方法

比如，我之前写的
<!--more-->
```html
{% raw %}
{% if variable %}
{% else %}
{% endif %}
{% endraw %}
```
就会因为包含了

```html
{% raw %}{% ... %}{% endraw %}
```

解决方法是：

在每一块包含 Liquid 语句的代码快前后用

```html
{{ "{% raw " }}%} 和 {{ "{% endraw " }}%
```

包括起来。

这样就能确保示例代码不会被错误的解析成 Jekyll Web Pages 生成语句。

但是如果我要显示

```html
{{ "{% raw " }}%} 和 {{ "{% endraw " }}%}
```

怎么办呢 ？

我试着使用使用如下方法：
```html
{{ "{% raw " }}%}
{{ "{% raw " }}%}
{{ "{% endraw " }}%}
{{ "{% endraw " }}%}
```
来显示一个

```html
{{ "{% raw " }}%} 和 {{ "{% endraw " }}%}
```

后来我在 [Stack Overflow](http://stackoverflow.com/questions/3426182/how-to-escape-liquid-template-tags) 找到了一个回答：

使用

```html
{% raw %}{{ "{% raw " }}%}{% endraw %}
```

就可以得到

```html
{{ "{% raw " }}%}
```

这种方法是正确的。

![awesomesauce](http://ww1.sinaimg.cn/large/6aa09e8fjw1evgxf183vrj20zk12sqa3.jpg)
