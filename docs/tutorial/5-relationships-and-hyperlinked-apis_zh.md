# Tutorial 5: 关系和超链接API

目前我们的API中的关系是用主键表示的。我们将通过使用超链接来提高我们API的内部联系。

## 为我们的API创建一个根路径

现在我们有'snippets'和'users'的路径，但是我们的API没有一个入口点。我们将使用一个常规的基于函数的视图和我们前面介绍的`@api_view`装饰器创建一个。在你的`snippets/views.py`中添加：

    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from rest_framework.reverse import reverse


    @api_view(['GET'])
    def api_root(request, format=None):
        return Response({
            'users': reverse('user-list', request=request, format=format),
            'snippets': reverse('snippet-list', request=request, format=format)
        })

这里应该注意两件事。首先，我们使用REST框架的`reverse`功能来返回完全限定的URL；第二，URL模式是通过方便的名称来标识的，我们稍后将在`snippets/urls.py`中声明。

## 为高亮显示的代码片段创建路径

我们的API中另一个明显缺少的是代码高亮显示路径。

与所有其他API路径不同，我们不想使用JSON，而只是需要HTML表示。REST框架提供了两种HTML渲染器，一种用于处理使用模板渲染的HTML，另一种用于处理预渲染的HTML。第二个渲染器是我们要用于此路径的渲染器。

创建代码高亮视图时需要考虑的另一件事是，我们没有可用的具体通用视图。我们不是返回对象实例，而是返回对象实例的属性。

不是使用具体的通用视图，我们将使用基类来表示实例，并创建我们自己的`.get()`方法。在你的`snippets/views.py`中添加：

    from rest_framework import renderers
    from rest_framework.response import Response

    class SnippetHighlight(generics.GenericAPIView):
        queryset = Snippet.objects.all()
        renderer_classes = (renderers.StaticHTMLRenderer,)

        def get(self, request, *args, **kwargs):
            snippet = self.get_object()
            return Response(snippet.highlighted)

像往常一样，我们需要在URLconf中添加我们创建的新视图。我们将在`snippets/urls.py`中为我们的新API根路径添加一个url模式：

    url(r'^$', views.api_root),

然后为高亮代码片段添加一个url模式：

    url(r'^snippets/(?P<pk>[0-9]+)/highlight/$', views.SnippetHighlight.as_view()),

## 超链接我们的API

处理好实体之间的关系是Web API设计中更具挑战性的方面。我们可以选择几种不同的方式来代表一种关系：

* 使用主键。
* 在实体之间使用超链接。
* 在相关实体上使用唯一的标识字段。
* 使用相关实体的默认字符串表示形式。
* 将相关实体嵌套在父表示中。
* 一些其他自定义表示。

REST框架支持所有这些方式，并且可以将它们应用于正向或反向关系，也可以在诸如通用外键之类的自定义管理器上应用。

在这种情况下，我们希望在实体之间使用超链接方式。这样的话，我们需要修改我们的序列化程序来扩展`HyperlinkedModelSerializer`而不是现有的`ModelSerializer`。

`HyperlinkedModelSerializer`与`ModelSerializer`有以下区别：

* 默认情况下不包括`id`字段。
* 它包含一个`url`字段，使用`HyperlinkedIdentityField`。
* 关联关系使用`HyperlinkedRelatedField`，而不是`PrimaryKeyRelatedField`。

我们可以轻松地重写我们现有的序列化程序以使用超链接。在你的`snippets/serializers.py`中添加：

    class SnippetSerializer(serializers.HyperlinkedModelSerializer):
        owner = serializers.ReadOnlyField(source='owner.username')
        highlight = serializers.HyperlinkedIdentityField(view_name='snippet-highlight', format='html')

        class Meta:
            model = Snippet
            fields = ('url', 'id', 'highlight', 'owner',
                      'title', 'code', 'linenos', 'language', 'style')


    class UserSerializer(serializers.HyperlinkedModelSerializer):
        snippets = serializers.HyperlinkedRelatedField(many=True, view_name='snippet-detail', read_only=True)

        class Meta:
            model = User
            fields = ('url', 'id', 'username', 'snippets')

请注意，我们还添加了一个新的`'highlight'`字段。该字段与`url`字段的类型相同，不同之处在于它指向`'snippet-highlight'`url模式，而不是`'snippet-detail'`url模式。

因为我们已经包含了格式后缀的URL，例如`'.json'`，我们还需要在`highlight`字段上指出任何格式后缀的超链接，它应该使用`'.html'`后缀。

## 确保我们的URL模式被命名

如果我们要使用超链接的API，那么需要确保为我们的URL模式命名。我们来看看我们需要命名的URL模式。

* 我们API的根路径是指`'user-list'`和`'snippet-list'`。
* 我们的代码片段序列化器包含一个指向`'snippet-highlight'`的字段。
* 我们的用户序列化器包含一个指向`'snippet-detail'`的字段。
* 我们的代码片段和用户序列化程序包括`'url'`字段，默认情况下将指向`'{model_name}-detail'`，在这个例子中就是`'snippet-detail'`和`'user-detail'`。

将所有这些名称添加到我们的URLconf中后，最终我们的`snippets/urls.py`文件应该如下所示：

    from django.conf.urls import url, include
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    # API endpoints
    urlpatterns = format_suffix_patterns([
        url(r'^$', views.api_root),
        url(r'^snippets/$',
            views.SnippetList.as_view(),
            name='snippet-list'),
        url(r'^snippets/(?P<pk>[0-9]+)/$',
            views.SnippetDetail.as_view(),
            name='snippet-detail'),
        url(r'^snippets/(?P<pk>[0-9]+)/highlight/$',
            views.SnippetHighlight.as_view(),
            name='snippet-highlight'),
        url(r'^users/$',
            views.UserList.as_view(),
            name='user-list'),
        url(r'^users/(?P<pk>[0-9]+)/$',
            views.UserDetail.as_view(),
            name='user-detail')
    ])

    # 可浏览API的登录和注销视图
    urlpatterns += [
        url(r'^api-auth/', include('rest_framework.urls',
                                   namespace='rest_framework')),
    ]

## 添加分页

用户和代码片段的列表视图可能会返回相当多的实例，因此我们希望确保对结果分页，并允许API客户端依次获取每个单独的页面。

我们可以通过稍微修改我们的`tutorial/settings.py`文件来更改默认列表展示样式来使用分页。添加以下设置：

    REST_FRAMEWORK = {
        'PAGE_SIZE': 10
    }

请注意，REST框架中的所有设置都放在一个名为“REST_FRAMEWORK”的字典中，这有助于区分项目中的其他设置。

如果需要的话，我们也可以自定义分页风格，但在这个例子中，我们将一直使用默认设置。

## 浏览API

如果我们打开浏览器并浏览我们的API，那么你可以简单的通过页面上的超链接来了解API。

你还可以看到代码片段实例上的'highlight'链接，它能带你跳转到高亮显示的代码HTML表示。

在[教程的第6部分][tut-6]我们将介绍如何使用ViewSets和Routers来减少构建API所需的代码量。

[tut-6]: 6-viewsets-and-routers_zh.md
