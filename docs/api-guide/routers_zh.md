URLsource: routers.py

# Routers

> 资源路由允许你快速声明给定的有足够控制器的所有公共路由。而不是为你的index...声明单独的路由，一个强大的路由能在一行代码中声明它们。
>
> &mdash; [Ruby on Rails 文档][cite]

某些Web框架（如Rails）提供了自动确定应用程序的URL应如何映射到处理传入请求的逻辑的功能。

REST框架添加了对自动URL路由到Django的支持，并为你提供了一种简单、快速和一致的方式来将视图逻辑连接到一组URL。

## Usage

这里有一个简单的URL conf的例子，它使用 `SimpleRouter`。

    from rest_framework import routers

    router = routers.SimpleRouter()
    router.register(r'users', UserViewSet)
    router.register(r'accounts', AccountViewSet)
    urlpatterns = router.urls

`register()` 方法有两个强制参数：

* `prefix` - 用于此组路由的URL前缀。
* `viewset` - 处理请求的viewset类。

还可以指定一个附加参数（可选）：

* `base_name` - 用于创建的URL名称的基本名称。如果不设置该参数，将根据视图集的`queryset`属性（如果有）来自动生成基本名称。注意，如果视图集不包括`queryset`属性，那么在注册视图集时必须设置`base_name`。

上面的示例将生成以下URL模式：

* URL pattern: `^users/$`  Name: `'user-list'`
* URL pattern: `^users/{pk}/$`  Name: `'user-detail'`
* URL pattern: `^accounts/$`  Name: `'account-list'`
* URL pattern: `^accounts/{pk}/$`  Name: `'account-detail'`

---

**注意**: `base_name` 参数用于指定视图名称模式的初始部分。在上面的例子中就是指 `user` 或 `account` 部分。

通常，你*不需要*指定`base_name`参数，但是如果你有自定义`get_queryset`方法的视图集，那么那个视图集可能没有设置`.queryset`属性。当你注册这个视图集的时候，你就有可能会看到类似如下的错误：

    'base_name' argument not specified, and could not automatically determine the name from the viewset, as it does not have a '.queryset' attribute.

这意味着你需要在注册视图集时显式设置`base_name`参数，因为无法从model名自动确定。

---

### 在路由中使用 `include`

路由器实例上的`.urls`属性只是一个URL模式的标准列表。对于如何添加这些URL，有很多不同的写法。

例如，你可以将`router.urls`附加到现有视图的列表中...

    router = routers.SimpleRouter()
    router.register(r'users', UserViewSet)
    router.register(r'accounts', AccountViewSet)
    
    urlpatterns = [
        url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
    ]
    
    urlpatterns += router.urls

或者，你可以使用Django的`include`函数，像这样...

    urlpatterns = [
        url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
        url(r'^', include(router.urls)),
    ]

路由器URL模式也支持命名空间的写法。

    urlpatterns = [
        url(r'^forgot-password/$', ForgotPasswordFormView.as_view()),
        url(r'^api/', include(router.urls, namespace='api')),
    ]

 如果使用带超链接序列化器的命名空间，你还需要确保序列化器上的任何`view_name`参数正确地反映命名空间。在上面的示例中，你需要让超链接到用户详细信息视图的序列化器字段包含一个参数，例如`view_name ='api：user-detail'`。

### 额外链接和操作

用`@detail_route`或`@list_route`装饰的视图集上的任何方法也将被路由。
例如，给定一个类似这样的方法在`UserViewSet`类：

    from myapp.permissions import IsAdminOrIsSelf
    from rest_framework.decorators import detail_route

    class UserViewSet(ModelViewSet):
        ...

        @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf])
        def set_password(self, request, pk=None):
            ...
将另外生成以下URL模式：

* URL pattern: `^users/{pk}/set_password/$`  Name: `'user-set-password'`

如果你不想让自定义的操作使用自动生成的默认URL，你可以改用url_path参数进行自定义。

