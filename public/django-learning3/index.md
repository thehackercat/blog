#  Django 学习笔记3-- Models 

## MTV vs MVC

正如在之前[这篇文章](http://thehackercat.me/blog/2015/11/14/django-learning1/)所提到的， 把数据存取逻辑、业务逻辑和表现逻辑组合在一起的概念有时被称为软件架构的 Model-View-Controller ( MVC )模式。 在这个模式中， Model 代表数据存取层，View 代表的是系统中选择显示什么和怎么显示的部分，Controller 指的是系统中根据用户输入并视需要访问模型，以决定使用哪个视图的那部分。

<!--more-->

而 Django 使用的更多的则是模型( Model )、模板( Template )和视图( Views )的软件设计模式，称为 MTV 模式。我在 Stack Overflow 的[这个回答](http://stackoverflow.com/questions/6621653/django-vs-model-view-controller)里找到了对于 MTV vs MVC 两种设计模式间的微妙的差别。

其中提到，不能简单的把 Django 视图认为是 MVC 控制器，把 Django 模板认为是 MVC 视图。

两者之间的差别在于，在 Django 中，视图( Views )不处理用户输入，而是用来选择要展示的哪些数据，而不是要如何展示数据。而 Django 模板 仅仅决定如何展现Django视图指定的数据。

或者说, Django 将 MVC 中的视图进一步分解为 Django 视图 和 Django 模板两个部分，分别决定 “展现哪些数据” 和 “如何展现”，使得 Django 的模板可以根据需要随时替换，而不仅仅限制于内置的模板。至于 MVC 控制器部分，由 Django 框架的 URLconf 来实现。

## 模型练手

为了深入了解 Django Models 对数据的操作，我写了一个简单的博客模型作为练手。

在新建模型时遇到了一个 App migrations 问题如下：

![Model CommandError](http://thehackercat-hackercat.stor.sinaapp.com/model%20error%201.png)

后来发现是由于 Django 版本问题，在最近版本把 migrations 移出了所创建的 App 的根目录，只需要执行```python manage.py makemigration```接着再执行```python manage.py migrate```即可解决。

写了个简单的博客的增删改查，代码如下：
``` python #view.py
#!/usr/bin/python
#-*-coding:utf-8 -*-

from django.shortcuts import render
from django.http import HttpResponseRedirect
from blog.models import Blog
from blog import forms
from django.template import RequestContext

def blog_list(request):
    blog_list = Blog.objects.all()
    return render(request,"blog_list.html",{'blog_list':blog_list})

def blog_form(request):
    if request.method == 'POST':
        form = forms.BlogForm(request.POST)

        if form.is_valid():
            data = form.cleaned_data

            if 'id' not in data:
                blog = Blog(title=data['title'],author=data['author'],content=data['content'])
                blog.save()
            else:
                blog = Blog.object.get(id=data.id)
                blog.title = data['title']
                blog.author = date['author']
                blog.content = data['content']
                blog.save()
            return HttpResponseRedirect('/blog/list')
        else:
            form = forms.BlogForm()
            return render(request,"blog_form.html",context_instance=RequestContext(request))

def blog_del(request):
    errors = []
    if 'id' in request.GET:
        bid_ = request.GET['id']
        Blog.objects.fileter(id=bid_).delete()
        return HttpResponseRedirect('/blog/list')

def blog_view(request):
    if 'id' in request.GET:
        bid_ = request.GET['id']
        blog = Blog.object.get(id=bid_)
        form = forms.BlogForm(
            initial = {'id':blog.id,'title':blog.title,'author':blog.author,'content':blog.content}
        )
        return render(request,"blog_form.html",{'form':form},context_instance=RequestContext(request))
    else:
        errors.append("参数异常请刷新后重试")
        return render(request,"blog_list.html",{'errors':errors})

def blog_edit(request):
    errors = []
    if 'id' in request.GET:
        bid_ = request.GET['id']
        blog = Blog.objects.get(id=bid_)
        form = forms.BlogForm(
                initial = {'id':blog.id,'title':blog.title,'author':blog.author,'content':blog.content}
        )
        return render_to_response("blog_form.html",{'form':form},context_instance=RequestContext(request))
    else:
        errors.append("参数异常请刷新后重试")
        return render(request,"blog_list.html",{'errors':errors})
```


``` python #form.py
#!/usr/bin/python
#-*-coding:utf-8 -*-
from django import forms

class BlogForm(forms.Form):
    title =  forms.CharField(label='标题')
    author = forms.CharField(label='作者')
    content = forms.CharField(label='正文',widget=forms.Textarea)
```
其中 CharField() 相当于赋予了 title 表段 varchar 的属性。

object.all() 相当于执行了一条```select * from blog```的 sql 语句。

object.get() 相当于执行了一条```select * from blog where id='bid_'```的获取单个对象的 sql 语句。

object.save() 相当于执行了```UPDATE blog SET ... ```的 sql 语句。

并用 errors[] 列表来捕捉错误信息，一般防止出现错误的 sql 语句时增加了 blog 表段中的 id 号而其余属性值为空的情况。

感觉相比于 ThinkinPHP 操作表单 GET/POST 请求以及处理数据库方面要方便得多。

## 疑惑

MVC 框架大大缩小了开发者对数据存储的直接操作，框架自动生成 sql 语句并空值数据的存取等。以后写 sql 感觉就跟 Excel 一样了，那应该怎么优化 sql 呢。

顺便吐槽一下，最近 GitHub Repositorie 换新的布局，天热噜，怎么能这么丑！

![Dont learn CS](http://thehackercat-hackercat.stor.sinaapp.com/dontstudyCS.jpg)

