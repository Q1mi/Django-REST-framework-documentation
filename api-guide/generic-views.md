source: mixins.py

```
    generics.py
```

\# Generic views

&gt; Django的标准 views...是为更快捷地使用常见的使用模式而开发的... 它们采用在view开发中找到的某些常见语法和模式，并对它们进行抽象，以便你可以快速写出数据的常见视图，而无需重复自己。

&gt;

&gt; — \[Django 文档\]\[cite\]

基于类的视图的主要优点之一是它们允许你组合一些可重用的行为。  REST framework通过提供许多预先构建的视图来提供常用的模式来利用这一优点。

REST framework 提供的通用视图允许您快速构建与数据库模型密切映射的API视图。

如果通用视图不适合你的API的需求，你可以选择使用常规`APIView`类，或重用通用视图使用的mixins和基类来组成你自己的一组可重用的通用视图。

\#\# 举个例子

通常在使用通用视图时，你将覆盖视图，并设置多个类属性。

```
from django.contrib.auth.models import User

from myapp.serializers import UserSerializer

from rest\_framework import generics

from rest\_framework.permissions import IsAdminUser



class UserList\(generics.ListCreateAPIView\):

    queryset = User.objects.all\(\)

    serializer\_class = UserSerializer

    permission\_classes = \(IsAdminUser,\)
```

对于更复杂的情况，您可能还想覆盖视图类上的各种方法。比如：

    class UserList\(generics.ListCreateAPIView\):

        queryset = User.objects.all\(\)

        serializer\_class = UserSerializer

        permission\_classes = \(IsAdminUser,\)



        def list\(self, request\):

            \# Note the use of \`get\_queryset\(\)\` instead of \`self.queryset\`

            queryset = self.get\_queryset\(\)

            serializer = UserSerializer\(queryset, many=True\)

            return Response\(serializer.data\)

对于非常简单的情况，你可能想使用`.as_view()`方法传递任何类属性。 比如：你的URLconf可能包括类似以下条目：

```
url\(r'^/users/', ListCreateAPIView.as\_view\(queryset=User.objects.all\(\), serializer\_class=UserSerializer\), name='user-list'\)
```

---

\# API Reference

\#\# GenericAPIView

此类扩展了REST框架的`APIView`类，为标准list和detail view 添加了通常需要的行为。

提供的每个具体通用视图是通过将`GenericAPIView`与一个或多个mixin类组合来构建的。

\#\#\# Attributes

\*\*基本设置\*\*:

以下属性控制着基本视图的行为。

\* \`queryset\` - 用于从视图返回对象的查询结果集。通常，你必须设置此属性或者重写 \`get\_queryset\(\)\` 方法。如果你重写了一个视图的方法，重要的是你应该调用 \`get\_queryset\(\)\` 方法而不是直接访问该属性，因为 \`queryset\` 将被计算一次，这些结果将为后续请求缓存起来。

\* \`serializer\_class\` - 用于验证和反序列化输入以及用于序列化输出的Serializer类。 通常，你必须设置此属性或者重写\`get\_serializer\_class\(\)\` 方法。

\* \`lookup\_field\` - 用于执行各个model实例的对象查找的model字段。默认为 \`'pk'\`。 请注意，在使用超链接API时，如果需要使用自定义的值，你需要确保在API视图\*和\*序列化类\*都\*设置查找字段。

\* \`lookup\_url\_kwarg\` - 应用于对象查找的URL关键字参数。它的 URL conf 应该包括一个与这个值相对应的关键字参数。如果取消设置，默认情况下使用与 \`lookup\_field\`相同的值。

\*\*Pagination\*\*:

以下属性用于在与列表视图一起使用时控制分页。

\* \`pagination\_class\` - 当分页列出结果时应使用的分页类。默认值与 \`DEFAULT\_PAGINATION\_CLASS\` 设置的值相同，即 \`'rest\_framework.pagination.PageNumberPagination'\`。

\*\*Filtering\*\*:

\* \`filter\_backends\` - 用于过滤查询集的过滤器后端类的列表。默认值与\`DEFAULT\_FILTER\_BACKENDS\` 设置的值相同。

\#\#\# Methods

\*\*Base methods\*\*:

\#\#\#\# \`get\_queryset\(self\)\`

返回列表视图中实用的查询集，该查询集还用作详细视图中的查找基础。默认返回由 \`queryset\` 属性指定的查询集。

这个方法应该总是被调用而不是直接访问 \`self.queryset\` ，因为 \`self.queryset\` 只会被计算一起，然后这些结果将为后续的请求缓存起来。

该方法可能会被重写以提供动态行为，比如返回基于发出请求的用户的结果集。

例如:

```
def get\_queryset\(self\):

    user = self.request.user

    return user.accounts.all\(\)
