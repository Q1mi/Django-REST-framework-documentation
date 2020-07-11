
# Responses

> 与基本的HttpResponse对象不同, TemplateResponse 对象保留view提供的上下文的详细信息以计算 response.  Response的最终输出直到它在稍后的响应过程中被需要才会计算。
>

> — [Django 文档][cite]

REST framework 通过提供一个 `Response` 类来支持 HTTP content negotiation，该类允许你返回可以呈现为多种内容类型的内容，具体取决于客户端的请求。

`Response` 类是 Django中 `SimpleTemplateResponse` 类的一个子类。`Response` 对象用Python基本数据类型初始化。 然后REST framework 使用标准的HTTP content negotiation 来确定如何呈现最终的响应内容。

你并不需要一定是用 `Response` 类，你可以从你的视图返回常规的 `HttpResponse` 或者 `StreamingHttpResponse` 对象。使用`Response`类只提供了一个可以呈现多种格式的更好的界面来返回 content-negotiated 的 Web API 响应。

除非由于某种原因你要对 REST framework 做大量的自定义，否则你应该始终对返回对象的views使用 `APIView` 类或者 `@api_view` 函数。这样做可以确保视图在返回之前能够执行 content negotiation 并且为响应选择适当的渲染器。

---

# 创建 responses

## Response()

**签名:** `Response(data, status=None, template_name=None, headers=None, content_type=None)`

与常规的 `HttpResponse` 对象不同，你不能使用渲染内容来实例化一个 `Response` 对象，而是传递未渲染的数据，包含任何Python基本数据类型。

`Response` 类使用的渲染器无法自行处理像 Django model 实例这样的复杂数据类型，因此你需要在创建 `Response` 对象之前将数据序列化为基本数据类型。

你可以使用 REST framework的 `Serializer` 类来执行此类数据的序列化，或者使用你自定义的来序列化。

参数:

* `data`: response的数列化数据.

* `status`:  response的状态码。默认是200.  另行参阅 [status codes][statuscodes].

* `template_name`: `HTMLRenderer` 选择要使用的模板名称。

* `headers`: A dictionary of HTTP headers to use in the response.

* `content_type`: response的内容类型。通常由渲染器自行设置，由content negotiation确定，但是在某些情况下，你需要明确指定内容类型。

---

# 属性

## .data

`Request` 对象的未渲染内容。

## .status_code

HTTP 响应的数字状态吗。

## .content

response的呈现内容。 `.render()` 方法必须先调用才能访问 `.content` 。

## .template_name

`template_name` 只有在使用 `HTMLRenderer` 或者其他自定义模板作为response的渲染器时才需要提供该属性。

## .accepted_renderer

将用于呈现response的render实例。

自动通过 `APIView` 或者 `@api_view` 在view返回response之前设置。

## .accepted_media_type

由 content negotiation 阶段选择的媒体类型。

自动通过 `APIView` 或者 `@api_view` 在view返回response之前设置。

## .renderer_context

一个将传递给渲染器的`.render()`方法的附加上下文信息字典。

自动通过 `APIView` 或者 `@api_view` 在view返回response之前设置。

---

# 标准的HttpResponse 属性

`Response` 类扩展了 `SimpleTemplateResponse`，并且所有常用的属性和方法都是提供的。比如你可以使用标准的方法设置response的header信息：

```
response = Response()

response['Cache-Control'] = 'no-cache'
```

## .render()

**Signature:** `.render()`

和其他的 `TemplateResponse` 一样，调用该方法将response的序列化数据呈现为最终的response内容。 当 `.render()` 被调用时， response的内容将被设置成在 `accepted_renderer`实例上调用 `.render(data, accepted_media_type, renderer_context)` 方法返回的结果。

你通常并不需要自己调用 `.render()` ，因为它是由Django的标准响应周期来处理的。

[cite]: [https://docs.djangoproject.com/en/stable/stable/template-response/](https://docs.djangoproject.com/en/stable/stable/template-response/)

[statuscodes]: status-codes.md

