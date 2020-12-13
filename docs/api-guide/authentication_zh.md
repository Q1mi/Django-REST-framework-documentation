
# 认证

> 身份验证功能需要可插拔。
>
> &mdash; Jacob Kaplan-Moss, ["REST worst practices"][cite]

身份验证是将传入请求与一组标识凭据（例如请求来自的用户或其签名的令牌）相关联的机制。然后，[权限][permission] 和 [限制][throttling] 可以使用这些凭据来确定是否应允许该请求。

REST framework 提供了一些开箱即用的身份验证方案，并且还允许你实现自定义方案。

验证始终在视图的最开始进行，在执行权限和限制检查之前以及允许任何其他代码继续执行之前。

`request.user` 属性通常被设置为`contrib.auth` 包中 `User` 类的一个实例。

`request.auth` 属性用于任何其他身份验证信息，例如，它可以用于表示请求签名的身份验证令牌。

---

**注意：** 不要忘了**认证本身不会允许或拒绝传入的请求**，它只是简单识别请求携带的凭证。

有关如何设置API权限策略的信息，请参阅 [权限文档][permission]。

---

## 如何确定身份验证

认证方案总是被定义为一个类的列表。REST framework 将尝试使用列表中的每个类进行身份验证，并使用成功完成验证的第一个类的返回值设置 `request.user` 和`request.auth`。

如果没有类进行验证，`request.user` 将被设置成 `django.contrib.auth.models.AnonymousUser`的实例，`request.auth` 将被设置成`None`。

未认证请求的`request.user` 和 `request.auth` 的值可以使用 `UNAUTHENTICATED_USER`和`UNAUTHENTICATED_TOKEN` 设置进行修改。

## 设置认证方案

可以使用 `DEFAULT_AUTHENTICATION_CLASSES` 设置全局的默认身份验证方案。比如：

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': (
            'rest_framework.authentication.BasicAuthentication',
            'rest_framework.authentication.SessionAuthentication',
        )
    }

你还可以使用基于`APIView`类视图的方式，在每个view或每个viewset基础上设置身份验证方案。

    from rest_framework.authentication import SessionAuthentication, BasicAuthentication
    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ExampleView(APIView):
        authentication_classes = (SessionAuthentication, BasicAuthentication)
        permission_classes = (IsAuthenticated,)

        def get(self, request, format=None):
            content = {
                'user': unicode(request.user),  # `django.contrib.auth.User` 实例。
                'auth': unicode(request.auth),  # None
            }
            return Response(content)

或者，如果你使用基于函数的视图，那就使用`@api_view`装饰器。

    @api_view(['GET'])
    @authentication_classes((SessionAuthentication, BasicAuthentication))
    @permission_classes((IsAuthenticated,))
    def example_view(request, format=None):
        content = {
            'user': unicode(request.user),  # `django.contrib.auth.User` 实例。
            'auth': unicode(request.auth),  # None
        }
        return Response(content)

## 未认证和禁止的响应

当未经身份验证的请求被拒绝时，有下面两种不同的错误代码可使用。

* [HTTP 401 未认证][http401]
* [HTTP 403 无权限][http403]

HTTP 401 响应必须始终包括一个`WWW-Authenticate`头，指示客户端如何进行身份验证。 HTTP 403响应不包括`WWW-Authenticate`。

具体使用哪种响应取决于认证方案。虽然可以使用多种认证方案，但是仅可以使用一种方案来确定响应的类型。**在确定响应类型时，将使用视图上设置的第一个认证类。**

注意，当一个请求通过了验证但是被拒绝执行请求的权限时，不管认证方案是什么，都要使用 `403 Permission Denied` 响应。

## Apache mod_wsgi 具体配置

注意，如果使用 [Apache using mod_wsgi][mod_wsgi_official]部署，认证头默认不会传递给WSGI应用程序，它假定由Apache处理认证，而不是在应用层面处理。

