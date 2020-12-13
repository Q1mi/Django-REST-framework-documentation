source: urlpatterns.py

# Format suffixes （格式后缀）

> Section 6.2.1 并没有说应当总是使用内容协商
>
> &mdash; Roy Fielding, [REST discuss mailing list][cite]

webapi的一种常见模式是在url上使用文件扩展名来为给定的媒体类型提供端点。例如，'http://example.com/api/users.json'来提供JSON显示。

将格式后缀模式添加到API的URLconf中的每个条目是容易出错的，而且是不DRY的，因此REST框架提供了一种快捷方式，可以将这些模式添加到URLconf中。

## format_suffix_patterns （格式后缀模式）

**Signature**: format_suffix_patterns(urlpatterns, suffix_required=False, allowed=None)

返回一个URL模式列表，其中的所提供的每个URL模式附加了格式后缀

参数：

* **urlpatterns**: 必需。一个URL模式列表。
* **suffix_required**:  可选。一个布尔值，指示URL中的后缀是可选的还是必需的。默认值为False，这意味着后缀在默认情况下是可选的。
* **allowed**:  Optional.  可选。一个有效格式后缀的列表或元组。如果未提供，将使用通配符格式后缀模式。

Example:

    from rest_framework.urlpatterns import format_suffix_patterns
    from blog import views

    urlpatterns = [
        url(r'^/$', views.apt_root),
        url(r'^comments/$', views.comment_list),
        url(r'^comments/(?P<pk>[0-9]+)/$', views.comment_detail)
    ]

    urlpatterns = format_suffix_patterns(urlpatterns, allowed=['json', 'html'])

当使用 `format_suffix_patterns` 时，必须确保将 `'format'` 关键字参数添加到相应的视图中。例如：

    @api_view(('GET', 'POST'))
    def comment_list(request, format=None):
        # do stuff...

或基于类的视图：

    class CommentList(APIView):
        def get(self, request, format=None):
            # do stuff...

        def post(self, request, format=None):
            # do stuff...

使用的kwarg名称可以通过使用 `FORMAT_SUFFIX_KWARG` 设置进行修改。

还要注意，`format_suffix_patterns` 不支持降为 `include` URL模式。

### Using with `i18n_patterns` (与i18n_模式一同使用)

如果使用Django提供的 `i18n_patterns` 函数，以及 `format_suffix_patterns`，则应确保将 `i18n_patterns` 函数被应用为最终函数或最外层函数。例如：

    url patterns = [
        …
    ]

    urlpatterns = i18n_patterns(
        format_suffix_patterns(urlpatterns, allowed=['json', 'html'])
    )

---

## Query parameter formats (查询参数格式)

格式后缀的替代方法是在查询参数中包含请求的格式。REST框架在默认情况下提供了这个选项，并且在可浏览API中使用以在不同的可用表现形式之间切换。

为了使用其简短格式选择表示形式，请使用 `format` 参数查询。例如：`http://example.com/organizations/？format=csv`。

可以使用 `URL_FORMAT_OVERRIDE` 设置来修改此查询参数的名称。将该值设置为 `None` 可禁用此行为。

---

## Accept headers vs. format suffixes

一些Web社区似乎有一种观点，即文件名扩展名不是RESTful模式，应当始终使用 `HTTP Accept`报文头。

这实际上是一种误解。例如，引用Roy Fielding的以下讨论了查询参数media type indicators与file extension media type indicators的相对优点言论：

&ldquo;这就是为什么我总是喜欢扩展。这两个选择都与REST无关。&ldquo; &mdash; 罗伊菲尔丁，[REST discuss mailing list][cite2]

这段引文没有提到Accept headers，但它明确表示格式后缀应该被视为可接受的模式。

[cite]: http://tech.groups.yahoo.com/group/rest-discuss/message/5857
[cite2]: http://tech.groups.yahoo.com/group/rest-discuss/message/14844
