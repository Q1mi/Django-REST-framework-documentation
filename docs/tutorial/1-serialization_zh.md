# Tutorial 1: 序列化

## 介绍

本教程将介绍如何创建一个简单的收录代码高亮展示的Web API。这个过程中, 将会介绍组成Rest框架的各个组件，并让你全面了解各个组件是如何一起工作的。

这个教程是相当深入的，所以你应该在开始之前准备些啤酒和花生毛豆什么的。如果你只是想来个快速预览，那你应该查看[quickstart]文档。

---

**注意**：本教程的代码可以在Github的[tomchristie/rest-framework-tutorial][repo]库中找到。完整的实现版本可以[在线][sandbox]查看作为一个沙盒版本进行测试。

---

## 创建一个新环境

在做其他事情之前，我们要用[virtualenv]创建一个新的虚拟环境。这样就能确保我们的包配置与我们正在开展的任何其他项目保持良好的隔离。

    virtualenv env
    source env/bin/activate

现在我们在一个virtualenv环境中，我们可以安装我们需要的包了。

    pip install django
    pip install djangorestframework
    pip install pygments  # 代码高亮插件

**注意：** 要随时退出virtualenv环境，只需输入`deactivate`。有关更多信息，请参阅[virtualenv documentation][virtualenv]文档。

## 正式开始

好了，我们现在要开始写代码了。
首先，我们来创建一个新的项目。

    cd ~
    django-admin.py startproject tutorial
    cd tutorial

完成上面的步骤以后我们要创建一个用于创建简单Web API的app。

    python manage.py startapp snippets

我们需要将新建的`snippets`app和`rest_framework`app添加到`INSTALLED_APPS`。让我们编辑`tutorial/settings.py`文件：

    INSTALLED_APPS = (
        ...
        'rest_framework',
        'snippets.apps.SnippetsConfig',
    )

请注意如果你使用的Django版本低于1.9，你需要使用`snippets.apps.SnippetsConfig`替换`snippets`。

好了，我们的准备工作做完了。

## 创建一个model

为了实现本教程的目的，我们将开始创建一个用于存储代码片段的简单的`Snippet` model。然后继续编辑`snippets/models.py`文件。注意：优秀的编程实践都会添加注释。虽然你将在本教程代码的存储库版本中找到它们，但我们在此忽略了它们，专注于代码本身。

    from django.db import models
    from pygments.lexers import get_all_lexers
    from pygments.styles import get_all_styles

    LEXERS = [item for item in get_all_lexers() if item[1]]
    LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
    STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


    class Snippet(models.Model):
        created = models.DateTimeField(auto_now_add=True)
        title = models.CharField(max_length=100, blank=True, default='')
        code = models.TextField()
        linenos = models.BooleanField(default=False)
        language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
        style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

        class Meta:
            ordering = ('created',)

我们还需要为我们的代码段模型创建初始迁移（initial migration），并首次同步数据库（migrate）。

    python manage.py makemigrations snippets
    python manage.py migrate

## 创建一个序列化类

开发我们的Web API的第一件事是为我们的Web API提供一种将代码片段实例序列化和反序列化为诸如`json`之类的表示形式的方式。我们可以通过声明与Django forms非常相似的序列化器（serializers）来实现。 在`snippets`的目录下创建一个名为`serializers.py`文件，并添加以下内容。

    from rest_framework import serializers
    from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


    class SnippetSerializer(serializers.Serializer):
        id = serializers.IntegerField(read_only=True)
        title = serializers.CharField(required=False, allow_blank=True, max_length=100)
        code = serializers.CharField(style={'base_template': 'textarea.html'})
        linenos = serializers.BooleanField(required=False)
        language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
        style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

        def create(self, validated_data):
            """
            根据提供的验证过的数据创建并返回一个新的`Snippet`实例。
            """
            return Snippet.objects.create(**validated_data)

        def update(self, instance, validated_data):
            """
            根据提供的验证过的数据更新和返回一个已经存在的`Snippet`实例。
            """
            instance.title = validated_data.get('title', instance.title)
            instance.code = validated_data.get('code', instance.code)
            instance.linenos = validated_data.get('linenos', instance.linenos)
            instance.language = validated_data.get('language', instance.language)
            instance.style = validated_data.get('style', instance.style)
            instance.save()
            return instance

