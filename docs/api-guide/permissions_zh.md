URLsource: permissions.py

# 权限

> 身份验证或身份识别本身通常不足以获取信息或代码的访问权限。因此，请求访问的实体必须具有授权。
>
> &mdash; [Apple Developer Documentation][cite]

连同[认证][authentication]和[限制][throttling]，权限决定是否应该接收请求或拒绝访问。

权限检查始终在视图的开始处运行，在允许继续执行任何其他代码之前运行。
权限检查通常会使用`request.user`和`request.auth`属性中的身份验证信息来确定是否允许传入请求。

权限用于授予或拒绝将不同类别的用户访问到API的不同部分。

最简单的权限是允许访问任何经过身份验证的用户，并拒绝访问任何未经身份验证的用户。这对应于REST框架中的`IsAuthenticated`类。

稍微严格的权限控制风格是对经过身份验证的用户的允许完全访问，但对未经身份验证的用户的允许只读访问。这对应于REST框架中的`IsAuthenticatedOrReadOnly`类。

## 如何确定权限

REST框架中的权限始终被定义为一个权限类的列表。

在运行视图的主体之前，检查列表中的每个权限。
如果任何权限检查失败，会抛出一个`exceptions.PermissionDenied`或`exceptions.NotAuthenticated`异常，并且视图的主体将不会运行。

当权限检查失败时，将返回"403 Forbidden"或"401 Unauthorized"响应，具体根据以下规则：

* 请求已成功通过身份验证，但权限被拒绝。 *&mdash; 将返回403 Forbidden响应。*
* 请求未成功认证，最高优先级的认证类*不使用*`WWW-Authenticate`标头。*&mdash; 将返回403 Forbidden响应。*
* 请求未成功认证，最高优先级的认证类*使用*`WWW-Authenticate`标头。*&mdash; 将返回HTTP 401未经授权的响应，并附带适当的`WWW-Authenticate`标头。*

## 对象级权限

REST框架权限还支持对象级的许可。对象级权限用于确定是否允许用户操作特定对象（通常是模型实例）。

当调用`.get_object()`时，由REST框架的通用视图运行对象级权限检测。
与视图级别权限一样，如果不允许用户操作给定对象，则会抛出`exceptions.PermissionDenied`异常。

如果你正在编写自己的视图并希望强制执行对象级权限检测，或者你想在通用视图中重写`get_object`方法，那么你需要在检索对象的时候显式调用视图上的`.check_object_permissions(request, obj)`方法。

这将抛出`PermissionDenied`或`NotAuthenticated`异常，或者如果视图具有适当的权限，则返回。

例如：

    def get_object(self):
        obj = get_object_or_404(self.get_queryset())
        self.check_object_permissions(self.request, obj)
        return obj

#### 对象级别的权限限制

出于性能原因，通用视图在返回对象列表时不会自动将对象级权限应用于查询集中的每个实例。

通常，当你使用对象级权限时，你还需要适当地[过滤查询集][filtering]过滤查询集，以确保用户只能看到他们被允许查看的实例。

## 设置权限策略

默认权限策略可以使用`DEFAULT_PERMISSION_CLASSES`设置进行全局设置。比如：

    REST_FRAMEWORK = {
        'DEFAULT_PERMISSION_CLASSES': (
            'rest_framework.permissions.IsAuthenticated',
        )
    }

如果未指定，则此设置默认为允许无限制访问：

    'DEFAULT_PERMISSION_CLASSES': (
       'rest_framework.permissions.AllowAny',
    )

你还可以使用基于`APIView`类的视图在每个视图或每个视图集的基础上设置身份验证策略。

    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ExampleView(APIView):
        permission_classes = (IsAuthenticated,)

        def get(self, request, format=None):
            content = {
                'status': 'request was permitted'
            }
            return Response(content)

或者你可以使用`@api_view`装饰器装饰基于函数的视图。

    from rest_framework.decorators import api_view, permission_classes
    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response

    @api_view(['GET'])
    @permission_classes((IsAuthenticated, ))
    def example_view(request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)

__注意：__ 当你通过类属性或装饰器设置新的权限类时，你要通知视图忽略__settings.py__文件中设置的默认列表。

---

# API 参考

## AllowAny

`AllowAny`权限类将允许不受限制的访问，而**不管该请求是否已通过身份验证或未经身份验证**。

此权限不是严格要求的，因为你可以通过使用空列表或元组进行权限设置来获得相同的结果，但你可能会发现指定此类很有用，因为它使意图更明确。

## IsAuthenticated

