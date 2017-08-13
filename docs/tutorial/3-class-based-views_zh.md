# Tutorial 3: 基于类的视图（CBV）

我们也可以使用基于类的视图编写我们的API视图，而不是基于函数的视图。我们将看到这是一个强大的模式，允许我们重用常用功能，并帮助我们保持代码[DRY][dry]。

## 使用基于类的视图重写我们的API

我们将首先将根视图重写为基于类的视图。所有这些都是对`views.py`文件的重构。

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from django.http import Http404
    from rest_framework.views import APIView
    from rest_framework.response import Response
    from rest_framework import status


    class SnippetList(APIView):
        """
        列出所有的snippets或者创建一个新的snippet。
        """
        def get(self, request, format=None):
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        def post(self, request, format=None):
            serializer = SnippetSerializer(data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

到现在为止还挺好。它看起来与以前的情况非常相似，但是我们在不同的HTTP方法之间有更好的分离。我们还需要更新`views.py`中的实例视图。

    class SnippetDetail(APIView):
        """
        检索，更新或删除一个snippet示例。
        """
        def get_object(self, pk):
            try:
                return Snippet.objects.get(pk=pk)
            except Snippet.DoesNotExist:
                raise Http404

        def get(self, request, pk, format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        def put(self, request, pk, format=None):
            snippet = self.get_object(pk)
            serializer = SnippetSerializer(snippet, data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        def delete(self, request, pk, format=None):
            snippet = self.get_object(pk)
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

看起来不错，它现在仍然非常类似于基于功能的视图。

我们还需要重构我们的`urls.py`，现在我们使用基于类的视图。

    from django.conf.urls import url
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.SnippetList.as_view()),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.SnippetDetail.as_view()),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns)

好了，我们完成了。如果你启动服务，那么它应​​该像之前一样可以运行。

## 使用混合（mixins）

使用基于类视图的最大优势之一是它可以轻松地创建可复用的行为。

到目前为止，我们使用的创建/获取/更新/删除操作和我们创建的任何基于模型的API视图非常相似。这些常见的行为是在REST框架的mixin类中实现的。

让我们来看看我们是如何通过使用mixin类编写视图的。这是我们的`views.py`模块。

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import mixins
    from rest_framework import generics

    class SnippetList(mixins.ListModelMixin,
                      mixins.CreateModelMixin,
                      generics.GenericAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.list(request, *args, **kwargs)

        def post(self, request, *args, **kwargs):
            return self.create(request, *args, **kwargs)

我们将花点时间好好看下这里的具体实现方式。我们使用`GenericAPIView`构建了我们的视图，并且用上了`ListModelMixin`和`CreateModelMixin`。

基类提供核心功能，而mixin类提供`.list()`和`.create()`操作。然后我们明确地将`get`和`post`方法绑定到适当的操作。都目前为止都是显而易见的。

    class SnippetDetail(mixins.RetrieveModelMixin,
                        mixins.UpdateModelMixin,
                        mixins.DestroyModelMixin,
                        generics.GenericAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

        def get(self, request, *args, **kwargs):
            return self.retrieve(request, *args, **kwargs)

        def put(self, request, *args, **kwargs):
            return self.update(request, *args, **kwargs)

        def delete(self, request, *args, **kwargs):
            return self.destroy(request, *args, **kwargs)

非常相似。这一次我们使用`GenericAPIView`类来提供核心功能，并添加mixins来提供`.retrieve()`），`.update()`和`.destroy()`操作。

## 使用通用的基于类的视图

通过使用mixin类，我们使用更少的代码重写了这些视图，但我们还可以再进一步。REST框架提供了一组已经混合好（mixed-in）的通用视图，我们可以使用它来简化我们的`views.py`模块。

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework import generics


    class SnippetList(generics.ListCreateAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer


    class SnippetDetail(generics.RetrieveUpdateDestroyAPIView):
        queryset = Snippet.objects.all()
        serializer_class = SnippetSerializer

哇，如此简洁。我们可以使用很多现成的代码，让我们的代码看起来非常清晰、简洁，地道的Django。

接下来，我们将移步[教程的第4部分][tut-4]，我们将介绍如何处理API的身份验证和权限。

[dry]: http://en.wikipedia.org/wiki/Don't_repeat_yourself
[tut-4]: 4-authentication-and-permissions_zh.md
