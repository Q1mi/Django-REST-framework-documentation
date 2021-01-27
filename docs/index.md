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

## Requirements

REST framework requires the following:

* Python (3.5, 3.6, 3.7, 3.8, 3.9)
* Django (2.2, 3.0, 3.1)

We **highly recommend** and only officially support the latest patch release of
each Python and Django series.

The following packages are optional:

* [PyYAML][pyyaml], [uritemplate][uriteemplate] (5.1+, 3.0.0+) - Schema generation support.
* [Markdown][markdown] (3.0.0+) - Markdown support for the browsable API.
* [Pygments][pygments] (2.4.0+) - Add syntax highlighting to Markdown processing.
* [django-filter][django-filter] (1.0.1+) - Filtering support.
* [django-guardian][django-guardian] (1.1.1+) - Object level permissions support.

## Installation

Install using `pip`, including any optional packages you want...

    pip install djangorestframework
    pip install markdown       # Markdown support for the browsable API.
    pip install django-filter  # Filtering support

...or clone the project from github.

    git clone https://github.com/encode/django-rest-framework

Add `'rest_framework'` to your `INSTALLED_APPS` setting.

    INSTALLED_APPS = [
        ...
        'rest_framework',
    ]

If you're intending to use the browsable API you'll probably also want to add REST framework's login and logout views.  Add the following to your root `urls.py` file.

    urlpatterns = [
        ...
        path('api-auth/', include('rest_framework.urls'))
    ]

Note that the URL path can be whatever you want.

## Example

Let's take a look at a quick example of using REST framework to build a simple model-backed API.

We'll create a read-write API for accessing information on the users of our project.

Any global settings for a REST framework API are kept in a single configuration dictionary named `REST_FRAMEWORK`.  Start off by adding the following to your `settings.py` module:

    REST_FRAMEWORK = {
        # Use Django's standard `django.contrib.auth` permissions,
        # or allow read-only access for unauthenticated users.
        'DEFAULT_PERMISSION_CLASSES': [
            'rest_framework.permissions.DjangoModelPermissionsOrAnonReadOnly'
        ]
    }

Don't forget to make sure you've also added `rest_framework` to your `INSTALLED_APPS`.

We're ready to create our API now.
Here's our project's root `urls.py` module:

    from django.urls import path, include
    from django.contrib.auth.models import User
    from rest_framework import routers, serializers, viewsets

    # Serializers define the API representation.
    class UserSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = User
            fields = ['url', 'username', 'email', 'is_staff']

    # ViewSets define the view behavior.
    class UserViewSet(viewsets.ModelViewSet):
        queryset = User.objects.all()
        serializer_class = UserSerializer

    # Routers provide an easy way of automatically determining the URL conf.
    router = routers.DefaultRouter()
    router.register(r'users', UserViewSet)

    # Wire up our API using automatic URL routing.
    # Additionally, we include login URLs for the browsable API.
    urlpatterns = [
        path('', include(router.urls)),
        path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]

You can now open the API in your browser at [http://127.0.0.1:8000/](http://127.0.0.1:8000/), and view your new 'users' API. If you use the login control in the top right corner you'll also be able to add, create and delete users from the system.

## Quickstart

Can't wait to get started? The [quickstart guide][quickstart] is the fastest way to get up and running, and building APIs with REST framework.
