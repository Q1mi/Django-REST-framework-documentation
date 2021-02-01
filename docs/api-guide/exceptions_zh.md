---
source:
    - exceptions.py
---

# Exceptions

> 异常... 让错误处理清晰地位于程序架构的中心位置或高层位置。
>
> &mdash; Doug Hellmann, [Python 异常处理方法][cite]

## 在REST framework views中的异常处理

REST framework的视图能处理各种各样的异常，能处理并返回合适的错误响应。

以下异常会被处理：

* 在REST framework内部产生的`APIException` 的子类异常。
* 原生Django的`Http404` 异常.
* 原生Django的`PermissionDenied` 异常.

在上述各类情况中，REST framework会返回一个带有合适状态码与content-type的响应。响应正文(Response body)会包含有关报错的任何额外信息。

大多数报错响应都会在响应正文里包含一个`detail`。

例如，下面的请求：

    DELETE http://api.example.com/foo/bar HTTP/1.1
    Accept: application/json

就可能会收到一个报错响应，表示`DELETE`方法在该资源上不可使用。

    HTTP/1.1 405 Method Not Allowed
    Content-Type: application/json
    Content-Length: 42
    
    {"detail": "Method 'DELETE' not allowed."}

验证类的报错在处理上略有不同，他们会把字段名作为keys包含在响应正文中。验证类的报错如果并不是针对特定的字段，则会使用"non_field_error"或者是setting中`NON_FIELD_ERRORS_KEY`的值作为keys返回。

一个验证类错误大概是这个样子：

    HTTP/1.1 400 Bad Request
    Content-Type: application/json
    Content-Length: 94
    
    {"amount": ["A valid integer is required."], "description": ["This field may not be blank."]}

## 自定义异常处理

当然你也可以自己实现自定义异常处理，只需要创建一个异常处理函数即可，它要能够将你API views里引发的异常转换成响应对象。这样你就可以自己控制你API的报错响应的样式了。

这个函数必须接受一对参数，第一个是需要处理的异常，第二个则是一个字典类型，这个字典需要包含一切额外相关信息，比如当前正在处理的view等。异常处理函数要么返回一个`Response`对象要么就直接返回`None`(比如异常无法正常处理的情况下)。如果处理函数返回了`None`，那么这个异常会继续向上报错，并由Django返回一个标准的HTTP 500的'server error'响应。

举个例子，你可能想确保你所有的响应正文里都会包含该次请求的HTTP状态码，就像这样：

    HTTP/1.1 405 Method Not Allowed
    Content-Type: application/json
    Content-Length: 62
    
    {"status_code": 405, "detail": "Method 'DELETE' not allowed."}

想要变成这样的响应，你可以写一个像下面这样的异常处理函数：

    from rest_framework.views import exception_handler
    
    def custom_exception_handler(exc, context):
        # 首先调用REST framework默认的异常处理，
        # 以获得标准的错误响应。
        response = exception_handler(exc, context)
    
        # 接下来将HTTP状态码加到响应中。
        if response is not None:
            response.data['status_code'] = response.status_code
    
        return response

这里默认的处理函数并没有用到参数context，但如果异常处理函数需要更多额外的信息就可以用到。比如需要当前处理异常的view的信息，就可以用`context['view']`来获取。

异常处理函数还必须通过settings来配置，设置中的key是`EXCEPTION_HANDLER`，就像下面这样：

    REST_FRAMEWORK = {
        'EXCEPTION_HANDLER': 'my_project.my_app.utils.custom_exception_handler'
    }

如果没有特别指出，`'EXCEPTION_HANDLER'`设置默认使用REST framework提供的标准异常处理：

    REST_FRAMEWORK = {
        'EXCEPTION_HANDLER': 'rest_framework.views.exception_handler'
    }

要注意的是异常处理只会由产生异常的响应调用。它无法处理view直接返回的响应，比如generic views在serializer验证出错时返回的响应`HTTP_400_BAD_REQUEST`。

---

# API 参考

## APIException

**签名:** `APIException()`

是所有`APIView`或者`@api_view`中产生的异常的**基类**。

如果想自定义一个异常类，首先要继承`APIException`，然后在该异常类中设置`.status_code`, `.default_detail`以及`default_code`属性。

举例，如果你的API依赖某个时不时会掉线的第三方服务，你可能想要自己实现一个异常"503 Service Unavailable"的HTTP响应码，你可以这么干：

    from rest_framework.exceptions import APIException
    
    class ServiceUnavailable(APIException):
        status_code = 503
        default_detail = 'Service temporarily unavailable, try again later.'
        default_code = 'service_unavailable'

#### 检查 API 异常

有很多属性可以用来检查一个API异常的状态，你可以用这些属性来构建专属你项目的自定义异常。

可用的属性和方法有：

* `.detail` - 以文字形式返回报错的细节描述。
* `.get_codes()` - 返回报错的标识码。
* `.get_full_details()` - 返回报错的细节描述以及报错的标识码。

大多数情况下，报错的细节的返回结果很简单：

    >>> print(exc.detail)
    You do not have permission to perform this action.
    >>> print(exc.get_codes())
    permission_denied
    >>> print(exc.get_full_details())
    {'message':'You do not have permission to perform this action.','code':'permission_denied'}

