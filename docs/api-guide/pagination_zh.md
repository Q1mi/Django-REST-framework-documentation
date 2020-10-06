source: pagination.py

# Pagination (分页)

>Django provides a few classes that help you manage paginated data – that is, data that’s split across several pages, with “Previous/Next” links.
>
> &mdash; [Django 文档][cite]

REST框架包括对可定制的分页样式的支持。 这使您可以调整使大型结果集被拆分为单独的数据页面。

分页API可以支持以下两者之一：

* 作为响应内容的一部分提供的分页链接。
* 包含在响应标头中的分页链接，例如 `Content-Range` 或 `Link`。

当前内置样式都使用作为响应内容一部分的链接。当使用可浏览的API时，这种样式更容易访问。

只有在使用常规视图或视图集时才自动执行分页。如果使用的是常规的 `APIView` ，则需要自行调用分页API以确保返回分页的响应。请参见 `mixins.ListModelMixin` 以及  `generics.GenericAPIView` 类作为示例。

Pagination can be turned off by setting the pagination class to `None`.
可以通过将Pagination类设置为 `None` 来关闭分页。

## Setting the pagination style (设置分页样式)

可以使用 `DEFAULT_PAGINATION_CLASS` 和 `PAGE_SIZE` 设置键全局设置分页样式。例如，为了使用内置的限制/偏移分页，可以执行以下操作

    REST_FRAMEWORK = {
        'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
        'PAGE_SIZE': 100
    }

注意，您需要同时设置分页类和应该使用的页面大小。默认情况下， `DEFAULT_PAGINATION_CLASS` 和 `PAGE_SIZE` 均为 `None` 。

也可以使用 `pagination_class` 属性在单个视图上设置分页类。通常，您希望在整个API中使用相同的分页样式，尽管您可能希望根据每个视图改变分页的各个方面，例如默认或最大页面大小。

## Modifying the pagination style (调整分页样式)

如果要修改分页样式的特定方面，则需要重写其中一个分页类，并设置要更改的属性。

    class LargeResultsSetPagination(PageNumberPagination):
        page_size = 1000
        page_size_query_param = 'page_size'
        max_page_size = 10000

    class StandardResultsSetPagination(PageNumberPagination):
        page_size = 100
        page_size_query_param = 'page_size'
        max_page_size = 1000

然后，可以使用 `.pagination_class` 属性将新样式应用于视图：

    class BillingRecordsView(generics.ListAPIView):
        queryset = Billing.objects.all()
        serializer_class = BillingRecordsSerializer
        pagination_class = LargeResultsSetPagination

或者使用默认的 `DEFAULT_PAGINATION_CLASS` 设置键全局应用样式。例如：

    REST_FRAMEWORK = {
        'DEFAULT_PAGINATION_CLASS': 'apps.core.pagination.StandardResultsSetPagination'
    }

---

# API Reference

## PageNumberPagination

此分页样式接受请求查询参数中的单个数字页码。

**Request**:

    GET https://api.example.org/accounts/?page=4

**Response**:

    HTTP 200 OK
    {
        "count": 1023
        "next": "https://api.example.org/accounts/?page=5",
        "previous": "https://api.example.org/accounts/?page=3",
        "results": [
           …
        ]
    }

#### Setup (设置)

为了全局启用 `PageNumberPagination` 样式，请使用以下配置，并根据需要设置 `PAGE_SIZE` ：

    REST_FRAMEWORK = {
        'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
        'PAGE_SIZE': 100
    }

在 `GenericAPIView` 子类中，还可以设置 `pagination_class` 属性，以在每个视图的基础上选择 `PageNumberPagination`。

#### Configuration (配置)

`PageNumberPagination` 类包含许多属性，通过重写其中的一些属性可以修改分页属性

为了设置这些属性，应当重写 `PageNumberPagination` 类，然后如上所述启用自定义分页类。