如果你正在部署到Apache，并且使用任何non-session的身份验证，则需要显式配置mod_wsgi才能将所需的头文件传递给应用程序。这可以通过在适当的上下文中指定`WSGIPassAuthorization`指令并将其设置为`'On'`来完成。

    # 这可能会在服务器配置，虚拟主机，目录或.htaccess
    WSGIPassAuthorization On

---

# API 参考

## BasicAuthentication

此认证方案使用[HTTP 基本认证][basicauth]，针对用户的用户名和密码进行认证。基本认证通常只适用于测试。

如果认证成功 `BasicAuthentication` 提供以下信息。

* `request.user` 将是一个 Django `User` 实例。
* `request.auth` 将是 `None`。

那些被拒绝的未经身份验证的请求会返回使用适当WWW-Authenticate标头的`HTTP 401 Unauthorized`响应。例如：

    WWW-Authenticate: Basic realm="api"

**注意：** 如果你在生产中使用`BasicAuthentication`，那么你必须确保你的API仅在`https`中可用。你还应确保你的API客户端始终在登录时重新请求用户名和密码，并且不会将这些详细信息存储到持久存储中。

## TokenAuthentication

该认证方案使用简单的基于Token的HTTP认证方案。Token认证适用于客户端 - 服务器设置，如本地桌面和移动客户端。

