URLsource: parsers.py

# 解析器

> 机器互动的Web服务往往使用更多结构化的格式发送数据而不是使用表单编码，因为它们发送的是比简单形式更复杂的数据。
>
> — Malcom Tredinnick, [Django developers group](https://groups.google.com/d/topic/django-developers/dxI4qVzrBY4/discussion)

REST 框架包括一些内置的Parser类，允许你接受各种媒体类型的请求。还支持定义自己的自定义解析器，这使你可以灵活地设计API接受的媒体类型。

## 解析器如何确定

一组视图的有效解析器总是被定义为一个类的列表。当访问`request.data`时，REST框架将检查传入请求中的`Content-Type`头，并确定用于解析请求内容的解析器。

---

**注意**: 开发客户端应用程序时应该始终记住在HTTP请求中发送数据时确保设置`Content-Type`头。

如果你不设置内容类型，大多数客户端将默认使用`'application/x-www-form-urlencoded'`，而这可能并不是你想要的。

举个例子，如果你使用jQuery的[.ajax\(\) 方法](http://api.jquery.com/jQuery.ajax/)发送`json`编码数据，你应该确保包含`contentType：'application / json'`设置。

---

## 设置解析器

可以使用`DEFAULT_PARSER_CLASSES`设置全局默认的解析器集。例如，以下设置将仅允许具有`JSON`内容的请求，而不是JSON或表单数据的默认值。

```
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework.parsers.JSONParser',
    )
}
```

你还可以设置用于单个视图或视图集的解析器， 使用`APIView`类视图。

```
from rest_framework.parsers import JSONParser
from rest_framework.response import Response
from rest_framework.views import APIView

class ExampleView(APIView):
    """
    可以接收JSON内容POST请求的视图。
    """
    parser_classes = (JSONParser,)

    def post(self, request, format=None):
        return Response({'received data': request.data})
```

或者，如果你使用基于方法的视图的`@api_view`装饰器。

```
from rest_framework.decorators import api_view
from rest_framework.decorators import parser_classes

@api_view(['POST'])
@parser_classes((JSONParser,))
def example_view(request, format=None):
    """
    可以接收JSON内容POST请求的视图
    """
    return Response({'received data': request.data})
```

---

# API参考

## JSONParser

解析 `JSON` 请求内容。

**.media\_type**: `application/json`

## FormParser

解析 HTML 表单内容。`request.data`将被填充一个`QueryDict`的数据。

通常，你需要使用`FormParser`和`MultiPartParser`两者，以便完全支持HTML表单数据。

**.media\_type**: `application/x-www-form-urlencoded`

## MultiPartParser

解析多部分HTML表单内容，支持文件上传。`request.data` 都将被一个 `QueryDict`填充。

你通常会同时使用`FormParser`和`MultiPartParser`两者，以便完全支持HTML表单数据。

**.media\_type**: `multipart/form-data`

## FileUploadParser

解析原始文件上传内容。 `request.data` 属性将是有单个key `'file'`的包含上传文件的字典。

如果与`FileUploadParser`一起使用的视图使用`filename` URL关键字参数调用，则该参数将用作文件名。

如果没有`filename` URL关键字参数调用，那么客户端必须在`Content-Disposition` HTTP头中设置文件名。例如 `Content-Disposition: attachment; filename=upload.jpg`.

**.media\_type**: `*/*`

##### 说明:

* `FileUploadParser` 用于与原始数据请求一起上传文件的本机客户端。对于基于Web的上传，或者对于具有多部分上传支持的本机客户端，您应该使用`MultiPartParser`解析器。
* 由于该解析器的`media_type`与任何内容类型匹配，所以`FileUploadParser`通常应该是API视图中唯一的解析器。
* `FileUploadParser` 遵循 Django 的标准 `FILE_UPLOAD_HANDLERS` 设置，和 `request.upload_handlers` 属性。参见 [Django 文档](https://docs.djangoproject.com/en/stable/topics/http/file-uploads/#upload-handlers) 获取更多细节。

##### 基本用法示例：

```
# views.py
class FileUploadView(views.APIView):
    parser_classes = (FileUploadParser,)

    def put(self, request, filename, format=None):
        file_obj = request.data['file']
        # ...
        # do some stuff with uploaded file
        # ...
        return Response(status=204)

# urls.py
urlpatterns = [
    # ...
    url(r'^upload/(?P<filename>[^/]+)$', FileUploadView.as_view())
]
```

---

# 自定义解析器

要实现一个自定义解析器，你应该重写`BaseParser`，设置`.media_type`属性，并实现`.parse(self，stream，media_type，parser_context)`方法。

该方法应该返回用于填充`request.data` 属性的数据。

传递给 `.parse()` 的参数是:

### stream

表示请求体的数据流。

### media\_type

可选的。如果提供，这是传入请求内容的媒体类型。

基于请求的`Content-Type:`头，这可能比渲染器的`media_type`属性更具体，可能包括媒体类型参数。例如 `"text/plain; charset=utf-8"`。

### parser\_context

可选的。如果提供，该参数将是一个包含解析请求内容可能需要的任何附加上下文的字典。

默认情况下将包含以下keys: `view`, `request`, `args`, `kwargs`。

## 例子

以下是一个Plain text的示例，它将使用表示请求正文的字符串填充`request.data`属性。

```
class PlainTextParser(BaseParser):
    """
    Plain text 解析器。
    """
    media_type = 'text/plain'

    def parse(self, stream, media_type=None, parser_context=None):
        """
        只需返回一个表示请求正文的字符串。
        """
        return stream.read()
```

---

# 第三方包

以下是可用的第三方包。

## YAML

[REST framework YAML](http://jpadilla.github.io/django-rest-framework-yaml/) 提供了 [YAML](http://www.yaml.org/) 解析和呈现的支持。之前它是直接包含在REST框架包中的，现在被替代为第三方包支持。

#### 安装和配置

使用pip进行安装。

```
$ pip install djangorestframework-yaml
```

修改你的 REST framework settings。

```
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework_yaml.parsers.YAMLParser',
    ),
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework_yaml.renderers.YAMLRenderer',
    ),
}
```

## XML

[REST Framework XML](http://jpadilla.github.io/django-rest-framework-xml/) 提供了一种简单的非正式XML格式。它以前是直接包含在REST框架包中，现在被替代为第三方包支持。

#### 安装和配置


使用pip进行安装。

```
$ pip install djangorestframework-xml
```

修改你的 REST framework settings.

```
REST_FRAMEWORK = {
    'DEFAULT_PARSER_CLASSES': (
        'rest_framework_xml.parsers.XMLParser',
    ),
    'DEFAULT_RENDERER_CLASSES': (
        'rest_framework_xml.renderers.XMLRenderer',
    ),
}
```

## MessagePack

[MessagePack](https://github.com/juanriaza/django-rest-framework-msgpack) 是一种快速，高效的二进制序列化格式。 [Juan Riaza](https://github.com/juanriaza) 维护了 [djangorestframework-msgpack](https://github.com/juanriaza/django-rest-framework-msgpack) 包，它为REST框架提供MessagePack渲染器和解析器支持。

## CamelCase JSON

[djangorestframework-camel-case](https://github.com/vbabiy/djangorestframework-camel-case) 为REST framework提供了驼峰类型JSON 生成和解析。这个库允许序列化程序使用Python风格的下划线字段名称，并且将其转变为Javascript风格的驼峰字段名称在API中公开。这个库由 [Vitaly Babiy](https://github.com/vbabiy)维护。

