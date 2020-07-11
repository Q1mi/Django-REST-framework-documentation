
# Versioning

> 对接口进行版本控制只是一种杀死已部署客户端的“礼貌”方式。
>
> &mdash; [Roy Fielding][cite].

API 版本控制允许你在不同的客户端之间更改行为. REST framework 提供了许多不同的版本控制方案。

版本控制由传入的客户端请求决定，可以基于请求的URL，也可以基于请求头。

有许多有效的方法可以进行版本控制。[无版本控制的系统也可能是合适的][roy-fielding-on-versioning]，特别是如果你正在为一个长期系统进行工程设计，而这个系统有多个超出你控制范围的客户端。

## 使用REST框架进行版本控制

当启用API版本控制，`request.version` 属性将包含与传入客户端请求中请求的版本相对应的字符串。

默认情况下，版本控制没有启用，`request.version` 将总是返回 `None`。

#### 根据版本改变行为

如何改变API的行为取决于你自己，但是一个你通常想要的例子是在新版本中切换到不同的序列化样式。例如:

    def get_serializer_class(self):
        if self.request.version == 'v1':
            return AccountSerializerVersion1
        return AccountSerializer

#### 反向解析版本化API的URL

REST framework 包含的 `reverse` 函数与版本控制方案相关联。你需要确保将当前 `request` 作为关键字参数传递进去，如下所示。

    from rest_framework.reverse import reverse

    reverse('bookings-list', request=request)

上述函数将应用任何适用于请求版本的URL转换。例如:

* 如果使用`NamespacedVersioning`，并且API的版本是'v1'，那么将会查找 `'v1:bookings-list'`，可能反向解析为类似`http://example.org/v1/bookings/` 这个URL。
* 如果使用`QueryParameterVersioning` 并且API的版本是`1.0`，那么返回的URL可能就像这样`http://example.org/bookings/?version=1.0`。

#### 版本化 API 和超链接序列化器 

当将超链接序列化样式与基于URL的版本控制方案一起使用时，请确保将请求作为上下文包含在序列化程序中。

    def get(self, request):
        queryset = Booking.objects.all()
        serializer = BookingsSerializer(queryset, many=True, context={'request': request})
        return Response({'all_bookings': serializer.data})

这样做将会在任何返回的URL中包含适当的版本控制。

## 配置版本控制方案

版本控制方案由settings中的`DEFAULT_VERSIONING_CLASS`为key来配置。

    REST_FRAMEWORK = {
        'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.NamespaceVersioning'
    }

除非显式设置，否则`DEFAULT_VERSIONING_CLASS`将会是`None`。在这种情况下`request.version`属性将总是返回`None`。

还可以在单个视图上设置版本控制方案。通常不需要这样做，因为全局使用单一版本控制方案更有意义。如果确实需要这样做，请在视图类中使用`versioning_class`属性。

    class ProfileList(APIView):
        versioning_class = versioning.QueryParameterVersioning

#### 其他版本控制设置

以下设置的key也用于控制版本控制:

* `DEFAULT_VERSION`. 当版本控制信息不存在时用于设置`request.version`的默认值，默认设置为`None`。
* `ALLOWED_VERSIONS`. 如果设置了此值将限制版本控制方案可能返回的版本集，如果客户端请求提供的版本不在此集中，则会引发错误。请注意，用于`DEFAULT_VERSION`的值应该总是在`ALLOWED_VERSIONS`设置的集合中（除非是`None`）。该配置默认是 `None`。
* `VERSION_PARAM`. 一个应当用于任何版本控制系统参数的字符串，例如媒体类型或URL查询参数。默认值是`'version'`。

你还可以通过自定义版本控制方案并使用`default_version`，`allowed_versions`和`version_param`类变量，在每个视图或每个视图集的基础上设置版本类加上这三个值。例如，如果你想使用`URLPathVersioning`：

    from rest_framework.versioning import URLPathVersioning
    from rest_framework.views import APIView

    class ExampleVersioning(URLPathVersioning):
        default_version = ...
        allowed_versions = ...
        version_param = ...

    class ExampleView(APIVIew):
        versioning_class = ExampleVersioning

---

# API参考

## AcceptHeaderVersioning

此方案要求客户端在`Accept` header 中将版本指定为媒体类型的一部分。该版本作为媒体类型参数包含在内，它补充了主要媒体类型。

下面是一个使用accept header版本化样式的HTTP请求示例。

    GET /bookings/ HTTP/1.1
    Host: example.com
    Accept: application/json; version=1.0

在上面的示例请求中`request.version`属性将返回字符串`'1.0'`。
基于 accept headers 的版本控制[通常被认为][klabnik-guidelines]是[最佳实践][heroku-guidelines]，尽管其他版本控制方式可能适合你的客户端需求。

#### Using accept headers with vendor media types

