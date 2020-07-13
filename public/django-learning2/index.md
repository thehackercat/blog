#  Django 学习笔记2-- Templates 

虽然 Django 中 Html 可以直接硬编码到 Python 中，但是这种行为并不利于前端开发人员进行维护。所以 Django 有了[流模板](http://liquidmarkup.org/) ( Liquid Templates )。

## 流模板基础

举个例子，下面这个模板大致含括了 Django 模板的几个特性。
<!--more-->

``` html
{% raw %}
{% load staticfiles %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}{% endblock %}</title>
</head>
<body>
    <p>Dear {{ person_name }},</p>

    <p>Thanks for placing an order from {{ company }}. It's scheduled to ship on {{ ship_date|date:"F J,Y " }}.</p>

    {% if ordered_warranty %}
    <p>Your warranty information will be included in the packaging.</p>
    <p>Here are the items you've ordered: </p>

    <ul>
    {% for item in item_list %}
        <li>
            {{ item }}
        </li>
    {% endfor %}
    </ul>
    {% else %}
    <p>You didnt order a warranty, so you're on your own when the products inevitably stop working. </p>
    {% endif %}

    <p>Sincerely,<br />{{ company }}</p>
    <p>footer</p>
    {% block footer %}
    <hr>
    <p>Thanks for visiting my site.</p>
    {% endblock %}
</body>
</html>
{% endraw %}
```

看出，模板是基于 Html 的，事实上它就是保存成一个 .html 文件，它跟我们所看到的 html 的区别就在于多了一些由 ```{% raw %}{{ }}{% endraw %}```括起来的变量以及由```{% raw %}{% %}{% endraw %}```括起来的模板标签，此外变量还通过过滤器 ```|```来对文本输出格式进行转换。而这里 ```{% raw %}{{ }}{% endraw %}``` 里的变量相当于一个形参，真正显示出来的是在我们渲染模板的 Python 文件里所传给它的值。

比如在下面的模板渲染代码里

```python
c = Context({'person_name':'LexusLee',
			'company': 'UESTC',
			'ship_date':datetime.date(2015,09,24),
			'ordered_warranty': False})
```

那么模板中的 ```person_name``` 最终显示的就是 ```LexusLee``` 。

## 模板标签

1. ```{% raw %}{% if variable %} {% else %} {% endif %}{% endraw %}```用于判断变量 variable 是否为真，为真则执行 else 标签前的内容，否则执行 else 便签内的内容，跟大部分编程语言中的条件语句用法一致。
2. 同理```{% raw %}{% for %} {% endfor %}{% endraw %}```的用法也和大部分编程语言中循环语句的用法一致。需要注意的是，每个 for 循环中还有一个成为```{% raw %}{% forloop %}{% endraw %}```的模板变量，这个变量能提示一些循环进度信息相关的属性，关于这个变量的详细统发可以参照[这一节](http://djangobook.py3k.cn/appendixF/)。
3. ```{% raw %}{% block content %} {% endblock %}{% endraw %}```是用来处理[模板继承](http://djangobook.py3k.cn/appendixF/)和重载的标签，来避免重复和冗余的代码。比如上述的实例模板( base.html )中，我希望在多个文件中都能显示 footer ，而不需要重复编码，故在该模板中写了```{% raw %}{% block footer %}{% endraw %}```,而在另一个文件中只需要写
``` html
{% raw %}
{% extends "base.html" %}
{% block footer %}
<a href="https://github.com/thehackercat">Github</a>
{% endblock %}
{% endraw %}
```
这样所有的 footer 中都会有
``` html
<a href="https://github.com/thehackercat">Github</a>
```
这行代码。而之前```{% raw %}{% block footer %} {% endblock %}{% endraw %}```框中的代码将会被 overwrite ，也就是说**对于重载模块，子模板可以重载这些部分，如果子模板不重载这些部分，则会按照默认的内容显示**。

4.```{% raw %}{% load staticfiles %}{% endraw %}```用来加载静态资源，比如加载 CSS 、 JS 等静态文件时会用到。

5.```{% raw %}{# #}{% endraw %}``` 用于注释。