例如，如果你要将自定义操作的URL更改为`^users/{pk}/change-password/$`, 你可以这样写：

    from myapp.permissions import IsAdminOrIsSelf
    from rest_framework.decorators import detail_route
    
    class UserViewSet(ModelViewSet):
        ...
        
        @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf], url_path='change-password')
        def set_password(self, request, pk=None):
            ...

以上示例将生成以下URL格式：

* URL pattern: `^users/{pk}/change-password/$`  Name: `'user-change-password'`

在你不想使用为自定义操作生成的默认名称的情况下，ni可以使用url_name参数来自定义它。

例如，如果要将自定义操作的名称更改为`'user-change-password'`，则可以写为：

    from myapp.permissions import IsAdminOrIsSelf
    from rest_framework.decorators import detail_route
    
    class UserViewSet(ModelViewSet):
        ...
        
        @detail_route(methods=['post'], permission_classes=[IsAdminOrIsSelf], url_name='change-password')
        def set_password(self, request, pk=None):
            ...

以上示例现在将生成以下URL格式：

* URL pattern: `^users/{pk}/set_password/$`  Name: `'user-change-password'`

你还可以同时设置url_path和url_name参数对自定义视图的URL生成进行额外的控制。

有关更多信息，请参阅viewset文档 [标记路由的额外操作][route-decorators].

# API 向导

## SimpleRouter

该路由器包括标准集合`list`, `create`, `retrieve`, `update`, `partial_update` 和 `destroy`动作的路由。视图集中还可以使用`@ detail_route`或`@ list_route`装饰器标记要被路由的其他方法。

<table border=1>
    <tr><th>URL 样式</th><th>HTTP 方法</th><th>动作</th><th>URL 名</th></tr>
    <tr><td rowspan=2>{prefix}/</td><td>GET</td><td>list</td><td rowspan=2>{basename}-list</td></tr></tr>
    <tr><td>POST</td><td>create</td></tr>
    <tr><td>{prefix}/{methodname}/</td><td>GET, or as specified by `methods` argument</td><td>`@list_route` decorated method</td><td>{basename}-{methodname}</td></tr>
    <tr><td rowspan=4>{prefix}/{lookup}/</td><td>GET</td><td>retrieve</td><td rowspan=4>{basename}-detail</td></tr></tr>
    <tr><td>PUT</td><td>update</td></tr>
    <tr><td>PATCH</td><td>partial_update</td></tr>
    <tr><td>DELETE</td><td>destroy</td></tr>
    <tr><td>{prefix}/{lookup}/{methodname}/</td><td>GET, or as specified by `methods` argument</td><td>`@detail_route` decorated method</td><td>{basename}-{methodname}</td></tr>
</table>

