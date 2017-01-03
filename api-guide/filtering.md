# Filtering

> “ 由Django Manager提供的根QuerySet描述了数据库表中的所有对象。可是通常你需要的只是选择完整对象中的一个子集而已。
>
> —— Django文档  
> ”

REST framework列表视图的默认行为是返回一个model的全部queryset。通常你却想要你的API来限制queryset返回的数据。

最简单的删选一个`GenericAPIView`子视图的queryset的方法就是复写他的`.get_queryset()`方法。

重写这个方法允许你使用很多不同的方式来定制视图返回的queryset。

## Filtering against the current user（根据当前用户进行过滤）

您可能想要过滤queryset，以确保只返回与发出请求的当前已验证用户相关的结果。

你可以通过基于request.user的值进行过滤来实现。

比如：


```
from myapp.models import Purchase
from myapp.serializers import PurchaseSerializer
from rest_framework import generics

class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases
        for the currently authenticated user.
        """
        user = self.request.user
        return Purchase.objects.filter(purchaser=user)
```

## Filtering against the URL（根据URL进行过滤）

另一种过滤方式可能包括基于URL的某些部分来限制queryset。

例如，如果你的URL配置包含一个参数如下:


```
url('^purchases/(?P<username>.+)/$', PurchaseList.as_view()),
```

你就可以写一个view，返回基于URL中的username参数进行过滤的结果。


```
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        This view should return a list of all the purchases for
        the user as determined by the username portion of the URL.
        """
        username = self.kwargs['username']
        return Purchase.objects.filter(purchaser__username=username)
```

## Filtering against query parameters（根据查询参数进行过滤）

过滤初始查询集的最后一个示例是基于url中的查询参数确定初始查询集。

我们可以覆盖`.get_queryset()`来处理像`http://example.com/api/purchases?username=denvercoder9`这样的网址，并且只有在URL中包含`username`参数时，才过滤queryset：


```
class PurchaseList(generics.ListAPIView):
    serializer_class = PurchaseSerializer

    def get_queryset(self):
        """
        Optionally restricts the returned purchases to a given user,
        by filtering against a `username` query parameter in the URL.
        """
        queryset = Purchase.objects.all()
        username = self.request.query_params.get('username', None)
        if username is not None:
            queryset = queryset.filter(purchaser__username=username)
        return queryset
```

# Generic Filtering（通用过滤）

除了能够覆盖默认queryset，REST框架还包括对通用过滤后端的支持，允许你轻松构建复杂的检索器和过滤器。

通用过滤器也可以在browsable API和admin API中显示为HTML控件。

![filter-controls.png](/assets/filter-controls.png)

## Setting filter backends（设置通用过滤后端）

默认过滤器后端可以在全局设置中使用`DEFAULT_FILTER_BACKENDS`来配置。例如。


```
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
}
```

你还可以使用基于`GenericAPIView`类的视图在每个view或每个viewset基础上设置过滤器后端。


```
import django_filters.rest_framework
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics

class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (django_filters.rest_framework.DjangoFilterBackend,)
```



# API Guide

## DjangoFilterBackend

`django-filter`库包含一个为REST framework提供高度可定制字段过滤的`DjangoFilterBackend`类。

要使用`DjangoFilterBackend`，首先要先安装`django-filter`。

```
pip install django-filter
```

现在，你需要将filter backend 添加到你django project的settings中：

```
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
}
```

或者你也可以将filter backend添加到一个单独的view或viewSet中：

```
from django_filters.rest_framework import DjangoFilterBackend

class UserListView(generics.ListAPIView):
    ...
    filter_backends = (DjangoFilterBackend,)
```

如果你正在使用 browsable API或 admin API，你还需要安装`django-crispy-forms`，通过使用Bootstarp3渲染来提高filter form在浏览器中的展示效果。

```
pip install django-crispy-forms
```

安装完成后，将`crispy-forms`添加到你Django project的`INSTALLED_APPS`中，browsable API将为`DjangoFilterBackend`提供一个像下面这样的filter control：  
![django filter](/assets/django-filter.png)

## 指定筛选字段（Specifying filter fields）

如果你的需求都是些简单相等类型的筛选，那么你可以在你的view或viewSet里面设置一个`filter_fields`属性，列出所有你想依靠筛选的字段集合。

```
class ProductList(generics.ListAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = (filters.DjangoFilterBackend,)
    filter_fields = ('category', 'in_stock')
```

这将为你列出来的字段自动创建一个filterSet，允许你想下面这样请求筛选结果：

```
http://example.com/api/products?category=clothing&in_stock=True
```

## 指定一个filterSet（Specifying a FilterSet）