要使用`TokenAuthentication`方案，你需要[配置认证类](#setting-the-authentication-scheme) 以便包含`TokenAuthentication`，另外在`INSTALLED_APPS`设置中还需要包含`rest_framework.authtoken`：

    INSTALLED_APPS = (
        ...
        'rest_framework.authtoken'
    )

---

**注意：** 确保在修改设置后运行一下`manage.py migrate`。`rest_framework.authtoken` app 会提交一些Django数据库迁移操作。

---

你还需要为你的用户创建令牌。

    from rest_framework.authtoken.models import Token

    token = Token.objects.create(user=...)
    print token.key

对客户端进行身份验证，token需要包含在 `Authorization`HTTP头中。密钥应该是以字符串"Token"为前缀，以空格分割的两个字符串。例如：

    Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b

**注意：** 如果你想在HTTP头中使用其他的关键字，比如`Bearer`，只需要继承`TokenAuthentication`类并设置 `keyword`类变量。

如果认证成功，`TokenAuthentication` 提供以下认证信息：

* `request.user` 将是一个Django `User` 实例。
* `request.auth` 将是一个`rest_framework.authtoken.models.Token` 实例。

那些被拒绝的未经身份验证的请求会返回使用适当WWW-Authenticate标头的`HTTP 401 Unauthorized`响应。例如：

    WWW-Authenticate: Token

命令行工具`curl` 可用于测试基于Token认证的API，例如：

    curl -X GET http://127.0.0.1:8000/api/example/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'

---

**注意：** 如果你在生产环境下使用`TokenAuthentication`认证，你必须确保你的API仅在`https`可用。

---

#### 生成Tokens

##### 通过使用信号

如果你希望每个用户拥有自动生成的令牌，你可以简单地捕获用户的`post_save`信号。

    from django.conf import settings
    from django.db.models.signals import post_save
    from django.dispatch import receiver
    from rest_framework.authtoken.models import Token

    @receiver(post_save, sender=settings.AUTH_USER_MODEL)
    def create_auth_token(sender, instance=None, created=False, **kwargs):
        if created:
            Token.objects.create(user=instance)

请注意，你需要确保将此代码段放置在已安装的`models.py`模块或Django在启动时导入的其他位置。

如果你已经创建了一些用户，则可以如下所示为所有现有用户生成令牌：

    from django.contrib.auth.models import User
    from rest_framework.authtoken.models import Token

    for user in User.objects.all():
        Token.objects.get_or_create(user=user)

##### 通过暴露 api 端点

当使用`TokenAuthentication`时，你可能希望为客户端提供一个获取给定用户名和密码的令牌的机制。 REST framework 提供了一个内置的视图来提供这个功能。要使用它，需要将 `obtain_auth_token` 视图添加到你的URLconf：

    from rest_framework.authtoken import views
    urlpatterns += [
        url(r'^api-token-auth/', views.obtain_auth_token)
    ]

注意URL正则匹配模式那里可以是你想要使用的任何内容。

当使用form表单或JSON将有效的`username`和`password`字段POST提交到视图时，`obtain_auth_token`视图将返回JSON响应：

    { 'token' : '9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b' }

请注意，默认的`obtain_auth_token`视图显式使用JSON请求和响应，而不是使用settings中配置的默认渲染器和解析器类。如果需要自定义版本的`obtain_auth_token`视图，可以通过重写`ObtainAuthToken`类，并在url conf中使用它来实现。

默认情况下，没有权限或限制应用于`obtain_auth_token`视图。如果你希望应用限制，则需要重写视图类，并使用`throttle_classes`属性包含它们。


##### With Django admin

也可以通过管理界面手动创建令牌。如果你使用的是大型用户群，我们建议你动态修改`TokenAdmin`类，以根据你的需要进行自定义，更具体地说，将`user`字段声明为`raw_field`。

`your_app/admin.py`:

    from rest_framework.authtoken.admin import TokenAdmin

    TokenAdmin.raw_id_fields = ('user',)


## Session认证

此认证方案使用Django的默认session后端进行身份验证。Session身份验证适用于与你的网站在相同的Session环境中运行的AJAX客户端。

如果成功验证，`SessionAuthentication` 提供以下凭据。

* `request.user` 是一个 Django `User` 实例。
* `request.auth` 是 `None`。

那些被拒绝的未经身份验证的请求会返回`HTTP 403 Forbidden`响应。

如果你正在使用带有SessionAuthentication的AJAX样式的API，你需要确保任何"任何"不安全的HTTP方法调用（如：`PUT`, `PATCH`, `POST` or `DELETE`请求）都包含有效的CSRF token。有关详细信息，请参阅[Django CSRF 文档][csrf-ajax]。

**警告**：在创建登陆页面时，始终要使用Django的标准登陆视图。这样才能确保你的登陆视图被正确的认证保护。

由于需要同时支持session和non-session非会话身份验证，REST框架中的CSRF验证与标准Django中的工作方式略有不同。这意味着只有经过身份验证的请求需要CSRF令牌，匿名请求可能不会发送CSRF令牌tokens。此行为不适用于始终需要使用CSRF验证的登录视图。

# 自定义认证

要实现自定义的认证方案，要继承`BaseAuthentication`类并且重写`.authenticate(self, request)` 方法。如果认证成功，该方法应返回`(user, auth)`的二元元组，否则返回`None`。

在某些情况下，你可能不想返回`None`，而是希望从`.authenticate()`方法抛出`AuthenticationFailed`异常。

通常你应该采取的方法是：

* 如果不尝试验证，返回`None`。还将检查任何其他正在使用的身份验证方案。
* 如果尝试验证但失败，则抛出`AuthenticationFailed`异常。无论任何权限检查也不检查任何其他身份验证方案，立即返回错误响应。

你也*可以*重写`.authenticate_header(self, request)`方法。如果实现该方法，则应返回一个字符串，该字符串将用作`HTTP 401 Unauthorized`响应中的`WWW-Authenticate`头的值。

如果`.authenticate_header()`方法未被重写，则认证方案将在未验证的请求被拒绝访问时返回`HTTP 403 Forbidden`响应。

## 示例

以下示例将以自定义请求标头中名称为'X_USERNAME'提供的用户名作为用户对任何传入请求进行身份验证。

	from django.contrib.auth.models import User
    from rest_framework import authentication
    from rest_framework import exceptions

    class ExampleAuthentication(authentication.BaseAuthentication):
        def authenticate(self, request):
            username = request.META.get('X_USERNAME')
            if not username:
                return None

            try:
                user = User.objects.get(username=username)
            except User.DoesNotExist:
                raise exceptions.AuthenticationFailed('No such user')

            return (user, None)

---

# 第三方包

以下第三方包都是可用的。

## Django OAuth Toolkit

[Django OAuth Toolkit][django-oauth-toolkit] 包提供了OAuth 2.0 认证支持，并且兼容Python 2.7和Python 3.3+。这个包使用优秀的[OAuthLib][oauthlib]，由[Evonove][evonove]维护。该软件包有很完善的文档，并得到很好的支持，目前是我们**推荐使用的OAuth 2.0支持软件包**。

#### 安装和配置

使用`pip`安装。

    pip install django-oauth-toolkit

把这个包添加到你的`INSTALLED_APPS`中，并且修改你的REST framework设置。

    INSTALLED_APPS = (
        ...
        'oauth2_provider',
    )

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': (
            'oauth2_provider.ext.rest_framework.OAuth2Authentication',
        )
    }