默认情况下，由`SimpleRouter`创建的URL将附加尾部斜杠。
在实例化路由器时，可以通过将`trailing_slash`参数设置为`False'来修改此行为。比如：

    router = SimpleRouter(trailing_slash=False)

尾部斜杠在Django中是常见的，但是在其他一些框架（如Rails）中默认不使用。你选择使用哪种风格在很大程度上是你个人偏好问题，虽然一些javascript框架可能需要一个特定的路由风格。

路由器将匹配包含除斜杠和句点字符以外的任何字符的查找值。对于更严格（或更宽松）的查找模式，请在视图集上设置`lookup_value_regex`属性。例如，你可以将查找限制为有效的UUID：

    class MyModelViewSet(mixins.RetrieveModelMixin, viewsets.GenericViewSet):
        lookup_field = 'my_model_id'
        lookup_value_regex = '[0-9a-f]{32}'

## DefaultRouter

这个路由器类似于上面的`SimpleRouter`，但是还包括一个默认返回所有列表视图的超链接的API根视图。它还生成可选的`.json`样式格式后缀的路由。

<table border=1>
    <tr><th>URL 样式</th><th>HTTP 方法</th><th>动作</th><th>URL 名称</th></tr>
    <tr><td>[.format]</td><td>GET</td><td>automatically generated root view</td><td>api-root</td></tr></tr>
    <tr><td rowspan=2>{prefix}/[.format]</td><td>GET</td><td>list</td><td rowspan=2>{basename}-list</td></tr></tr>
    <tr><td>POST</td><td>create</td></tr>
    <tr><td>{prefix}/{methodname}/[.format]</td><td>GET, or as specified by `methods` argument</td><td>`@list_route` decorated method</td><td>{basename}-{methodname}</td></tr>
    <tr><td rowspan=4>{prefix}/{lookup}/[.format]</td><td>GET</td><td>retrieve</td><td rowspan=4>{basename}-detail</td></tr></tr>
    <tr><td>PUT</td><td>update</td></tr>
    <tr><td>PATCH</td><td>partial_update</td></tr>
    <tr><td>DELETE</td><td>destroy</td></tr>
    <tr><td>{prefix}/{lookup}/{methodname}/[.format]</td><td>GET, or as specified by `methods` argument</td><td>`@detail_route` decorated method</td><td>{basename}-{methodname}</td></tr>
</table>

与`SimpleRouter`一样，在实例化路由器时，可以通过将`trailing_slash`参数设置为`False'来删除URL路由的尾部斜杠。

    router = DefaultRouter(trailing_slash=False)

# 自定义 Routers

通常你并不需要实现自定义路由器，但如果你对API的网址结构有特定的要求，那它就十分有用了。这样做允许你以可重用的方式封装URL结构，确保你不必为每个新视图显式地编写URL模式。

实现自定义路由器的最简单的方法是继承一个现有的路由器类。`.routes`属性用于模板将被映射到每个视图集的URL模式。`.routes`属性是一个名为tuples的Route对象的列表。

`Route`命名元组的参数是：

**url**: 表示要路由的URL的字符串。可能包括以下格式字符串：

* `{prefix}` - 用于此组路由的URL前缀。
* `{lookup}` - 用于与单个实例进行匹配的查找字段。
* `{trailing_slash}` - 可以是一个'/'或一个空字符串，这取决于`trailing_slash`参数。

**mapping**: HTTP方法名称到视图方法的映射

**name**: 在`reverse`调用中使用的URL的名称。可能包括以下格式字符串：

* `{basename}` - 用于创建的URL名称的基本名称

**initkwargs**: 实例化视图时应传递的任何其他参数的字典。注意，`suffix`参数被保留用于标识视图集类型，在生成视图名称和面包屑链接时使用。

## 自定义动态路由

你还可以定制`@ list_route`和`@detail_route`装饰器的路由。要路由这些装饰器中的一个或两个，请在`.routes`列表中包含一个`DynamicListRoute`和/或`DynamicDetailRoute`命名的元组。

`DynamicListRoute`和`DynamicDetailRoute`的参数是：

**url**: 表示要路由的URL的字符串。可以包括与“Route”相同的格式字符串，并且另外接受`{methodname}`和`{methodnamehyphen}`格式字符串。

**name**: 在`reverse`调用中使用的URL的名称。可能包括以下格式字符串：`{basename}`，`{methodname}`和`{methodnamehyphen}`。

**initkwargs**: 实例化视图时应传递的任何其他参数的字典。

## 例子

以下示例将只路由到`list`和`retrieve`操作，并且不使用尾部斜线约定。

    from rest_framework.routers import Route, DynamicDetailRoute, SimpleRouter

    class CustomReadOnlyRouter(SimpleRouter):
        """
        A router for read-only APIs, which doesn't use trailing slashes.
        """
        routes = [
            Route(
            	url=r'^{prefix}$',
            	mapping={'get': 'list'},
            	name='{basename}-list',
            	initkwargs={'suffix': 'List'}
            ),
            Route(
            	url=r'^{prefix}/{lookup}$',
                mapping={'get': 'retrieve'},
                name='{basename}-detail',
                initkwargs={'suffix': 'Detail'}
            ),
            DynamicDetailRoute(
            	url=r'^{prefix}/{lookup}/{methodnamehyphen}$',
            	name='{basename}-{methodnamehyphen}',
            	initkwargs={}
        	)
        ]

