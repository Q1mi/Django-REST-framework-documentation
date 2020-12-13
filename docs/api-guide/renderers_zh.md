
# 渲染器

> 在将TemplateResponse实例返回给客户端之前，必须渲染它。渲染过程采用模板和上下文的中间表示，并将其转换为可以提供给客户端的最终字节流。
>
> &mdash; [Django 文档][cite]

REST框架包括许多内置的Renderer类，它们允许你使用各种媒体类型返回响应。还支持定义你自己的自定义渲染器，这样可以灵活地设计你自己的媒体类型。

## 渲染器的确定方式

视图的有效渲染器集合始终被定义为一个元素都是类的列表。当输入视图时，REST框架将对传入请求执行内容协商，并确定最适合的渲染器来满足请求。

内容协商的基本过程包括检查请求的`Accept`头，以确定响应中期望的媒体类型。URL上可选的格式后缀可以用于显式请求特定表示。例如URL`http://example.com/api/users_count.json`可能是始终返回JSON数据的路径。

有关详细信息，请参阅有关[内容协商][conneg]的文档。

## 设置渲染器

可以使用`DEFAULT_RENDERER_CLASSES`设置全局默认的渲染器集。例如，以下设置将使用`JSON`作为主要媒体类型，并且还包括自描述API。

    REST_FRAMEWORK = {
        'DEFAULT_RENDERER_CLASSES': (
            'rest_framework.renderers.JSONRenderer',
            'rest_framework.renderers.BrowsableAPIRenderer',
        )
    }

你还可以设置用于单个视图或视图集的渲染器，使用`APIView`类视图。

    from django.contrib.auth.models import User
    from rest_framework.renderers import JSONRenderer
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class UserCountView(APIView):
        """
        返回JSON格式活动用户数的视图。
        """
        renderer_classes = (JSONRenderer, )

        def get(self, request, format=None):
            user_count = User.objects.filter(active=True).count()
            content = {'user_count': user_count}
            return Response(content)

或者，如果你使用基于功能的视图的`@api_view`装饰器。

    @api_view(['GET'])
    @renderer_classes((JSONRenderer,))
    def user_count_view(request, format=None):
        """
        返回JSON格式活动用户数的视图。
        """
        user_count = User.objects.filter(active=True).count()
        content = {'user_count': user_count}
        return Response(content)

## 渲染器类的排序

指定你的API的渲染器类时要考虑到每个媒体类型要分配哪些优先级，这一点非常重要。如果一个客户端不能指定它可以接受的表示形式，例如发送一个`Accept: */*`头，或者不包含一个`Accept`头，那么REST框架将选择列表中用于响应的第一个渲染器。

例如，如果你的API提供JSON响应和HTML可浏览的API，则可能需要将`JSONRenderer`设置为你的默认渲染器，以便向不指定`Accept`标头的客户端发送`JSON`响应。

如果你的API包含可以根据请求提供常规网页和API响应的视图，那么你就可以考虑使用`TemplateHTMLRenderer`作为你的默认渲染器，以便能在那些发送[破坏的接收头][browser-accept-headers]的旧版本的浏览器上能很好的展示。 

---

# API 参考

## JSONRenderer

使用utf-8编码将请求的数据渲染成`JSON`。

请注意，默认样式是包括unicode字符，并使用没有不必要空格的紧凑样式渲染响应:

    {"unicode black star":"★","value":999}

客户端还可以包含`'indent'`媒体类型参数，在这种情况下，返回的`JSON`将被缩进。例如`Accept: application/json; indent=4`。

    {
        "unicode black star": "★",
        "value": 999
    }

可以使用`UNICODE_JSON`和`COMPACT_JSON`更改默认JSON编码样式。

**.media_type**: `application/json`

**.format**: `'.json'`

**.charset**: `None`

## TemplateHTMLRenderer

使用Django的标准模板将数据渲染成HTML。

与其他渲染器不同，传递给`Response`的数据不需要序列化。此外，与其他渲染器不同，你可能希望在创建`Response`时包含一个`template_name`参数。

TemplateHTMLRenderer将创建一个`RequestContext`，使用`response.data`作为上下文字典，并确定用于渲染上下文的模板名称。

模板名称由（按优先顺序）确定：

1. 一个显式的`template_name`参数传递给响应。
2. 在类中显式定义`.template_name`属性。
3. 调用`view.get_template_names（）`的返回结果。

