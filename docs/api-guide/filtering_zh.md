
# Filtering

> “ 由Django Manager提供的根QuerySet描述了数据库表中的所有对象。可是通常你需要的只是选择完整对象中的一个子集而已。
>
> —— Django文档
> ”

REST framework列表视图的默认行为是返回一个model的全部queryset。通常你却想要你的API来限制queryset返回的数据。

最简单的过滤任意`GenericAPIView`子视图queryset的方法就是重写它的`.get_queryset()`方法。

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

我们可以通过重写`.get_queryset()`方法来处理像`http://example.com/api/purchases?username=denvercoder9`这样的网址，并且只有在URL中包含`username`参数时，才过滤queryset：


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

除了能够重写默认的queryset，REST框架还包括对通用过滤后端的支持，允许你轻松构建复杂的检索器和过滤器。

通用过滤器也可以在browsable API和admin API中显示为HTML控件。

![filter-controls.png](../img/filter-controls.png)

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

请注意，你可以同时重写`.get_queryset()`方法或使用通用过滤，并且一切都会按照预期生效。 例如，如果`Product`与`User`（名为`purchase`）具有多对多关系，则可能需要编写如下所示的view：


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

## DjangoFilterBackend（Django过滤后端）

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
![django filter](../img/django-filter.png)

## Specifying filter fields（指定筛选字段）

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
现在你可以下面这样执行：


```
http://example.com/api/products?manufacturer=foo
```

有关使用过滤器集的更多详细信息，请参阅[django-filter文档](https://django-filter.readthedocs.io/en/latest/index.html)。

---

### Hints & Tips（提示）

* 默认情况下未启用过滤。如果你想使用`DjangoFilterBackend`记得确保它是通过使用`DEFAULT_FILTER_BACKENDS`设置安装的。

* 使用布尔字段时，您应该在网址查询参数中使用`True`和`False`值，而不是`0`，`1`，`true`或`false`。 （允许的布尔值目前硬连线在Django的[NullBooleanSelect实现中](https://github.com/django/django/blob/master/django/forms/widgets.py)。）

* `django-filter`支持使用Django的双下划线语法对关系进行过滤。

---

## SearchFilter（搜索过滤）

`SearchFilter`类支持基于简单单查询参数的搜索，并且基于[Django admin的搜索功能](https://docs.djangoproject.com/en/stable/ref/contrib/admin/#django.contrib.admin.ModelAdmin.search_fields)。

在使用时， browsable API将包括一个`SearchFilter`控件：
![search-filter.png](../img/search-filter.png)

仅当view中设置了`search_fields`属性时，才应用`SearchFilter`类。`search_fields`属性应该是model中文本类型字段的名称列表，例如`CharField`或`TextField`。


```
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.SearchFilter,)
    search_fields = ('username', 'email')
```

这将允许客户端通过进行以下查询来过滤列表中的项目：


```
http://example.com/api/users?search=russell
```

你还可以在查找API中使用双下划线符号对ForeignKey或ManyToManyField执行相关查找：


```
search_fields = ('username', 'email', 'profile__profession')
```

默认情况下，搜索将使用不区分大小写的部分匹配。 搜索参数可以包含多个搜索项，其应该是空格和/或逗号分隔。 如果使用多个搜索术语，则仅当所有提供的术语都匹配时才在列表中返回对象。

可以通过在`search_fields`前面添加各种字符来限制搜索行为。

* '^' 以指定内容开始.

* '=' 完全匹配

* '@' 全文搜索（目前只支持Django的MySQL后端）

* '$' 正则搜索

例如：


```
search_fields = ('=username', '=email')
```

默认情况下，搜索参数名为`'search'`，但这可以通过使用`SEARCH_PARAM`设置覆盖。

有关更多详细信息，请参阅[Django文档](https://docs.djangoproject.com/en/stable/ref/contrib/admin/#django.contrib.admin.ModelAdmin.search_fields)。

---

## OrderingFilter（排序筛选）

`OrderingFilter`类支持简单的查询参数控制结果排序。
![ordering-filter.png](../img/ordering-filter.png)

默认情况下，查询参数名为`'ordering'`，但这可以通过使用`ORDERING_PARAM`设置覆盖。

例如，按用户名排序用户：


```
http://example.com/api/users?ordering=username
```

客户端还可以通过为字段名称加上'-'来指定反向排序，如下所示：


```
http://example.com/api/users?ordering=-username
```

还可以指定多个排序：


```
http://example.com/api/users?ordering=account,username
```

## Specifying which fields may be ordered against（指定支持排序的字段）

建议你明确指定API应在ordering filter中允许哪些字段。您可以通过在view中设置`ordering_fields`属性来实现这一点，如下所示：


```
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = ('username', 'email')
```

这有助于防止意外的数据泄漏，例如允许用户针对密码哈希字段或其他敏感数据进行排序。

如果不在视图上指定`ordering_fields`属性，过滤器类将默认允许用户对`serializer_class`属性指定的serializer上的任何可读字段进行过滤。

如果你确信视图正在使用的queryset不包含任何敏感数据，则还可以通过使用特殊值`'__all__'`来明确指定view应允许对任何model字段或queryset进行排序。


```
class BookingsListView(generics.ListAPIView):
    queryset = Booking.objects.all()
    serializer_class = BookingSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = '__all__'
```

## Specifying a default ordering（指定默认排序）

如果在view中设置了`ordering`属性，则将把它用作默认排序。

通常，你可以通过在初始queryset上设置`order_by`来控制此操作，但是使用view中的`ordering`参数允许你以某种方式指定排序，然后可以将其作为上下文自动传递到呈现的模板。如果它们用于排序结果的话就能使自动渲染不同的列标题成为可能。


```
class UserListView(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    filter_backends = (filters.OrderingFilter,)
    ordering_fields = ('username', 'email')
    ordering = ('username',)
```

`ordering`属性可以是字符串或字符串的列表/元组。

---

# DjangoObjectPermissionsFilte（Django对象权限过滤）


