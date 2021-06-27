---
source:
    - validators.py
---

# 验证器

> 验证器对于在不同类型的字段之间重用验证逻辑非常有用。
>
> &mdash; [Django documentation][cite]

在使用REST framework过程中，你大部分时间会使用默认的验证器，或者在序列化器或字段类上显式的书写验证方法。

然而，有时候你会想要把你的验证逻辑放到可重用的组件里面去，可以在代码库中更方便的使用。你可以用验证函数或者验证类来实现。

## REST framework中的验证器

Django REST framework序列化程序中的验证器的处理方式和Django中的`ModelForm`类中的验证方式不太相同。

`ModelForm`的验证器部分作用在表单中，部分作用在模型实例中。在Django REST framework中的验证器完全作用在序列化类中。这有一下几个优势：

* 他让代码的关注点更分离，让你的代码行为更加显而易见。
* 在快捷`ModelSerializer`类和精确类`Serializer`之间的切换更加容易。每个在`ModelSerializer`类中使用的方法替换起来都很简单。
* 打印序列化实例的`repr`信息会确切的显示使用了什么验证规则。在模型实例上没有调用任何额外的隐藏验证行为。

当你使用`ModelSerializer`时，所有这些都会自动为你处理。如果你想弃用他使用`Serializer`类来替代，那你需要定义确切的每个验证规则。

#### 例子

作为如何使用Django REST framework中验证器的例子，我们将要使用一个有唯一约束字段的模型类。

    class CustomerReportRecord(models.Model):
        time_raised = models.DateTimeField(default=timezone.now, editable=False)
        reference = models.CharField(unique=True, max_length=20)
        description = models.TextField()

这里有一个基础的`ModelSerializer`可以用来创建和更新`CustomerReportRecord`示例：

    class CustomerReportSerializer(serializers.ModelSerializer):
        class Meta:
            model = CustomerReportRecord

如果我们使用`manage.py shell`打开Django命令行，我们可以看到：

    >>> from project.example.serializers import CustomerReportSerializer
    >>> serializer = CustomerReportSerializer()
    >>> print(repr(serializer))
    CustomerReportSerializer():
        id = IntegerField(label='ID', read_only=True)
        time_raised = DateTimeField(read_only=True)
        reference = CharField(max_length=20, validators=[<UniqueValidator(queryset=CustomerReportRecord.objects.all())>])
        description = CharField(style={'type': 'textarea'})

有趣的点在`reference`字段。我们可以看到唯一约束在序列化器的字段中被一个验证器强制执行了。

正因这种确切的风格，REST framework包含了一些Django核心不具备的验证器类。这些类的详细信息如下。

---

## UniqueValidator

这个验证器可以用来强制执行模型字段的`unqieru=True`约束。
他只需要一个必须的参数，和一个`message`参数：

* `queryset` *必要参数* - 这是应该对其强制执行唯一性检查的结果集。
* `message` - 当验证器失败时应该使用的错误信息。
* `lookup` - 用于查找现存的实例中有验证值的查询。默认是`'exact'`。

这个验证器应该使用在`序列化器字段`中，类似于：

    from rest_framework.validators import UniqueValidator

    slug = SlugField(
        max_length=100,
        validators=[UniqueValidator(queryset=BlogPost.objects.all())]
    )

##  UniqueTogetherValidator

这个验证器可以用来强制执行模型实例的`unqieru=True`约束。
她有两个必要参数，和一个客人参数`message`：

* `queryset` *必要参数* - 这是应该对其强制执行唯一性检查的结果集。
* `fields` *必要参数* - 一个需要联合唯一字段的列表或者元组。他们必须是序列化类中的字段。
* `message` - 当验证器失败时应该使用的错误信息。

这个验证器应该使用在`序列化器类`中，类似于：

    from rest_framework.validators import UniqueTogetherValidator

    class ExampleSerializer(serializers.Serializer):
        # ...
        class Meta:
            # ToDo项目属于一个父列表，并且被一个定义的`position`字段排序。
            # 同一个列表中不能共有同一个位置。
            validators = [
                UniqueTogetherValidator(
                    queryset=ToDoItem.objects.all(),
                    fields=['list', 'position']
                )
            ]

---

**提示** `UniqueTogetherValidator`类始终施加一个隐式约束，即它应用的所有字段始终被视为必需的。具有`default`值的字段是个例外，因为即使在用户输入中省略了这些字段，它们也总是提供一个默认值。

---

## UniqueForDateValidator

## UniqueForMonthValidator

## UniqueForYearValidator

这些验证器可以用来对模型实例强制执行`unique_for_date`，`unique_for_month`和`unique_for_year`约束。他们接受如下参数：

* `queryset` *必要参数* - 这是应该对其强制执行唯一性检查的结果集。
* `field` *必要参数* - 将根据其验证给定日期范围中的唯一性的字段名。这必须是序列化类中的字段。
* `date_field` *必要参数* - 将用于确定唯一性约束的日期范围的字段名称这必须是序列化类中的字段。
* `message` - 当验证器失败时应该使用的错误信息。

这个验证器应该使用在`序列化器类`中，类似于：

    from rest_framework.validators import UniqueForYearValidator

    class ExampleSerializer(serializers.Serializer):
        # ...
        class Meta:
            # 博客发布应该有一个当前年份唯一的别名
            validators = [
                UniqueForYearValidator(
                    queryset=BlogPostItem.objects.all(),
                    field='slug',
                    date_field='published'
                )
            ]

用于验证的日期字段始终需要出现在序列化程序类上。你不能简单地依赖模型类`default=.`，因为用于默认值的值只有在验证运行之后才会生成。

