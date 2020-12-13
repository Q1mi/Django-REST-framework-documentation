# 缓存

> 某个女人意识非常敏锐，但是几乎没有记忆 ... 她只记得如何工作，并且很努力。
> - Lydia Davis

REST Framework中的缓存是使用Django提供的缓存工具来实现的。

---

## 在apiview和viewset中使用缓存

Django提供了一个[`method_decorator`][decorator]的装饰器，来结合类视图使用。
它也可以和其他缓存重视起一起使用，例如[`cache_page`][page]和[`vary_on_cookie`][cookie]。

```python
from django.utils.decorators import method_decorator
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_cookie

from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework import viewsets


class UserViewSet(viewsets.ViewSet):

    # 为每个请求该接口的用户设置2个小时缓存
    @method_decorator(cache_page(60*60*2))
    @method_decorator(vary_on_cookie)
    def list(self, request, format=None):
        content = {
            'user_feed': request.user.get_user_feed()
        }
        return Response(content)


class PostView(APIView):

    # 为接口设置缓存
    @method_decorator(cache_page(60*60*2))
    def get(self, request, format=None):
        content = {
            'title': 'Post title',
            'body': 'Post content'
        }
        return Response(content)
```

**NOTE:** [`cache_page`][page]装饰器只缓存`GET`和`HEAD`的200状态码的返回

[page]: https://docs.djangoproject.com/en/dev/topics/cache/#the-per-view-cache
[cookie]: https://docs.djangoproject.com/en/dev/topics/http/decorators/#django.views.decorators.vary.vary_on_cookie
[decorator]: https://docs.djangoproject.com/en/dev/topics/class-based-views/intro/#decorating-the-class
