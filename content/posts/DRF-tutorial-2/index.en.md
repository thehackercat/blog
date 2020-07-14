---
layout: post
title: "Django REST Framework 2-请求和响应"
date: 2016-04-02 02:01:33 +0800
comments: true
weight: 5
draft: false
categories: ["Coding"]
tags: ["Python", "Django", "RESTful"]
lightgallery: true
---

## 请求与响应

### 请求对象

DRF 提供了一个 ```request``` 对象，它继承自 ```HttpRequest``` 并且提供了更丰富的对 request 的解析处理的方法。其中最核心的是 ```request``` 对象的 ```request.data``` 属性，它看起来和 Django 的```request.POST``` 相似，但是在处理 Web API 上更强大些。

```python
request.POST  # Only handles form data.  Only works for 'POST' method.
request.data  # Handles arbitrary data.  Works for 'POST', 'PUT' and 'PATCH' methods.
```

```request.data``` 相比于 ```request.POST``` 能够处理 api 中的 「POST」、「PUT」、「PATCH」等请求。

### 返回对象

DRF 也提供了一个 ```response``` 对象，它能把未 render 的对象(数据)通过一定方式转化为正确的数据格式返回给客户端。

```python
return Response(data)  # Renders to content type as requested by the client.
```

### 状态码

如果单独使用 Http 状态码的话代码会很难度，比如像我这种万年记不住几个很奇怪的状态码的人，在看到它们的时候还要 google 这就很伤！所以 DRF 提供了一个可读性更好的状态码标识，比如 `HTTP_400_BAD_REQUEST`  ，是不是一下就看出来这是 bad request 了。这些状态码都封装在了 ```status``` 模块里，使用它们比使用纯数字的 Http 状态码更安逸。

### 封装的 API views

DRF 提供了两个封装好的 API views

1.  `@api_view`  这个装饰器用于基于函数的视图
2.  `APIView`  这个类用于基于类的视图

这两个 views 提供了一些函数如确保在视图中接收到 ```request``` 实例和自动在 ```response``` 对象中添加 context 使其能够被 render 。

### 开始撸代码吧

紧接着[上节的教程]()我们要在 views 中添加一些新功能

先把 ```JSONResponse``` 扔掉，这东西太难用了，我们不再需要它。

接着在 ```douban/views.py``` 中加入以下代码:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from douban.models import Movies
from douban.serializer import MoviesSerializer

@api_view(['GET', 'POST'])
def movies_list(request):
    """
    罗列出所有的 Movies 或者 能新建一个 Movies
    """
    if request.method == 'GET':
        movies = Movies.objects.all()
        serializer = MoviesSerializer(movies, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = MoviesSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def movies_detail(request, pk):
    """
    展示\更新或删除一个 Movies
    """
    try:
        movies = Movies.objects.get(pk=pk)
    except Movies.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = MoviesSerializer(movies)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = MoviesSerializer(movies, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        movies.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

用上了 @api_view 后代码比之前更简洁了。

需要注意的是: 我们不再指明 request 和 response 中的内容类型。 request.DATA 即可用来处理 json 数据类型类型, 也可以处理 yaml 或其他数据类型。我们只需要在 response 中指定要返回的数据， DRF 能根据不同情况，自动在 response 中呈现正确的数据类型。

### 在 URLs 中添加可选后缀

现在我们的 response 对象不是像教程1中的对数据类型进行强制要求了。

并且对 url 也不是硬连接的。

那么就可以定制可选的 url 后缀，如:

通过 [http://example.com/api/items/4/.json](http://example.com/api/items/4/.json) 来访问 Web API。

我们所需做的就是在 views 中添加 ```format``` 关键字:

```python
def movies_list(request, format=None):
```

还有

```python
def movies_detail(request, pk, format=None):
```

然后在 ```douban/urls.py``` 中加入 ```format_suffix_patterns``` :

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from douban import views

urlpatterns = [
    url(r'^dbmovies/$', views.movies_list),
    url(r'^dbmovies/(?P<pk>[0-9]+)/$', views.movies_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

不过，一般情况下，我们耶不会用到那么奇葩的 url 访问方式，以上的例子只是说明了用奇葩的 url 方式也是可以访问的 ：D

### 再测试下我们的 API

在终端中输入

```bash
$ python manage.py runserver
```

接着来浏览器中访问 ```http://127.0.0.1/dbmovies/```

![apitest2](http://7xse6j.com1.z0.glb.clouddn.com/apitest2.png)

如果出现如图所示的 api 则说明 Web api 返回成功。

然后我们可以在这个页面中 POST 一个新的 Movies :

在表单中选择 Media type 为 json 格式并输入

```json
{
    "id": 3,
    "title": "Carol",
    "year": "2015",
    "country": "US",
    "type": "Romance",
    "rating": "8.3"
}
```

![apipost](http://7xse6j.com1.z0.glb.clouddn.com/apitest3.png)

如果返回如下图所示，则说明 POST 成功！

![postsuccess](http://7xse6j.com1.z0.glb.clouddn.com/apitest4.png)

你或许会注意到，每个访问这个页面的人都能 POST 一个新的 Movies ，这是不合理的，所以需要赋予权限，这个我们日后再说。

### 可浏览性

Because the API chooses the content type of the response based on the client request, it will, by default, return an HTML-formatted representation of the resource when that resource is requested by a web browser. This allows for the API to return a fully web-browsable HTML representation.

Having a web-browsable API is a huge usability win, and makes developing and using your API much easier. It also dramatically lowers the barrier-to-entry for other developers wanting to inspect and work with your API.

See the [browsable api](http://www.django-rest-framework.org/topics/browsable-api/) topic for more information about the browsable API feature and how to customize it.

