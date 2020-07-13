# Django REST Framework 1-序列化


## 序列化

### 环境搭建

首先我们先新建一个 ```restapi```  项目并安装上 django-rest-framework (DRF) 环境

```bash
$ pip install djangorestframework
$ python manage.py startnewproject restapi
$ cd restapi
$ python manage.py startnewapp douban
```

接着，我们需要在 ```setting.py``` 里的加入如下代码:

```python
INSTALLED_APPS = (
    ...
    'rest_framework',
    'douban',
)
```

### 建立模型

由于我炒鸡喜欢看电影，所以仿着  ```douban-API``` 来做个简易的豆瓣电影的 rest-api 。

所以我们就用这个「仿豆瓣电影 api 」来作为栗子开始教程吧！

编辑 ```douban/models.py``` 文件并加入以下代码:

``` python
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
    ('Romance', 'Romance' ),
    ('Comedy', 'Comedy')
)
GENDER_CHOICES = (
    ('male', 'male'),
    ('female', 'female')
)

class movies(models.Model):
    title = models.CharField(max_length=100, blank=True, default='')
    year = models.CharField(max_length=20)
    # 在 director 关联了 movies 类 和 celecrity 类, 在第4章会用到 celebrity 类
    # director = models.ForeignKey('celebrity', related_name='movies')
    country = models.CharField(choices=COUNTRY_CHOICES, default='US', max_length=20)
    type = models.CharField(choices=TYPE_CHOICES, default='Romance', max_length=20)
    rating = models.DecimalField(max_digits=3, decimal_places=1)
    created = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ('created',)

# class celebrity(models.Model):
#     name = models.CharField(max_length=100, blank=True, default='')
#     age = models.IntegerField()
#     gender = models.CharField(choices=GENDER_CHOICES, default='男', max_length=20)
```

接着在终端中运行:

```bash
$ python manage.py makemigrations douban
$ python manage.py migrate
$ python manage.py syncdb
```

来创建一个新的 migrations 并在数据库中生成表。

### 创建序列化类

在开始构建 Web API 时，我们首先要做的就是提供对 ```movies``` 实例的序列化和反序列化(即对序列化后的实例进行「解码」)，这样才能生成可供浏览的 ```json``` 格式的 api 。我们可以通过声明「序列器」(一个和 Django 表单十分类似的玩意儿)来做到这一点。

在 ```restapi``` 目录中创建一个 ```serializer.py``` 文件，加入以下代码:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from rest_framework import serializers
from douban.models import movies, COUNTRY_CHOICES, TYPE_CHOICES

