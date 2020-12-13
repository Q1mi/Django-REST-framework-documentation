
# Generic views

> Django的标准 views...是为更快捷地使用常见的使用模式而开发的... 它们采用在view开发中找到的某些常见语法和模式，并对它们进行抽象，以便你可以快速写出数据的常见视图，而无需重复自己。
>

> — [Django 文档][cite]

基于类的视图的主要优点之一是它们允许你组合一些可重用的行为。  REST framework通过提供许多预先构建的视图来提供常用的模式来利用这一优点。

REST framework 提供的通用视图允许您快速构建与数据库模型密切映射的API视图。

如果通用视图不适合你的API的需求，你可以选择使用常规 `APIView` 类，或重用通用视图使用的mixins和基类来组成你自己的一组可重用的通用视图。

## 举个例子

通常在使用通用视图时，你将覆盖视图，并设置多个类属性。

```
from django.contrib.auth.models import User
from myapp.serializers import UserSerializer
from rest_framework import generics
from rest_framework.permissions import IsAdminUser

class UserList(generics.ListCreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = (IsAdminUser, )
```

对于更复杂的情况，您可能还想覆盖视图类上的各种方法。比如：

    class UserList(generics.ListCreateAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer
        permission_classes = (IsAdminUser, )

        def list(self, request):
            # Note the use of `get_queryset()` instead of `self.queryset`
            queryset = self.get_queryset()
            serializer = UserSerializer(queryset, many=True)
            return Response(serializer.data)

对于非常简单的情况，你可能想使用 `.as_view()` 方法传递任何类属性。 比如：你的URLconf可能包括类似以下条目：

```
url(r'^/users/', ListCreateAPIView.as_view\(queryset=User.objects.all(), serializer_class=UserSerializer), name='user-list')
```

---

# API Reference

## GenericAPIView

此类扩展了REST框架的 `APIView` 类，为标准list和detail view 添加了通常需要的行为。

提供的每个具体通用视图是通过将 `GenericAPIView` 与一个或多个mixin类组合来构建的。

### Attributes

**基本设置**:

以下属性控制着基本视图的行为。

* `queryset` - 用于从视图返回对象的查询结果集。通常，你必须设置此属性或者重写 `get_queryset()` 方法。如果你重写了一个视图的方法，重要的是你应该调用 `get_queryset()` 方法而不是直接访问该属性，因为 `queryset` 将被计算一次，这些结果将为后续请求缓存起来。
* `serializer_class` - 用于验证和反序列化输入以及用于序列化输出的Serializer类。 通常，你必须设置此属性或者重写`get_serializer_class()` 方法。
* `lookup_field` - 用于执行各个model实例的对象查找的model字段。默认为 `'pk'`。 请注意，在使用超链接API时，如果需要使用自定义的值，你需要确保在API视图*和*序列化类*都*设置查找字段。
* `lookup_url_kwarg` - 应用于对象查找的URL关键字参数。它的 URL conf 应该包括一个与这个值相对应的关键字参数。如果取消设置，默认情况下使用与 `lookup_field`相同的值。

**Pagination**:

以下属性用于在与列表视图一起使用时控制分页。

* `pagination_class` - 当分页列出结果时应使用的分页类。默认值与 `DEFAULT_PAGINATION_CLASS` 设置的值相同，即 `'rest_framework.pagination.PageNumberPagination'`。

**Filtering**:

* `filter_backends` - 用于过滤查询集的过滤器后端类的列表。默认值与`DEFAULT_FILTER_BACKENDS` 设置的值相同。

### Methods

**Base methods**:

#### `get_queryset(self)`

返回列表视图中实用的查询集，该查询集还用作详细视图中的查找基础。默认返回由 `queryset` 属性指定的查询集。
这个方法应该总是被调用而不是直接访问 `self.queryset` ，因为 `self.queryset` 只会被计算一起，然后这些结果将为后续的请求缓存起来。
该方法可能会被重写以提供动态行为，比如返回基于发出请求的用户的结果集。

例如:

```
def get_queryset(self):
    user = self.request.user
    return user.accounts.all()
```

#### `get_object(self)`

返回应用于详细视图的对象实例。默认使用 `lookup_field` 参数过滤基本的查询集。

该方法可以被重写以提供更复杂的行为，例如基于多个 URL 参数的对象查找。

例如:

```
def get_object(self):
    queryset = self.get_queryset()
    filter = {}
    for field in self.multiple_lookup_fields:
        filter[field] = self.kwargs[field]

    obj = get_object_or_404(queryset, **filter)
    self.check_object_permissions(self.request, obj)
    return obj
```

请注意，如果你的API不包含任何对象级的权限控制，你可以选择不执行`self.check_object_permissions`，简单的返回 `get_object_or_404` 查找的对象即可。

#### `filter_queryset(self, queryset)`

给定一个queryset，使用任何过滤器后端进行过滤，返回一个新的queryset。

例如:

```
def filter_queryset(self, queryset):
    filter_backends = (CategoryFilter, )

    if 'geo_route' in self.request.query_params:
        filter_backends = (GeoRouteFilter, CategoryFilter)
    elif 'geo_point' in self.request.query_params:
        filter_backends = (GeoPointFilter, CategoryFilter)

    for backend in list(filter_backends):
        queryset = backend().filter_queryset(self.request, queryset, view=self)

    return queryset
```

#### `get_serializer_class(self)`

返回应用于序列化的类。默认为返回 `serializer_class` 属性的值。

可以被重写以提供动态的行为，例如对于读取和写入操作使用不同的序列化器，或者为不同类型的用户提供不同的序列化器。

例如:

```
def get_serializer_class(self\):
    if self.request.user.is_staff:
        return FullAccountSerializer
    return BasicAccountSerializer
```

**Save and deletion hooks**:

以下方法由mixin类提供，并提供对象保存或删除行为的简单重写。

* `perform_create(self, serializer)` - 在保存新对象实例时由 `CreateModelMixin` 调用。
* `perform_update(self, serializer)` - 在保存现有对象实例时由 `UpdateModelMixin` 调用。
* `perform_destroy(self, instance)` - 在删除对象实例时由 `DestroyModelMixin` 调用。

这些钩子对于设置请求中隐含的但不是请求数据的一部分的属性特别有用。例如，你可以根据请求用户或基于URL关键字参数在对象上设置属性。

```
def perform_create(self, serializer):
    serializer.save(user=self.request.user)
```

这些可重写的关键点对于添加在保存对象之前或之后发生的行为（例如通过电子邮件发送确认或记录更新日志）也特别有用。

```
def perform_update(self, serializer):
    instance = serializer.save()
    send_email_confirmation(user=self.request.user, modified=instance)
```

你还可以使用这些钩子通过抛出 `ValidationError()` 来提供额外的验证。当你需要在数据库保存时应用一些验证逻辑时，这会很有用。 例如:

```
def perform_create(self, serializer):
    queryset = SignupRequest.objects.filter(user=self.request.user)
    if queryset.exists():
        raise ValidationError('You have already signed up')
    serializer.save(user=self.request.user)
```

**注意**: 这些方法替换了旧版本2.x中的 `pre_save`, `post_save`, `pre_delete` 和 `post_delete` 方法，这些方法不再可用。

**其他方法**:

你通常并不需要重写以下方法，虽然在你使用 `GenericAPIView` 编写自定义视图的时候可能会调用它们。

* `get_serializer_context(self)` - 返回包含应该提供给序列化程序的任何额外上下文的字典。默认包含 `'request'`, `'view'` 和 `'format'` 这些keys。.
* `get_serializer(self, instance=None, data=None, many=False, partial=False)` - 返回一个序列化器的实例。
* `get_paginated_response(self, data)` - 返回分页样式的 `Response` 对象。
* `paginate_queryset(self, queryset)` - 如果需要分页查询，返回页面对象，如果没有为此视图配置分页，则返回 `None`。
* `filter_queryset(self, queryset)` - 给定查询集，使用任何过滤器后端进行过滤，返回一个新的查询集。

---

# Mixins

Mixin 类提供用于提供基本视图行为的操作。注意mixin类提供动作方法，而不是直接定义处理程序方法，例如 `.get()` 和 `.post()`， 这允许更灵活的行为组成。

Mixin 类可以从 `rest_framework.mixins`导入。

## ListModelMixin

提供一个 `.list(request, *args, **kwargs)` 方法，实现列出结果集。

如果查询集被填充了数据，则返回 `200 OK` 响应，将查询集的序列化表示作为响应的主体。相应数据可以任意分页。

## CreateModelMixin

提供 `.create(request, *args, **kwargs)` 方法，实现创建和保存一个新的model实例。

如果创建了一个对象，这将返回一个 `201 Created` 响应，将该对象的序列化表示作为响应的主体。如果序列化的表示中包含名为 `url`的键，则响应的 `Location` 头将填充该值。

如果为创建对象提供的请求数据无效，将返回 `400 Bad Request`，其中错误详细信息作为响应的正文。

## RetrieveModelMixin

提供一个 `.retrieve(request, *args, **kwargs)` 方法，实现返回响应中现有模型的实例。

如果可以检索对象，则返回 `200 OK` 响应，将该对象的序列化表示作为响应的主体。否则将返回 `404 Not Found`。

## UpdateModelMixin

提供 `.update(request, *args, **kwargs)` 方法，实现更新和保存现有模型实例。

同时还提供了一个 `.partial_update(request, *args, **kwargs)` 方法，这个方法和 `update` 方法类似，但更新的所有字段都是可选的。这允许支持 HTTP `PATCH` 请求。

如果一个对象被更新，这将返回一个 `200 OK` 响应，将对象的序列化表示作为响应的主体。

如果为更新对象提供的请求数据无效，将返回一个 `400 Bad Request` 响应，错误详细信息作为响应的正文。

## DestroyModelMixin

提供一个 `.destroy(request, *args, **kwargs)` 方法，实现删除现有模型实例。

如果删除对象，则返回 `204 No Content` 响应，否则返回 `404 Not Found`。

---

# Concrete View Classes

以下类是具体的通用视图。这通常是你真正用到的那些，除非你需要深度定制的行为。

这些视图类可以从 `rest_framework.generics`导入。

## CreateAPIView

用于 **仅创建** 端点。

提供一个 `post` 方法处理程序。

扩展: [GenericAPIView], [CreateModelMixin]

## ListAPIView

用于** 只读 ** 端点以表示**模型实例集合 **。

提供一个 `get` 方法处理程序。

扩展: [GenericAPIView], [ListModelMixin]

## RetrieveAPIView

用于**只读** 端点以表示**单个模型实例**。

提供一个 `get` 方法处理程序。

扩展: [GenericAPIView], [RetrieveModelMixin]

## DestroyAPIView

用于**只删除**端点以表示**单个模型实例**。

提供一个 `delete` 方法处理程序。

扩展: [GenericAPIView], [DestroyModelMixin]

## UpdateAPIView

用于**只更新**端点以表示**单个模型实例**。

提供一个 `put`和`patch`方法处理程序。

扩展: [GenericAPIView], [UpdateModelMixin]

## ListCreateAPIView

用于**读写端点**以表示**模型实例的集合**。

提供一个 `get` 和 `post` 方法的处理程序。

扩展: [GenericAPIView], [ListModelMixin], [CreateModelMixin]

## RetrieveUpdateAPIView

用于 **读取或更新** 端点以表示 **单个模型实例**。

提供 `get`, `put` 和 `patch` 方法的处理程序。

扩展: [GenericAPIView], [RetrieveModelMixin], [UpdateModelMixin]

## RetrieveDestroyAPIView

用于 **读取或删除** 端点以表示 **单个模型实例**。

提供 `get` 和 `delete` 方法的处理程序。

扩展: [GenericAPIView], [RetrieveModelMixin], [DestroyModelMixin]

## RetrieveUpdateDestroyAPIView

用于 **读写删除** 端点以表示 **单个模型实例**。

提供 `get`, `put`, `patch` 和 `delete`方法的处理程序。

扩展: [GenericAPIView], [RetrieveModelMixin], [UpdateModelMixin], [DestroyModelMixin]

---

# 自定义通用视图

通常你会想使用现有的通用视图，但是使用一些简单的自定义的行为。如果你发现自己在多个地方重复使用了一些自定义行为，你可能想将行为重构为一个公共类，然后只需根据需要应用到任何视图或视图集。

## 创建自定义 mixins

例如，如果你需要基于 URL conf中的多个字段查找对象，则可以创建一个如下所示的 mixin类：

    class MultipleFieldLookupMixin(object):
        """
        Apply this mixin to any view or viewset to get multiple field filtering
        based on a `lookup_fields` attribute, instead of the default single field filtering.
        """
        def get_object(self):
            queryset = self.get_queryset()             # 获取基本的queryset
            queryset = self.filter_queryset(queryset)  # 应用任何过滤器后端
            filter = {}
            for field in self.lookup_fields:
                if self.kwargs[field]: # Ignore empty fields.
                    filter[field] = self.kwargs[field]
            return get_object_or_404(queryset, **filter)  # 查找对象

然后，你可以在需要应用自定义行为时随时将此mixin类应用于视图或视图集。

```
class RetrieveUserView(MultipleFieldLookupMixin, generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
    lookup_fields = ('account', 'username')
```

如果你需要使用自定义行为，那么使用自定义mixin是一个不错的选择。

## 创建自定义基类

如果你在多个视图中使用mixin，你可以进一步创建你自己的一组基本视图，然后可以在整个项目中使用。举个例子:

```
class BaseRetrieveView(MultipleFieldLookupMixin,
                       generics.RetrieveAPIView):
    pass

class BaseRetrieveUpdateDestroyView(MultipleFieldLookupMixin,
                                    generics.RetrieveUpdateDestroyAPIView):
    pass
```

如果你的自定义行为始终需要在整个项目中的大量视图中重复，则使用自定义基类是一个不错的选择。

---

# PUT as create

在3.0版本之前，REST framework mixins 将 `PUT` 视为更新或创建操作，具体取决于对象是否已存在。

允许 `PUT` 作为创建操作是有问题的，因为它必然会暴露关于对象的存在与不存在的信息。同样不明显的是，透明地允许重新创建以前删除的实例是比简单地返回`404`响应更好的默认行为。

两种形式 "`PUT` as 404" 和 "`PUT` as create" 可以在不同的情况下有效，但从3.0版本起，我们现在使用 404作为默认值，因为它更简单和更明显。

如果你需要通用的 PUT-as-create行为，你可能想要包括像 [这个`AllowPUTAsCreateMixin` 类]([https://gist.github.com/tomchristie/a2ace4577eff2c603b1b](https://gist.github.com/tomchristie/a2ace4577eff2c603b1b)) 一样的mixin在你的视图里。

---

# 第三方包

以下第三方包提供了其他通用视图实现。

## Django REST Framework bulk

The [django-rest-framework-bulk package][django-rest-framework-bulk] 包实现通用视图mixins以及一些常见的具体通用视图，以允许通过API请求应用批量操作。

## Django Rest Multiple Models

[Django Rest Multiple Models][django-rest-multiple-models] 提供了通过单个API请求发送多个序列化模型和/或者查询集的通用视图（和mixin）。 

[cite]: [https://docs.djangoproject.com/en/stable/ref/class-based-views/\\#base-vs-generic-views](https://docs.djangoproject.com/en/stable/ref/class-based-views/\#base-vs-generic-views)

[GenericAPIView]: #genericapiview

[ListModelMixin]: #listmodelmixin

[CreateModelMixin]: #createmodelmixin

[RetrieveModelMixin]: #retrievemodelmixin

[UpdateModelMixin]: #updatemodelmixin

[DestroyModelMixin]: #destroymodelmixin

[django-rest-framework-bulk]: [https://github.com/miki725/django-rest-framework-bulk](https://github.com/miki725/django-rest-framework-bulk)

[django-rest-multiple-models]: [https://github.com/Axiologue/DjangoRestMultipleModels](https://github.com/Axiologue/DjangoRestMultipleModels)

