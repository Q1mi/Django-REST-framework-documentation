# Requests（请求）

> 如果你正在做基于REST的Web服务...你最好忽略request.POST。

> &mdash; Malcom Tredinnick, [Django developers group][cite]

REST framework的`Request`类扩展了标准的`HttpRequest`，添加对REST framework的灵活请求解析和请求身份验证的支持。

---

# Request parsing（请求解析）


REST framework的请求对象提供灵活的请求解析，允许你以与通常处理表单数据相同的方式使用JSON数据或其他媒体类型处理请求。

## .data

`request.data` 返回请求正文的解析内容。这与标准的 `request.POST` 和 `request.FILES` 属性类似，除了下面的：

* 它包括所有解析的内容, 包括 *文件或非文件* 输入。
* 它支持解析除`POST`之外的HTTP方法的内容，这意味着你可以访问`PUT`和`PATCH`请求的内容。
* 它支持REST framework灵活的请求解析，而不仅仅支持表单数据。 例如，你可以以与处理传入表单数据相同的方式处理传入的JSON数据。

 更多详细信息请参阅[parsers documentation].

## .query_params

`request.query_params`是`request.GET`的一个更准确的同义词。

为了让你的代码清晰明了, 我们建议使用 `request.query_params` 而不是Django标准的`request.GET`。这样做有助于保持代码库更加正确和明了——任何HTTP方法类型可能包括查询参数，而不仅仅是`GET`请求。

## .parsers

`APIView`类或`@api_view`装饰器将根据view中设置的`parser_classes`集合或基于`DEFAULT_PARSER_CLASSES`设置，确保此属性自动设置为`Parser`实例列表。

你通常并不需要访问这个属性。

---

**Note:** 如果客户端发送格式错误的内容，则访问`request.data`可能会引发`ParseError`。默认情况下REST framework的 `APIView`类或`@api_view`装饰器将捕获错误并返回`400 Bad Request`响应。

如果客户端发送具有无法解析的呃逆荣类型的请求，则会引发 `UnsupportedMediaType` 异常, 默认情况下会捕获该异常并返回 `415 Unsupported Media Type` 响应。

---

# Content negotiation（内容协商）

请求提供了一些属性允许你确定内容协商阶段的结果。这允许你实现具体的行为，例如为不同的媒体类型选择不用的序列化方案。

## .accepted_renderer

由内容协商阶段选择的render实例。

## .accepted_media_type

由内容协商阶段接受的媒体类型的字符串。

---

# Authentication（认证）

REST framework 提供了灵活的，每次请求的验证，让你能够：
* 对API的不同部分使用不同的身份验证策略。
* 支持使用多个身份验证策略。
* 提供与传入请求相关联的用户和令牌信息。

## .user

`request.user` 通常返回一个 `django.contrib.auth.models.User` 实例, 尽管该行为取决于所使用的的认证策略。

如果请求未认证则 `request.user` 的默认值为 `django.contrib.auth.models.AnonymousUser`的一个实例。

更多详细信息请查阅 [authentication documentation].

## .auth

`request.auth` 返回任何其他身份验证上下文。 `request.auth` 的确切行为取决于所使用的的认证策略，但它通常可以是请求被认证的token的实例。

如果请求未认证或者没有其他上下文，则 `request.auth` 的默认值为 `None`.

更多详细信息请查阅 [authentication documentation].

## .authenticators

`APIView` 类或 `@api_view` 装饰器将根据在view中设置的 `authentication_classes` 或基于`DEFAULT_AUTHENTICATORS` 设置，确保此属性自动设置为 `Authentication` 实例的列表。

你通常并不需要访问此属性。

---

# Browser enhancements（浏览器增强）

REST framework 支持一些浏览器增强功能，例如基于浏览器的 `PUT`, `PATCH` 和 `DELETE` 表单。

## .method

`request.method` 返回请求的HTTP方法的 **大写** 字符串表示形式。

透明地支持基于浏览器的 `PUT`, `PATCH` 和 `DELETE` 表单。

更多详细信息请查阅 [browser enhancements documentation].

## .content_type

`request.content_type` 返回表示HTTP请求正文的媒体类型的字符串对象，如果未提供媒体类型，则返回空字符串。

你通常不需要直接访问请求的内容类型，因为你通常将依赖于REST framework的默认请求解析行为。

如果你确实需要访问请求的内容类型，你应该使用 `.content_type` 属性，而不是使用 `request.META.get('HTTP_CONTENT_TYPE')`, 因为它为基于浏览器的非表单内容提供了透明的支持。

更多详细信息请查阅 [browser enhancements documentation].

## .stream

`request.stream` 返回一个表示请求主体内容的流。

你通常不需要直接访问请求的内容类型，因为你通常将依赖于REST framework的默认请求解析行为。


---

# Standard HttpRequest attributes（标准HttpRequest属性）

由于 REST framework 的 `Request` 扩展了 Django的 `HttpRequest`, 所以所有其他标准属性和方法也是可用的。例如 `request.META` 和 `request.session` 字典正常可用。

请注意，由于实现原因， `Request` 类并不会从 `HttpRequest` 类继承, 而是使用合成扩展类。


[cite]: https://groups.google.com/d/topic/django-developers/dxI4qVzrBY4/discussion
[parsers documentation]: ./parsers_zh.md
[authentication documentation]: authentication.md
[browser enhancements documentation]: ../topics/browser-enhancements.md