`IsAuthenticated` 权限类将拒绝任何未经身份验证的用户的权限，并允许其他权限。 如果你希望你的API仅供注册用户访问，则此权限适用。

如果你希望你的API允许匿名用户读取权限，并且只允许对已通过身份验证的用户进行写入权限，则此权限是适合的。

## IsAdminUser

除非`user.is_staff`为`True`，否则`IsAdminUser`权限类将拒绝任何用户的权限，在这种情况下将允许权限。

如果你希望你的API只能被部分受信任的管理员访问，则此权限是适合的。

## IsAuthenticatedOrReadOnly

`IsAuthenticatedOrReadOnly` 将允许经过身份验证的用户执行任何请求。只有当请求方法是“安全”方法（`GET`, `HEAD` 或 `OPTIONS`）之一时，才允许未经授权的用户请求。

如果你希望你的API允许匿名用户读取权限，并且只允许对已通过身份验证的用户进行写入权限，则此权限是适合的。

## DjangoModelPermissions

此权限类与Django的标准`django.contrib.auth`[model权限][contribauth]相关联。此权限只能应用于具有`.queryset`属性集的视图。只有在用户*通过身份验证*并分配了*相关模型权限*的情况下，才会被授予权限。

* `POST` 请求要求用户对模型具有`添加`权限。
* `PUT` 和 `PATCH` 请求要求用户对模型具有`更改`权限。
* `DELETE` 请求想要求用户对模型具有`删除`权限。

默认行为也可以被重写以支持自定义模型权限。例如，你可能希望为`GET`请求包含一个`查看`模型的权限。

要使用自定义模型权限，请覆盖`DjangoModelPermissions`并设置`.perms_map`属性。有关详细信息，请参阅源代码。

#### 使用不包含`queryset`属性的视图。

如果你在重写了`get_queryset()`方法的视图中使用此权限，这个视图上可能没有`queryset`属性。在这种情况下，我们建议还使用保护性的查询集来标记视图，以便此类可以确定所需的权限。比如：

    queryset = User.objects.none()  # DjangoModelPermissions需要一个queryset

## DjangoModelPermissionsOrAnonReadOnly

与`DjangoModelPermissions`类似，但也允许未经身份验证的用户具有对API的只读访问权限。

## DjangoObjectPermissions

此权限类与Django的标准[对象权限框架][objectpermissions]相关联，该框架允许模型上的每个对象的权限。为了使用此权限类，你还需要添加支持对象级权限的权限后端，例如[django-guardian][guardian]。

与`DjangoModelPermissions`一样，此权限只能应用于具有`.queryset`属性或`.get_queryset()`方法的视图。只有在用户通过身份验证并且具有*相关的每个对象权限*和相关的*模型权限*后，才会被授予权限。

* `POST` 请求要求用户对模型实例具有`添加`权限。
* `PUT` 和`PATCH` 请求要求用户对模型示例具有`更改`权限。
* `DELETE` 请求要求用户对模型示例具有`删除`权限。

请注意，`DjangoObjectPermissions` **不需要** `django-guardian`软件包，并且应当同样支持其他对象级别的后端。

与`DjangoModelPermissions`一样，你可以通过重写`DjangoObjectPermissions`并设置`.perms_map`属性来使用自定义模型权限。有关详细信息，请参阅源代码。

---

**注意**：如果你需要`GET`,`HEAD`和`OPTIONS`请求的对象级`视图`权限，那么你还需要考虑添加`DjangoObjectPermissionsFilter`类，以确保相应API只返回包含用户具有适当视图权限的对象的结果。

---

---

# 自定义权限

要实现自定义权限，请重写`BasePermission`并实现以下方法中的一个或两个

* `.has_permission(self, request, view)`
* `.has_object_permission(self, request, view, obj)`

如果请求被授予访问权限，方法应该返回`True`，否则返回`False`。

如果你需要测试请求是读取操作还是写入操作，则应该根据常量`SAFE_METHODS`检查请求方法，`SAFE_METHODS`是包含`'GET'`, `'OPTIONS'`和`'HEAD'`的元组。例如：

    if request.method in permissions.SAFE_METHODS:
        # 检查只读请求的权限
    else:
        # 检查读取请求的权限

---

**注意**: 仅当视图级`has_permission`检查已通过时，才会调用实例级`has_object_permission`方法。另请注意，为了运行实例级别检查，视图代码应显式调用`.check_object_permissions(request, obj)`。如果你使用的是通用视图，那么默认会为你处理。

---

