# Tutorial 2: 请求和响应

从现在开始，我们将真正开始接触REST框架的核心。
我们来介绍几个基本的构建模块。

## 请求对象（Request objects）

REST框架引入了一个扩展了常规`HttpRequest`的`Request`对象，并提供了更灵活的请求解析。`Request`对象的核心功能是`request.data`属性，它与`request.POST`类似，但对于使用Web API更为有用。

    request.POST  # 只处理表单数据  只适用于'POST'方法
    request.data  # 处理任意数据  适用于'POST'，'PUT'和'PATCH'方法

## 响应对象（Response objects）

REST框架还引入了一个`Response`对象，这是一种获取未渲染（unrendered）内容的`TemplateResponse`类型，并使用内容协商来确定返回给客户端的正确内容类型。

    return Response(data)  # 渲染成客户端请求的内容类型。

## 状态码（Status codes）

在你的视图（views）中使用纯数字的HTTP 状态码并不总是那么容易被理解。而且如果错误代码出错，很容易被忽略。REST框架为`status`模块中的每个状态代码（如`HTTP_400_BAD_REQUEST`）提供更明确的标识符。使用它们来代替纯数字的HTTP状态码是个很好的主意。

## 包装（wrapping）API视图

REST框架提供了两个可用于编写API视图的包装器（wrappers）。

1. 用于基于函数视图的`@api_view`装饰器。
2. 用于基于类视图的`APIView`类。

这些包装器提供了一些功能，例如确保你在视图中接收到`Request`实例，并将上下文添加到`Response`，以便可以执行内容协商。

包装器还提供了诸如在适当时候返回`405 Method Not Allowed`响应，并处理在使用格式错误的输入来访问`request.data`时发生的任何`ParseError`异常。

## 组合在一起

好的，我们开始使用这些新的组件来写几个视图。

我们在`views.py`中不再需要`JSONResponse`类了，所以把它删除掉。删除之后，我们就可以开始重构我们的视图了。

    from rest_framework import status
    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer


    @api_view(['GET', 'POST'])
    def snippet_list(request):
        """
        列出所有的snippets，或者创建一个新的snippet。
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        elif request.method == 'POST':
            serializer = SnippetSerializer(data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

我们的实例视图比前面的示例有所改进。它稍微简洁一点，现在的代码与我们使用Forms API时非常相似。我们还使用了命名状态代码，这使得响应意义更加明显。

以下是`views.py`模块中单个snippet的视图。

    @api_view(['GET', 'PUT', 'DELETE'])
    def snippet_detail(request, pk):
        """
        获取，更新或删除一个snippet实例。
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return Response(status=status.HTTP_404_NOT_FOUND)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        elif request.method == 'PUT':
            serializer = SnippetSerializer(snippet, data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        elif request.method == 'DELETE':
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

这对我们来说应该都是非常熟悉的，-它和正常Django视图并没有什么不同。

注意，我们不再显式地将请求或响应绑定到给定的内容类型。`request.data`可以处理传入的`json`请求，但它也可以处理其他格式。同样，我们返回带有数据的响应对象，但允许REST框架将响应给我们渲染成正确的内容类型。

## 给我们的网址添加可选的格式后缀

为了充分利用我们的响应不再与单一内容类型连接，我们可以为API路径添加对格式后缀的支持。使用格式后缀给我们明确指定了给定格式的URL，这意味着我们的API将能够处理诸如[http://example.com/api/items/4.json][json-url]之类的URL。

像下面这样在这两个视图中添加一个`format`关键字参数。

    def snippet_list(request, format=None):

和

    def snippet_detail(request, pk, format=None):

现在更新`urls.py`文件，给现有的URL后面添加一组`format_suffix_patterns`。

    from django.conf.urls import url
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.snippet_list),
        url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns)

我们不一定需要添加这些额外的url模式，但它给了我们一个简单，清晰的方式来引用特定的格式。

## 怎么查看结果？

从命令行开始测试API，就像我们在[教程第一部分][tut-1]中所做的那样。一切操作都很相似，尽管我们发送无效的请求也会有一些更好的错误处理了。

我们可以像以前一样获取所有snippet的列表。

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

我们可以通过使用`Accept`标头来控制我们回复的响应格式：

    http http://127.0.0.1:8000/snippets/ Accept:application/json  # 请求JSON
    http http://127.0.0.1:8000/snippets/ Accept:text/html         # 请求HTML

或者通过附加格式后缀：

    http http://127.0.0.1:8000/snippets.json  # JSON后缀
    http http://127.0.0.1:8000/snippets.api   # 浏览器可浏览API后缀

类似地，我们可以使用`Content-Type`头控制我们发送的请求的格式。

    # POST表单数据
    http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

    {
      "id": 3,
      "title": "",
      "code": "print 123",
      "linenos": false,
      "language": "python",
      "style": "friendly"
    }

    # POST JSON数据
    http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

    {
        "id": 4,
        "title": "",
        "code": "print 456",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

如果你向上述`http`请求添加了`--debug`，则可以在请求标头中查看请求类型。

现在可以在浏览器中访问[http://127.0.0.1:8000/snippets/][devserver]查看API。

### 浏览功能

由于API根据客户端请求选择响应的内容类型，因此默认情况下，当Web浏览器请求该资源时，它将返回资源的HTML格式表示。这允许API返回完全浏览器可浏览（web-browsable）的HTML表示。

拥有支持浏览器可浏览的API在可用性方面完胜并使开发和使用你的API更容易。它也大大降低了其他开发人员要检查和使用API​​的障碍。

有关支持浏览器可浏览的API功能以及如何自定义API的更多信息，请参阅[可浏览的api][browsable-api]主题。

## 下一步是什么？

在[教程的第3部分][tut-3]，我们将开始使用基于类的视图，并查看通用视图如何减少需要编写的代码量。

[json-url]: http://example.com/api/items/4.json
[devserver]: http://127.0.0.1:8000/snippets/
[browsable-api]: ../topics/browsable-api.md
[tut-1]: 1-serialization_zh.md
[tut-3]: 3-class-based-views_zh.md
