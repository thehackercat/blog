# Django 高级 views 和 URLconf 配置

由于官网教程讲得迷迷糊糊的，所以我提炼了下代码，发现便于理解很多。

## URLconf 技巧
``` python urls.py
# 在模块开始导入关联的视图函数，直接传递函数对象
from django.conf.urls import include, url
from django.contrib import admin
from mysite.views import hello, current_datetime, hours_ahead

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^hello/$', hello),
    url(r'^time/$', current_datetime),
    url(r'^time/plus/(\d{1,2})/$', hours_ahead),
]
```
<!--more-->

``` python urls.py
# 在模块开始导入 views 模块，传递 views.视图函数
from django.conf.urls import include, url
from django.contrib import admin
from mysite import views

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^hello/$', views.hello),
    url(r'^time/$', views.current_datetime),
    url(r'^time/plus/(\d{1,2})/$', views.hours_ahead),
]
```

``` python urls.py
# 传入一个包含模块名+函数名的对象
from django.conf.urls import include, url
from django.contrib import admin

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^hello/$', 'mysite.views.hello'),
    url(r'^time/$', 'mysite.views.current_datetime'),
    url(r'^time/plus/(\d{1,2})/$', 'mysite.views.hours_ahead'),
]
```

``` python urls.py
# 开启 URLconf 调试模式
from django.conf.urls import include, url
from django.contrib import admin
from mysite import views

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^hello/$', views.hello),
    url(r'^time/$', views.current_datetime),
    url(r'^time/plus/(\d{1,2})/$', views.hours_ahead),
]

if settings.DEBUG:
	urlpatterns += url(r'^debuginfo/$', views.debug),
	)
```
## 命名组
我觉得命名组的模式增加了代码冗余度，且语义化也不好。对于我这种懒人完全不需要 ：D ~~(其实就是我懒的借口)~~

而它的目的在于，将变量以**位置参数**的方式传递给视图函数变为以**关键字参数**的方式传递。

``` python urls.py
# 无名组，以位置参数传递变量
from django.conf.urls import include, url
from django.contrib import admin
from mysite import views

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^articles/(\d{4})/$'， view.year_archive),
    url(r'^articles/(\d{4})/(\d{2})/$', views.month_archive),
]
```

``` python urls.py
# 命名组，以关键字参数传递变量
from django.conf.urls import include, url
from django.contrib import admin
from mysite import views

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^articles/(?P<year>\d{4})/$'， view.year_archive),
    url(r'^articles/(?P<year>\d{4})/(?<month>\d{2})/$', views.month_archive),
]
```