使用 `TemplateHTMLRenderer`的视图的例子：

    class UserDetail(generics.RetrieveAPIView):
        """
        返回给定用户的模板HTML表示的视图。
        """
        queryset = User.objects.all()
        renderer_classes = (TemplateHTMLRenderer,)

        def get(self, request, *args, **kwargs):
            self.object = self.get_object()
            return Response({'user': self.object}, template_name='user_detail.html')

你可以使用`TemplateHTMLRenderer`来返回使用REST框架的常规HTML页面，或者从单个路径返回HTML和API响应。

如果你正在构建使用 `TemplateHTMLRenderer` 和其他渲染类的网站，你应该考虑将`TemplateHTMLRenderer`列为`renderer_classes`列表中的第一个类，这样即使对于发送格式不正确的`ACCEPT:`头文件的浏览器它也将被优先排序。

有关`TemplateHTMLRenderer`用法的更多示例，请参阅[_HTML&Forms_主题页] [html-and-forms]。

**.media_type**: `text/html`

**.format**: `'.html'`

**.charset**: `utf-8`

也可以看看: `StaticHTMLRenderer`

## StaticHTMLRenderer

一个简单的渲染器，只需返回预渲染的HTML。与其他渲染器不同，传递给响应对象的数据应该是表示要返回的内容的字符串。

一个使用 `StaticHTMLRenderer`的视图的例子：

    @api_view(('GET',))
    @renderer_classes((StaticHTMLRenderer,))
    def simple_html_view(request):
        data = '<html><body><h1>Hello, world</h1></body></html>'
        return Response(data)

你可以使用`StaticHTMLRenderer`使用REST框架返回常规HTML页面，也可以从单个路径返回HTML和API响应。

**.media_type**: `text/html`

**.format**: `'.html'`

**.charset**: `utf-8`

也可以看看： `TemplateHTMLRenderer`

## BrowsableAPIRenderer

将数据渲染成可浏览的API：

![The BrowsableAPIRenderer](../img/quickstart.png)

此渲染器将确定哪个其他渲染器将被赋予最高优先级，并使用它在HTML页面中显示API。

**.media_type**: `text/html`

**.format**: `'.api'`

**.charset**: `utf-8`

**.template**: `'rest_framework/api.html'`

#### 自定义 BrowsableAPIRenderer

默认情况下，响应内容将以与`BrowsableAPIRenderer`不同的最高优先级渲染器渲染。如果你需要自定义此行为，例如使用HTML作为默认返回格式，但在可浏览的API中使用JSON，则可以通过重写`get_default_renderer()`方法来实现。例如：

    class CustomBrowsableAPIRenderer(BrowsableAPIRenderer):
        def get_default_renderer(self, view):
            return JSONRenderer()

##  AdminRenderer

将数据渲染给HTML以进行类似管理的显示：

![The AdminRender view](../img/admin.png)

此渲染器适用于CRUD风格的Web API，还应提供用于管理数据的用户友好界面。

请注意，包含嵌套或列表序列化器的输入视图对于`AdminRenderer`将无法正常工作，因为HTML表单无法正确支持它们。

**注意**: 当数据中存在正确配置的`URL_FIELD_NAME`（缺省`url`）属性时，`AdminRenderer`才能够包含指向详细页面的链接。对于`HyperlinkedModelSerializer`，这将是这种情况，但是对于`ModelSerializer` 或者简单的`Serializer`类，你需要确保明确地包含该字段。例如，我们使用模型`get_absolute_url`方法：

    class AccountSerializer(serializers.ModelSerializer):
        url = serializers.CharField(source='get_absolute_url', read_only=True)

        class Meta:
            model = Account


**.media_type**: `text/html`

**.format**: `'.admin'`

**.charset**: `utf-8`

**.template**: `'rest_framework/admin.html'`

## HTMLFormRenderer

将序列化程序返回的数据渲染为HTML表单。此渲染器的输出不包括封闭的`<form>`标签，隐藏的CSRF输入或任何提交按钮。

此渲染器不是直接使用，而是可以通过将序列化器实例传递给`render_form`模板标记来替代模板。

    {% load rest_framework %}

    <form action="/submit-report/" method="post">
        {% csrf_token %}
        {% render_form serializer %}
        <input type="submit" value="Save" />
    </form>

