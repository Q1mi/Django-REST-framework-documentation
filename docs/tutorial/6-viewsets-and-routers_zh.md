# Tutorial 6: 视图集和路由器

REST框架包括一个用于处理`ViewSets`的抽象，它允许开发人员集中精力对API的状态和交互进行建模，并根据常规约定自动处理URL构造。

`ViewSet`类与`View`类几乎相同，不同之处在于它们提供诸如`read`或`update`之类的操作，而不是`get`或`put`等方法处理程序。

最后一个`ViewSet`类只绑定到一组方法处理程序，当它被实例化成一组视图的时候，通常通过使用一个`Router`类来处理自己定义URL conf的复杂性。

## 使用ViewSets重构

我们看一下目前的视图，把它们重构成视图集。

首先让我们将`UserList`和`UserDetail`视图重构为一个`UserViewSet`。我们可以删除这两个视图，并用一个类替换它们：

    from rest_framework import viewsets

    class UserViewSet(viewsets.ReadOnlyModelViewSet):
        """
        此视图自动提供`list`和`detail`操作。
        """
        queryset = User.objects.all()
        serializer_class = UserSerializer

这里，我们使用`ReadOnlyModelViewSet`类来自动提供默认的“只读”操作。我们仍然像使用常规视图那样设置`queryset`和`serializer_class`属性，但我们不再需要向两个不同的类提供相同的信息。

接下来，我们将替换`SnippetList`，`SnippetDetail`和`SnippetHighlight`视图类。我们可以删除三个视图，并再次用一个类替换它们。

    from rest_framework.decorators import detail_route

    class SnippetViewSet(viewsets.ModelViewSet):
        """
        此视图自动提供`list`，`create`，`retrieve`，`update`和`destroy`操作。

        另外我们还提供了一个额外的`highlight`操作。
        """
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer
        permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                              IsOwnerOrReadOnly,)

        @detail_route(renderer_classes=[renderers.StaticHTMLRenderer])
        def highlight(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted)

        def perform_create(self, serializer):
            serializer.save(owner=self.request.user)

这次我们使用了`ModelViewSet`类来获取完整的默认读写操作。

请注意，我们还使用`@detail_route`装饰器创建一个名为`highlight`的自定义操作。这个装饰器可用于添加不符合标准`create`/`update`/`delete`样式的任何自定义路径。

默认情况下，使用`@detail_route`装饰器的自定义操作将响应`GET`请求。如果我们想要一个响应`POST`请求的动作，我们可以使用`methods`参数。

默认情况下，自定义操作的URL取决于方法名称本身。如果要更改URL的构造方式，可以为装饰器设置url_path关键字参数。

## 明确地将ViewSets绑定到URL

当我们定义URLConf时，处理程序方法只能绑定到那些动作上。
要了解在幕后发生的事情，首先显式地从我们的视图集中创建一组视图。

在`urls.py`文件中，我们将`ViewSet`类绑定到一组具体视图中。

    from snippets.views import SnippetViewSet, UserViewSet, api_root
    from rest_framework import renderers

    snippet_list = SnippetViewSet.as_view({
        'get': 'list',
        'post': 'create'
    })
    snippet_detail = SnippetViewSet.as_view({
        'get': 'retrieve',
        'put': 'update',
        'patch': 'partial_update',
        'delete': 'destroy'
    })
    snippet_highlight = SnippetViewSet.as_view({
        'get': 'highlight'
    }, renderer_classes=[renderers.StaticHTMLRenderer])
    user_list = UserViewSet.as_view({
        'get': 'list'
    })
    user_detail = UserViewSet.as_view({
        'get': 'retrieve'
    })

请注意，我们是如何通过将http方法绑定到每个视图所需的操作，从每个`ViewSet`类创建多个视图的。

现在我们将资源绑定到具体的视图中，我们可以像往常一样在URL conf中注册视图。

    urlpatterns = format_suffix_patterns([
        url(r'^$', api_root),
        url(r'^snippets/$', snippet_list, name='snippet-list'),
        url(r'^snippets/(?P<pk>[0-9]+)/$', snippet_detail, name='snippet-detail'),
        url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', snippet_highlight, name='snippet-highlight'),
        url(r'^users/$', user_list, name='user-list'),
        url(r'^users/(?P<pk>[0-9]+)/$', user_detail, name='user-detail')
    ])

## 使用路由器

因为我们使用的是`ViewSet`类而不是`View`类，我们实际上不需要自己设计URL。将资源连接到视图和url的约定可以使用`Router`类自动处理。我们需要做的就是使用路由器注册相应的视图集，然后让它执行其余操作。

这是我们重写的`urls.py`文件。

    from django.conf.urls import url, include
    from snippets import views
    from rest_framework.routers import DefaultRouter

    # 创建路由器并注册我们的视图。
    router = DefaultRouter()
    router.register(r'snippets', views.SnippetViewSet)
    router.register(r'users', views.UserViewSet)

    # API URL现在由路由器自动确定。
    # 另外，我们还要包含可浏览的API的登录URL。
    urlpatterns = [
        url(r'^', include(router.urls)),
        url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]

使用路由器注册viewsets类似于提供urlpattern。我们包含两个参数 - 视图的URL前缀和视图本身。

我们使用的`DefaultRouter`类也会自动为我们创建API根视图，因此我们现在可以从我们的`views`模块中删除`api_root`方法。

## 视图（views）vs视图集（viewsets）之间的权衡

使用视图集可以是一个非常有用的抽象。它有助于确保URL约定在你的API中保持一致，最大限度地减少编写所需的代码量，让你能够专注于API提供的交互和表示，而不是URLconf的细节。

这并不意味着采用视图集总是正确的方法。在使用基于类的视图而不是基于函数的视图时，有一个类似的权衡要考虑。使用视图集不像单独构建视图那样明确。

在[本教程的第7部分][tut-7]中，我们将介绍如何添加API模式，并使用客户端库或命令行工具与我们的API进行交互。

[tut-7]: 7-schemas-and-client-libraries_zh.md
