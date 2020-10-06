source: reverse.py

# Returning URLs （返回URL）

> The central feature that distinguishes the REST architectural style from other network-based styles is its emphasis on a uniform interface between components.
>
> &mdash; Roy Fielding, [Architectural Styles and the Design of Network-based Software Architectures][cite]

通常，从webapi返回绝对uri可能是更好的做法，例如 `http://example.com/foobar`，而不是返回相对url，例如 `/foobar`。

这样做的好处是：
* 更加明确。
* 为API客户端留下了更少的工作量。
* 当在JSON等没有本机URI类型的表示中找到字符串时，它的含义没有任何歧义。
* 使得使用超链接标记HTML表示之类的事情变得很容易。

REST框架提供了两个实用的函数，使从webapi返回绝对URI变得更加简便。

使用它们不是必要的，但是如果这样做，那么自描述API将能够自动将输出转为超链接，这使得浏览API更加容易。

## reverse

**Signature:** `reverse(viewname, *args, **kwargs)`

Has the same behavior as [`django.urls.reverse`][reverse], except that it returns a fully qualified URL, using the request to determine the host and port.

与 [`django.urls.reverse`][reverse] 具有相同的行为，但它使用请求来确定主机和端口,返回一个完全合格的URL，。

You should **include the request as a keyword argument** to the function, for example:
应当**将请求作为关键字参数包含到**函数中，例如：

    from rest_framework.reverse import reverse
    from rest_framework.views import APIView
	from django.utils.timezone import now

	class APIRootView(APIView):
	    def get(self, request):
	        year = now().year
			data = {
 				...
    		    'year-summary-url': reverse('year-summary', args=[year], request=request)
            }
    		return Response(data)

## reverse_lazy

**Signature:** `reverse_lazy(viewname, *args, **kwargs)`

与 [`django.urls.reverse_lazy`][reverse-lazy] 具有相同的行为，但它使用请求来确定主机和端口,返回一个完全合格的URL。

与 `reverse` 函数一样，应该**将请求作为关键字参数包含**到函数中，例如：

    api_root = reverse_lazy('api-root', request=request)

[cite]: http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm#sec_5_1_5
[reverse]: https://docs.djangoproject.com/en/stable/topics/http/urls/#reverse
[reverse-lazy]: https://docs.djangoproject.com/en/stable/topics/http/urls/#reverse-lazy