序列化器类的第一部分定义了序列化/反序列化的字段。`create()`和`update()`方法定义了在调用`serializer.save()`时如何创建和修改完整的实例。

序列化器类与Django `Form`类非常相似，并在各种字段中包含类似的验证标志，例如`required`，`max_length`和`default`。

字段标志还可以控制serializer在某些情况下如何显示，比如渲染HTML的时候。上面的`{'base_template': 'textarea.html'}`标志等同于在Django `Form`类中使用`widget=widgets.Textarea`。这对于控制如何显示可浏览器浏览的API特别有用，我们将在本教程的后面看到。

我们实际上也可以通过使用`ModelSerializer`类来节省时间，就像我们后面会用到的那样。但是现在我们还继续使用我们明确定义的serializer。

## 使用序列化类

在我们进一步了解之前，我们先来熟悉使用我们新的Serializer类。输入下面的命令进入Django shell。

    python manage.py shell

好的，像下面一样导入几个模块，然后开始创建一些代码片段来处理。

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser

    snippet = Snippet(code='foo = "bar"\n')
    snippet.save()

    snippet = Snippet(code='print "hello, world"\n')
    snippet.save()

我们现在已经有几个片段实例了，让我们看一下将其中一个实例序列化。

    serializer = SnippetSerializer(snippet)
    serializer.data
    # {'id': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}

此时，我们将模型实例转换为Python原生数据类型。要完成序列化过程，我们将数据转换成`json`。

    content = JSONRenderer().render(serializer.data)
    content
    # '{"id": 2, "title": "", "code": "print \\"hello, world\\"\\n", "linenos": false, "language": "python", "style": "friendly"}'

反序列化是类似的。首先我们将一个流（stream）解析为Python原生数据类型...

    from django.utils.six import BytesIO

    stream = BytesIO(content)
    data = JSONParser().parse(stream)

...然后我们要将Python原生数据类型恢复成正常的对象实例。

    serializer = SnippetSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.validated_data
    # OrderedDict([('title', ''), ('code', 'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
    serializer.save()
    # <Snippet: Snippet object>

可以看到API和表单(forms)是多么相似。当我们开始使用我们的序列化类编写视图的时候，相似性会变得更加明显。

我们也可以序列化查询结果集（querysets）而不是模型实例。我们只需要为serializer添加一个`many=True`标志。

    serializer = SnippetSerializer(Snippet.objects.all(), many=True)
    serializer.data
    # [OrderedDict([('id', 1), ('title', u''), ('code', u'foo = "bar"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 2), ('title', u''), ('code', u'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')]), OrderedDict([('id', 3), ('title', u''), ('code', u'print "hello, world"'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])]

## 使用ModelSerializers

我们的`SnippetSerializer`类中重复了很多包含在`Snippet`模型类（model）中的信息。如果能保证我们的代码整洁，那就更好了。

就像Django提供了`Form`类和`ModelForm`类一样，REST framework包括`Serializer`类和`ModelSerializer`类。

我们来看看使用`ModelSerializer`类重构我们的序列化类。再次打开`snippets/serializers.py`文件，并将`SnippetSerializer`类替换为以下内容。


    class SnippetSerializer(serializers.ModelSerializer):
        class Meta:
            model = Snippet
            fields = ('id', 'title', 'code', 'linenos', 'language', 'style')

序列一个非常棒的属性就是可以通过打印序列化器类实例的结构(representation)查看它的所有字段。

    from snippets.serializers import SnippetSerializer
    serializer = SnippetSerializer()
    print(repr(serializer))
    # SnippetSerializer():
    #    id = IntegerField(label='ID', read_only=True)
    #    title = CharField(allow_blank=True, max_length=100, required=False)
    #    code = CharField(style={'base_template': 'textarea.html'})
    #    linenos = BooleanField(required=False)
    #    language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
    #    style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...

重要的是要记住，`ModelSerializer`类并不会做任何特别神奇的事情，它们只是创建序列化器类的快捷方式：

* 一组自动确定的字段。
* 默认简单实现的`create()`和`update()`方法。

## 使用我们的序列化（Serializer）来编写常规的Django视图（views）

让我们看看如何使用我们新的Serializer类编写一些API视图。目前我们不会使用任何REST框架的其他功能，我们只需将视图作为常规Django视图编写。