有关更多信息，请参阅[HTML和表单][html-and-forms]文档。

**.media_type**: `text/html`

**.format**: `'.form'`

**.charset**: `utf-8`

**.template**: `'rest_framework/horizontal/form.html'`

## MultiPartRenderer

此渲染器用于渲染HTML multipart表单数据。 **它不适合作为响应渲染器**，而是用于创建测试请求，使用REST framework的 [测试客户端和测试请求工厂][testing]。

**.media_type**: `multipart/form-data; boundary=BoUnDaRyStRiNg`

**.format**: `'.multipart'`

**.charset**: `utf-8`

---

# Custom renderers

要实现自定义渲染器，你应该重写`BaseRenderer`，设置 `.media_type`和`.format`属性，并且实现 `.render(self, data, media_type=None, renderer_context=None)` 方法。

这个方法应当返回一个字节bytestring，它将被用于HTTP响应的主体。

传递给 `.render()` 方法的参数是：

### `data`

请求数据，由 `Response()` 实例化设置。

### `media_type=None`

可选的。如果提供，这是由内容协商阶段确定的所接受的媒体类型。

根据客户端的 `Accept:` 头，这可能比渲染器的 `media_type` 属性更具体，可能包括媒体类型参数。例如 `"application/json; nested=true"`。

### `renderer_context=None`

可选的。如果提供，这是一个由view提供的上下文信息的字典。

默认情况下这个字典会包括以下键： `view`, `request`, `response`, `args`, `kwargs`。

## 例子

下面是一个示例明文渲染器，它将使用参数作为响应 `data` 的内容返回响应。 

    from django.utils.encoding import smart_unicode
    from rest_framework import renderers


    class PlainTextRenderer(renderers.BaseRenderer):
        media_type = 'text/plain'
        format = 'txt'

        def render(self, data, media_type=None, renderer_context=None):
            return data.encode(self.charset)

## 设置字符集

假设默认的渲染器类正在使用 `UTF-8` 编码。要使用其他编码，请在渲染器设置 `charset` 属性。

    class PlainTextRenderer(renderers.BaseRenderer):
        media_type = 'text/plain'
        format = 'txt'
        charset = 'iso-8859-1'

        def render(self, data, media_type=None, renderer_context=None):
            return data.encode(self.charset)

注意，如果一个渲染类返回了一个unicode字符串，则响应内容将被`Response`类强制转换成bytestring，渲染器上的设置的 `charset` 属性将用于确定编码。

如果渲染器返回一个bytestring表示原始的二进制内容，则应该设置字符集的值为 `None`，确保响应请求头的 `Content-Type` 中不会设置 `charset` 值。

在某些情况下你可能还需要将 `render_style` 属性设置成 `'binary'`。这么做也将确保可浏览的API不会尝试将二进制内容显示为字符串。

    class JPEGRenderer(renderers.BaseRenderer):
        media_type = 'image/jpeg'
        format = 'jpg'
        charset = None
        render_style = 'binary'

        def render(self, data, media_type=None, renderer_context=None):
            return data

---

# 高级渲染器使用

你可以使用REST framework的渲染器做一些非常灵活的事情。一些例子...

* 根据请求的媒体类型，从同一个路径既能提供单独的或者嵌套的表示。
* 提供常规HTML网页和来自同一路径的基于JSON的API响应。
* 为API客户端指定要使用的多种类型的HTML表示形式。
* 未指定渲染器的媒体类型，例如使用 `media_type = 'image/*'`，并使用 `Accept` 标头来更改响应的编码。

## 媒体类型的不同行为

在某些情况下，你可能希望视图根据所接受的媒体类型使用不同的序列化样式。如果你需要实现这个功能，你可以根据 `request.accepted_renderer` 来确定将用于响应的协商渲染器。

例如:

    @api_view(('GET',))
    @renderer_classes((TemplateHTMLRenderer, JSONRenderer))
    def list_users(request):
        """
        一个可以返回系统中用户的JSON或HTML表示的视图。
        """
        queryset = Users.objects.filter(active=True)

        if request.accepted_renderer.format == 'html':
            # TemplateHTMLRenderer 采用一个上下文的字典，
            # 并且额外需要一个 'template_name'。
            # 它不需要序列化。
            data = {'users': queryset}
            return Response(data, template_name='list_users.html')

        # JSONRenderer 需要正常的序列化数据。
        serializer = UserSerializer(instance=queryset)
        data = serializer.data
        return Response(data)