* `django_paginator_class` —— 要使用的django paginator类。默认为 `django.core.paginator.Paginator` ，适用于大多数情况。
* `page_size` —— 表示页面大小的数值。如果设置，则覆盖 `PAGE_SIZE` 设置。其默认值与 `PAGE_SIZE` 设置键相同。
* `page_query_param` —— 一个字符串值，指示用于分页控件的查询参数的名称。
* `page_size_query_param` —— 如果设置，这是一个字符串值，指示允许客户端根据每个请求设置页面大小的查询参数的名称。默认值为 `None`，表示客户端可能无法控制请求的页面大小。
* `max_page_size` —— 如果设置，这是一个数字值，指示允许的最大请求页面大小。此属性仅在设置了 `page_size_query_param` 时有效。
* `last_page_strings` —— 一个字符串值的列表或元组，指示可与`page_query_param` 一起使用的值，以请求集合中的最后一页。默认为 `('last',)`。
* `template` —— 在可浏览API中呈现分页控件时要使用的模板名称。可以重写以修改呈现样式，或设置为 `None` 以完全禁用HTML分页控件。默认为 `"rest_framework/pagination/numbers.html"`。

---

## LimitOffsetPagination

这种分页样式反映了查找多个数据库记录时使用的语法。客户端包括“limit”和“offset”查询参数。limit指示要返回的项目的最大数目，这相当于其他样式中的 `page_size`。offset指示查询相对于完整的未分页项的起始位置。

**Request**:

    GET https://api.example.org/accounts/?limit=100&offset=400

**Response**:

    HTTP 200 OK
    {
        "count": 1023
        "next": "https://api.example.org/accounts/?limit=100&offset=500",
        "previous": "https://api.example.org/accounts/?limit=100&offset=300",
        "results": [
           …
        ]
    }

#### Setup (设置)

为了全局启用 `LimitOffsetPagination`样式，请使用以下配置：

    REST_FRAMEWORK = {
        'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination'
    }

可选地，您也可以设置 `PAGE_SIZE` 键。如果还使用了 `PAGE_SIZE` 参数，那么 `limit` query参数将是可选的，并且可以被客户端忽略。

在 `GenericAPIView` 子类中，您还可以设置 `pagination_class` 属性，以便在每个视图的基础上选择 `LimitOffsetPagination`。

#### Configuration (配置)

`LimitOffsetPagination` 类包含许多可以重写以修改分页样式的属性。

为了设置这些属性，需要重写 `LimitOffsetPagination` 类，然后如上所述启用自定义分页类。

* `default_limit` —— 数值，指示在客户端未在查询参数中提供限制时使用的限制。默认为与 `PAGE_SIZE` 设置键相同的值。
* `limit_query_param` —— 字符串值，指示“limit”查询参数的名称。默认为 `'limit'`。
* `offset_query_param` —— 字符串值，指示“offset”查询参数的名称。默认为 `'offset'`。
* `max_limit` —— 如果设置，这是一个数值，指示客户端可能请求的最大允许限制。默认为 `None`。
* `template` —— 在可浏览API中呈现分页控件时要使用的模板名称。可以重写以修改呈现样式，或设置为 `None` 以完全禁用HTML分页控件。默认为 `"rest_framework/pagination/numbers.html"`。

---

## CursorPagination

基于cursor的分页提供了一个不透明的“游标”指示器，客户端可以使用它对结果集进行分页。这种分页样式只显示正向和反向控件，而不允许客户端导航到任意位置。

基于cursor的分页要求结果集中的项具有唯一的、不变的顺序。此顺序通常可能是记录上的创建时间戳，因为它提为分页供了一致的顺序。

基于cursor的分页比其他方案更复杂。它还要求结果集呈现固定的顺序，并且不允许客户端任意索引结果集。但是，它确实提供了以下好处：

* 提供一致的分页视图。当恰当地使用时，`CursorPagination` 可以确保客户端在翻阅记录时永远不会看到同一个项目两次，即使在分页过程中其他客户端正在插入新的项目时亦是如此。
* 支持使用非常大的数据集。对于极其庞大的数据集，使用基于offset的分页样式进行分页可能会变得效率低下或无法使用。基于cursor的分页方案具有固定时间的属性，并且不会随着数据集大小的增加而减慢。

#### Details and limitations

正确使用基于cursor的分页需要注意一些细节。你需要考虑你希望该方案适用于什么样的顺序。默认值是按 `"-created"` 排序。这假设模型实例上**必须有一个“created”时间戳字段**，并且将呈现一个“timeline”样式的，最新添加的项在前面的分页视图。

