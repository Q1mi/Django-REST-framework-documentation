source: response.py

\# Responses

&gt; 与基本的HttpResponse对象不同, TemplateResponse 对象保留view提供的上下文的详细信息以计算 response.  Response的最终输出直到它在稍后的响应过程中被需要才会计算。

&gt;

&gt; — \[Django 文档\]\[cite\]

REST framework 通过提供一个\`Response\`类来支持 HTTP content negotiation，该类允许你返回可以呈现为多种内容类型的内容，具体取决于客户端的请求。

\`Response\` 类的子类 Django的 \`SimpleTemplateResponse\`。  \`Response\` 对象用Python基本数据类型初始化。 然后REST framework 使用标准的HTTP content negotiation 来确定如何呈现最终的相应内容。

你并不需要一定是用 \`Response\` 类， 你可以从你的视图返回常规的 \`HttpResponse\` 或者 \`StreamingHttpResponse\` 对象。  使用 \`Response\` 类只提供了一个可以呈现多种格式的更好的界面来返回 content-negotiated 的 Web API 响应。

除非由于某种原因你要对 REST framework 做大量的自定义，否则你应该始终对返回对象的views使用 \`APIView\` 类或者\`@api\_view\` 函数。这样做可以确保视图在返回之前能够执行 content negotiation 并且为响应选择适当的渲染器。

---

\# 创建 responses

\#\# Response\(\)

\*\*Signature:\*\* \`Response\(data, status=None, template\_name=None, headers=None, content\_type=None\)\`

与常规的 \`HttpResponse\` 对象不同，你不能使用渲染内容来实例化一个 \`Response\` 对象，而是传递未渲染的数据，包含任何Python基本数据类型。

\`Response\` 类使用的渲染器无法自行处理想Django model实例这样的复杂数据类型，因此你需要在创建\`Response\`对象之前将数据序列化为基本数据类型。

你可以使用 REST framework的\`Serializer\` 类来执行此类数据的序列化，或者使用你自定义的来序列化。

Arguments:

\* \`data\`: The serialized data for the response.

\* \`status\`: A status code for the response.  Defaults to 200.  See also \[status codes\]\[statuscodes\].

\* \`template\_name\`: A template name to use if \`HTMLRenderer\` is selected.

\* \`headers\`: A dictionary of HTTP headers to use in the response.

\* \`content\_type\`: The content type of the response.  Typically, this will be set automatically by the renderer as determined by content negotiation, but there may be some cases where you need to specify the content type explicitly.

---

\# Attributes

\#\# .data

The unrendered content of a \`Request\` object.

\#\# .status\_code

The numeric status code of the HTTP response.

\#\# .content

The rendered content of the response.  The \`.render\(\)\` method must have been called before \`.content\` can be accessed.

\#\# .template\_name

The \`template\_name\`, if supplied.  Only required if \`HTMLRenderer\` or some other custom template renderer is the accepted renderer for the response.

\#\# .accepted\_renderer

The renderer instance that will be used to render the response.

Set automatically by the \`APIView\` or \`@api\_view\` immediately before the response is returned from the view.

\#\# .accepted\_media\_type

The media type that was selected by the content negotiation stage.

Set automatically by the \`APIView\` or \`@api\_view\` immediately before the response is returned from the view.

\#\# .renderer\_context

A dictionary of additional context information that will be passed to the renderer's \`.render\(\)\` method.

Set automatically by the \`APIView\` or \`@api\_view\` immediately before the response is returned from the view.

---

\# Standard HttpResponse attributes

The \`Response\` class extends \`SimpleTemplateResponse\`, and all the usual attributes and methods are also available on the response.  For example you can set headers on the response in the standard way:

```
response = Response\(\)

response\['Cache-Control'\] = 'no-cache'
```

\#\# .render\(\)

\*\*Signature:\*\* \`.render\(\)\`

As with any other \`TemplateResponse\`, this method is called to render the serialized data of the response into the final response content.  When \`.render\(\)\` is called, the response content will be set to the result of calling the \`.render\(data, accepted\_media\_type, renderer\_context\)\` method on the \`accepted\_renderer\` instance.

You won't typically need to call \`.render\(\)\` yourself, as it's handled by Django's standard response cycle.

\[cite\]: [https://docs.djangoproject.com/en/stable/stable/template-response/](https://docs.djangoproject.com/en/stable/stable/template-response/)

\[statuscodes\]: status-codes.md