让我们来看看我们定义的`CustomReadOnlyRouter`为简单视图生成的路由。

`views.py`:

    class UserViewSet(viewsets.ReadOnlyModelViewSet):
        """
        A viewset that provides the standard actions
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer
        lookup_field = 'username'

        @detail_route()
        def group_names(self, request):
            """
            Returns a list of all the group names that the given
            user belongs to.
            """
            user = self.get_object()
            groups = user.groups.all()
            return Response([group.name for group in groups])

`urls.py`:

    router = CustomReadOnlyRouter()
    router.register('users', UserViewSet)
    urlpatterns = router.urls

将生成以下映射...

<table border=1>
    <tr><th>URL</th><th>HTTP 方法</th><th>动作</th><th>URL 名称</th></tr>
    <tr><td>/users</td><td>GET</td><td>list</td><td>user-list</td></tr>
    <tr><td>/users/{username}</td><td>GET</td><td>retrieve</td><td>user-detail</td></tr>
    <tr><td>/users/{username}/group-names</td><td>GET</td><td>group_names</td><td>user-group-names</td></tr>
</table


有关设置`.routes`属性的另一个示例，请参阅`SimpleRouter`类的源代码。

## 高级自定义路由器

如果你想提供完全自定义的行为，你可以覆写`BaseRouter`并写`get_urls（self）`方法。该方法应该检查注册的视图并返回一个URL模式列表。可以通过访问`self.registry`属性来检查注册的前缀，视图集和基本名元组。

你可能还想覆盖`get_default_base_name（self，viewset）`方法，或者在路由器注册你的视图时总是显式地设置`base_name`参数。

# 第三方包

还提供以下第三方软件包。

## DRF嵌套路由器

[drf-nested-routers 包][drf-nested-routers] 提供了使用嵌套资源的路由器和关系字段。

## ModelRouter (wq.db.rest)

The [wq.db 包][wq.db] 提供了一个使用`register_model()`API扩展`DefaultRouter`的高级[ModelRouter] [wq.db-router]类（和singleton实例）。很像Django的`admin.site.register`，`rest.router.register_model`唯一需要的参数是一个model类。url前缀，序列化器和视图集的合理默认值将从模型和全局配置中推断出来。

    from wq.db import rest
    from myapp.models import MyModel

    rest.router.register_model(MyModel)

## DRF-extensions

[`DRF-extensions`包][drf-extensions] 提供用于创建 [嵌套视图][drf-extensions-nested-viewsets]的[路由器][drf-extensions-routers],  具有 [可定制端点名称][drf-extensions-customizable-endpoint-names]的[集合级别控制器][drf-extensions-collection-level-controllers]。

[cite]: http://guides.rubyonrails.org/routing.html
[route-decorators]: viewsets.md#marking-extra-actions-for-routing
[drf-nested-routers]: https://github.com/alanjds/drf-nested-routers
[wq.db]: http://wq.io/wq.db
[wq.db-router]: http://wq.io/docs/router
[drf-extensions]: http://chibisov.github.io/drf-extensions/docs/
[drf-extensions-routers]: http://chibisov.github.io/drf-extensions/docs/#routers
[drf-extensions-nested-viewsets]: http://chibisov.github.io/drf-extensions/docs/#nested-routes
[drf-extensions-collection-level-controllers]: http://chibisov.github.io/drf-extensions/docs/#collection-level-controllers
[drf-extensions-customizable-endpoint-names]: http://chibisov.github.io/drf-extensions/docs/#controller-endpoint-name