严格地说`json`媒体类型未指定为[包含其他参数][json-parameters]。如果要构建明确指定的公共API，则可以考虑使用[vendor media type][vendor-media-type]。为此，请将渲染器配置为使用自定义媒体类型的基于JSON的渲染器：

    class BookingsAPIRenderer(JSONRenderer):
        media_type = 'application/vnd.megacorp.bookings+json'

你的客户端请求现在会是这样的:

    GET /bookings/ HTTP/1.1
    Host: example.com
    Accept: application/vnd.megacorp.bookings+json; version=1.0

## URLPathVersioning

此方案要求客户端将版本指定为URL路径的一部分。

    GET /v1/bookings/ HTTP/1.1
    Host: example.com
    Accept: application/json

你的URL conf中必须包含一个使用`'version'`关键字参数的匹配模式，，以便版本控制方案可以使用此版本信息。

    urlpatterns = [
        url(
            r'^(?P<version>(v1|v2))/bookings/$',
            bookings_list,
            name='bookings-list'
        ),
        url(
            r'^(?P<version>(v1|v2))/bookings/(?P<pk>[0-9]+)/$',
            bookings_detail,
            name='bookings-detail'
        )
    ]

## NamespaceVersioning

对于客户端，此方案与`URLPathVersioning`相同。唯一的区别是，它是如何在 Django 应用程序中配置的，因为它使用URL conf中的命名空间而不是URL conf中的关键字参数。

    GET /v1/something/ HTTP/1.1
    Host: example.com
    Accept: application/json

使用此方案，`request.version` 属性是根据与传入请求的路径匹配的 `namespace` 确定的。

在下面的示例中，我们给一组视图提供了两个可能出现的不同URL前缀，每个前缀在不同的命名空间下:

    # bookings/urls.py
    urlpatterns = [
        url(r'^$', bookings_list, name='bookings-list'),
        url(r'^(?P<pk>[0-9]+)/$', bookings_detail, name='bookings-detail')
    ]

    # urls.py
    urlpatterns = [
        url(r'^v1/bookings/', include('bookings.urls', namespace='v1')),
        url(r'^v2/bookings/', include('bookings.urls', namespace='v2'))
    ]

如果你只需要一个简单的版本控制方案`URLPathVersioning`和`NamespaceVersioning`都是合适的。`URLPathVersioning` 这种方法可能更适合小型项目，对于更大的项目来说`NamespaceVersioning`可能更容易管理。

## HostNameVersioning

主机名版本控制方案要求客户端将请求的版本指定为URL中主机名的一部分。
例如，以下是对`http://v1.example.com/bookings/`的HTTP请求：

    GET /bookings/ HTTP/1.1
    Host: v1.example.com
    Accept: application/json

默认情况下，此实现期望主机名与以下简单正则表达式匹配:

    ^([a-zA-Z0-9]+)\.[a-zA-Z0-9]+\.[a-zA-Z0-9]+$

注意，第一组用括号括起来，表示这是主机名的匹配部分。

`HostNameVersioning`这种方案在调试模式下使用方案可能会很尴尬，因为你通常会访问原始IP地址，例如`127.0.0.1`。有各种在线教程，介绍[使用自定义子域名访问本地主机][lvh]，这种情况下你可能会发现这很有帮助。

如果你有基于版本将传入请求路由到不同服务器的需求，那么基于主机名的版本控制会特别有用，因为你可以为不同的 API 版本配置不同的 DNS 记录。

## QueryParameterVersioning

此方案是一种在 URL 中包含版本信息作为查询参数的简单方案。例如：
This scheme is a simple style that includes the version as a query parameter in the URL. For example:

    GET /something/?version=0.1 HTTP/1.1
    Host: example.com
    Accept: application/json

---

# Custom versioning schemes

要实现自定义版本控制方案，请继承 `BaseVersioning` 并重写 `.determine_version` 方法。

## 举个例子

下面的例子中使用一个自定义的 `X-API-Version` header来确定所请求的版本。

    class XAPIVersionScheme(versioning.BaseVersioning):
        def determine_version(self, request, *args, **kwargs):
            return request.META.get('HTTP_X_API_VERSION', None)

如果你的版本化方案基于请求URL，你还需要改变版本化URL的确定方式。为此，你还应该重写类中的`.reverse()`方法。有关示例，请参见源代码。

[cite]: http://www.slideshare.net/evolve_conference/201308-fielding-evolve/31
[roy-fielding-on-versioning]: http://www.infoq.com/articles/roy-fielding-on-versioning
[klabnik-guidelines]: http://blog.steveklabnik.com/posts/2011-07-03-nobody-understands-rest-or-http#i_want_my_api_to_be_versioned
[heroku-guidelines]: https://github.com/interagent/http-api-design/blob/master/en/foundations/require-versioning-in-the-accepts-header.md
[json-parameters]: http://tools.ietf.org/html/rfc4627#section-6
[vendor-media-type]: http://en.wikipedia.org/wiki/Internet_media_type#Vendor_tree
[lvh]: https://reinteractive.net/posts/199-developing-and-testing-rails-applications-with-subdomains
