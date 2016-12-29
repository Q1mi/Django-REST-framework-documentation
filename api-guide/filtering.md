# Filtering


> “ 由Manager提供的根QuerySet描述了数据库表中的所有对象。通常，可是你只需要选择完整对象的一个子集而已。
> 
—— Django文档
>”
 
 REST framework列表视图的默认行为是返回一个model的全部queryset。通常你却想要你的API来限制queryset返回的数据。

最简单的删选一个`GenericAPIView`子视图的queryset的方法就是复写他的`.get_queryset()`方法。

重写这个方法允许你使用很多不同的方式来定制视图返回的queryset。


# API Guide

## DjangoFilterBackend

`django-filter`库包含一个为REST framework提供高度可定制字段过滤的`DjangoFilterBackend`类。

要使用`DjangoFilterBackend`，首先要先安装`django-filter`。


```
pip install django-filter
```

现在，你需要将filter backend 添加到你django project的settings.py文件中：


```
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ('django_filters.rest_framework.DjangoFilterBackend',)
}
```





