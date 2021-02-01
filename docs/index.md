## Django REST framework 中文文档

最开始是2015年还在搜狗上班的时候，有个项目用到DRF框架的时候随手翻译的。

如果本文对你有帮助，请在[github](https://github.com/Q1mi/Django-REST-framework-documentation/tree/master/)上 **star** 该项目。

当然我也十分欢迎你加入该项目并提交PR推送你的翻译。

（说句题外话，一个翻译项目就没有必要抄袭了吧。。。我都无奈了。。。）

## Django REST framework 技术交流QQ群

QQ群号：930578836

## 翻译人员

* [Q1mi](https://github.com/Q1mi)
* [ScienceYuan](https://github.com/ScienceYuan)
* [leveleven](https://github.com/leveleven)
* [Liniian](https://github.com/Liniian)
* [belingud](https://github.com/belingud)
* [Core00077](https://github.com/Core00077)

## 以下章节已完成翻译

### 教程

* [Quickstart](/Django-REST-framework-documentation/tutorial/quickstart_zh/)
* [1-Serialization](/Django-REST-framework-documentation/tutorial/1-serialization_zh/)
* [2-requests-and-responses](/Django-REST-framework-documentation/tutorial/2-requests-and-responses_zh/)
* [3-class-based-views](/Django-REST-framework-documentation/tutorial/3-class-based-views_zh/)
* [4-authentication-and-permissions](/Django-REST-framework-documentation/tutorial/4-authentication-and-permissions_zh/)
* [5-relationships-and-hyperlinked-apis](/Django-REST-framework-documentation/tutorial/5-relationships-and-hyperlinked-apis_zh/)
* [6-viewsets-and-routers](/Django-REST-framework-documentation/tutorial/6-viewsets-and-routers_zh/)
* [7-schemas-and-client-libraries](/Django-REST-framework-documentation/tutorial/7-schemas-and-client-libraries_zh/)

### API指南

* [Requests](/Django-REST-framework-documentation/api-guide/requests_zh/)
* [Responses](/Django-REST-framework-documentation/api-guide/responses_zh/)
* [Views](/Django-REST-framework-documentation/api-guide/views_zh/)
* [Generic Views](/Django-REST-framework-documentation/api-guide/generic-views_zh/)
* [Viewets](/Django-REST-framework-documentation/api-guide/viewsets_zh/)
* [Routers](/Django-REST-framework-documentation/api-guide/routers_zh/)
* [Parsers](/Django-REST-framework-documentation/api-guide/parsers_zh/)
* [Renderers](/Django-REST-framework-documentation/api-guide/renderers_zh/)
* [Serializers](/Django-REST-framework-documentation/api-guide/serializers_zh/)
* [Serializer fields](/Django-REST-framework-documentation/api-guide/fields_zh/)
* [Throttling](/Django-REST-framework-documentation/api-guide/throttling_zh/)
* [Filtering](/Django-REST-framework-documentation/api-guide/filtering_zh/)
* [Pagination](/Django-REST-framework-documentation/api-guide/pagination_zh/)
* [Authentication](/Django-REST-framework-documentation/api-guide/authentication_zh/)
* [Permissions](/Django-REST-framework-documentation/api-guide/permissions_zh/)
* [Caching](/Django-REST-framework-documentation/api-guide/caching_zh/)
* [Versioning](/Django-REST-framework-documentation/api-guide/versioning_zh/)
* [Content negotiation](/Django-REST-framework-documentation/api-guide/content-negotiation_zh/)
* [Format suffixes](/Django-REST-framework-documentation/api-guide/format-suffixes_zh/)
* [Metadata](/Django-REST-framework-documentation/api-guide/metadata_zh/)
* [Returning URLs](/Django-REST-framework-documentation/api-guide/reverse_zh/)
* [Exceptions](/Django-REST-framework-documentation/api-guide/exceptions_zh/)
* [Status codes](/Django-REST-framework-documentation/api-guide/status-codes_zh/)
* [Testing](/Django-REST-framework-documentation/api-guide/testing_zh/)

### 主题

* [ajax-csrf-cors](/Django-REST-framework-documentation/topics/ajax-csrf-cors_zh/)

## 要求

REST framework 需要满足以下要求:

* Python (3.5, 3.6, 3.7, 3.8, 3.9)
* Django (2.2, 3.0, 3.1)

我们**强烈推荐**且仅官方支持Python和Django的每个系列(大版本)的最新版本。

以下包为可选项：

* [PyYAML][pyyaml], [uritemplate][uriteemplate] (5.1+, 3.0.0+) - Schema生成支持。
* [Markdown][markdown] (3.0.0+) - 为browsable API 提供Markdown支持。
* [Pygments][pygments] (2.4.0+) - 为Markdown处理提供语法高亮。
* [django-filter][django-filter] (1.0.1+) - Filtering支持。
* [django-guardian][django-guardian] (1.1.1+) - 对象级别的权限支持。

## 安装

用 `pip`来安装，可以包括任何你想安装的可选包…

    pip install djangorestframework
    pip install markdown       # 为browsable API 提供Markdown支持。
    pip install django-filter  # Filtering支持。

...或者直接从github clone该项目。

    git clone https://github.com/encode/django-rest-framework

在`INSTALLED_APPS`中添加 `'rest_framework'` 项。

    INSTALLED_APPS = [
        ...
        'rest_framework',
    ]

如果你正打算用browsable API，你可能也想用REST framework的登录注销视图。把下面的代码添加到你根目录的`urls.py`文件。

    urlpatterns = [
        ...
        path('api-auth/', include('rest_framework.urls'))
    ]

请注意，URL路径可以随意改。

## 举个例子

让我们来看看一个用REST framework构建的简单的模型支撑的API后端案例。

我们会创建一个读写API来访问我们项目的用户信息。

REST framework API的所有全局设定都会放在一个叫`REST_FRAMEWORK`的配置词典里。首先把下面的内容添加到你的`settings.py`模块中：

    REST_FRAMEWORK = {
        # Use Django's standard `django.contrib.auth` permissions,
        # or allow read-only access for unauthenticated users.
        'DEFAULT_PERMISSION_CLASSES': [
            'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'
        ]
    }

不要忘了你还要`rest_framework`添加到你的`INSTALLED_APPS`里。

这样我们就可以开始创建我们的API了。

这是我们根目录的`urls.py`模块：

    from django.urls import path, include
    from django.contrib.auth.models import User
    from rest_framework import routers, serializers, viewsets
    
    # 序列化器是用来定义API的表示形式。
    class UserSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = User
            fields = ['url', 'username', 'email', 'is_staff']
    
    # ViewSets定义视图的行为。
    class UserViewSet(viewsets.ModelViewSet):
        queryset = User.objects.all()
        serializer_class = UserSerializer
    
    # 路由器提供一个简单自动的方法来决定URL的配置。
    router = routers.DefaultRouter()
    router.register(r'users', UserViewSet)
    
    # 通过URL自动路由来给我们的API布局。
    # 此外，我们还要把登录的URL包含进来。
    urlpatterns = [
        path('', include(router.urls)),
        path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]

现在你可以在浏览器中打开[http://127.0.0.1:8000/](http://127.0.0.1:8000/)，查看你的'user' API了。如果你用了右上角的登录控制，那你还可以在系统中添加、创建并删除用户。

## Quickstart

迫不及待想上手了？[快速入门指南][quickstart]是你上手、运行并构建REST framewrok API的最快方法。

[pyyaml]: https://pypi.org/project/PyYAML/
[uriteemplate]: https://pypi.org/project/uritemplate/
[markdown]: https://pypi.org/project/Markdown/
[pygments]: https://pypi.org/project/Pygments/
[django-filter]: https://pypi.org/project/django-filter/
[django-guardian]: https://github.com/django-guardian/django-guardian
[quickstart]: tutorial/quickstart_zh.md