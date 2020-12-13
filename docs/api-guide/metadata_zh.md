source: metadata.py

# Metadata （元数据）


> [The `OPTIONS`]方法允许客户端确定与资源或服务器功能相关的选项和/或需求，而无需暗示资源操作或启动资源检索。
> 
> &mdash; [RFC7231, Section 4.3.7.][cite]

REST框架包含一个可配置的机制，用于确定API应该如何响应 `OPTIONS` 请求。这允许您返回API模式或其他资源信息。

对于HTTP `OPTIONS` 请求究竟应该返回什么样的响应样式，目前还没有广泛采用的约定，因此我们提供了一种返回一些有用信息的点对点样式。

下面是一个示例响应，它演示了默认情况下返回的信息。

    HTTP 200 OK
    Allow: GET, POST, HEAD, OPTIONS
    Content-Type: application/json

    {
        "name": "To Do List",
        "description": "List existing 'To Do' items, or create a new item.",
        "renders": [
            "application/json",
            "text/html"
        ],
        "parses": [
            "application/json",
            "application/x-www-form-urlencoded",
            "multipart/form-data"
        ],
        "actions": {
            "POST": {
                "note": {
                    "type": "string",
                    "required": false,
                    "read_only": false,
                    "label": "title",
                    "max_length": 100
                }
            }
        }
    }

## Setting the metadata scheme （设置元数据模式）

您可以使用 `'DEFAULT_METADATA_CLASS'` 设置键全局设置元数据类：

    REST_FRAMEWORK = {
        'DEFAULT_METADATA_CLASS': 'rest_framework.metadata.SimpleMetadata'
    }

也可以为视图单独设置元数据类：

    class APIRoot(APIView):
        metadata_class = APIRootMetadata

        def get(self, request, format=None):
            return Response({
                ...
            })

REST框架包只包含一个名为 `SimpleMetadata` 的元数据类实现方式。如果要使用其他样式，则需要实现自定义元数据类。

## Creating schema endpoints （创建模式终端）

如果对使用常规 `GET` 请求访问的模式终端的创建有特定的要求，则可以考虑重新使用元数据API来执行此操作。

例如，可以在视图集上使用以下附加路由来提供可链接的模式终端。

    @list_route(methods=['GET'])
    def schema(self, request):
        meta = self.metadata_class()
        data = meta.determine_metadata(request, self)
        return Response(data)

有许多可能会选择采用这种方法的理由，其中包括 `OPTIONS` 响应[不可缓存][no-options]。

---

# Custom metadata classes （自定义元数据类）

如果要提供自定义元数据类，则应重写 `BaseMetadata` 并应用 `determine_metadata(self, request, view)` 方法。

你可能希望做的有用的事情包括返回模式信息、使用[JSON schema （JSON 模式）][json-schema]之类的格式，或者将调试信息返回给管理员用户。

## Example （举个例子）

以下的类可被用于限制返回给 `OPTIONS` 的信息。

    class MinimalMetadata(BaseMetadata):
        """
        Don't include field and other information for `OPTIONS` requests.
        Just return the name and description.
        """
        def determine_metadata(self, request, view):
            return {
                'name': view.get_view_name(),
                'description': view.get_view_description()
            }

然后配置设置以使用此自定义类：

    REST_FRAMEWORK = {
        'DEFAULT_METADATA_CLASS': 'myproject.apps.core.MinimalMetadata'
    }

# Third party packages

The following third party packages provide additional metadata implementations.

## DRF-schema-adapter

[drf-schema-adapter][drf-schema-adapter] is a set of tools that makes it easier to provide schema information to frontend frameworks and libraries. It provides a metadata mixin as well as 2 metadata classes and several adapters suitable to generate [json-schema][json-schema] as well as schema information readable by various libraries.

You can also write your own adapter to work with your specific frontend.
If you wish to do so, it also provides an exporter that can export those schema information to json files.

[cite]: http://tools.ietf.org/html/rfc7231#section-4.3.7
[no-options]: https://www.mnot.net/blog/2012/10/29/NO_OPTIONS
[json-schema]: http://json-schema.org/
[drf-schema-adapter]: https://github.com/drf-forms/drf-schema-adapter