编辑`snippets/views.py`文件，并且添加以下内容。

    from django.http import HttpResponse
    from django.views.decorators.csrf import csrf_exempt
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer

    class JSONResponse(HttpResponse):
        """
        An HttpResponse that renders its content into JSON.
        """
        def __init__(self, data, **kwargs):
            content = JSONRenderer().render(data)
            kwargs['content_type'] = 'application/json'
            super(JSONResponse, self).__init__(content, **kwargs)

我们API的根视图支持列出所有现有的snippet或创建一个新的snippet。

    @csrf_exempt
    def snippet_list(request):
        """
        列出所有的code snippet，或创建一个新的snippet。
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return JSONResponse(serializer.data)

        elif request.method == 'POST':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data, status=201)
            return JSONResponse(serializer.errors, status=400)

请注意，因为我们希望能够从不具有CSRF令牌的客户端对此视图进行POST，因此我们需要将视图标记为`csrf_exempt`。这不是你通常想要做的事情，并且REST框架视图实际上比这更实用的行为，但它现在足够达到我们的目的。

我们也需要一个与单个snippet对象相应的视图，并且用于获取，更新和删除这个snippet。

    @csrf_exempt
    def snippet_detail(request, pk):
        """
        获取，更新或删除一个 code snippet。
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return HttpResponse(status=404)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return JSONResponse(serializer.data)

        elif request.method == 'PUT':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(snippet, data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data)
            return JSONResponse(serializer.errors, status=400)

        elif request.method == 'DELETE':
            snippet.delete()
            return HttpResponse(status=204)

最终，我们需要把这些视图连起来。创建一个`snippets/urls.py`文件：

    from django.conf.urls import url
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.snippet_list),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
    ]

我们也需要在根URL配置`tutorial/urls.py`文件中，添加我们的snippet应用的URL。

    from django.conf.urls import url, include

    urlpatterns = [
        url(r'^', include('snippets.urls')),
    ]

值得注意的是，目前我们还没有正确处理好几个边缘事项。如果我们发送格式错误的`json`，或者如果使用视图不处理的方法发出请求，那么我们最终会出现一个500“服务器错误”响应。不过，现在这样做。

## 测试在我们Web API进行第一次尝试

现在为我们的snippets启动一个服务器。

退出所有的shell...

    quit()

...启动Django的开发服务器。

    python manage.py runserver

    Validating models...

    0 errors found
    Django version 1.8.3, using settings 'tutorial.settings'
    Development server is running at http://127.0.0.1:8000/
    Quit the server with CONTROL-C.

在另一个终端窗口中，我们可以测试服务器。

我们可以使用[curl][curl]或[httpie][httpie]测试我们的服务器。Httpie是用Python编写的用户友好的http客户端，我们安装它。

你可以使用pip来安装httpie：

    pip install httpie

最后，我们可以得到所有snippet的列表：

    http http://127.0.0.1:8000/snippets/

    HTTP/1.1 200 OK
    ...
    [
      {
        "id": 1,
        "title": "",
        "code": "foo = \"bar\"\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
      },
      {
        "id": 2,
        "title": "",
        "code": "print \"hello, world\"\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
      }
    ]

或者我们可以通过引用其id来获取特定的snippet：

    http http://127.0.0.1:8000/snippets/2/

    HTTP/1.1 200 OK
    ...
    {
      "id": 2,
      "title": "",
      "code": "print \"hello, world\"\n",
      "linenos": false,
      "language": "python",
      "style": "friendly"
    }

同样，你可以通过在网络浏览器中访问这些URL来显示相同​​的json。

## 我们现在的位置

我们到目前为止做得不错，我们有一个与Django的Forms API非常相似序列化API，和一些常规的Django视图。

现在，我们的API视图除了服务`json`响应外，不会做任何其他特别的东西，并且有一些错误我们仍然需要清处理，但是它仍是一个可用的Web API。

我们将在[本教程的第2部分][tut-2]来完善这些东西。

[quickstart]: quickstart_zh.md
[repo]: https://github.com/tomchristie/rest-framework-tutorial
[sandbox]: http://restframework.herokuapp.com/
[virtualenv]: http://www.virtualenv.org/en/latest/index.html
[tut-2]: 2-requests-and-responses_zh.md
[httpie]: https://github.com/jakubroztocil/httpie#installation
[curl]: http://curl.haxx.se
