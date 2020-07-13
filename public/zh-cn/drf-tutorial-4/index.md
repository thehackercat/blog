# Django REST Framework 4-验证和授权


## 验证与授权

目前来看，我们的 API 并没有权限上的限制(即任何人都可以编辑或删除我们的 Movies )，这不是我们想要的。所以我们需要在 API 上做些限制以确保:

- Movies 与 Users 关联起来。
- 只有授权了的用户才能创建新的 Movies。
- 只有 Movies 的创建者才可以更新或删除它。
- 未授权的用户只能进行查看。

### 在 models 中增加以下信息

我们先把之前注释掉的

```python
director = models.ForeignKey('celebrity', related_name='Movies')

class celebrity(models.Model):
    name = models.CharField(max_length=100, blank=True, default='')
    age = models.IntegerField()
    gender = models.CharField(choices=GENDER_CHOICES, default='male', max_length=20)

```

关联导演类的注释解开，来看看多张表在生成的 api 里的关联性。

接着在 ```models.py``` 中的 Movies 类中加入以下代码来确定 Movies 的创建者:

```python
owner = models.ForeignKey('auth.User', related_name='Movies')
```

最后 ```models.py``` 代码为:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from django.db import models

# 举个栗子
COUNTRY_CHOICES = (
    ('US', 'US'),
    ('Asia', 'Asia'),
    ('CN', 'CN'),
    ('TW', 'TW'),
)
TYPE_CHOICES = (
    ('Drama', 'Drama'),
    ('Thriller', 'Thriller'),
    ('Sci-Fi', 'Sci-Fi'),
    ('Romance', 'Romance'),
    ('Comedy', 'Comedy')
)
GENDER_CHOICES = (
    ('male', 'male'),
    ('female', 'female')
)

class Movies(models.Model):
    title = models.CharField(max_length=100, blank=True, default='')
    year = models.CharField(max_length=20)
    # 在 director 关联了 Movies 类 和 celecrity 类, 在第4章会用到 celebrity 类
    director = models.ForeignKey('celebrity', related_name='movies')
    # 关联 User 类来确定 Movies 的创建者
    owner = models.ForeignKey('auth.User', related_name='movies')
    country = models.CharField(choices=COUNTRY_CHOICES, default='US', max_length=20)
    type = models.CharField(choices=TYPE_CHOICES, default='Romance', max_length=20)
    rating = models.DecimalField(max_digits=3, decimal_places=1)
    created = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ('created',)

class celebrity(models.Model):
    name = models.CharField(max_length=100, blank=True, default='')
    age = models.IntegerField()
    gender = models.CharField(choices=GENDER_CHOICES, default='male', max_length=20)
```

修改完了模型，我们需要更新一下数据表。

通常来讲，我们会创建一个数据库 migration 来更新数据表，但是为了图省事儿，宝宝我索性删了整张 Movies 表直接重建！

在数据库中删除 douban_movies 表后在终端中执行以下命令:

```bash
$ python manage.py syncdb
```

接着我们可能会需要多个 User 来测试 API ，如果之前你没有创建 Django Super User 的话，用以下命令创建:

```bash
$ python manage.py createsuperuser
```

然后进入 ```http://127.0.0.1/admin/``` 界面，登录并找到  ```/user/``` 表，然后在里面手动创建 user 并赋予权限。

### 为新增的模型增加 endpoints

既然现在我们已经有了 users 模型和 celebrity 模型，那么现在需要做的就是在 ```serializer.py``` 中让他们在 API 中展现出来，加入以下代码:

```python
class UserSerializer(serializers.ModelSerializer):
    movies = serializers.PrimaryKeyRelatedField(many=True, queryset=Movies.objects.all())

    class Meta:
        model = User
        fields = ('id', 'username', 'movies')

class DirectorSerializer(serializers.ModelSerializer):
    movies = serializers.PrimaryKeyRelatedField(many=True, queryset=Movies.objects.all())

    class Meta:
        model = celebrity
        fields = ('id', 'name', 'age', 'gender', 'movies')
```

因为我们之前在 ```models.py``` 中添加了 ```owner = models.ForeignKey('auth.User', related_name='movies')``` 其中 ```related_name``` 设置了可以通过 User.movies 来逆向访问到 movies 表。所以在 ```ModelSerializer``` 类中我们需要在 fields 中添加一个 ```movies``` 来实现逆向访问。同理 ```DirectorSerializer``` 类中也进行相应修改。

接着，我们还需要在 ```views.py``` 中添加相应的视图。

为 User 添加只读 API ，使用 ```ListAPIView``` 和 ```RetrieveAPIView```

为 Director 添加读写 API ，使用 ```ListCreateAPIView``` 和 ```RetrieveUpdateDestroyAPIView```

