
# ViewSets

> 在路由已经确定了用于请求的控制器之后，你的控制器负责理解请求并产生适当的输出。 
>
> — [Ruby on Rails 文档](http://guides.rubyonrails.org/routing.html)

Django REST framework允许你将一组相关视图的逻辑组合在单个类（称为 `ViewSet`）中。 在其他框架中，你也可以找到概念上类似于 'Resources' 或 'Controllers'的类似实现。

 `ViewSet` 只是**一种基于类的视图，它不提供任何方法处理程序**（如 `.get()`或`.post()`）,而是提供诸如 `.list()` 和 `.create()` 之类的操作。

 `ViewSet` 的方法处理程序仅使用 `.as_view()` 方法绑定到完成视图的相应操作。

通常不是在 urlconf 中的视图集中显示注册视图，而是要使用路由类注册视图集，该类会自动为你确定 urlconf。

## Example

让我们定义一个简单的视图集，可以用来列出或检索系统中的所有用户。

```
from django.contrib.auth.models import User
from django.shortcuts import get_object_or_404
from myapps.serializers import UserSerializer
from rest_framework import viewsets
from rest_framework.response import Response

class UserViewSet(viewsets.ViewSet):
    """
    A simple ViewSet for listing or retrieving users.
    """
    def list(self, request):
        queryset = User.objects.all()
        serializer = UserSerializer(queryset, many=True)
        return Response(serializer.data)

    def retrieve(self, request, pk=None):
        queryset = User.objects.all()
        user = get_object_or_404(queryset, pk=pk)
        serializer = UserSerializer(user)
        return Response(serializer.data)
```

如果我们需要，我们可以将这个viewset绑定到两个单独的视图，想这样：

```
user_list = UserViewSet.as_view({'get': 'list'})
user_detail = UserViewSet.as_view({'get': 'retrieve'})
```

通常我们不会这么做，我们会用一个router来注册我们的viewset，让urlconf自动生成。

```
from myapp.views import UserViewSet
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'users', UserViewSet)
urlpatterns = router.urls
```

不需要编写自己的视图集，你通常会想要使用提供默认行为的现有基类。例如：

```
class UserViewSet(viewsets.ModelViewSet):
    """
    用于查看和编辑用户实例的视图集。
    """
    serializer_class = UserSerializer
    queryset = User.objects.all()
```

与使用 `View` 类相比，使用 `ViewSet` 类有两个主要优点。

* 重复的逻辑可以组合成一个类。在上面的例子中，我们只需要指定一次 `queryset`，它将在多个视图中使用。
* 通过使用 routers, 哦们不再需要自己处理URLconf。

这两者都有一个权衡。使用常规的 views 和 URL confs 更明确也能够为你提供更多的控制。ViewSets有助于快速启动和运行，或者当你有大型的API，并且希望在整个过程中执行一致的 URL 配置。

## 标记路由的额外操作

REST framework 中包含的默任 routes 将为标准的 create/retrieve/update/destroy 类型操作提供路由, 如下所示：

    class UserViewSet(viewsets.ViewSet):
        """
        示例 viewset 演示了将由路由器类处理的标准动作。

        如果你使用格式后缀，请务必为每个动作包含一个`format=None` 的关键字参数。
        """

        def list(self, request):
            pass

        def create(self, request):
            pass

        def retrieve(self, request, pk=None):
            pass

        def update(self, request, pk=None):
            pass

        def partial_update(self, request, pk=None):
            pass

        def destroy(self, request, pk=None):
            pass

如果你有需要被路由到的特别方法，你可以使用 `@detail_route` 或 `@list_route` 装饰器将它们标记为需要路由。

`@detail_route` 装饰器在其URL模式中包含 `pk` 用于需要单个实例的方法。The `@list_route` 装饰器用于对对象列表进行操作的方法。

例如：

```
from django.contrib.auth.models import User
from rest_framework import status
from rest_framework import viewsets
from rest_framework.decorators import detail_route, list_route
from rest_framework.response import Response
from myapp.serializers import UserSerializer, PasswordSerializer

class UserViewSet(viewsets.ModelViewSet):
    """
    一个提供标准动作的 viewset
    """
    queryset = User.objects.all()
    serializer_class = UserSerializer

    @detail_route(methods=['post'])
    def set_password(self, request, pk=None):
        user = self.get_object()
        serializer = PasswordSerializer(data=request.data)
        if serializer.is_valid():
            user.set_password(serializer.data['password'])
            user.save()
            return Response({'status': 'password set'})
        else:
            return Response(serializer.errors,
                            status=status.HTTP_400_BAD_REQUEST)

    @list_route()
    def recent_users(self, request):
        recent_users = User.objects.all().order('-last_login')

        page = self.paginate_queryset(recent_users)
        if page is not None:
            serializer = self.get_serializer(page, many=True)
            return self.get_paginated_response(serializer.data)

        serializer = self.get_serializer(recent_users, many=True)
        return Response(serializer.data)
```

装饰器可以另外获取为路由视图设置的额外参数。例如...

```
    @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf])
    def set_password(self, request, pk=None):
       ...
```

这些装饰器将默认路由 `GET` 请求，但也可以通过使用 `methods` 参数接受其他 HTTP 方法。例如：

```
    @detail_route(methods=['post', 'delete'])
    def unset_password(self, request, pk=None):
       ...
```

这两个新动作将在 urls `^users/{pk}/set_password/$` 和 `^users/{pk}/unset_password/$`上可用。

---

# API 参考

## ViewSet

`ViewSet` 继承自 `APIView`。你可以使用任何标准属性，如 `permission_classes`, `authentication_classes` 以便控制视图集上的 API 策略。

`ViewSet` 类不提供任何操作的实现。为了使用 `ViewSet` 类，你将重写该类并显式地定义动作实现。

## GenericViewSet

`GenericViewSet` 类继承自 `GenericAPIView`，并提供了 `get_object`， `get_queryset` 方法和其他通用视图基本行为的默认配置，但默认情况不包括任何操作。

In order to use a `GenericViewSet` class you'll override the class and either mixin the required mixin classes, or define the action implementations explicitly.

## ModelViewSet

The `ModelViewSet` class inherits from `GenericAPIView` and includes implementations for various actions, by mixing in the behavior of the various mixin classes.

The actions provided by the `ModelViewSet` class are `.list()`, `.retrieve()`,  `.create()`, `.update()`, `.partial_update()`, and `.destroy()`.

#### Example

Because `ModelViewSet` extends `GenericAPIView`, you'll normally need to provide at least the `queryset` and `serializer_class` attributes.  For example:

```
class AccountViewSet(viewsets.ModelViewSet):
    """
    A simple ViewSet for viewing and editing accounts.
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]
```

Note that you can use any of the standard attributes or method overrides provided by `GenericAPIView`.  For example, to use a `ViewSet` that dynamically determines the queryset it should operate on, you might do something like this:

```
class AccountViewSet(viewsets.ModelViewSet):
    """
    A simple ViewSet for viewing and editing the accounts
    associated with the user.
    """
    serializer_class = AccountSerializer
    permission_classes = [IsAccountAdminOrReadOnly]

    def get_queryset(self):
        return self.request.user.accounts.all()
```

Note however that upon removal of the `queryset` property from your `ViewSet`, any associated [router](routers.md) will be unable to derive the base\_name of your Model automatically, and so you will have to specify the `base_name` kwarg as part of your [router registration](routers.md).

Also note that although this class provides the complete set of create/list/retrieve/update/destroy actions by default, you can restrict the available operations by using the standard permission classes.

## ReadOnlyModelViewSet

The `ReadOnlyModelViewSet` class also inherits from `GenericAPIView`.  As with `ModelViewSet` it also includes implementations for various actions, but unlike `ModelViewSet` only provides the 'read-only' actions, `.list()` and `.retrieve()`.

#### Example

As with `ModelViewSet`, you'll normally need to provide at least the `queryset` and `serializer_class` attributes.  For example:

```
class AccountViewSet(viewsets.ReadOnlyModelViewSet):
    """
    A simple ViewSet for viewing accounts.
    """
    queryset = Account.objects.all()
    serializer_class = AccountSerializer
```

Again, as with `ModelViewSet`, you can use any of the standard attributes and method overrides available to `GenericAPIView`.

# Custom ViewSet base classes

You may need to provide custom `ViewSet` classes that do not have the full set of `ModelViewSet` actions, or that customize the behavior in some other way.

## Example

To create a base viewset class that provides `create`, `list` and `retrieve` operations, inherit from `GenericViewSet`, and mixin the required actions:

    class CreateListRetrieveViewSet(mixins.CreateModelMixin,
                                    mixins.ListModelMixin,
                                    mixins.RetrieveModelMixin,
                                    viewsets.GenericViewSet):
        """
        A viewset that provides `retrieve`, `create`, and `list` actions.

        To use it, override the class and set the `.queryset` and
        `.serializer_class` attributes.
        """
        pass

By creating your own base `ViewSet` classes, you can provide common behavior that can be reused in multiple viewsets across your API.