## 不明确的媒体类型

在某些情况下你可能希望渲染器提供一些列媒体类型。
在这种情况下，你可以通过为 `media_type` 设置诸如 `image/*` 或 `*/*`这样的值来指定应该响应的媒体类型。

如果你指定了渲染器的媒体类型，你应该确保在返回响应时使用 `content_type` 属性明确指定媒体类型。 例如:

    return Response(data, content_type='image/png')

## 设计你的媒体类型

许多Web API的目标，简单的具有超链接的 `JSON` 响应可能就已经足够了。如果你想完全拥抱RESTful设计和[HATEOAS]则需要更详细地考虑媒体类型的设计和使用。

用[Roy Fielding的话][quote]来说，"REST API 应该花费所有的描述性努力来定义用于表示资源和驱动应用程序状态的媒体类型（们），或者为现有的标准媒体类型定义扩展关系名称和/或超文本启用标记。"。

有关自定义媒体类型的优秀示例，请参阅GitHub关于自定义 [application/vnd.github+json] 媒体类型的应用以及 Mike Amundsen的IANA认可的 [application/vnd.collection+json] JSON超媒体。

## HTML错误视图

通常渲染器都具有相同的行为，无论它处理的是正常的响应还是由异常引起的响应，如 `Http404` 或 `PermissionDenied` 异常，或者一个 `APIException` 的子类。

如果你正在使用 `TemplateHTMLRenderer` 或 `StaticHTMLRenderer` 时抛出了异常，行为略有不同。并且反映 [Django对错误视图的默认处理][django-error-views].

由HTML渲染器引发和处理的异常将尝试按照优先顺序使用以下方法之一进行渲染。

* 加载并渲染一个名为 `{status_code}.html`的模板。
* 加载并渲染一个名为 `api_exception.html`的模板。
* 渲染HTTP状态码和文本，例如 "404 Not Found"。

模板们将使用一个包括`status_code`和`details`的 `RequestContext` 渲染。

**注意**: 如果设置了 `DEBUG=True`，Django将展示它的标准回溯错误页面而不是渲染HTTP状态码和文本。

---

# 第三方包

以下第三方包都是可用的。

## YAML

[REST framework YAML][rest-framework-yaml] 提供 [YAML][yaml] 解析和渲染支持。它之前直接包含在REST framework 包中，现在被替代为第三方包支持。
#### 安装和配置

使用pip安装。

    $ pip install djangorestframework-yaml

修改你的REST framework设置。

    REST_FRAMEWORK = {
        'DEFAULT_PARSER_CLASSES': (
            'rest_framework_yaml.parsers.YAMLParser',
        ),
        'DEFAULT_RENDERER_CLASSES': (
            'rest_framework_yaml.renderers.YAMLRenderer',
        ),
    }

## XML

[REST Framework XML][rest-framework-xml] 提供了一个简单的非正式XML格式。它之前直接包含在REST framework 包中，现在被替代为第三方包支持。

#### 安装和配置

使用pip安装。

    $ pip install djangorestframework-xml

修改你的REST framework设置。

    REST_FRAMEWORK = {
        'DEFAULT_PARSER_CLASSES': (
            'rest_framework_xml.parsers.XMLParser',
        ),
        'DEFAULT_RENDERER_CLASSES': (
            'rest_framework_xml.renderers.XMLRenderer',
        ),
    }

## JSONP

[REST framework JSONP][rest-framework-jsonp] 提供JSONP渲染支持。它之前直接包含在REST framework 包中，现在被替代为第三方包支持。

---

**警告**: 如果你需要跨域的AJAX请求，你通常应该使用更现代化的[CORS][cors]方法代替`JSONP`。更多详细信息请参阅[CORS文档][cors-docs]。

`jsonp` 本质上是一个浏览器hack方法，[仅适用于全局可读的API路径][jsonp-security]，其中`GET`请求未经身份验证，并且不需要任何用户权限。

---

#### 安装和配置

使用pip安装。

    $ pip install djangorestframework-jsonp

修改你的REST framework设置。

    REST_FRAMEWORK = {
        'DEFAULT_RENDERER_CLASSES': (
            'rest_framework_jsonp.renderers.JSONPRenderer',
        ),
    }