可以通过重写分页类中的 `'ordering'` 属性，或将 `OrderingFilter` 筛选器类与 `CursorPagination` 一起使用来修改排序。当与 `OrderingFilter` 一起使用时，您应该考虑严格限制用户可以按其排序的字段。

正确使用cursor分页应使用有满足以下条件的排序字段：

* 应当是一个不变的值，例如时间戳、slug或其他在创建时只设置一次的字段。
* 应当是独一无二的，或者几乎是独一无二的。毫秒精度的时间戳就是一个很好的例子。cursor分页的这种实现使用了一种智能的"position plus offset" 样式，允许它正确地支持不严格唯一的值作为排序。
* 应当为可强制转化为字符串的不可为null的值。
* 不应当是浮点数。精度误差容易导致不正确的结果。提示：改用小数。（如果已经有一个浮点数字段，并且必须根据它进行分页，那么[这里提供了一个使用小数限制精度的 `CursorPagination` 子类示例。][example]）
* 字段应具有数据库索引。

使用不满足这些约束的排序字段通常仍然有效，但是您将失去游标分页的一些好处。

关于我们用于游标分页的实现的更多技术细节，["Building cursors for the Disqus API"][disqus-cursor-api] 博客文章对基本方法进行了很好的概述。

#### Setup （设置）

为了全局启用 `CursorPagination` 样式，请使用以下配置，根据需要修改 `PAGE_SIZE` ：

    REST_FRAMEWORK = {
        'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.CursorPagination',
        'PAGE_SIZE': 100
    }

在 `GenericAPIView` 子类中，还可以设置 `pagination_class` 属性，以在每个视图的基础上选择 `CursorPagination`。

#### Configuration (配置)

`CursorPagination` 类包含许多可以重写以修改分页样式的属性，。

为了设置这些属性，需要重写 `CursorPagination` 类，然后如上所述启用自定义分页类。

* `page_size` = 表示 `PAGE_SIZE` 的数值。如果设置，则覆盖页面大小设置。默认为与 `PAGE_SIZE` 设置键相同的值。
* `cursor_query_param` = 字符串值，指示“cursor”查询参数的名称。默认为 `'cursor'`。
* `ordering` = 这应当是一个字符串或字符串列表，指示将对其应用基于cursor的分页的字段。例如： `ordering='slug'`。默认为 `-created`。也可以通过在视图上使用 `OrderingFilter` 覆盖此值。
* `template` = 在可浏览API中呈现分页控件时要使用的模板的名称。可以重写以修改呈现样式，或设置为“无”以完全禁用HTML分页控件。默认为 `"rest_framework/pagination/previous_and_next.html"`。

---

# Custom pagination styles (自定义分页样式)

为了创建自定义分页序列化程序类，应将 `pagination.BasePagination` 子类化，并重写  `paginate_queryset(self, queryset, request, view=None)` 和 `get_paginated_response(self, data)` 方法：

* `paginate_queryset` 方法被传递给初始的queryset，它应该返回一个iterable对象，该对象只包含请求页中的数据。
* The `get_paginated_response` m方法传递序列化的页数据，并应返回一个 `Response` 。

注意，`paginate_queryset` 方法可能会在分页实例上设置state，该state （？）稍后可能会由`get_paginated_response` 方法使用。

## Example

假设我们想用一个修改过的格式替换默认的分页输出样式，该格式在嵌套的“links”键中包含下一页和上一页的链接。我们可以规定一个自定义分页类，如下所示：

    class CustomPagination(pagination.PageNumberPagination):
        def get_paginated_response(self, data):
            return Response({
                'links': {
                   'next': self.get_next_link(),
                   'previous': self.get_previous_link()
                },
                'count': self.page.paginator.count,
                'results': data
            })

然后我们需要在配置中设置自定义类：

    REST_FRAMEWORK = {
        'DEFAULT_PAGINATION_CLASS': 'my_project.apps.core.pagination.CustomPagination',
        'PAGE_SIZE': 100
    }

注意，如果您关心在可浏览API中如何在响应中显示键的顺序，那么在构造分页响应主体时，您可能会选择使用 `OrderedDict`，但这是可选的。

<!-- ## Header based pagination

