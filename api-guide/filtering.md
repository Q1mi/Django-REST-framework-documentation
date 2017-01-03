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

---

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

## Filtering and object lookups（过滤和对象查找）

请注意，如果为view配置过滤器后端，并且用于过滤list views，则它也将用于过滤用于返回单个对象的queryset。

例如，给定前面的示例以及一个ID为`4675`的产品，以下URL将返回相应的对象或者返回404，具体取决于给定的产品实例是否满足筛选条件。


```
http://example.com/api/products/4675/?category=clothing&max_price=10.00
```

## Overriding the initial queryset（覆盖初始queryset）

请注意，你可以同时使用覆盖的`.get_queryset()`和通用过滤，并且一切都将按预期生效。 例如，如果`Product`与`User`（名为`purchase`）具有多对多关系，则可能需要编写如下所示的view：


```
class PurchasedProductsList(generics.ListAPIView):
    """
    Return a list of all the products that the authenticated
    user has ever purchased, with optional filtering.
    """
    model = Product
    serializer_class = ProductSerializer
    filter_class = ProductFilter

    def get_queryset(self):
        user = self.request.user
        return user.purchase_set.all()
```

---


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

## Specifying a FilterSet（指定FilterSet）

对于更高级的过滤要求，你可以指定在view中应该使用的`FilterSet`类。例如：


```
import django_filters
from myapp.models import Product
from myapp.serializers import ProductSerializer
from rest_framework import generics

class ProductFilter(django_filters.rest_framework.FilterSet):
    min_price = django_filters.NumberFilter(name="price", lookup_expr='gte')
    max_price = django_filters.NumberFilter(name="price", lookup_expr='lte')
    class Meta:
        model = Product
        fields = ['category', 'in_stock', 'min_price', 'max_price']

class ProductList(generics.ListAPIView):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
    filter_backends = (django_filters.rest_framework.DjangoFilterBackend,)
    filter_class = ProductFilter
```

这将允许你向下面一样发出请求，如：


```
http://example.com/api/products?category=clothing&max_price=10.00
```

你也可以使用`django-filter`跨越关系，让我们假设每个产品都有`Manufacturer`模型的外键，所以我们创建过滤器使用`Manufacturer`名称过滤。例如：


```
import django_filters
from myapp.models import Product
from myapp.serializers import ProductSerializer
from rest_framework import generics

class ProductFilter(django_filters.rest_framework.FilterSet):
    class Meta:
        model = Product
        fields = ['category', 'in_stock', 'manufacturer__name']
```

这使我们能够进行如下查询：


```
http://example.com/api/products?manufacturer__name=foo
```

这很好，但它将Django的双下划线约定作为API的一部分暴露出来。如果你想显式地命名过滤器参数，你可以显式地将它包含在`FilterSet`类中：


```
import django_filters
from myapp.models import Product
from myapp.serializers import ProductSerializer
from rest_framework import generics

class ProductFilter(django_filters.rest_framework.FilterSet):
    manufacturer = django_filters.CharFilter(name="manufacturer__name")

    class Meta:
        model = Product
        fields = ['category', 'in_stock', 'manufacturer']
```