如果是验证类报错，那报错细节就会是一个列表或者字典：

    >>> print(exc.detail)
    {"name":"This field is required.","age":"A valid integer is required."}
    >>> print(exc.get_codes())Zh
    {"name":"required","age":"invalid"}
    >>> print(exc.get_full_details())
    {"name":{"message":"This field is required.","code":"required"},"age":{"message":"A valid integer is required.","code":"invalid"}}

## ParseError

**签名:** `ParseError(detail=None, code=None)`

在访问`request.data`时，如果请求中包含格式不正确的数据，则该异常会被抛出。

默认情况下该异常会返回HTTP状态码为"400 Bad Request"的响应。

## AuthenticationFailed

**签名:** `AuthenticationFailed(detail=None, code=None)`

当请求包含错误的认证信息时抛出。

默认情况下该异常会返回HTTP状态码为"401 Unauthenticated"的响应，但也有可能返回状态码为"403 Forbidden"的响应，这个主要取决于当前使用的认证模式。查看[authentication 文档][authentication]以了解更多细节。

## NotAuthenticated

**签名:** `NotAuthenticated(detail=None, code=None)`

当未带认证信息的请求检查权限出错时抛出。

默认情况下该异常会返回HTTP状态码为"401 Unauthenticated"的响应，但也有可能返回状态码为"403 Forbidden"的响应，这个主要取决于当前使用的认证模式。查看[authentication 文档][authentication]以了解更多细节。

## PermissionDenied

**签名:** `PermissionDenied(detail=None, code=None)`

Raised when an authenticated request fails the permission checks.

当一个带认证信息的请求检查权限出错时抛出。

默认情况下该异常会返回HTTP状态码为"403 Forbidden"的响应。

## NotFound

**签名:** `NotFound(detail=None, code=None)`

当给定URL的资源不存在的时候抛出。这个异常和Django标准的`Http404`异常等同。

默认情况下该异常会返回HTTP状态码为"404 Not Found"的响应。

## MethodNotAllowed

**签名:** `MethodNotAllowed(method, detail=None, code=None)`

当一个请求产生且没有view映射了该请求需要的对应方法来处理时抛出。

默认情况下该异常会返回HTTP状态码为"405 Method Not Allowed"的响应。

## NotAcceptable

**签名:** `NotAcceptable(detail=None, code=None)`

当一个带有`Accept`头的请求无法被任何当前可用的renderer满足时抛出。

默认情况下该异常会返回HTTP状态码为"406 Not Acceptable"的响应。

## UnsupportedMediaType

**签名:** `UnsupportedMediaType(media_type, detail=None, code=None)`

当没有解析器能够在访问`request.data`时处理其content type时抛出。

默认情况下该异常会返回HTTP状态码为"415 Unsupported Media Type"的响应。

## Throttled

**签名:** `Throttled(wait=None, detail=None, code=None)`

当访问的请求无法通过throttling检查时抛出。

默认情况下该异常会返回HTTP状态码为"429 Too Many Requests"的响应。

## ValidationError

**签名:** `ValidationError(detail, code=None)`

验证错误的异常要和其他`APIException`的类有所不同。

- `detail`参数是必须的，而不是可选的。
- `detail`参数可以用列表或者字典来存放错误信息，也可以使用嵌套的数据结构。
- 依据习惯，为了区分它与Django内置的验证错误，你应当引入 serializers 模块并且用一个完全合格的`ValidationError` 。比如`raise serializers.ValidationError('This field must be an integer value.')`。

`ValidationError`类应当被serialzier, field validation和validator类使用。它可以在调用`serializer.is_valid`时抛出，当然前提是要附带有`raise_exception`关键字参数：

    serializer.is_valid(raise_exception=True)

REST framework的generic views使用的是`raise_exception=True`标志，这意味着你可以在你的API里全局覆盖验证类错误响应的样式。要这么做，你需要像上文所述去自定义一个异常处理函数。

默认情况下该异常会返回HTTP状态码为"400 Bad Request"的响应。


---

# Generic Error Views

Django REST Framework提供两种合适的方法来生成通用的JSON格式的`500` Server Error和`400` Bad Request responses. (Django默认的报错views提供的是HTML格式的响应，一般来说不太适用于API-only的web后端。)

请参考[Django自定义错误view文档][django-custom-error-views]来分别使用这两种方法。

## `rest_framework.exceptions.server_error`

返回一个带有HTTP状态码`500`且content-type为`application/json`的响应。

设置`handler500`如下：

    handler500 = 'rest_framework.exceptions.server_error'

## `rest_framework.exceptions.bad_request`

返回一个带有HTTP状态码`400`且content-type为`application/json`的响应。

设置`handler400`如下：

    handler400 = 'rest_framework.exceptions.bad_request'

[cite]: https://doughellmann.com/blog/2009/06/19/python-exception-handling-techniques/
[authentication]: authentication_zh.md
[django-custom-error-views]: https://docs.djangoproject.com/en/dev/topics/http/views/#customizing-error-views