根据你希望API的行为方式，你可能需要使用几种样式。如果你使用的是`Model Serializer`，你可能只会依赖 REST framework生成的默认值，但如果使用的是`Serializer`或只是想要更明确的控制，请使用下面演示的样式。

#### 使用一个可写的日期字段

如果你希望日期字段时可以写的，唯一需要注意的事情是你需要确保传入的数据中是可用的，或者设置一个`default`参数，或者设置`required=True`

    published = serializers.DateTimeField(required=True)

### 使用只读日期字段

如果你希望是可见的，但是不可被用户编辑，那就设置`read_only=True`另外设置`default=...`参数

    published = serializers.DateTimeField(read_only=True, default=timezone.now)

#### 使用隐藏日期字段

如果你希望日期字段完全中用户不可见，那么使用`HiddenField`。这个字段类型不接受用户输入，取而代之的是在序列化器中的`validated_data`中始终返回他的默认值。

    published = serializers.HiddenField(default=timezone.now)

---

**注意**：`UniqueFor<Range>Validator`类会强加一个隐式约束，他们应用的字段总是必须的。除了有`default`值的字段，因为他们总会提供一个返回值，即使用户没有输入。

# 高级字段默认值

在序列化器中跨多个字段应用的序列化器，有时可能需要一个不由API客户端提供的字段，但这*是*可以作为验证期的输入的。

你可以希望应用与这种验证期两种模式包括：

* 使用`HiddenField`。这个字段会在`validated_data`中体现，但是*不会*在序列化器输出时使用。
* 使用设置了`read_only=True`的标准字段，同时包含`default=…`参数。这个字段*将会*在序列化器输出中使用，但是不会被用户直接设置。

REST framework包含一系列默认值在上下文中很有用。

#### CurrentUserDefault

一个可以用来代表当前用户的默认类。为了使用他，在实例化序列化器时，'request'必须包含在上下文(context)的字典中

    owner = serializers.HiddenField(
        default=serializers.CurrentUserDefault()
    )

#### CreateOnlyDefault

一个*仅在创建操作中设置默认值*的默认类。在更新操作中会被省略。

他只接收一个参数，可以是一个默认值或者在创建操作中可调用。

    created_at = serializers.DateTimeField(
        default=serializers.CreateOnlyDefault(timezone.now)
    )

---

# 验证器的限制

一些模棱两可的情况下你需要精确自定义验证器，而不是使用`ModelSerializer`默认序列化类生成的。

这时候你可以希望关闭自动生成的验证器，可以通过给序列化器的`Meta.validators`参数指定一个空列表。

## 可选字段

在默认的“联合唯一”验证器强制要求所有的字段是`required=True`。一些情况下，你可能希望对确切的字段使用`required=False`，这种情况下验证器所期望的行为是模棱两可的。

这种情况下你需要在序列化器中排除这个验证器，取而代之你需要在`.validate()`方法或者在视图中精确控制验证逻辑。

例如：

    class BillingRecordSerializer(serializers.ModelSerializer):
        def validate(self, attrs):
            # 在这里或者在视图中，使用自定义的验证器。

        class Meta:
            fields = ['client', 'date', 'amount']
            extra_kwargs = {'client': {'required': False}}
            validators = []  # 移除默认的“联合唯一”约束。

## 更新级联序列化器

当对存在的对象使用更新时，唯一性验证器将会在唯一性检查中排除当前实例。当前的实例在唯一性检查的上下文中是可用的，因为他在序列化器中作为一个属性存在，在最初实例化是作为`instance=...`参数传入。

在这种情况下更新操作在*级联*的序列化器中是不能排除的，因为这个对象是不可用的。

重申，你可能明确希望从序列化器中移除验证器，在`.validate()`方法或者视图中明确编码验证器的约束。

## 调试复杂案例

如果你无法确定一个`ModelSerializer`类会如何生成，一个点子是使用`manage.py shell`，打印出序列化器的实例，这样你就可以看到字段和验证器是如何自动生成的。

    >>> serializer = MyComplexModelSerializer()
    >>> print(serializer)
    class MyComplexModelSerializer:
        my_fields = ...

同时考虑到复杂案例通常在你的序列化类中更明确会更好，而不是使用默认的`ModelSerializer`类来自动生成。这会让你写更多的代码，但是行为结果更加可视。

---

# 自定义验证器

你可以使用Django存在的验证器，或者自己自定义验证器。

# 基于函数

一个验证器可以是一个可调用对象，在验证失败是抛出`serializers.ValidationError`异常。

    def even_number(value):
        if value % 2 != 0:
            raise serializers.ValidationError('This field must be an even number.')

#### 字段级别的验证器

你可以使用`.validate_<field_name>`的规则，在`Serializer`的子类中自定义字段级别的验证器方法，这部分的文档在[Serializer docs](https://www.django-rest-framework.org/api-guide/serializers/#field-level-validation)

## 基于类

使用`__call__`方法来些基于类的验证器。基于类的验证器可以让你参数化并且可以重复使用。

    class MultipleOf:
        def __init__(self, base):
            self.base = base

        def __call__(self, value):
            if value % self.base != 0:
                message = 'This field must be a multiple of %d.' % self.base
                raise serializers.ValidationError(message)

#### 访问上下文

在一些高级的案例中，你可能希望验证器将序列化器字段作为附加上下文使用。你可以通过在验证器上设置 `requires_context = True` 属性来实现。 `_call_` 方法将使用 `serializer_field` 或 `serializer` 作为附加参数被调用。

    requires_context = True

    def __call__(self, value, serializer_field):
        ...

[cite]: https://docs.djangoproject.com/en/stable/ref/validators/