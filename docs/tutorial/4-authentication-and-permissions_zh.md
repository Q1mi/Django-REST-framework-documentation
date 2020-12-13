# Tutorial 4: 认证和权限

目前，我们的API对谁可以编辑或删除代码段没有任何限制。我们希望有更高级的行为，以确保：

* 代码片段始终与创建者相关联。
* 只有通过身份验证的用户可以创建片段。
* 只有代码片段的创建者可以更新或删除它。
* 未经身份验证的请求应具有完全只读访问权限。

## 在我们的模型（model）中添加信息

我们将对我们的`Snippet`模型类进行几次更改。首先，我们添加几个字段。其中一个字段将用于表示创建代码段的用户，另一个字段将用于存储代码的高亮显示的HTML内容。

将以下两个字段添加到`models.py`文件中的`Snippet`模型中。

    owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
    highlighted = models.TextField()

我们还需要确保在保存模型时，使用`pygments`代码高亮显示库填充要高亮显示的字段。

我们需要导入额外的模块：

    from pygments.lexers import get_lexer_by_name
    from pygments.formatters.html import HtmlFormatter
    from pygments import highlight

现在我们可以在我们的模型类中添加一个`.save()`方法：

    def save(self, *args, **kwargs):
        """
        使用`pygments`库创建一个高亮显示的HTML表示代码段。
        """
        lexer = get_lexer_by_name(self.language)
        linenos = self.linenos and 'table' or False
        options = self.title and {'title': self.title} or {}
        formatter = HtmlFormatter(style=self.style, linenos=linenos,
                                  full=True, **options)
        self.highlighted = highlight(self.code, lexer, formatter)
        super(Snippet, self).save(*args, **kwargs)

完成这些工作后，我们需要更新我们的数据库表。
通常这种情况我们会创建一个数据库迁移（migration）来实现这一点，但现在我们只是个教程示例，所以我们选择直接删除数据库并重新开始。

    rm -f tmp.db db.sqlite3
    rm -r snippets/migrations
    python manage.py makemigrations snippets
    python manage.py migrate

你可能还需要创建几个不同的用户，以用于测试API。最快的方法是使用`createsuperuser`命令。

    python manage.py createsuperuser

## 为我们的用户模型添加路径

现在我们已经创建了一些用户，我们最好在API中添加这些用户的表示。创建一个新的序列化器非常简单，在`serializers.py`文件中添加：

    from django.contrib.auth.models import User

    class UserSerializer(serializers.ModelSerializer):
        snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())

        class Meta:
            model = User
            fields = ('id', 'username', 'snippets')

因为`'snippets'` 在用户模型中是一个*反向*关联关系。在使用 `ModelSerializer` 类时它默认不会被包含，所以我们需要为它添加一个显式字段。

我们还会在`views.py`中添加几个视图。我们只想将用户展示为只读视图，因此我们将使用`ListAPIView`和`RetrieveAPIView`通用的基于类的视图。

    from django.contrib.auth.models import User


    class UserList(generics.ListAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer


    class UserDetail(generics.RetrieveAPIView):
        queryset = User.objects.all()
        serializer_class = UserSerializer

确保导入了`UserSerializer`类

    from snippets.serializers import UserSerializer

最后，我们需要通过在URL conf中引用它们来将这些视图添加到API中。将以下内容添加到`urls.py`文件的patterns中。

    url(r'^users/$', views.UserList.as_view()),
    url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()),

## 将Snippet和用户关联

现在，如果我们创建了一个代码片段，并不能将创建该代码片段的用户与代码段实例相关联。用户不是作为序列化表示的一部分发送的，而是作为传入请求的属性。（译者注：user不在传过来的数据中，而是通过request.user获得）

我们处理的方式是在我们的代码片段视图中重写一个`.perform_create()`方法，这样我们可以修改实例保存的方法，并处理传入请求或请求URL中隐含的任何信息。

在`SnippetList`视图类中，添加以下方法：

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)

我们的序列化器的`create()`方法现在将被传递一个附加的`'owner'`字段以及来自请求的验证数据。

## 更新我们的序列化器

现在，这些代码片段和创建它们的用户相关联，让我们更新我们的`SnippetSerializer`来体现这个关联关系。将以下字段添加到`serializers.py`中的序列化器定义：
 Add the following field to the serializer definition in `serializers.py`:

    owner = serializers.ReadOnlyField(source='owner.username')

**注意**：确保你还将`'owner',`添加到内部`Meta`类的字段列表中。

这个字段非常有趣。`source`参数控制哪个属性用于填充字段，并且可以指向序列化实例上的任何属性。它也可以采用如上所示点加下划线的方式，在这种情况下，它将以与Django模板语言一起使用的相似方式遍历给定的属性。