更多详情请参阅[Django REST framework - Getting started][django-oauth-toolkit-getting-started]文档。

## Django REST framework OAuth

[Django REST framework OAuth][django-rest-framework-oauth]包提供OAuth1和OAuth2支持。

这个软件包以前直接包含在REST framework中，但现在已被作为第三方软件包支持和维护。

#### 安装和配置

使用`pip`进行安装。

    pip install djangorestframework-oauth

更多配置和使用信息请查阅Django REST framework OAuth文档中的[authentication][django-rest-framework-oauth-authentication]和[permissions][django-rest-framework-oauth-permissions]。

## Digest Authentication

HTTP摘要认证是一种广泛实现的方案，旨在替代HTTP基本认证，并提供简单的加密认证机制。[Juan Riaza][juanriaza]维护着[djangorestframework-digestauth][djangorestframework-digestauth]为REST framework提供了HTTP摘要认证支持。

## Django OAuth2 Consumer

[Rediker Software][rediker]的[Django OAuth2 Consumer][doac]是另一个为REST框架提供[OAuth 2.0 support for REST framework][doac-rest-framework]的软件包。该包包含tokens范围限制权限，允许对你的API进行更细粒度的访问。

## JSON Web Token Authentication

JSON Web Token是一种相当新的标准，可用于基于token的身份验证。与内置的TokenAuthentication方案不同，JWT身份验证不需要使用数据库来验证令牌。[Blimp][blimp]维护[djangorestframework-jwt][djangorestframework-jwt]软件包，它提供了一个JWT Authentication类以及一个机制，客户端获得一个给定用户名和密码的JWT。

## Hawk HTTP Authentication

[HawkREST][hawkrest]库基于[Mohawk][mohawk]库，让你可以在API中使用[Hawk][hawk]签名的请求和响应。[Hawk][hawk]让双方使用共享密钥签名的消息彼此安全地进行通信。它基于[HTTP MAC access authentication][mac]访问认证（它基于[OAuth 1.0][oauth-1.0a]的部分）。

## HTTP Signature Authentication

HTTP签名（目前为[IETF草案][http-signature-ietf-draft]）提供了一种实现HTTP消息的源认证和消息完整性的方法。与[Amazon的HTTP签名方案][amazon-http-signature]类似，许多服务使用它，它允许无状态的每个请求的身份验证。[Elvio Toccalino][etoccalino]维护了[djangorestframework-httpsignature][djangorestframework-httpsignature]包，提供了一个易于使用的HTTP签名身份验证机制。

## Djoser

[Djoser][djoser]库提供一组视图来处理基本操作，如注册，登录，注销，密码重置和帐户激活。该包使用自定义用户模型，它使用基于token的身份验证。这是一个可以使用REST实现的Django认证系统。

## django-rest-auth

[Django-rest-auth][django-rest-auth]库提供了一组REST API端点，用于注册，身份验证（包括社交媒体身份验证），密码重置，检索和更新用户详细信息等。有了这些API端点之后，你的客户端应用程序（如AngularJS，iOS，Android和其他）可以通过REST API独立通信到Django后端站点，以进行用户管理。

