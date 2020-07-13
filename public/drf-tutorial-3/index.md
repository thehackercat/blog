# Django REST Framework 3-基于类的视图


## 基于类的视图

基于类的视图比先前基于函数的视图的可重用性更强，可以更多快好省地 ( [DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself) )地写出简洁的代码。

### 把 API 用基于类的视图的方式重写

编辑 ```douban/views.py``` 进行如下重写

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-
from douban.models import Movies
from douban.serializer import MoviesSerializer
from django.http import Http404
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status


class MoviesList(APIView):
    """
    罗列出所有的 Movies 或者 能新建一个 Movies
    """
    def get(self, request, format=None):
        movies = Movies.objects.all()
        serializer = MoviesSerializer(movies, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = MoviesSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class MoviesDetail(APIView):
    """
    展示\更新或删除一个 Movies
    """
    def get_object(self, pk):
        try:
            return Movies.objects.get(pk=pk)
        except Movies.DoesNotExist:
            raise Http404

    def get(self, request, pk, format=None):
        movies = self.get_object(pk)
        serializer = MoviesSerializer(movies)
        return Response(serializer.data)

    def put(self, request, pk, format=None):
        movies = self.get_object(pk)
        serializer = MoviesSerializer(movies, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        movies = self.get_object(pk)
        movies.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

并更新 ```douban/urls.py```

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-

from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from douban import views

urlpatterns = [
    url(r'^dbmovies/$', views.MoviesList.as_view()),
    url(r'^dbmovies/(?P<pk>[0-9]+)/$', views.MoviesDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

重写完毕！

### 使用 Mixins

使用基于类的视图的一大好处是，我们可以使用各种 mixins

DRF 为我们提供了许多现成的 mixins ，方便我们像使用 model-backed API 一样构建 "创建/获取/更新/删除" API. 我们试着使用 Mixins 改写原先的 views

GenericAPIView 为我们提供了 views 核心的功能, 而 ListModelMixin 和 CreateModelMixin 为我们提供了 .list() 和 .create() 功能，我们将这些功能与 http 动作的 GET 和 POST 相绑定:

```python
from douban.models import Movies
from douban.serializer import MoviesSerializer
from rest_framework import mixins
from rest_framework import generics

class MoviesList(mixins.ListModelMixin,
                  mixins.CreateModelMixin,
                  generics.GenericAPIView):
    queryset = Movies.objects.all()
    serializer_class = MoviesSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

同样的, 我们使用GenericAPIView, RetrieveModelMixin, UpdateModelMixin和DestroyModelMixin改写views.py:

```python
class MoviesDetail(mixins.RetrieveModelMixin,
                    mixins.UpdateModelMixin,
                    mixins.DestroyModelMixin,
                    generics.GenericAPIView):
    queryset = Movies.objects.all()
    serializer_class = MoviesSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
```

可看出，这三个 Mixin 分别对应 GET/PUT/DELETE 动作。

### 使用通用类视图

使用 Mixin 来重写 views 减少了代码量，但是还可以更少！

那就是使用「通用类视图」—「generic class based views」

同 Django 一样，DRF为我们提供了现成的通用类视图，接下来我们使用这些通用类视图再一次修改原有的 ```views.py``` :

```python
from douban.models import Movies
from douban.serializer import MoviesSerializer
from rest_framework import generics

class MoviesList(generics.ListCreateAPIView):
    queryset = Movies.objects.all()
    serializer_class = MoviesSerializer

class MoviesDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Movies.objects.all()
    serializer_class = MoviesSerializer
```

这样，代码已经非常的精简了，不过坏处在于，你不知道他具体执行了什么。

