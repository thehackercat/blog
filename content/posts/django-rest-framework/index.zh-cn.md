---
title: "Django REST Framework 快速上手"
date: 2016-03-29 15:43:36 +0800
comments: true
weight: 5
draft: false
categories: ["Code", "Python"]
tags: ["Python", "Django"]
lightgallery: true
---
## Django REST Framework 快速上手
### 背景
这几天正好在研究 RESTful 的方式来写 API，然后上手 Django REST 框架。

Django REST Framework (以下简称 DRF )是一个轻量级的库，熟悉 Django 的话可以很容易的用它来构建 Web API。
<!--more-->

### 安装前提

Django REST Framework 安装需要以下前提:

- Python (2.7, 3.2, 3.3, 3.4, 3.5)
- Django (1.7+, 1.8, 1.9)

我自己的环境是:

- Python 2.7.10
- Django 1.8.2


### 安装配置

安装 DRF 需要用到 ```pip``` 命令

```shell
pip install djangorestframework
pip install markdown	# Markdown support for the browsable API.
pip install django-filter	# Filtering support
```

或者在 GitHub 上 clone 它
```shell
git clone git@github.com:tomchristie/django-rest-framework.git
```

接着在 Django Project 根目录的 ```setting.py``` 文件中的 ```INSTALLED_APPS``` 加入 ```'rest_framework'```

``` Python
INSTALLED_APPS = (
    ...
    'rest_framework',
)
```

如果你要使用 DRF 的 browsable API 的话，你可能还需要添加 REST 框架的登录登出视图 ( views )，辣么需要在 ```url.py``` 文件中加入以下代码:

```python
urlpatterns = [
    ...
    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

注: 这个 URL 地址可以是任意的，但是必须 include ```'rest_framework.urls'``` 和 ```namespace='rest_framework'``` 。

### 举个栗子
现在我们来看一下一个简单的用 DRF 来构建一个模型支持较好的 API 的栗子。

任何一个对 REST 框架的全局设置都被放在 ```REST_FRAMEWORK``` 的模块内，所以你需要在 ```settings.py``` 文件中添加以下代码来通过 ```REST_FRAMEWORK``` 入口进行全局设置:

``` python
REST_FRAMEWORK = {
    # Use Django's standard `django.contrib.auth` permissions,
    # or allow read-only access for unauthenticated users.
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'
    ]
}
```

现在我们可以构建 API 了，编辑 Django 项目根目录的 ```url.py``` 文件:

``` python
from django.conf.urls import url, include
from django.contrib.auth.models import User
from rest_framework import routers, serializers, viewsets

# Serializers define the API representation.
class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ('url', 'username', 'email', 'is_staff')

# ViewSets define the view behavior.
class UserViewSet(viewsets.ModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer

# Routers provide an easy way of automatically determining the URL conf.
router = routers.DefaultRouter()
router.register(r'users', UserViewSet)

# Wire up our API using automatic URL routing.
# Additionally, we include login URLs for the browsable API.
urlpatterns = [
    url(r'^', include(router.urls)),
    url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```
解释一下，

每个 ```xxxSerializer``` 都要继承 ```ModelSerializer``` 来选择模型和模型字段。

UserSerializer 类继承了更符合 RESTful 设计的 ```HyperlinkedModelSerializer``` 超链接模型 Serializer 类，它和普通的 ```ModelSerializer``` 类有以下区别:

- 缺省状态下不包含 pk 字段
- 具有一个 url 字段，即HyperlinkedIdentityField类型
- 用HyperlinkedRelatedField表示关系，而非PrimaryKeyRelatedField

然后在 ```class Meta``` 中选择模型和要展现的模型元素

```ViewSet``` 用来定义 View 的行为，和 Django 的 views 类似，用来处理 API 的 read 、write、 update 等方法(而 Django views 则处理 http 的 GET 和 POST )

在 ViewSet 实例化之后，通过 ```Router``` 类，最终将 URL 和 ViewSet 方法绑定起来。

ok，现在你可以通过在浏览器中访问 ```http://127.0.0.1:8000/``` 来查看你的 'users' API 了。