Let's modify the built-in `PageNumberPagination` style, so that instead of include the pagination links in the body of the response, we'll instead include a `Link` header, in a [similar style to the GitHub API][github-link-pagination].

    class LinkHeaderPagination(pagination.PageNumberPagination):
        def get_paginated_response(self, data):
            next_url = self.get_next_link()
            previous_url = self.get_previous_link()

            if next_url is not None and previous_url is not None:
                link = '<{next_url}>; rel="next", <{previous_url}>; rel="prev"'
            elif next_url is not None:
                link = '<{next_url}>; rel="next"'
            elif previous_url is not None:
                link = '<{previous_url}>; rel="prev"'
            else:
                link = ''

            link = link.format(next_url=next_url, previous_url=previous_url)
            headers = {'Link': link} if link else {}

            return Response(data, headers=headers) -->

## Using your custom pagination class （使用自定义分页类）


为了在默认情况下使用自定义分页类，请使用 `DEFAULT_PAGINATION_CLASS` 设置：

    REST_FRAMEWORK = {
        'DEFAULT_PAGINATION_CLASS': 'my_project.apps.core.pagination.LinkHeaderPagination',
        'PAGE_SIZE': 100
    }

列表终端的API响应现在将包括一个 `Link` 头，而不是将分页链接作为响应正文的一部分，例如：

![Link Header][link-header]

*A custom pagination style, using the 'Link' header'*

---

## Pagination & schemas （分页和模式）

还可以通过应用 `get_schema_fields()` 方法，使分页控件可用于REST框架提供的模式自动生成。此方法应当有以下签名：

`get_schema_fields(self, view)`

此方法应当返回一个coreapi.Field实例的列表。

---

# HTML pagination controls （HTML分页控件）

默认情况下，使用分页类将导致HTML分页控件在可浏览的API中显示。有两种内置的显示样式。 `PageNumberPagination` 和 `LimitOffsetPagination` 类显示带有上一页和下一页控件的页码列表。 `CursorPagination` 类显示的样式更简单，仅显示上一页和下一页控件。

## Customizing the controls （自定义控件)

您可以重写呈现HTML分页控件的模板。两种内置样式是：

* `rest_framework/pagination/numbers.html`
* `rest_framework/pagination/previous_and_next.html`

在全局模板目录中提供具有这些路径之一的模板将覆盖相关分页类的默认渲染。

或者，可以通过以下方式完全禁用HTML分页控件：将现有类的子类化，将 `template = None` 设置为该类的属性。然后，您需要配置 `DEFAULT_PAGINATION_CLASS` 设置键，以将自定义类作为默认的分页样式使用。

#### Low-level API

当 `display_page_controls` 属性在分页实例中时，决定分页类是否显示控件的低级API被暴露出来。如果自定义分页类要求显示HTML分页控件，则应在 `paginate_queryset` 方法中将其设置为 `True`。

`.to_html()` 和 `.get_html_context()` 方法也可以在自定义分页类中重写，以便进一步自定义控件的渲染方式。


---

# Third party packages

The following third party packages are also available.

## DRF-extensions

The [`DRF-extensions` package][drf-extensions] includes a [`PaginateByMaxMixin` mixin class][paginate-by-max-mixin] that allows your API clients to specify `?page_size=max` to obtain the maximum allowed page size.

## drf-proxy-pagination

The [`drf-proxy-pagination` package][drf-proxy-pagination] includes a `ProxyPagination` class which allows to choose pagination class with a query parameter.

[cite]: https://docs.djangoproject.com/en/stable/topics/pagination/
[github-link-pagination]: https://developer.github.com/guides/traversing-with-pagination/
[link-header]: ../img/link-header-pagination.png
[drf-extensions]: http://chibisov.github.io/drf-extensions/docs/
[paginate-by-max-mixin]: http://chibisov.github.io/drf-extensions/docs/#paginatebymaxmixin
[drf-proxy-pagination]: https://github.com/tuffnatty/drf-proxy-pagination
[disqus-cursor-api]: http://cramer.io/2011/03/08/building-cursors-for-the-disqus-api

[example]: https://gist.github.com/keturn/8bc88525a183fd41c73ffb729b8865be#file-fpcursorpagination-py