class MoviesSerializer(serializers.Serializer):
    pk = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    year = serializers.CharField(max_length=20)
    country = serializers.ChoiceField(choices=COUNTRY_CHOICES, default='US')
    type = serializers.ChoiceField(choices=TYPE_CHOICES, default='Romance')
    rating = serializers.DecimalField(max_digits=3, decimal_places=1)

    def create(self, validated_data):
        """
        根据接收到的 validated_data 创建一个 movies 实例
        """
        return movies.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        根据接收到的 validated_data 更新并返回一个 movies 实例
        """
        instance.title = validated_data.get('title', instance.title)
        instance.year = validated_data.get('year', instance.year)
        instance.country = validated_data.get('country', instance.country)
        instance.type = validated_data.get('type', instance.type)
        instance.rating = validated_data.get('rating', instance.rating)
        instance.save()
        return instance
```

序列器的第一个部分定义了要进行序列化/反序列化的字段。

```create()``` 和 ```update()``` 方法定义了符合规范的 movies 实例的创建和更新的方法。

序列器非常类似于 Django ```Form``` 表单，它包含了几种对字段常见的验证标识符，如 ```required``` 、 ```max_length``` 、 ```default``` 等。这些标识符实现的功能类似于 Django 表单，就不详细解释了。

所以序列器实现了以下两个功能:

- 选择相应的模型
- 选择要展现的字段(验证后的)

我们也可以通过使用 ```ModelSerializer``` 多快好省地的构建序列器，这个我们日后再说。

### 开始使用序列器

在开始项目之前，我们先熟悉下序列器，在终端中启动 Django shell :

```bash
$ python manage.py shell
```

输入以下代码来创建2个 Movies 实例

「荒野猎人」和「蝙蝠侠爱上超人」

```python
from douban.models import Movies
from douban.serializer import MoviesSerializer
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser

movies = Movies(title='The Revenant', year='2015', country='US', type='Drama', rating=7.9)
movies.save()

movies = Movies(title='Batman v Superman: Dawn of Justice',  year='2016', country='US', type='Romance', rating=6.7)
movies.save()
```

然后将其中一个实例序列化

```python
serializer = MoviesSerializer(movies)
serializer.data

#{'rating': u'7.9', 'title': u'The Revenant', 'country': 'US', 'year': u'2015', 'pk': None, 'type': 'Drama'}
```

接着我们将以上数据转换为 JSON 格式，实现序列化

```python
content = JSONRenderer().render(serializer.data)
content

#{"pk":null,"title":"The Revenant","year":"2015","country":"US","type":"Drama","rating":"7.9"}'
```

反序列化也类似，通过解析 Python 数据流并将数据流"引入"实例中即可

```python
from django.utils.six import BytesIO

stream = BytesIO(content)
data = JSONParser().parse(stream)
serializer = MoviesSerializer(data=data)
serializer.is_valid()
# True
serializer.validated_data
#OrderedDict([(u'title', u'The Revenant'), (u'year', u'2015'), (u'country', 'US'), (u'type', 'Drama'), (u'rating', Decimal('7.9'))])
```

可见, serializer和django form 有多么相似, 当我们写view时, 这一相似性会更加明显.

当我们输入参数many=True时, serializer还能序列化queryset:

```python
serializer = MoviesSerializer(Movies.objects.all(), many=True)
serializer.data
[OrderedDict([('pk', 1), ('title', u'Batman v Superman: Dawn of Justice'), ('year', u'2016'), ('country', 'US'), ('type', 'Romance'), ('rating', u'6.7')]), OrderedDict([('pk', 2), ('title', u'The Revenant'), ('year', u'2015'), ('country', 'US'), ('type', 'Drama'), ('rating', u'7.9')])]
```

### 使用更高级的 ModelSerializers

接着如果你按照官网的教程走下去，你会发现上面的 ```serializer.py``` 是个代码冗杂的序列器，这不符合 Python 的风格。

所以我们要做的就是简化代码。

DRF 提供了更为简便的 ```ModelSerializer``` 类可以解决这个问题。

所以我们修改之前的 ```serializer.py``` :

```python
class MoviesSerializer(serializers.ModelSerializer):
    class Meta:
        model = Movies
        fields = ('id', 'title', 'year', 'country', 'type', 'rating')
```

这种模式的序列器可以很方便地检查 fields 中的每个字段

然后在终端中打开 Django shell

```bash
$ python manage.py shell
```

输入以下代码

```python
from douban.serializer import MoviesSerializer
serializer = MoviesSerializer()
print(repr(serializer))

#MoviesSerializer():
    id = IntegerField(label='ID', read_only=True)
    title = CharField(allow_blank=True, max_length=100, required=False)
    year = CharField(max_length=20)
    country = ChoiceField(choices=(('US', 'US'), ('Asia', 'Asia'), ('CN', 'CN'), ('TW', 'TW')), required=False)
    type = ChoiceField(choices=(('Drama', 'Drama'), ('Thriller', 'Thriller'), ('Sci-Fi', 'Sci-Fi'), ('Romance', 'Romance'), ('Comedy', 'Comedy')), required=False)
    rating = DecimalField(decimal_places=1, max_digits=3)
```

注: ```ModelSerializer``` 类仅仅是创建 ```serializer``` 类的一个快捷方法，它除了实现以下两种方法外并没有其余的功能:

- 声明需要展现的字段
- 定义  `create()` 和 `update()` 方法

### 使用 Django views 编写序列器视图

为了更好理解序列器，我们不使用 DRF 的其他特性，仅仅用 Django views 模式来编写序列器的视图。

我们会创建一个 HttpResponse 的子类，这样就能将数据以 json 格式返回。

编辑 ```douban/views.py``` 加入以下代码:

```python
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser
from douban.models import Movies
from douban.serializer import MoviesSerializer

class JSONResponse(HttpResponse):
    """
    将数据转为 JSON 格式的 HttpResponse 子类
    """
    def __init__(self, data, **kwargs):
        content = JSONRenderer().render(data)
        kwargs['content_type'] = 'application/json'
        super(JSONResponse, self).__init__(content, **kwargs)
```

讲道理的话，我们 api 的根目录应该能罗列出所有的 Movies 或者 能新建一个 Movies

并且还需要一个用于展示、更新和删除 Movies 的 views

编辑 ```douban/views.py``` 加入以下代码:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser
from douban.models import Movies
from douban.serializer import MoviesSerializer

class JSONResponse(HttpResponse):
    """
    将数据转为 JSON 格式的 HttpResponse 子类
    """
    def __init__(self, data, **kwargs):
        content = JSONRenderer().render(data)
        kwargs['content_type'] = 'application/json'
        super(JSONResponse, self).__init__(content, **kwargs)

@csrf_exempt
def movies_list(request):
    """
    罗列出所有的 Movies 或者 能新建一个 Movies
    """
    if request.method == 'GET':
        movies = Movies.objects.all()
        serializer = MoviesSerializer(movies, many=True)
        return JSONResponse(serializer.data)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = MoviesSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data, status=201)
        return JSONResponse(serializer.errors, status=400)

@csrf_exempt
def movies_detail(request, pk):
    """
    展示\更新或删除一个 Movies
    """
    try:
        movies = Movies.objects.get(pk=pk)
    except Movies.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = MoviesSerializer(movies)
        return JSONResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = MoviesSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data)
        return JSONResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        movies.delete()
        return HttpResponse(status=204)
```

我不是很弄明白这里关掉 csrf 的意义，那不如直接就不用 csrf 不就好了？

不管了，先放着，以后回来看 ( 吐舌头

最后修改 ```douban/url.py``` 导入相应的视图

```python
from django.conf.urls import url
from douban import views

urlpatterns = [
    url(r'^dbmovies/$', views.movies_list),
    url(r'^dbmovies/(?P<pk>[0-9]+)/$', views.movies_detail),
]
```

并在 ```restapi/url.py``` 中 include 一下

```python
from django.conf.urls import url, include

urlpatterns = [
    url(r'^', include('douban.urls')),
]
```

这样 url 和 views 就绑定好了。

### 测试 Web API

在终端中输入

```bash
$ python manage.py runserver
```

接着来浏览器中访问 ```http://127.0.0.1/dbmovies/```

![apitest](http://7xse6j.com1.z0.glb.clouddn.com/apitest.png)

如果出现如图所示的 api 则说明 Web api 返回成功。

(顺便安利一个 chrome 插件 — [FeHelper](https://www.baidufe.com/fehelper) 可以自动格式化 JSON 代码)