我们添加的字段是无类型的`ReadOnlyField`类，区别于其他类型的字段（如`CharField`，`BooleanField`等）。无类型的`ReadOnlyField`始终是只读的，只能用于序列化表示，不能用于在反序列化时更新模型实例。我们也可以在这里使用`CharField(read_only=True)`。

## 添加视图所需的权限

现在，代码片段与用户是相关联的，我们希望确保只有经过身份验证的用户才能创建，更新和删除代码片段。

REST框架包括许多权限类，我们可以使用这些权限类来限制谁可以访问给定的视图。
在这种情况下，我们需要的是`IsAuthenticatedOrReadOnly`类，这将确保经过身份验证的请求获得读写访问权限，未经身份验证的请求将获得只读访问权限。

首先要在视图模块中导入以下内容

    from rest_framework import permissions

然后，将以下属性添加到`SnippetList`**和**`SnippetDetail`视图类中。

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,)

## 给Browsable API添加登陆

如果你打开浏览器并浏览我们的API，那么你会发现不能创建新的代码片段。只有登陆用户才能创建新的代码片段。

我们可以通过编辑项目级别的`urls.py`文件中的URLconf来添加可浏览的API使用的登录视图。

在文件顶部添加以下导入：

    from django.conf.urls import include

而且，在文件末尾添加一个模式（pattern）以包括可浏览的API的登录和注销视图。

    urlpatterns += [
        url(r'^api-auth/', include('rest_framework.urls',
                                   namespace='rest_framework')),
    ]

模式的`r'^api-auth/'`部分实际上可以是你要使用的任何URL。唯一的限制是包含的URL必须使用`'rest_framework'`命名空间。在Django 1.9以上的版本中，REST框架将设置命名空间，因此你可以将其删除。

现在，如果你再次打开浏览器并刷新页面，你将在页面右上角看到一个“登录”链接。如果你用早期创建的用户登录，就可以再次创建代码片段。

一旦你创建了一些代码片段后，在'/users/'路径下你会注意到每个用户的'snippets'字段都包含与每个用户相关联的代码片段的列表。


## 对象级别的权限

我们希望所有的代码片段都可以被任何人看到，但也要确保只有创建代码片段的用户才能更新或删除它。

为此，我们将需要创建一个自定义权限。

在snippets app中，创建一个新文件`permissions.py`。

    from rest_framework import permissions


    class IsOwnerOrReadOnly(permissions.BasePermission):
        """
        自定义权限只允许对象的所有者编辑它。
        """

        def has_object_permission(self, request, view, obj):
            # 读取权限允许任何请求，
            # 所以我们总是允许GET，HEAD或OPTIONS请求。
            if request.method in permissions.SAFE_METHODS:
                return True

            # 只有该snippet的所有者才允许写权限。
            return obj.owner == request.user

现在，我们可以通过在`SnippetDetail`视图类中编辑`permission_classes`属性将该自定义权限添加到我们的代码片段实例路径：

    permission_classes = (permissions.IsAuthenticatedOrReadOnly,
                          IsOwnerOrReadOnly,)

确保要先导入`IsOwnerOrReadOnly`类。

    from snippets.permissions import IsOwnerOrReadOnly

现在，如果再次打开浏览器，你会发现如果你以代码片段创建者的身份登录的话，“DELETE”和“PUT”操作才会显示在代码片段实例路径上。

## 使用API​​进行身份验证

现在因为我们在API上有一组权限，如果我们要编辑任何代码片段，我们都需要验证我们的请求。我们还没有设置任何[身份验证类][authentication]，所以应用的是默认的`SessionAuthentication`和`BasicAuthentication`。

当我们通过Web浏览器与API进行交互时，我们可以登录，然后浏览器会话将为请求提供所需的身份验证。

如果我们在代码中与API交互，我们需要在每次请求上显式提供身份验证凭据。

如果我们通过没有验证就尝试创建一个代码片段，我们会像下面展示的那样收到报错：

    http POST http://127.0.0.1:8000/snippets/ code="print 123"

    {
        "detail": "Authentication credentials were not provided."
    }

我们可以通过加上我们之前创建的一个用户的用户名和密码来成功创建：

    http -a tom:password123 POST http://127.0.0.1:8000/snippets/ code="print 789"

    {
        "id": 5,
        "owner": "tom",
        "title": "foo",
        "code": "print 789",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

## 总结

我们现在已经在我们的Web API上获得了相当精细的一组权限控制，并为系统的用户和他们创建的代码片段提供了API路径。

在[本教程的第5部分][tut-5]中，我们将介绍如何通过为高亮显示的代码片段创建一个HTML路径来将所有内容联系起来，并在系统内部通过使用超链接将API联系起来。

[authentication]: ../api-guide/authentication_zh.md
[tut-5]: 5-relationships-and-hyperlinked-apis_zh.md