```

\#\#\#\# \`get\_object\(self\)\`

返回应用于详细视图的对象实例。默认使用 \`lookup\_field\` 参数过滤基本的查询集。

该方法可以被重写以提供更复杂的行为，例如基于多个 URL 参数的对象查找。

例如:

```
def get\_object\(self\):

    queryset = self.get\_queryset\(\)

    filter = {}

    for field in self.multiple\_lookup\_fields:

        filter\[field\] = self.kwargs\[field\]



    obj = get\_object\_or\_404\(queryset, \*\*filter\)

    self.check\_object\_permissions\(self.request, obj\)

    return obj
```

请注意，如果你的API不包含任何对象级的权限控制，你可以选择不执行\`self.check\_object\_permissions\`，简单的返回 \`get\_object\_or\_404\` 查找的对象即可。

\#\#\#\# \`filter\_queryset\(self, queryset\)\`

给定一个queryset，使用任何过滤器后端进行过滤，返回一个新的queryset。

例如:

```
def filter\_queryset\(self, queryset\):

    filter\_backends = \(CategoryFilter,\)



    if 'geo\_route' in self.request.query\_params:

        filter\_backends = \(GeoRouteFilter, CategoryFilter\)

    elif 'geo\_point' in self.request.query\_params:

        filter\_backends = \(GeoPointFilter, CategoryFilter\)



    for backend in list\(filter\_backends\):

        queryset = backend\(\).filter\_queryset\(self.request, queryset, view=self\)



    return queryset
```

\#\#\#\# \`get\_serializer\_class\(self\)\`

返回应用于序列化的类。默认为返回\`serializer\_class\` 属性的值。

可以被重写以提供动态的行为，例如对于读取和写入操作使用不同的序列化器，或者为不同类型的用户提供不同的序列化器。

例如:

```
def get\_serializer\_class\(self\):

    if self.request.user.is\_staff:

        return FullAccountSerializer

    return BasicAccountSerializer
```

\*\*Save and deletion hooks\*\*:

以下方法由mixin类提供，并提供对象保存或删除行为的简单重写。

\* \`perform\_create\(self, serializer\)\` - 在保存新对象实例时由 \`CreateModelMixin\` 调用。

\* \`perform\_update\(self, serializer\)\` - 在保存现有对象实例时由 \`UpdateModelMixin\` 调用。

\* \`perform\_destroy\(self, instance\)\` - 在删除对象实例时由 \`DestroyModelMixin\` 调用。

这些钩子对于设置请求中隐含的但不是请求数据的一部分的属性特别有用。例如，你可以根据请求用户或基于URL关键字参数在对象上设置属性。

```
def perform\_create\(self, serializer\):

    serializer.save\(user=self.request.user\)
```

这些可重写的关键点对于添加在保存对象之前或之后发生的行为（例如通过电子邮件发送确认或记录更新日志）也特别有用。

```
def perform\_update\(self, serializer\):

    instance = serializer.save\(\)

    send\_email\_confirmation\(user=self.request.user, modified=instance\)
```

你还可以使用这些钩子通过抛出\`ValidationError\(\)\`来提供额外的验证。当你需要在数据库保存时应用一些验证逻辑时，这会很有用。 例如:

```
def perform\_create\(self, serializer\):

    queryset = SignupRequest.objects.filter\(user=self.request.user\)

    if queryset.exists\(\):

        raise ValidationError\('You have already signed up'\)

    serializer.save\(user=self.request.user\)
