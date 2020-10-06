source: negotiation.py

# Content negotiation （内容协商）

>HTTP为“内容协商” ——即当存在多个可用表示方法时，为给定响应选择最佳表示的过程，提供了多种机制。
>
> &mdash; [RFC 2616][cite], Fielding 等人.

[cite]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec12.html

内容协商是根据用户端或服务器的首选项，从多个可能的表示方法中选择一个返回给客户端的过程。

## Determining the accepted renderer （确定可接受的渲染器）

REST框架使用一种简单的内容协商方式，根据可用的渲染器、每个渲染器的优先级以及客户端的 `Accept:` 头来确定应该将哪个媒体类型返回给客户端。使用的方式部分是客户端驱动的，部分是服务器驱动的。

1. 更具体的媒体类型优先于不太具体的媒体类型。
2. 如果多个媒体类型具有相同的特异性，则根据为给定视图配置的渲染器的顺序优先选择。

例如，给定以下 `Accept` 头：

    application/json; indent=4, application/json, application/yaml, text/html, */*

每种给定媒体类型的优先级为：

* `application/json; indent=4`
* `application/json`, `application/yaml` and `text/html`
* `*/*`

如果被请求的视图只配置了 `YAML` 和 `HTML` 的渲染器，那么REST framework将选择`renderer_classes` 列表或默认的 `DEFAULT_RENDERER_CLASSES` 设置中最先列出的渲染器。

有关 `HTTP Accept` 头的更多信息，请参见[RFC 2616]
[accept-header]

---




**Note**: REST框架在确定优先级时不考虑“q”值。“q”值的使用会对缓存产生负面影响，在作者看来，它们是一种不必要的、过于复杂的内容协商方法。

这是一种有效的方法，因为HTTP规范未明确说明服务器应该如何权衡基于服务器的首选项和基于客户端的首选项。

---

# Custom content negotiation （自定义内容协商）

您不太可能希望为REST框架提供一个定制的内容协商方案，但如果需要，您可以这样做。若要应用自定义内容协商方案，请重写 `BaseContentNegotiation`。

REST框架的内容协商类处理对请求的适当解析器和响应的适当渲染器的选择，因此您应该同时应用 `.select_parser(request, parsers)` 和 `.select_renderer(request, renderers, format_suffix)` 方法。

`select_parser()` 方法应该从可用解析器列表中返回一个解析器实例，如果没有解析器能够处理传入的请求，则返回 `None`。

`select_renderer()` 方法应返回一个双元组（渲染器实例，媒体类型），或抛出 `NotAcceptable` 异常。

## Example （举个例子）

下面是一个自定义内容协商类，它在选择适当的解析器或渲染器时忽略客户端请求。

    from rest_framework.negotiation import BaseContentNegotiation

    class IgnoreClientContentNegotiation(BaseContentNegotiation):
        def select_parser(self, request, parsers):
            """
            Select the first parser in the `.parser_classes` list.
            """
            return parsers[0]

        def select_renderer(self, request, renderers, format_suffix):
            """
            Select the first renderer in the `.renderer_classes` list.
            """
            return (renderers[0], renderers[0].media_type)

## Setting the content negotiation （设置内容协商）

可以使用`DEFAULT_CONTENT_NEGOTIATION_CLASS` 设置来全局设置默认的“内容协商”类。例如，以下设置将使用示例`IgnoreClientContentNegotiation`类。

    REST_FRAMEWORK = {
        'DEFAULT_CONTENT_NEGOTIATION_CLASS': 'myapp.negotiation.IgnoreClientContentNegotiation',
    }

还可以对使用基于 `APIView` 类的视图设置用于单个视图或视图集的内容协商。

	from myapp.negotiation import IgnoreClientContentNegotiation
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class NoNegotiationView(APIView):
        """
        An example view that does not perform content negotiation.
        """
        content_negotiation_class = IgnoreClientContentNegotiation

        def get(self, request, format=None):
            return Response({
                'accepted media type': request.accepted_renderer.media_type
            })

[accept-header]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html
