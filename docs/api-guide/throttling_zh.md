source: throttling.py

# Throttling （限流）

> HTTP/1.1 420 Enhance Your Calm
>
> [Twitter API rate limiting response][cite]

类似于[权限]，限流器决定是否应当授权请求。限流器指示临时状态，并被用于控制客户端能够向API发出请求的频率。

与权限一样，多个限流器可能会被应用。您的API或许会对未经认证的请求进行限制，而对于已经认证的请求进行相对更少的限制。

另一个您可能会想要使用多个限流器的情况是由于某些服务特别耗费资源，因此您需要对API的不同部分施加不同的限制。

如果需要同时应用突发限制流量和持续限制流量，也可以使用多个限流器。例如，您可能希望将用户限制为每分钟最多60个请求，每天最多1000个请求。

限流器不一定仅指限制请求的频次。例如，存储服务可能还需要限制带宽，而付费数据服务则可能需要限制被访问的记录的特定数量。

## How throttling is determined （如何决定限流）

与权限和身份验证一样，REST框架中的限制总是被定义为一系列的类。

在运行视图主体之前，列表中的每一个限流器会被检查。如果任何限流器检查异常，则将抛出`exceptions.Throttled`，并且视图主体将不会运行。

## Setting the throttling policy （设定限流策略）

通过使用 `DEFAULT_THROTTLE_CLASSES` 和 `DEFAULT_THROTTLE_RATES` 默认限流策略将被全局设定,例如：

    REST_FRAMEWORK = {
        'DEFAULT_THROTTLE_CLASSES': (
            'rest_framework.throttling.AnonRateThrottle',
            'rest_framework.throttling.UserRateThrottle'
        ),
        'DEFAULT_THROTTLE_RATES': {
            'anon': '100/day',
            'user': '1000/day'
        }
    }

被用于 `DEFAULT_THROTTLE_RATES` 中的流量描述可能包括 `second`, `minute`, `hour` or `day` 作为限流周期

对于基于 `APIView` 类的视图，您可以以每个视图或每个视图集为基础设置限流策略

	from rest_framework.response import Response
    from rest_framework.throttling import UserRateThrottle
	from rest_framework.views import APIView

    class ExampleView(APIView):
        throttle_classes = (UserRateThrottle,)

        def get(self, request, format=None):
            content = {
                'status': 'request was permitted'
            }
            return Response(content)

或者，如果您使用带有 `@api_view` 装饰器的基于函数的视图

    @api_view(['GET'])
    @throttle_classes([UserRateThrottle])
    def example_view(request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)

## How clients are identified （如何识别客户端)

`X-Forwarded-For HTTP` 请求头和 `REMOTE_ADDR WSGI` 变量用于唯一地标识用户的IP地址。如果存在 `X-Forwarded-For` 请求头，则将使用它，否则将使用WSGI环境中 `REMOTE_ADDR` 变量的值。

如果需要严格标识唯一的客户端IP地址，则需要首先通过设置 `NUM_proxies` 设置来配置API后运行的应用程序代理的数量。此设置应为零或更大的整数。如果设置为非零，那么一旦第一次排除任何应用程序代理IP地址，客户端IP将被标识为 `·X-Forwarded-For·` 标头中的最后一个IP地址。如果设置为零，则 `Remote-Addr` 头将始终用于标识IP地址。