```

\*\*注意\*\*: 这些方法替换了旧版本 2.x中的 \`pre\_save\`, \`post\_save\`, \`pre\_delete\` 和 \`post\_delete\` 方法，这些方法不再可用。

\*\*其他方法\*\*:

你通常并不需要重写以下方法，虽然在你使用\`GenericAPIView\`编写自定义视图的时候可能会调用它们。

\* \`get\_serializer\_context\(self\)\` - 返回包含应该提供给序列化程序的任何额外上下文的字典。默认包含 \`'request'\`, \`'view'\` 和 \`'format'\` 这些keys。.

\* \`get\_serializer\(self, instance=None, data=None, many=False, partial=False\)\` - 返回一个序列化器的实例。

\* \`get\_paginated\_response\(self, data\)\` - 返回分页样式的 \`Response\` 对象。

\* \`paginate\_queryset\(self, queryset\)\` - 如果需要分页查询，返回页面对象，如果没有为此视图配置分页，则返回 \`None\`。

\* \`filter\_queryset\(self, queryset\)\` - 给定查询集，使用任何过滤器后端进行过滤，返回一个新的查询集。

---

\# Mixins

Mixin 类提供用于提供基本视图行为的操作。注意mixin类提供动作方法，而不是直接定义处理程序方法，例如 \`.get\(\)\` 和\`.post\(\)\`， 这允许更灵活的行为组成。

Mixin 类能够从 \`rest\_framework.mixins\`导入。

\#\# ListModelMixin

Provides a \`.list\(request, \*args, \*\*kwargs\)\` method, that implements listing a queryset.

If the queryset is populated, this returns a \`200 OK\` response, with a serialized representation of the queryset as the body of the response.  The response data may optionally be paginated.

\#\# CreateModelMixin

Provides a \`.create\(request, \*args, \*\*kwargs\)\` method, that implements creating and saving a new model instance.

If an object is created this returns a \`201 Created\` response, with a serialized representation of the object as the body of the response.  If the representation contains a key named \`url\`, then the \`Location\` header of the response will be populated with that value.

If the request data provided for creating the object was invalid, a \`400 Bad Request\` response will be returned, with the error details as the body of the response.

\#\# RetrieveModelMixin

Provides a \`.retrieve\(request, \*args, \*\*kwargs\)\` method, that implements returning an existing model instance in a response.

If an object can be retrieved this returns a \`200 OK\` response, with a serialized representation of the object as the body of the response.  Otherwise it will return a \`404 Not Found\`.

\#\# UpdateModelMixin

Provides a \`.update\(request, \*args, \*\*kwargs\)\` method, that implements updating and saving an existing model instance.

Also provides a \`.partial\_update\(request, \*args, \*\*kwargs\)\` method, which is similar to the \`update\` method, except that all fields for the update will be optional.  This allows support for HTTP \`PATCH\` requests.

If an object is updated this returns a \`200 OK\` response, with a serialized representation of the object as the body of the response.

If the request data provided for updating the object was invalid, a \`400 Bad Request\` response will be returned, with the error details as the body of the response.

\#\# DestroyModelMixin

Provides a \`.destroy\(request, \*args, \*\*kwargs\)\` method, that implements deletion of an existing model instance.

If an object is deleted this returns a \`204 No Content\` response, otherwise it will return a \`404 Not Found\`.

---

\# Concrete View Classes

The following classes are the concrete generic views.  If you're using generic views this is normally the level you'll be working at unless you need heavily customized behavior.

The view classes can be imported from \`rest\_framework.generics\`.

\#\# CreateAPIView

Used for \*\*create-only\*\* endpoints.

Provides a \`post\` method handler.

Extends: \[GenericAPIView\], \[CreateModelMixin\]

\#\# ListAPIView

Used for \*\*read-only\*\* endpoints to represent a \*\*collection of model instances\*\*.

Provides a \`get\` method handler.

Extends: \[GenericAPIView\], \[ListModelMixin\]

\#\# RetrieveAPIView

Used for \*\*read-only\*\* endpoints to represent a \*\*single model instance\*\*.

Provides a \`get\` method handler.

Extends: \[GenericAPIView\], \[RetrieveModelMixin\]

\#\# DestroyAPIView

Used for \*\*delete-only\*\* endpoints for a \*\*single model instance\*\*.

Provides a \`delete\` method handler.

Extends: \[GenericAPIView\], \[DestroyModelMixin\]

\#\# UpdateAPIView

Used for \*\*update-only\*\* endpoints for a \*\*single model instance\*\*.

Provides \`put\` and \`patch\` method handlers.

Extends: \[GenericAPIView\], \[UpdateModelMixin\]

\#\# ListCreateAPIView

Used for \*\*read-write\*\* endpoints to represent a \*\*collection of model instances\*\*.

Provides \`get\` and \`post\` method handlers.

Extends: \[GenericAPIView\], \[ListModelMixin\], \[CreateModelMixin\]

\#\# RetrieveUpdateAPIView

Used for \*\*read or update\*\* endpoints to represent a \*\*single model instance\*\*.

Provides \`get\`, \`put\` and \`patch\` method handlers.

Extends: \[GenericAPIView\], \[RetrieveModelMixin\], \[UpdateModelMixin\]

\#\# RetrieveDestroyAPIView

Used for \*\*read or delete\*\* endpoints to represent a \*\*single model instance\*\*.

Provides \`get\` and \`delete\` method handlers.

Extends: \[GenericAPIView\], \[RetrieveModelMixin\], \[DestroyModelMixin\]

\#\# RetrieveUpdateDestroyAPIView

Used for \*\*read-write-delete\*\* endpoints to represent a \*\*single model instance\*\*.

Provides \`get\`, \`put\`, \`patch\` and \`delete\` method handlers.

Extends: \[GenericAPIView\], \[RetrieveModelMixin\], \[UpdateModelMixin\], \[DestroyModelMixin\]

---

\# Customizing the generic views

Often you'll want to use the existing generic views, but use some slightly customized behavior.  If you find yourself reusing some bit of customized behavior in multiple places, you might want to refactor the behavior into a common class that you can then just apply to any view or viewset as needed.

\#\# Creating custom mixins

For example, if you need to lookup objects based on multiple fields in the URL conf, you could create a mixin class like the following:

    class MultipleFieldLookupMixin\(object\):

        """

        Apply this mixin to any view or viewset to get multiple field filtering

        based on a \`lookup\_fields\` attribute, instead of the default single field filtering.

        """

        def get\_object\(self\):

            queryset = self.get\_queryset\(\)             \# Get the base queryset

            queryset = self.filter\_queryset\(queryset\)  \# Apply any filter backends

            filter = {}

            for field in self.lookup\_fields:

                if self.kwargs\[field\]: \# Ignore empty fields.

                    filter\[field\] = self.kwargs\[field\]

            return get\_object\_or\_404\(queryset, \*\*filter\)  \# Lookup the object

You can then simply apply this mixin to a view or viewset anytime you need to apply the custom behavior.

```
class RetrieveUserView\(MultipleFieldLookupMixin, generics.RetrieveAPIView\):

    queryset = User.objects.all\(\)

    serializer\_class = UserSerializer

    lookup\_fields = \('account', 'username'\)