## django-rest-framework-social-oauth2

[Django-rest-framework-social-oauth2][django-rest-framework-social-oauth2]库提供了一种将社交插件（facebook，twitter，google等）集成到你的身份验证系统和简单的oauth2设置的简单方法。使用这个库，你将能够根据外部token（例如，Facebook访问token）对用户进行身份验证，将这些令牌转换为“内部”oauth2 tokens，并使用和生成oauth2 tokens来验证用户。

## django-rest-knox

[Django-rest-knox][django-rest-knox]库提供了模型和视图，以比内置的TokenAuthentication方案更安全和可扩展的方式来处理基于token的身份验证 - 使用单页面应用程序和移动客户端能够一起。它为每个客户端提供tokens，以及在提供一些其他身份验证（通常是基本身份验证）时生成tokens，删除token（提供服务器强制注销）和删除所有tokens（注销用户登录的所有客户端）的视图。

[cite]: http://jacobian.org/writing/rest-worst-practices/
[http401]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.2
[http403]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.4
[basicauth]: http://tools.ietf.org/html/rfc2617
[oauth]: http://oauth.net/2/
[permission]: permissions_zh.md
[throttling]: throttling.md
[csrf-ajax]: https://docs.djangoproject.com/en/stable/ref/csrf/#ajax
[mod_wsgi_official]: http://code.google.com/p/modwsgi/wiki/ConfigurationDirectives#WSGIPassAuthorization
[django-oauth-toolkit-getting-started]: https://django-oauth-toolkit.readthedocs.io/en/latest/rest-framework/getting_started.html
[django-rest-framework-oauth]: http://jpadilla.github.io/django-rest-framework-oauth/
[django-rest-framework-oauth-authentication]: http://jpadilla.github.io/django-rest-framework-oauth/authentication/
[django-rest-framework-oauth-permissions]: http://jpadilla.github.io/django-rest-framework-oauth/permissions/
[juanriaza]: https://github.com/juanriaza
[djangorestframework-digestauth]: https://github.com/juanriaza/django-rest-framework-digestauth
[oauth-1.0a]: http://oauth.net/core/1.0a
[django-oauth-plus]: http://code.larlet.fr/django-oauth-plus
[django-oauth2-provider]: https://github.com/caffeinehit/django-oauth2-provider
[django-oauth2-provider-docs]: https://django-oauth2-provider.readthedocs.io/en/latest/
[rfc6749]: http://tools.ietf.org/html/rfc6749
[django-oauth-toolkit]: https://github.com/evonove/django-oauth-toolkit
[evonove]: https://github.com/evonove/
[oauthlib]: https://github.com/idan/oauthlib
[doac]: https://github.com/Rediker-Software/doac
[rediker]: https://github.com/Rediker-Software
[doac-rest-framework]: https://github.com/Rediker-Software/doac/blob/master/docs/integrations.md#
[blimp]: https://github.com/GetBlimp
[djangorestframework-jwt]: https://github.com/GetBlimp/django-rest-framework-jwt
[etoccalino]: https://github.com/etoccalino/
[djangorestframework-httpsignature]: https://github.com/etoccalino/django-rest-framework-httpsignature
[amazon-http-signature]: http://docs.aws.amazon.com/general/latest/gr/signature-version-4.html
[http-signature-ietf-draft]: https://datatracker.ietf.org/doc/draft-cavage-http-signatures/
[hawkrest]: https://hawkrest.readthedocs.io/en/latest/
[hawk]: https://github.com/hueniverse/hawk
[mohawk]: https://mohawk.readthedocs.io/en/latest/
[mac]: http://tools.ietf.org/html/draft-hammer-oauth-v2-mac-token-05
[djoser]: https://github.com/sunscrapers/djoser
[django-rest-auth]: https://github.com/Tivix/django-rest-auth
[django-rest-framework-social-oauth2]: https://github.com/PhilipGarnero/django-rest-framework-social-oauth2
[django-rest-knox]: https://github.com/James1345/django-rest-knox