## MessagePack

[MessagePack][messagepack]是一种快速，高效的二进制序列化格式。[Juan Riaza][juanriaza]维护着[djangorestframework-msgpack][djangorestframework-msgpack] 包，它为REST framework提供MessagePack渲染器和解析器支持。

## CSV

逗号分隔的值是纯文本数据格式，可以轻松导入到电子表格应用中。[Mjumbe Poe][mjumbewu]维护着[djangorestframework-csv][djangorestframework-csv]包，它为REST framework提供了CSV渲染器支持。

## UltraJSON

[UltraJSON][ultrajson]是一个优化的C JSON编码器，可以显著提高JSON渲染速度。[Jacob Haslehurst][hzy]维护着使用UJSON包实现JSON渲染的[drf-ujson-renderer][drf-ujson-renderer]包。

## CamelCase JSON

[djangorestframework-camel-case]为REST framework提供了驼峰样式的JSON渲染器和解析器。这使序列化程序可以使用Python风格的下划线字段名，但是在API中显示成Javascript样式的驼峰字段名。它被[Vitaly Babiy][vbabiy]维护着。

## Pandas (CSV, Excel, PNG)

[Django REST Pandas]提供了一个序列化器和渲染器，通过[Pandas] DataFrame API提供额外的数据处理和输出。Django REST Pandas包括Pandas风格的CSV文件，Excel表格(包括 `.xls` 和 `.xlsx`)以及许多[其他格式][other formats]的渲染器。作为[wq 项目][wq]的一部分由[S. Andrew Sheppard][sheppard]维护着。

## LaTeX

[Rest Framework Latex]提供了一个使用Laulatex输出PDF的渲染器。它由[Pebble (S/F Software)][mypebble]维护着。


[cite]: https://docs.djangoproject.com/en/stable/stable/template-response/#the-rendering-process
[conneg]: content-negotiation.md
[html-and-forms]: ../topics/html-and-forms.md
[browser-accept-headers]: http://www.gethifi.com/blog/browser-rest-http-accept-headers
[testing]: testing.md
[HATEOAS]: http://timelessrepo.com/haters-gonna-hateoas
[quote]: http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven
[application/vnd.github+json]: http://developer.github.com/v3/media/
[application/vnd.collection+json]: http://www.amundsen.com/media-types/collection/
[django-error-views]: https://docs.djangoproject.com/en/stable/topics/http/views/#customizing-error-views
[rest-framework-jsonp]: http://jpadilla.github.io/django-rest-framework-jsonp/
[cors]: http://www.w3.org/TR/cors/
[cors-docs]: http://www.django-rest-framework.org/topics/ajax-csrf-cors/
[jsonp-security]: http://stackoverflow.com/questions/613962/is-jsonp-safe-to-use
[rest-framework-yaml]: http://jpadilla.github.io/django-rest-framework-yaml/
[rest-framework-xml]: http://jpadilla.github.io/django-rest-framework-xml/
[messagepack]: http://msgpack.org/
[juanriaza]: https://github.com/juanriaza
[mjumbewu]: https://github.com/mjumbewu
[vbabiy]: https://github.com/vbabiy
[rest-framework-yaml]: http://jpadilla.github.io/django-rest-framework-yaml/
[rest-framework-xml]: http://jpadilla.github.io/django-rest-framework-xml/
[yaml]: http://www.yaml.org/
[djangorestframework-msgpack]: https://github.com/juanriaza/django-rest-framework-msgpack
[djangorestframework-csv]: https://github.com/mjumbewu/django-rest-framework-csv
[ultrajson]: https://github.com/esnme/ultrajson
[hzy]: https://github.com/hzy
[drf-ujson-renderer]: https://github.com/gizmag/drf-ujson-renderer
[djangorestframework-camel-case]: https://github.com/vbabiy/djangorestframework-camel-case
[Django REST Pandas]: https://github.com/wq/django-rest-pandas
[Pandas]: http://pandas.pydata.org/
[other formats]: https://github.com/wq/django-rest-pandas#supported-formats
[sheppard]: https://github.com/sheppard
[wq]: https://github.com/wq
[mypebble]: https://github.com/mypebble
[Rest Framework Latex]: https://github.com/mypebble/rest-framework-latex
