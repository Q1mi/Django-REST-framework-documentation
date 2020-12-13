# 基于类的视图

> Django中基于类的视图对于旧式风格的视图来说是良好的替代品。
>
> &mdash; [reinout van rees][cite]

REST framework提供了一个`APIView`类，它是Django的`View`类的子类。

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

下面这些属性控制了API视图可拔插的那些方面。

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

下面这些方法会在请求被分发到具体的处理方法之前调用。

### .check_permissions(self, request)

### .check_throttles(self, request)

### .perform_content_negotiation(self, request, force=False)

## Dispatch methods

下面这些方法会被视图的`.dispatch()`方法直接调用。它们在调用`.get`, `.post()`, `put()`, `patch()`和`delete()`之类的请求处理方法之前或者之后执行任何需要执行的操作。

### .initial(self, request, \*args, **kwargs)

在处理方法调用之前进行任何需要的动作。
这个方法用于执行权限认证和限制，并且执行内容协商。
你通常不需要重写此方法。

### .handle_exception(self, exc)

任何被处理请求的方法抛出的异常都会被传递给这个方法，这个方法既不返回`Response`的实例，也不重新抛出异常。

默认的实现会处理`rest_framework.expceptions.APIException`的任何子类异常，以及Django的`Http404`和`PermissionDenied`异常，并且返回一个适当的错误响应。

如果你需要在自己的API中自定义返回的错误响应，你需要重写这个方法。

### .initialize_request(self, request, \*args, **kwargs)

确保传递给请求处理方法的请求对象是`Request`的实例，而不是通常的Django`HttpResquest`的实例。

你通常不需要重写这个方法。

### .finalize_response(self, request, response, \*args, **kwargs)

确保任何从处理请求的方法返回的`Response`对象被渲染到由内容协商决定的正确内容类型。

你通常不需要重写这个方法。

---

# 基于函数的视图

> 说[基于类的视图]不管什么时候都是更好的解决方案，那是错误的。
>
> &mdash; [Nick Coghlan][cite2]

在REST framework中，你也可以使用常规的基于函数的视图。它提供了一组简单的装饰器，用来包装你的视图函数，以确保视图函数会收到`Request`（而不是Django一般的`HttpRequest`)对象，并且返回`Response`（而不是Django的`HttpResponse`）对象，同时允许你设置这个请求的处理方式。

## @api_view()

**函数签名:** `@api_view(http_method_names=['GET'], exclude_from_schema=False)`

此功能的核心是`api_view`装饰器，它接受视图应该响应的HTTP方法列表的参数。
比如，你可以像这样写一个返回一些数据的非常简单的视图。

    from rest_framework.decorators import api_view

    @api_view()
    def hello_world(request):
        return Response({"message": "Hello, world!"})

这个视图会使用[settings]中指定的默认的渲染器，解析器，认证类等等。

默认的情况下，只有`GET`请求会被接受。其他的请求方法会得到一个"405 Method Not Allowed"响应。可以像下面的示例代码一样改变默认行为：

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

REST framework提供了一组可以加到视图上的装饰器来重写默认设置。这些装饰器必须放在`@api_view`的*后*(下)面。比如，要创建一个使用[限制器][throttling]确保特定用户每天只能调用一次的视图，可以用`@throttle_classes`装饰器并给它传递一个限制器类的列表。

    from rest_framework.decorators import api_view, throttle_classes
    from rest_framework.throttling import UserRateThrottle

    class OncePerDayUserThrottle(UserRateThrottle):
            rate = '1/day'

    @api_view(['GET'])
    @throttle_classes([OncePerDayUserThrottle])
    def view(request):
        return Response({"message": "Hello for today! See you tomorrow!"})

这些装饰器和上文中的`APIView`的子类中设置的属性相对应。

可用的装饰器有：

* `@renderer_classes(...)`
* `@parser_classes(...)`
* `@authentication_classes(...)`
* `@throttle_classes(...)`
* `@permission_classes(...)`

这些装饰器中的每一个都接受一个参数，这个参数必须是类的列表或元组。

[cite]: http://reinout.vanrees.org/weblog/2011/08/24/class-based-views-usage.html
[cite2]: http://www.boredomandlaziness.org/2012/05/djangos-cbvs-are-not-mistake-but.html
[settings]: settings.md
[throttling]: throttling.md
[schemas]: schemas.md