```python
class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer

class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer

class DirectorList(generics.ListCreateAPIView):
    queryset = celebrity.objects.all()
    serializer_class = DirectorSerializer

class DirectorDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = celebrity.objects.all()
    serializer_class = DirectorSerializer
```

最后，修改 ```urls.py``` 把视图关联起来，在 ```urlpatterns``` 中加入以下4个 patterns:

```python
urlpatterns = [
    url(r'^users/$', views.UserList.as_view()),
    url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()),
    url(r'^directors/$', views.DirectorList.as_view()),
    url(r'^directors/(?P<pk>[0-9]+)/$', views.DirectorDetail.as_view()),
]
```

### 把 Movies 和 Director 、 User 关联起来

现在，如果我们新建一部 movie ，那它和 director 还有 user 是没有关联的，因为 director 和 user 信息是通过 request 接收到的，而不是通过序列器接收的，这意味着，数据库中收到 director 和 user 信息是没有(和 movies 存在)外键关系的。

而要让他们发生关系 ，我们的做法是在视图中重写 ```.perform_create()``` 方法。

```.perform_create()``` 方法允许我们处理 request 或 requested URL 中的任何信息。

在 ```MoviesList``` 和 ```MoviesDetail``` 中添加以下代码:

```python
def perform_create(self, serializer):
    serializer.save(owner=self.request.user)
```

这样 ```create()``` 方法就能够在接收到 request.data 时将其传回给序列器里的 owner 和 director 了。

### 更新序列器

在视图中重写了 ```.perform_create()``` 方法后还需要更新下序列器才能实现他们之间的关联，在 ```serializer.py``` 中的 ```MoviesSerializer``` 类添加以下代码:

```python
owner = serializers.ReadOnlyField(source='owner.username')
director = serializers.CharField(source='celebrity.name')
```

接着在 ```class Meta``` 的 fields 中加入 owner 和 director :

```python
class Meta:
	model = Movies
    fields = ('id', 'title', 'director', 'year', 'country', 'type', 'rating', 'owner')
```

```source``` 关键字负责控制在 fields 中展现的数据的源，它可以指向这个序列器实例的任意一个属性。

对 owner 属性，我们用的是 ```ReadOnlyField``` 在确保它始终是只读的，我们也可以用 ```CharField(read_only=True)``` 来等效替代，但是我嫌它太长了，其余的 Field 还有诸如 ```CharField``` 、 ```BooleanField``` 等，你可以在 [「这里」](http://www.django-rest-framework.org/api-guide/fields/)查到。

### 添加权限

我们希望授权的用户才能新建、更新和删除 movies，所以需要添加权限管理的功能。

DRF 包含了一系列的 permission 类来实现权限管理，你可以在[「这里」](http://www.django-rest-framework.org/api-guide/permissions/) 查到。

在这个栗子中，我们使用  `IsAuthenticatedOrReadOnly` 来确保授权的请求得到读写的权限，未授权的请求只有只读权限。

首先，在 ```views.py``` 中 import 以下模块:

```python
from rest_framework import permissions
```

接着，在 ```MoviesList``` 和 ```MoviesDetail``` 中加入以下代码:

```python
permission_classes = (permissions.IsAuthenticatedOrReadOnly,)
```

### 添加可浏览的授权 api

如果你在浏览器中访问我们的 api Web 界面，你会发现我们没法创建新的 movies 了，因为在上一步我们设置了权限管理。

所以我需要在浏览器中添加用户登录来实现带界面的权限管理。(之所以说带界面是因为可以在终端中直接使用 httpie 来访问 api )

在 ```restapi/urls.py``` 中加入以下代码:

```python
urlpatterns += [
    url(r'^api-auth/', include('rest_framework.urls',
                               namespace='rest_framework')),
]
```

这样通过在浏览器中访问 Web api 界面就能在右上角发现一个登录按钮，进行登录授权了。

### 对象级权限

之前提到要使 movies 可以被任何人访问，但是只能被创建者编辑，所以需要赋予其游客访问的权限以及创建者编辑权限。

下面我们新建一个 ```permissions.py``` 来详细解决这个权限问题:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-
from rest_framework import permissions


class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    游客访问权限及创建者编辑权限
    """

    def has_object_permission(self, request, view, obj):
        # 游客权限
        if request.method in permissions.SAFE_METHODS:
            return True

        # 编辑权限
        return obj.owner == request.user
```

修改 ```views.py``` 中 ```MoviesDetail``` 的 ```permission_class``` :

```python
from douban.permissions import IsOwnerOrReadOnly

permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                      IsOwnerOrReadOnly,)
```

终于，我们完成了整个 api 授权的过程！

