---
layout: post
title: "Django REST Framework 5-关联性和超链接"
date: 2016-04-06 02:01:33 +0800
comments: true
weight: 5
draft: false
categories: ["Code", "Python"]
tags: ["Python", "Django", "RESTful"]
lightgallery: true
---

## 关联性和超链接

之前我们的 api 都是用外键关联，然而实际上用超链接的方式更符合 RESTful 的思想。

所以在这一章中我们将要用超链接(代替外键的方式)来提高关联性。

### 为 api 提供根路径

由于要采用超链接的方式，而之前我们的 'movies' / 'directors' / 'users' 虽然有了 endpoints ，但 api 本身却没有一个整体的根路径，所以我们使用 ```@api_view``` 装饰器来创建一个根路径。

在 ```douban/views.py``` 中添加如下代码:
<!--more-->

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.reverse import reverse

# api 根目录
@api_view(['GET'])
def api_root(request, format=None):
    return Response({
        'user': reverse('user-list', request=request, format=format),
        'movies': reverse('movies-list', request=request, format=format),
        'director': reverse('director-list', request=request, format=format)
    })
```

在这里需要注意两样东西：

1. 我们用了 DRF 的 ```reverse``` 方法而不是 Django 自带的 ```reverse``` 方法来返回一个正确的 URLs。
2. 此时如果打开 Web api 界面会报错, 因为我们还没有为 url 进行绑定， 稍后我们会添加。

接着在 ```doubt/urls.py``` 中添加对应路径

```python
url(r'^$', views.api_root),
```

### 使用炫酷的超链接

DRF 提供了以下几种方式来处理实体间的关系:

- 主键
- 超链接
- 相关项使用单一标识符
- 相关项默认文本信息
- 子项在母项中显示出来
- 其他方式

在这个栗子中我们使用超链接的方式来处理实体关系。

首先在序列器中使用 ```HyperlinkedModelSerializer``` 替代 ```ModelSerializer```

注:  ```HyperlinkedModelSerializer```  和 ```ModelSerializer``` 有以下几点区别:

- 它没有主键域 ( pk field )
- 它默认包含一个 url 域
- 关联时使用的是 ```HyperlinkedRelatedField``` 而不是 ```PrimaryKeyRelatedField```

在 ```doubt/serializer.py``` 中进行如下改写:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from django.contrib.auth.models import User
from rest_framework import serializers
from douban.models import Movies, celebrity, COUNTRY_CHOICES, TYPE_CHOICES

class MoviesSerializer(serializers.HyperlinkedModelSerializer):
    owner = serializers.ReadOnlyField(source='owner.username')
    director = serializers.HyperlinkedRelatedField(many=False, queryset=celebrity.objects.all(), view_name='director-detail')
    class Meta:
        model = Movies
        fields = ('id', 'title', 'director', 'year', 'country', 'type', 'rating', 'owner')

class UserSerializer(serializers.HyperlinkedModelSerializer):
    movies = serializers.HyperlinkedRelatedField(many=True, view_name='movies-detail', read_only=True)

    class Meta:
        model = User
        fields = ('id', 'username', 'movies')

class DirectorSerializer(serializers.HyperlinkedModelSerializer):
    movies = serializers.HyperlinkedRelatedField(many=True, view_name='movies-detail')
    class Meta:
        model = celebrity
        fields = ('id', 'name', 'age', 'gender', 'movies')
```

### 绑定 url

使用超链接 api 有个前提条件，我们需要确保 URL pattern 都已命名。

编辑 ```douban/urls.py``` 进行如下修改:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from douban import views

urlpatterns = format_suffix_patterns([
    url(r'^$', views.api_root),
    url(r'^movies/$', views.MoviesList.as_view(), name='movies-list'),
    url(r'^movies/(?P<pk>[0-9]+)/$', views.MoviesDetail.as_view(), name='movies-detail'),
    url(r'^users/$', views.UserList.as_view(), name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view(), name='user-detail'),
    url(r'^directors/$', views.DirectorList.as_view(), name='director-list'),
    url(r'^directors/(?P<pk>[0-9]+)/$', views.DirectorDetail.as_view(), name='director-detail'),
])

```

确保每个 URL pattern 都正确的与 ```views.py``` 中对应视图的命名进行绑定。

### 增加分页

对于大量的数据在单页显示体验很不好，所以要设置分页。

编辑 ```restapit/settings.py``` :

```python
REST_FRAMEWORK = {
    'PAGE_SIZE': 10
}
```

现在用浏览器访问我们的 api 界面，不断地添数据，就可以看到分页效果辣。

### 为什么使用超链接

因为用超链接的方式有个明确的指向，比如该栗子中 movies 的 director 字段由外键变为超链接的关联形式允许直接跳转到 director 的 api 页面。