如果校验失败，自定义权限将引发`PermissionDenied`异常。要更改与异常关联的错误消息，请直接在自定义权限上实现消息属性。否则将使用`PermissionDenied`的`default_detail`属性。

    from rest_framework import permissions

    class CustomerAccessPermission(permissions.BasePermission):
        message = 'Adding customers not allowed.'
        
        def has_permission(self, request, view):
             ...
        
## 示例

以下是根据黑名单检查传入请求的IP地址的权限类的示例，如果IP已被列入黑名单，则拒绝该请求。

    from rest_framework import permissions

    class BlacklistPermission(permissions.BasePermission):
        """
        对列入黑名单的IP进行全局权限检查。
        """

        def has_permission(self, request, view):
            ip_addr = request.META['REMOTE_ADDR']
            blacklisted = Blacklist.objects.filter(ip_addr=ip_addr).exists()
            return not blacklisted

除了针对所有传入请求运行的全局权限外，还可以创建对象级权限，这些权限仅针对影响特定对象实例的操作运行。例如：

    class IsOwnerOrReadOnly(permissions.BasePermission):
        """
        对象级权限仅允许对象的所有者对其进行编辑
        假设模型实例具有`owner`属性。
        """

        def has_object_permission(self, request, view, obj):
            # 任何请求都允许读取权限，
            # 所以我们总是允许GET，HEAD或OPTIONS 请求.
            if request.method in permissions.SAFE_METHODS:
                return True

            # 示例必须要有一个名为`owner`的属性
            return obj.owner == request.user

请注意，通用视图将检查适当的对象级权限，但如果你正在编写自己的自定义视图，则需要确保检查自己的对象级权限检查。你可以通过在获得对象实例后从视图中调用`self.check_object_permission(request，obj)`来执行此操作.如果任何对象级权限检查失败，此调用将引发适当的`APIException`，否则将简单地返回。

另请注意，通用视图仅检查检索单个模型实例的视图的对象级权限。如果你需要列表视图的对象级过滤，则需要单独过滤查询集。有关详细信息，请参阅[filtering文档][filtering]过滤文档。

---

# 第三方包

以下第三方包都可以使用。

## Composed Permissions

[Composed Permissions][composed-permissions]提供了一种简单的方法，使用小的可重用组件来定义复杂和多深度（使用逻辑运算符）权限对象。

## REST Condition

[REST Condition][rest-condition]包是用于以简单方便的方式构建复杂权限的另一个扩展。该扩展允许你将权限与逻辑运算符组合在一起。

## DRY Rest Permissions

[DRY Rest Permissions][dry-rest-permissions]包提供了为单个默认和自定义操作定义不同权限的功能。此包适用于具有从应用程序数据模型中定义的关系派生的权限的应用程序。它还支持通过API的序列化程序返回到客户端应用程序的权限检查。此外，它还支持向默认和自定义列表操作添加权限，以限制他们为每个用户检索的数据。

## Django Rest Framework Roles

[Django Rest Framework Roles][django-rest-framework-roles]包使你可以更轻松地通过多种类型的用户对API进行参数化。

## Django Rest Framework API Key

[Django Rest Framework API Key][django-rest-framework-api-key]包允许你确保向服务器发出的每个请求都需要一个API密钥标头。您可以从django管理界面生成一个。

[cite]: https://developer.apple.com/library/mac/#documentation/security/Conceptual/AuthenticationAndAuthorizationGuide/Authorization/Authorization.html
[authentication]: authentication_zh.md
[throttling]: throttling.md
[filtering]: filtering_zh.md
[contribauth]: https://docs.djangoproject.com/en/stable/topics/auth/customizing/#custom-permissions
[objectpermissions]: https://docs.djangoproject.com/en/stable/topics/auth/customizing/#handling-object-permissions
[guardian]: https://github.com/lukaszb/django-guardian
[get_objects_for_user]: http://pythonhosted.org/django-guardian/api/guardian.shortcuts.html#get-objects-for-user
[2.2-announcement]: ../topics/2.2-announcement.md
[filtering]: filtering_zh.md
[drf-any-permissions]: https://github.com/kevin-brown/drf-any-permissions
[composed-permissions]: https://github.com/niwibe/djangorestframework-composed-permissions
[rest-condition]: https://github.com/caxap/rest_condition
[dry-rest-permissions]: https://github.com/Helioscene/dry-rest-permissions
[django-rest-framework-roles]: https://github.com/computer-lab/django-rest-framework-roles
[django-rest-framework-api-key]: https://github.com/manosim/django-rest-framework-api-key
