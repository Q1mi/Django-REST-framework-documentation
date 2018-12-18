# 基于类的视图

> django的基于类的视图对于旧式风格的视图来说是良好的替代品。
>
> &mdash; [reinout van rees][cite]

REST framework提供Django的`APIView`类，它是Django的`View`类的子类。

`APIView`类和一般的`View`类有以下不同：

* 被传入到处理方法的请求不会是Django的`HttpRequest`类的实例，而是REST framework的`Request`类的实例。

* 处理方法可以返回REST framework的`Response`，而不是Django的`HttpRequest`。视图会管理内容协议，给响应设置正确的渲染器。

* 任何`APIException`异常都会被捕获，并且传递给合适的响应。

* 进入的请求将会经过认证，合适的权限和（或）节流检查会在请求被派发到处理方法之前运行。

使用`APIView`类和使用一般的`View`类非常相似，通常，进入的请求会被分发到合适处理方法比如`.get()`，或者`.post`。另外，很多属性会被设定在控制API策略的各种切面的类上。

比如：

    from rest_framework.views import APIView
    from rest_framework.response import Response
    from rest_framework import authentication, permissions

    class ListUsers(APIView):
        """
        列出系统中的所有用户的视图。

        * 需要token认证
        * 只有管理员用户可以访问这个视图。
        """
        authentication_classes = (authentication.TokenAuthentication,)
        permission_classes = (permissions.IsAdminUser,)

        def get(self, request, format=None):
            """
            Return a list of all users.
            """
            usernames = [user.username for user in User.objects.all()]
            return Response(usernames)

## API策略属性

下面这些属性控制API视图的可拔插的切面。

### .renderer_classes

### .parser_classes

### .authentication_classes

### .throttle_classes

### .permission_classes

### .content_negotiation_class

## API policy instantiation methods

下面这些方法被REST framework用来实例化各种可拔插的API策略。你通常不需要重写这些方法。

### .get_renderers(self)

### .get_parsers(self)

### .get_authenticators(self)

### .get_throttles(self)

### .get_permissions(self)

### .get_content_negotiator(self)

### .get_exception_handler(self)

## API policy implementation methods

下面这些方法会在请求被分发到处理方法之前被调用。

### .check_permissions(self, request)

### .check_throttles(self, request)

### .perform_content_negotiation(self, request, force=False)

## Dispatch methods

下面这些方法会被视图的`.dispatch()`方法直接调用。
在调用像`.get`, `.post()`, `put()`, `patch()`和`delete()`之类的请求处理方法之前或者之后,这些方法会进行任何需要所需的任何动作。

### .initial(self, request, \*args, **kwargs)

进行任何需要在处理方法被调用之前进行的动作。
这个方法被用来进行权限认证和请求丢弃，并且实施内容协议。

### .handle_exception(self, exc)

任何被处理请求的方法抛出的异常都会被传递给这个方法，这个方法既不返回`Response`的实例，也不重新抛出异常。

默认的实现处理任何`rest_framework.expceptions.APIException`的子类，比如Django的`Http404`和`PermissionDenied`异常，之后返回一个合适的错误响应。

如果你需要自定义你的API返回的错误响应，你需要重写这个方法。

### .initialize_request(self, request, \*args, **kwargs)

确保传递给处理请求的函数的请求对象的`Request`的实例，而不是一般的Django的`HttpResquest`的实例。

你通常不需要重写这个方法。

### .finalize_response(self, request, response, \*args, **kwargs)

确保任何从处理请求的方法返回的`Response`对象被渲染到正确的，被内容协议决定的内容类型。

你通常不需要重写这个方法。

---

# 基于

> 说[基于类的视图]不管什么时候都是更好的解决方案，那是错误的。
>
> &mdash; [Nick Coghlan][cite2]

在REST framework中，你也可以使用一般的基于函数的视图。它提供了一个简单的装饰器，用来包转你的基于函数的视图，以确保基于函数的视图会收到`Request`（而不是Django一般的`HttpRequest`)的实例，并且允许基于函数的视图返回`Response`（而不是Django的`HttpResponse`），以及允许你设置这个请求如何被处理。

## @api_view()

**函数签名:** `@api_view(http_method_names=['GET'], exclude_from_schema=False)`

这个函数的核心功能是提供`api_view`装饰器，这个装饰器设置你的视图需要响应的http动词。比如，你可以像这样写一个返回一些数据的非常简单的视图。

    from rest_framework.decorators import api_view

    @api_view()
    def hello_world(request):
        return Response({"message": "Hello, world!"})

这个视图会使用[settings]设置的默认的渲染器，解析器，认证类等等。

在默认的情况下，只有`GET`请求会被接受。其他的http动词会得到一个"405 Method Not Allowed"响应。可以指定这个视图允许的动词，来改变这一行为，就像是

    @api_view(['GET', 'POST'])
    def hello_world(request):
        if request.method == 'POST':
            return Response({"message": "Got some data!", "data": request.data})
        return Response({"message": "Hello, world!"})

你也可以用`exclude_from_schema`参数标记API视图来忽略任何[自动生成的视图][schemas],

    @api_view(['GET'], exclude_from_schema=True)
    def api_docs(request):
        ...

## API 策略装饰器

REST framework提供了一组可以加到视图上的装饰器来重写默认设置。这些装饰器必须放在`@api_view`的*后*(下)面。比如，可以用`@throttle_classes`装饰器来传递一个节流器类的列表，使用[节流器][throtting]来保证一个视图每天只能被每个用户调用一次。

    from rest_framework.decorators import api_view, throttle_classes
    from rest_framework.throttling import UserRateThrottle

    class OncePerDayUserThrottle(UserRateThrottle):
            rate = '1/day'

    @api_view(['GET'])
    @throttle_classes([OncePerDayUserThrottle])
    def view(request):
        return Response({"message": "Hello for today! See you tomorrow!"})

这些装饰器和上文中的`APIView`的子类的属性相对应。

可用的装饰器有：

* `@renderer_classes(...)`
* `@parser_classes(...)`
* `@authentication_classes(...)`
* `@throttle_classes(...)`
* `@permission_classes(...)`

每个参数都只有一个参数，这个参数必须是类的列表或元组。

[cite]: http://reinout.vanrees.org/weblog/2011/08/24/class-based-views-usage.html
[cite2]: http://www.boredomandlaziness.org/2012/05/djangos-cbvs-are-not-mistake-but.html
[settings]: settings.md
[throttling]: throttling.md
[schemas]: schemas.md
