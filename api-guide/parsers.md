source: parsers.py

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

Parses multipart HTML form content, which supports file uploads.  Both `request.data` will be populated with a `QueryDict`.

You will typically want to use both `FormParser` and `MultiPartParser` together in order to fully support HTML form data.

**.media\_type**: `multipart/form-data`

## FileUploadParser

Parses raw file upload content.  The `request.data` property will be a dictionary with a single key `'file'` containing the uploaded file.

If the view used with `FileUploadParser` is called with a `filename` URL keyword argument, then that argument will be used as the filename.

If it is called without a `filename` URL keyword argument, then the client must set the filename in the `Content-Disposition` HTTP header.  For example `Content-Disposition: attachment; filename=upload.jpg`.

**.media\_type**: `*/*`

##### Notes:

* The `FileUploadParser` is for usage with native clients that can upload the file as a raw data request.  For web-based uploads, or for native clients with multipart upload support, you should use the `MultiPartParser` parser instead.
* Since this parser's `media_type` matches any content type, `FileUploadParser` should generally be the only parser set on an API view.
* `FileUploadParser` respects Django's standard `FILE_UPLOAD_HANDLERS` setting, and the `request.upload_handlers` attribute.  See the [Django documentation](https://docs.djangoproject.com/en/stable/topics/http/file-uploads/#upload-handlers) for more details.

##### Basic usage example:

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

# Custom parsers

To implement a custom parser, you should override `BaseParser`, set the `.media_type` property, and implement the `.parse(self, stream, media_type, parser_context)` method.

The method should return the data that will be used to populate the `request.data` property.

The arguments passed to `.parse()` are:

### stream

A stream-like object representing the body of the request.

### media\_type

Optional.  If provided, this is the media type of the incoming request content.

Depending on the request's `Content-Type:` header, this may be more specific than the renderer's `media_type` attribute, and may include media type parameters.  For example `"text/plain; charset=utf-8"`.

### parser\_context

Optional.  If supplied, this argument will be a dictionary containing any additional context that may be required to parse the request content.

By default this will include the following keys: `view`, `request`, `args`, `kwargs`.

## Example

The following is an example plaintext parser that will populate the `request.data` property with a string representing the body of the request.

```
class PlainTextParser(BaseParser):
    """
    Plain text parser.
    """
    media_type = 'text/plain'

    def parse(self, stream, media_type=None, parser_context=None):
        """
        Simply return a string representing the body of the request.
        """
        return stream.read()
```

---

# Third party packages

The following third party packages are also available.

## YAML

[REST framework YAML](http://jpadilla.github.io/django-rest-framework-yaml/) provides [YAML](http://www.yaml.org/) parsing and rendering support. It was previously included directly in the REST framework package, and is now instead supported as a third-party package.

#### Installation & configuration

Install using pip.

```
$ pip install djangorestframework-yaml
```

Modify your REST framework settings.

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

[REST Framework XML](http://jpadilla.github.io/django-rest-framework-xml/) provides a simple informal XML format. It was previously included directly in the REST framework package, and is now instead supported as a third-party package.

#### Installation & configuration

Install using pip.

```
$ pip install djangorestframework-xml
```

Modify your REST framework settings.

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

[MessagePack](https://github.com/juanriaza/django-rest-framework-msgpack) is a fast, efficient binary serialization format.  [Juan Riaza](https://github.com/juanriaza) maintains the [djangorestframework-msgpack](https://github.com/juanriaza/django-rest-framework-msgpack) package which provides MessagePack renderer and parser support for REST framework.

## CamelCase JSON

[djangorestframework-camel-case](https://github.com/vbabiy/djangorestframework-camel-case) provides camel case JSON renderers and parsers for REST framework.  This allows serializers to use Python-style underscored field names, but be exposed in the API as Javascript-style camel case field names.  It is maintained by [Vitaly Babiy](https://github.com/vbabiy).