重要的是要了解，如果配置 `NUM_PROXIES` 设置，那么一个唯一的[NAT'd](http://en.wikipedia.org/wiki/Network_address_translation)网关后面的所有客户端都将被视为单个客户端。

关于 `X-Forwarded-For` 请求头如何工作以及标识远程客户端IP的更多上下文可以在[这里][identifing-clients]找到。

## Setting up the cache (设置缓存)

REST框架提供的节流类使用Django的缓存后端。您应该确保已设置适当的[缓存设置][cache-setting]。对于简单设置，后端的默认值 `LocMemCache` 即可。有关更多详细信息，请参见Django的[缓存文档][cache-docs]。

如果您需要使用 `'default'` 以外的缓存，则可以通过创建自定义的节流类并设置缓存属性来实现。 例如：

    class CustomAnonRateThrottle(AnonRateThrottle):
        cache = get_cache('alternate')

您需要记住，还需要在 `'DEFAULT_THROTTLE_CLASSES'` 设置键中设置您的自定义限流器，或使用 `throttle_classes` 视图属性。

---

# API Reference

## AnonRateThrottle

`AnonRateThrottle` 会限制未经身份验证的用户。传入请求的IP地址用于生成唯一的密钥以进行限制。

允许的请求频次由以下之一确定（按优先顺序）。

* 类的 `rate` 属性，可以通过重写 `AnonRateThrottle` 并设置该属性来提供。
* `DEFAULT_THROTTLE_RATES['anon']` 设置。

如果要限制来自未知源的请求速率，则`AnonRateThrottle` 是合适的。

## UserRateThrottle

`UserRateThrottle` 将通过API将用户限制为给定的请求速率。用户ID用于生成唯一的密钥以进行限制。未经身份验证的请求将退回到使用传入请求的IP地址来生成唯一密钥以进行限制。

允许的请求速率由以下之一确定（按优先顺序）。

* 类的 `rate` 属性，可以通过重写 `UserRateThrottle` 并设置该属性来提供。
* `DEFAULT_THROTTLE_RATES['user']` 设置。

一个API可能同时具有多个 `UserRateThrottles`。为此，请重写 `UserRateThrottle` 并为每个类设置一个唯一的“作用域”。

例如，可以通过使用以下类来实现多个用户节流率...

    class BurstRateThrottle(UserRateThrottle):
        scope = 'burst'

    class SustainedRateThrottle(UserRateThrottle):
        scope = 'sustained'

...和以下设置。

    REST_FRAMEWORK = {
        'DEFAULT_THROTTLE_CLASSES': (
            'example.throttles.BurstRateThrottle',
            'example.throttles.SustainedRateThrottle'
        ),
        'DEFAULT_THROTTLE_RATES': {
            'burst': '60/min',
            'sustained': '1000/day'
        }
    }

`UserRateThrottle` is suitable if you want simple global rate restrictions per-user.

## ScopedRateThrottle

`ScopedRateThrottle` 类可用于限制对API特定部分的访问。仅当所访问的视图包含. `throttle_scope` 属性时，才会应用此限流器。然后，通过将请求的“范围”与唯一的用户ID或IP地址串联起来，即可形成唯一的限制键。

允许的请求速率由 `DEFAULT_THROTTLE_RATES`设置使用请求“scope”中的键确定。

例如，给定以下视图...

    class ContactListView(APIView):
        throttle_scope = 'contacts'
        ...

    class ContactDetailView(APIView):
        throttle_scope = 'contacts'
        ...

    class UploadView(APIView):
        throttle_scope = 'uploads'
        ...

...和以下设置。

    REST_FRAMEWORK = {
        'DEFAULT_THROTTLE_CLASSES': (
            'rest_framework.throttling.ScopedRateThrottle',
        ),
        'DEFAULT_THROTTLE_RATES': {
            'contacts': '1000/day',
            'uploads': '20/day'
        }
    }

用户对 `ContactListView` 或 `ContactDetailView` 的请求将被限制为每天共计1000个请求。用户对 `UploadView` 的请求将被限制为每天共计20个请求。

---

# Custom throttles （自定义限流）

若要创建自定义限流器，请重写 `BaseThrottle` 并使用 `.allow_request(self, request, view)`。如果允许请求，则该方法应返回 `True` ，否则返回 `False`。

你也可以选择性地重写 `.wait()`方法。如果被使用，`.wait()` 应返回尝试下一个请求前建议的等待秒数，或者 `None`。只有在 `.allow_request()` 先前已返回 `False` 时才会调用 `.wait()`方法。

如果使用了 `.wait()` 方法并限制了请求，则响应中将包含 `Retry-After` 头。

## Example

下面是一个限流的例子，它将在每10个请求中随机地限制1个。

    import random

    class RandomRateThrottle(throttling.BaseThrottle):
        def allow_request(self, request, view):
            return random.randint(1, 10) != 1

[cite]: https://dev.twitter.com/docs/error-codes-responses
[permissions]: permissions.md
[identifing-clients]: http://oxpedia.org/wiki/index.php?title=AppSuite:Grizzly#Multiple_Proxies_in_front_of_the_cluster
[cache-setting]: https://docs.djangoproject.com/en/stable/ref/settings/#caches
[cache-docs]: https://docs.djangoproject.com/en/stable/topics/cache/#setting-up-the-cache
