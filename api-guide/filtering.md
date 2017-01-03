# Filtering

> “ 由Django Manager提供的根QuerySet描述了数据库表中的所有对象。可是通常你需要的只是选择完整对象中的一个子集而已。
>
> —— Django文档  
> ”

REST framework列表视图的默认行为是返回一个model的全部queryset。通常你却想要你的API来限制queryset返回的数据。

最简单的删选一个`GenericAPIView`子视图的queryset的方法就是复写他的`.get_queryset()`方法。

重写这个方法允许你使用很多不同的方式来定制视图返回的queryset。

## Filtering against the current user（根据当前用户进行过滤）

您可能想要过滤queryset确保那些只与当前被认证的请求用户有关的结果被返回。

你可以通过使用`request.user`的值来过滤实现。

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

另一个风格的过滤可能涉及限制queryset基于URL中的某些参数。

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

## Filtering against query parameters（根据query 参数进行过滤）



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