```

Using custom mixins is a good option if you have custom behavior that needs to be used.

\#\# Creating custom base classes

If you are using a mixin across multiple views, you can take this a step further and create your own set of base views that can then be used throughout your project.  For example:

```
class BaseRetrieveView\(MultipleFieldLookupMixin,

                       generics.RetrieveAPIView\):

    pass



class BaseRetrieveUpdateDestroyView\(MultipleFieldLookupMixin,

                                    generics.RetrieveUpdateDestroyAPIView\):

    pass
```

Using custom base classes is a good option if you have custom behavior that consistently needs to be repeated across a large number of views throughout your project.

---

\# PUT as create

Prior to version 3.0 the REST framework mixins treated \`PUT\` as either an update or a create operation, depending on if the object already existed or not.

Allowing \`PUT\` as create operations is problematic, as it necessarily exposes information about the existence or non-existence of objects. It's also not obvious that transparently allowing re-creating of previously deleted instances is necessarily a better default behavior than simply returning \`404\` responses.

Both styles "\`PUT\` as 404" and "\`PUT\` as create" can be valid in different circumstances, but from version 3.0 onwards we now use 404 behavior as the default, due to it being simpler and more obvious.

If you need to generic PUT-as-create behavior you may want to include something like \[this \`AllowPUTAsCreateMixin\` class\]\([https://gist.github.com/tomchristie/a2ace4577eff2c603b1b\](https://gist.github.com/tomchristie/a2ace4577eff2c603b1b\)\) as a mixin to your views.

---

\# Third party packages

The following third party packages provide additional generic view implementations.

\#\# Django REST Framework bulk

The \[django-rest-framework-bulk package\]\[django-rest-framework-bulk\] implements generic view mixins as well as some common concrete generic views to allow to apply bulk operations via API requests.

\#\# Django Rest Multiple Models

\[Django Rest Multiple Models\]\[django-rest-multiple-models\] provides a generic view \(and mixin\) for sending multiple serialized models and/or querysets via a single API request.

\[cite\]: [https://docs.djangoproject.com/en/stable/ref/class-based-views/\\#base-vs-generic-views](https://docs.djangoproject.com/en/stable/ref/class-based-views/\#base-vs-generic-views)

\[GenericAPIView\]: \#genericapiview

\[ListModelMixin\]: \#listmodelmixin

\[CreateModelMixin\]: \#createmodelmixin

\[RetrieveModelMixin\]: \#retrievemodelmixin

\[UpdateModelMixin\]: \#updatemodelmixin

\[DestroyModelMixin\]: \#destroymodelmixin

\[django-rest-framework-bulk\]: [https://github.com/miki725/django-rest-framework-bulk](https://github.com/miki725/django-rest-framework-bulk)

\[django-rest-multiple-models\]: [https://github.com/Axiologue/DjangoRestMultipleModels](https://github.com/Axiologue/DjangoRestMultipleModels)

