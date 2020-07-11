
# 序列化器

> 扩展serializers的有用性是我们想要解决的问题。但是，这不是一个微不足道的问题，而是需要一些严肃的设计工作。
>
> &mdash; Russell Keith-Magee, [Django用户组][cite]

序列化器允许把像查询集和模型实例这样的复杂数据转换为可以轻松渲染成`JSON`，`XML`或其他内容类型的原生Python类型。序列化器还提供反序列化，在验证传入的数据之后允许解析数据转换回复杂类型。 

REST framework中的serializers与Django的`Form`和`ModelForm`类非常像。我们提供了一个`Serializer`类，它为你提供了强大的通用方法来控制响应的输出，以及一个`ModelSerializer`类，它为创建用于处理模型实例和查询集的序列化程序提供了有用的快捷实现方式。

## 声明序列化器

让我们从创建一个简单的对象开始，我们可以使用下面的例子：

    from datetime import datetime

    class Comment(object):
        def __init__(self, email, content, created=None):
            self.email = email
            self.content = content
            self.created = created or datetime.now()

    comment = Comment(email='leila@example.com', content='foo bar')

我们将声明一个序列化器，我们可以使用它来序列化和反序列化与`Comment`对象相应的数据。

声明一个序列化器看起来非常像声明一个form：

    from rest_framework import serializers

    class CommentSerializer(serializers.Serializer):
        email = serializers.EmailField()
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

## 序列化对象

我们现在可以用`CommentSerializer`去序列化一个comment或comment列表。同样，使用`Serializer`类看起来很像使用`Form`类。

    serializer = CommentSerializer(comment)
    serializer.data
    # {'email': 'leila@example.com', 'content': 'foo bar', 'created': '2016-01-27T15:17:10.375877'}

此时，我们将模型实例转换为Python原生的数据类型。为了完成序列化过程，我们将数据转化为`json`。

    from rest_framework.renderers import JSONRenderer

    json = JSONRenderer().render(serializer.data)
    json
    # b'{"email":"leila@example.com","content":"foo bar","created":"2016-01-27T15:17:10.375877"}'

## 反序列化对象

反序列化是类似的。首先我们将一个流解析为Python原生的数据类型...

    from django.utils.six import BytesIO
    from rest_framework.parsers import JSONParser

    stream = BytesIO(json)
    data = JSONParser().parse(stream)

...然后我们将这些原生数据类型恢复到已验证数据的字典中。

    serializer = CommentSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.validated_data
    # {'content': 'foo bar', 'email': 'leila@example.com', 'created': datetime.datetime(2012, 08, 22, 16, 20, 09, 822243)}

## 保存实例

如果我们希望能够返回基于验证数据的完整对象实例，我们需要实现其中一个或全部实现`.create()`和`update()`方法。例如：

    class CommentSerializer(serializers.Serializer):
        email = serializers.EmailField()
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

        def create(self, validated_data):
            return Comment(**validated_data)

        def update(self, instance, validated_data):
            instance.email = validated_data.get('email', instance.email)
            instance.content = validated_data.get('content', instance.content)
            instance.created = validated_data.get('created', instance.created)
            return instance

如果你的对象实例对应Django的模型，你还需要确保这些方法将对象保存到数据库。例如，如果`Comment`是一个Django模型的话，具体的方法可能如下所示：

        def create(self, validated_data):
            return Comment.objects.create(**validated_data)

        def update(self, instance, validated_data):
            instance.email = validated_data.get('email', instance.email)
            instance.content = validated_data.get('content', instance.content)
            instance.created = validated_data.get('created', instance.created)
            instance.save()
            return instance

现在当我们反序列化数据的时候，基于验证过的数据我们可以调用`.save()`方法返回一个对象实例。

    comment = serializer.save()

调用`.save()`方法将创建新实例或者更新现有实例，具体取决于实例化序列化器类的时候是否传递了现有实例：

    # .save() will create a new instance.
    serializer = CommentSerializer(data=data)

    # .save() will update the existing `comment` instance.
    serializer = CommentSerializer(comment, data=data)

`.create()`和`.update()`方法都是可选的。你可以根据你序列化器类的用例不实现、实现它们之一或都实现。

#### 传递附加属性到`.save()`

有时你会希望你的视图代码能够在保存实例时注入额外的数据。此额外数据可能包括当前用户，当前时间或不是请求数据一部分的其他信息。

你可以通过在调用`.save()`时添加其他关键字参数来执行此操作。例如：

    serializer.save(owner=request.user)

在`.create()`或`.update()`被调用时，任何其他关键字参数将被包含在`validated_data`参数中。

#### 直接重写`.save()`

在某些情况下`.create()`和`.update()`方法可能无意义。例如在contact form中，我们可能不会创建新的实例，而是发送电子邮件或其他消息。

在这些情况下，你可以选择直接重写`.save()`方法，因为那样更可读和有意义。

例如：

    class ContactForm(serializers.Serializer):
        email = serializers.EmailField()
        message = serializers.CharField()

        def save(self):
            email = self.validated_data['email']
            message = self.validated_data['message']
            send_email(from=email, message=message)

请注意在上述情况下，我们现在不得不直接访问serializer的`.validated_data`属性。

## 验证

反序列化数据的时候，你始终需要先调用`is_valid()`方法，然后再尝试去访问经过验证的数据或保存对象实例。如果发生任何验证错误，`.errors`属性将包含表示生成的错误消息的字典。例如：

    serializer = CommentSerializer(data={'email': 'foobar', 'content': 'baz'})
    serializer.is_valid()
    # False
    serializer.errors
    # {'email': [u'Enter a valid e-mail address.'], 'created': [u'This field is required.']}

字典里的每一个键都是字段名称，值是与该字段对应的任何错误消息的字符串列表。`non_field_errors`键可能存在，它将列出任何一般验证错误信息。`non_field_errors`的名称可以通过REST framework设置中的`NON_FIELD_ERRORS_KEY`来自定义。
当对对象列表进行序列化时，返回的错误是每个反序列化项的字典列表。

#### 抛出无效数据的异常

`.is_valid()`方法使用可选的`raise_exception`标志，如果存在验证错误将会抛出一个`serializers.ValidationError`异常。

这些异常由REST framework提供的默认异常处理程序自动处理，默认情况下将返回`HTTP 400 Bad Request`响应。

    # 如果数据无效就返回400响应
    serializer.is_valid(raise_exception=True)

#### 字段级别的验证

你可以通过向你的`Serializer`子类中添加`.validate_<field_name>`方法来指定自定义字段级别的验证。这些类似于Django表单中的`.clean_<field_name>`方法。

这些方法采用单个参数，即需要验证的字段值。

你的`validate_<field_name>`方法应该返回一个验证过的数据或者抛出一个`serializers.ValidationError`异常。例如：

    from rest_framework import serializers

    class BlogPostSerializer(serializers.Serializer):
        title = serializers.CharField(max_length=100)
        content = serializers.CharField()

        def validate_title(self, value):
            """
            Check that the blog post is about Django.
            """
            if 'django' not in value.lower():
                raise serializers.ValidationError("Blog post is not about Django")
            return value

---

**注意：** 如果你在序列化器中声明`<field_name>`的时候带有`required=False`参数，字段不被包含的时候这个验证步骤就不会执行。

---

#### 对象级别的验证

要执行需要访问多个字段的任何其他验证，请添加一个`.validate()`方法到你的`Serializer`子类中。这个方法采用字段值字典的单个参数，如果需要应该抛出一个 `ValidationError`异常，或者只是返回经过验证的值。例如：

    from rest_framework import serializers

    class EventSerializer(serializers.Serializer):
        description = serializers.CharField(max_length=100)
        start = serializers.DateTimeField()
        finish = serializers.DateTimeField()

        def validate(self, data):
            """
            Check that the start is before the stop.
            """
            if data['start'] > data['finish']:
                raise serializers.ValidationError("finish must occur after start")
            return data

#### 验证器

序列化器上的各个字段都可以包含验证器，通过在字段实例上声明，例如：

    def multiple_of_ten(value):
        if value % 10 != 0:
            raise serializers.ValidationError('Not a multiple of ten')

    class GameRecord(serializers.Serializer):
        score = IntegerField(validators=[multiple_of_ten])
        ...

序列化器类还可以包括应用于一组字段数据的可重用的验证器。这些验证器要在内部的`Meta`类中声明，如下所示：

    class EventSerializer(serializers.Serializer):
        name = serializers.CharField()
        room_number = serializers.IntegerField(choices=[101, 102, 103, 201])
        date = serializers.DateField()

        class Meta:
            # 每间屋子每天只能有1个活动。
            validators = UniqueTogetherValidator(
                queryset=Event.objects.all(),
                fields=['room_number', 'date']
            )

更多信息请参阅 [validators文档](validators.md)。

## 访问初始数据和实例

将初始化对象或者查询集传递给序列化实例时，可以通过`.instance`访问。如果没有传递初始化对象，那么`.instance`属性将是`None`。

将数据传递给序列化器实例时，未修改的数据可以通过`.initial_data`获取。如果没有传递data关键字参数，那么`.initial_data`属性就不存在。

## 部分更新

默认情况下，序列化器必须传递所有必填字段的值，否则就会引发验证错误。你可以使用 `partial`参数来允许部分更新。

    # 使用部分数据更新`comment` 
    serializer = CommentSerializer(comment, data={'content': u'foo bar'}, partial=True)

## 处理嵌套对象

前面的实例适用于处理只有简单数据类型的对象，但是有时候我们也需要表示更复杂的对象，其中对象的某些属性可能不是字符串、日期、整数这样简单的数据类型。

`Serializer`类本身也是一种`Field`，并且可以用来表示一个对象类型嵌套在另一个对象中的关系。

    class UserSerializer(serializers.Serializer):
        email = serializers.EmailField()
        username = serializers.CharField(max_length=100)

    class CommentSerializer(serializers.Serializer):
        user = UserSerializer()
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

如果嵌套表示可以接收 `None`值，则应该将 `required=False`标志传递给嵌套的序列化器。

    class CommentSerializer(serializers.Serializer):
        user = UserSerializer(required=False)  # 可能是匿名用户。
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

类似的，如果嵌套的关联字段可以接收一个列表，那么应该将`many=True`标志传递给嵌套的序列化器。

    class CommentSerializer(serializers.Serializer):
        user = UserSerializer(required=False)
        edits = EditItemSerializer(many=True)  # edit'项的嵌套列表
        content = serializers.CharField(max_length=200)
        created = serializers.DateTimeField()

## 可写的嵌套表示

当处理支持反序列化数据的嵌套表示时，嵌套对象的任何错误都嵌套在嵌套对象的字段名下。

    serializer = CommentSerializer(data={'user': {'email': 'foobar', 'username': 'doe'}, 'content': 'baz'})
    serializer.is_valid()
    # False
    serializer.errors
    # {'user': {'email': [u'Enter a valid e-mail address.']}, 'created': [u'This field is required.']}

类似的，`.validated_data` 属性将包括嵌套数据结构。

#### 为嵌套关系定义`.create()`方法

如果你支持可写的嵌套表示，则需要编写`.create()`或`.update()`处理保存多个对象的方法。

下面的示例演示如何处理创建一个具有嵌套的概要信息对象的用户。

    class UserSerializer(serializers.ModelSerializer):
        profile = ProfileSerializer()

        class Meta:
            model = User
            fields = ('username', 'email', 'profile')

        def create(self, validated_data):
            profile_data = validated_data.pop('profile')
            user = User.objects.create(**validated_data)
            Profile.objects.create(user=user, **profile_data)
            return user

#### 为嵌套关系定义`.update()`方法

对于更新，你需要仔细考虑如何处理关联字段的更新。 例如，如果关联字段的值是`None`，或者没有提供，那么会发生下面哪一项？

* 在数据库中将关联字段设置成`NULL`。
* 删除关联的实例。
* 忽略数据并保留这个实例。
* 抛出验证错误。

下面是我们之前`UserSerializer`类中`update()`方法的一个例子。 

        def update(self, instance, validated_data):
            profile_data = validated_data.pop('profile')
            # 除非应用程序正确执行，
            # 保证这个字段一直被设置，
            # 否则就应该抛出一个需要处理的`DoesNotExist`。
            profile = instance.profile

            instance.username = validated_data.get('username', instance.username)
            instance.email = validated_data.get('email', instance.email)
            instance.save()

            profile.is_premium_member = profile_data.get(
                'is_premium_member',
                profile.is_premium_member
            )
            profile.has_support_contract = profile_data.get(
                'has_support_contract',
                profile.has_support_contract
             )
            profile.save()

            return instance

因为嵌套关系的创建和更新行为可能不明确，并且可能需要关联模型间的复杂依赖关系，REST framework 3 要求你始终明确的定义这些方法。默认的`ModelSerializer` `.create()`和`.update()`方法不包括对可写嵌套关联的支持。

提供自动支持某种类型的自动写入嵌套关联的第三方包可能与3.1版本一同放出。

#### 处理在模型管理类中保存关联实例

在序列化器中保存多个相关实例的另一种方法是编写处理创建正确实例的自定义模型管理器类。

例如，假设我们想确保`User`实例和`Profile`实例总是作为一对一起创建。我们可能会写一个类似这样的自定义管理器类：

    class UserManager(models.Manager):
        ...

        def create(self, username, email, is_premium_member=False, has_support_contract=False):
            user = User(username=username, email=email)
            user.save()
            profile = Profile(
                user=user,
                is_premium_member=is_premium_member,
                has_support_contract=has_support_contract
            )
            profile.save()
            return user

这个管理器类现在更好的封装了用户实例和用户信息实例总是在同一时间创建。我们在序列化器类上的`.create()`方法现在能够用新的管理器方法重写。

    def create(self, validated_data):
        return User.objects.create(
            username=validated_data['username'],
            email=validated_data['email']
            is_premium_member=validated_data['profile']['is_premium_member']
            has_support_contract=validated_data['profile']['has_support_contract']
        )

有关此方法的更多详细信息，请参阅Django文档中的 [模型管理器][model-managers]和[使用模型和管理器类的相关博客][encapsulation-blogpost]。

## 处理多个对象

`Serializer`类还可以序列化或反序列化对象的列表。

#### 序列化多个对象

为了能够序列化一个查询集或者一个对象列表而不是一个单独的对象，应该在实例化序列化器类的时候传一个`many=True`参数。这样就能序列化一个查询集或一个对象列表。

    queryset = Book.objects.all()
    serializer = BookSerializer(queryset, many=True)
    serializer.data
    # [
    #     {'id': 0, 'title': 'The electric kool-aid acid test', 'author': 'Tom Wolfe'},
    #     {'id': 1, 'title': 'If this is a man', 'author': 'Primo Levi'},
    #     {'id': 2, 'title': 'The wind-up bird chronicle', 'author': 'Haruki Murakami'}
    # ]

#### 反序列化多个对象

反序列化多个对象默认支持多个对象的创建，但是不支持多个对象的更新。有关如何支持或自定义这些情况的更多信息，请查阅这个文档[ListSerializer](#listserializer)。

## 包括额外的上下文

在某些情况下，除了要序列化的对象之外，还需要为序列化程序提供额外的上下文。一个常见的情况是，如果你使用包含超链接关系的序列化程序，这需要序列化器能够访问当前的请求以便正确生成完全限定的URL。

你可以在实例化序列化器的时候传递一个`context`参数来传递任意的附加上下文。例如：

    serializer = AccountSerializer(account, context={'request': request})
    serializer.data
    # {'id': 6, 'owner': u'denvercoder9', 'created': datetime.datetime(2013, 2, 12, 09, 44, 56, 678870), 'details': 'http://example.com/accounts/6/details'}

这个上下文的字典可以在任何序列化器字段的逻辑中使用，例如`.to_representation()`方法中可以通过访问`self.context`属性获取上下文字典。

---

# ModelSerializer

通常你会想要与Django模型相对应的序列化类。

`ModelSerializer`类能够让你自动创建一个具有模型中相应字段的`Serializer`类。

**这个`ModelSerializer`类和常规的`Serializer`类一样，不同的是**：

* 它根据模型自动生成一组字段。
* 它自动生成序列化器的验证器，比如unique_together验证器。
* 它默认简单实现了`.create()`方法和`.update()`方法。

声明一个`ModelSerializer`如下：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ('id', 'account_name', 'users', 'created')

默认情况下，所有的模型的字段都将映射到序列化器上相应的字段。

模型中任何关联字段比如外键都将映射到`PrimaryKeyRelatedField`字段。默认情况下不包括反向关联，除非像[serializer relations][relations]文档中规定的那样显式包含。

#### 检查`ModelSerializer`

序列化类生成有用的详细表示字符串，允许你全面检查其字段的状态。 这在使用`ModelSerializers`时特别有用，因为你想确定自动创建了哪些字段和验证器。

要检查的话，打开Django shell,执行 `python manage.py shell`，然后导入序列化器类，实例化它，并打印对象的表示：

    >>> from myapp.serializers import AccountSerializer
    >>> serializer = AccountSerializer()
    >>> print(repr(serializer))
    AccountSerializer():
        id = IntegerField(label='ID', read_only=True)
        name = CharField(allow_blank=True, max_length=100, required=False)
        owner = PrimaryKeyRelatedField(queryset=User.objects.all())

## 指定要包括的字段

如果你希望在模型序列化器中使用默认字段的一部分，你可以使用`fields`或`exclude`选项来执行此操作，就像使用`ModelForm`一样。强烈建议你使用`fields`属性显式的设置要序列化的字段。这样就不太可能因为你修改了模型而无意中暴露了数据。

例如：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ('id', 'account_name', 'users', 'created')

你还可以将`fields`属性设置成`'__all__'`来表明使用模型中的所有字段。

例如：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = '__all__'

你可以将`exclude`属性设置成一个从序列化器中排除的字段列表。

例如：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            exclude = ('users',)

在上面的例子中，如果`Account`模型有三个字段`account_name`，`users`和`created`，那么只有 `account_name`和`created`会被序列化。

在`fields`和`exclude`属性中的名称，通常会映射到模型类中的模型字段。

或者`fields`选项中的名称可以映射到模型类中不存在任何参数的属性或方法。

## 指定嵌套序列化

默认`ModelSerializer`使用主键进行关联，但是你也可以使用`depth`选项轻松生成嵌套关联：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ('id', 'account_name', 'users', 'created')
            depth = 1

`depth`选项应该设置一个整数值，表明应该遍历的关联深度。

如果要自定义序列化的方式你需要自定义该子段。

## 明确指定字段

你可以通过在`ModelSerializer`类上声明字段来增加额外的字段或者重写默认的字段，就和在`Serializer`类一样的。

    class AccountSerializer(serializers.ModelSerializer):
        url = serializers.CharField(source='get_absolute_url', read_only=True)
        groups = serializers.PrimaryKeyRelatedField(many=True)

        class Meta:
            model = Account

额外的字段可以对应模型上任何属性或可调用的方法。

## 指定只读字段

你可能希望将多个字段指定为只读，而不是显式的为每个字段添加`read_only=True`属性，这种情况你可以使用Meta的`read_only_fields`选项。

该选项应该是字段名称的列表或元祖，并像下面这样声明：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ('id', 'account_name', 'users', 'created')
            read_only_fields = ('account_name',)

模型中已经设置`editable=False`的字段和默认就被设置为只读的`AutoField`字段都不需要添加到`read_only_fields`选项中。

---

**注意**: 有一种特殊情况，其中一个只读字段是模型级别`unique_together`约束的一部分。在这种情况下，序列化器类需要该字段才能验证约束，但也不能由用户编辑。

处理此问题的正确方法是在序列化器上显式指定该字段，同时提供`read_only=True`和`default=…`关键字参数。

这种情况的一个例子就是对于一个和其他标识符`unique_together`的当前认证的`User`是只读的。 在这种情况下你可以像下面这样声明user字段：

    user = serializers.PrimaryKeyRelatedField(read_only=True, default=serializers.CurrentUserDefault())

有关[UniqueTogetherValidator](/api-guide/validators/#uniquetogethervalidator)和[CurrentUserDefault](/api-guide/validators/#currentuserdefault)类的详细文档，请查阅[验证器的文档](/api-guide/validators/)。

---


## 附加关键字参数

还可以通过使用`extra_kwargs`选项快捷地在字段上指定任意附加的关键字参数。在`read_only_fields`这种情况下，你不需要在序列化器上式的声明该字段。

这个选项是一个将具体字段名称当作键值的字典。例如：

    class CreateUserSerializer(serializers.ModelSerializer):
        class Meta:
            model = User
            fields = ('email', 'username', 'password')
            extra_kwargs = {'password': {'write_only': True}}

        def create(self, validated_data):
            user = User(
                email=validated_data['email'],
                username=validated_data['username']
            )
            user.set_password(validated_data['password'])
            user.save()
            return user

## 关联字段

在序列化模型实例的时候，你可以选择多种不同的方式来表示关联关系。对于`ModelSerializer`默认是使用相关实例的主键。

替代的其他方法包括使用超链接序列化，序列化完整的嵌套表示或者使用自定义表示的序列化。

更多详细信息请查阅[serializer relations][relations]文档。

## 自定义字段映射

ModelSerializer类还公开了一个可以覆盖的API，以便在实例化序列化器时改变序列化器字段的自动确定。

通常情况下，如果`ModelSerializer`没有生成默认情况下你需要的字段，那么你应该将它们显式地添加到类中，或者直接使用常规的`Serializer`类。但是在某些情况下，你可能需要创建一个新的基类，来定义给任意模型创建序列化字段的方式。

### `.serializer_field_mapping`

将Django model类映射到REST framework serializer类。你可以覆写这个映射来更改每个模型应该使用的默认序列化器类。

### `.serializer_related_field`

这个属性应是序列化器字段类，默认情况下用于关联字段。

对于`ModelSerializer`此属性默认是`PrimaryKeyRelatedField`。

对于`HyperlinkedModelSerializer`此属性默认是`serializers.HyperlinkedRelatedField`。

### `serializer_url_field`

应该用于序列化器上任何`url`字段的序列化器字段类。

默认是 `serializers.HyperlinkedIdentityField`

### `serializer_choice_field`

应用于序列化器上任何选择字段的序列化器字段类。

默认是`serializers.ChoiceField`

### The field_class和field_kwargs API

调用下面的方法来确定应该自动包含在序列化器类中每个字段的类和关键字参数。这些方法都应该返回两个 `(field_class, field_kwargs)`元祖。

### `.build_standard_field(self, field_name, model_field)`

调用后生成对应标准模型字段的序列化器字段。

默认实现是根据`serializer_field_mapping`属性返回一个序列化器类。

### `.build_relational_field(self, field_name, relation_info)`

调用后生成对应关联模型字段的序列化器字段。

默认实现是根据`serializer_relational_field`属性返回一个序列化器类。

这里的`relation_info`参数是一个命名元祖，包含`model_field`，`related_model`，`to_many`和`has_through_model`属性。

### `.build_nested_field(self, field_name, relation_info, nested_depth)`

当`depth`选项被设置时，被调用后生成一个对应到关联模型字段的序列化器字段。

默认实现是动态的创建一个基于`ModelSerializer`或`HyperlinkedModelSerializer`的嵌套的序列化器类。

`nested_depth`的值是`depth`的值减1。

`relation_info`参数是一个命名元祖，包含 `model_field`，`related_model`，`to_many`和`has_through_model`属性。

### `.build_property_field(self, field_name, model_class)`

被调用后生成一个对应到模型类中属性或无参数方法的序列化器字段。

默认实现是返回一个`ReadOnlyField`类。

### `.build_url_field(self, field_name, model_class)`

被调用后为序列化器自己的`url`字段生成一个序列化器字段。默认实现是返回一个`HyperlinkedIdentityField`类。

### `.build_unknown_field(self, field_name, model_class)`

当字段名称没有对应到任何模型字段或者模型属性时调用。
默认实现会抛出一个错误，尽管子类可能会自定义该行为。

---

# HyperlinkedModelSerializer

`HyperlinkedModelSerializer`类类似于`ModelSerializer`类，不同之处在于它使用超链接来表示关联关系而不是主键。

默认情况下序列化器将包含一个`url`字段而不是主键字段。

url字段将使用`HyperlinkedIdentityField`字段来表示，模型的任何关联都将使用`HyperlinkedRelatedField`字段来表示。

你可以通过将主键添加到`fields`选项中来显式的包含，例如：

    class AccountSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = Account
            fields = ('url', 'id', 'account_name', 'users', 'created')

## 绝对和相对URL

当实例化一个`HyperlinkedModelSerializer`时，你必须在序列化器的上下文中包含当前的`request`值，例如：

    serializer = AccountSerializer(queryset, context={'request': request})

这样做将确保超链接可以包含恰当的主机名，一边生成完全限定的URL，例如：

    http://api.example.com/accounts/1/

而不是相对的URL，例如：

    /accounts/1/

如果你*真的*要使用相对URL，你应该明确的在序列化器上下文中传递一个`{'request': None}`。

## 如何确定超链接视图

需要一种确定哪些视图能应用超链接到模型实例的方法。

默认情况下，超链接期望对应到一个样式能匹配`'{model_name}-detail'`的视图，并通过`pk`关键字参数查找实例。

你可以通过在`extra_kwargs`中设置`view_name`和`lookup_field`中的一个或两个来重写URL字段视图名称和查询字段。如下所示：

    class AccountSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = Account
            fields = ('account_url', 'account_name', 'users', 'created')
            extra_kwargs = {
                'url': {'view_name': 'accounts', 'lookup_field': 'account_name'},
                'users': {'lookup_field': 'username'}
            }

或者你可以显式的设置序列化器上的字段。例如：

    class AccountSerializer(serializers.HyperlinkedModelSerializer):
        url = serializers.HyperlinkedIdentityField(
            view_name='accounts',
            lookup_field='slug'
        )
        users = serializers.HyperlinkedRelatedField(
            view_name='user-detail',
            lookup_field='username',
            many=True,
            read_only=True
        )

        class Meta:
            model = Account
            fields = ('url', 'account_name', 'users', 'created')

---

**提示**：正确匹配超链接表示和你的URL配置有时可能会有点困难。打印一个`HyperlinkedModelSerializer`实例的`repr`是一个特别有用的方式来检查关联关系映射的那些视图名称和查询字段。

---

## 更改URL字段名称

URL字段的名称默认为'url'。你可以通过使用`URL_FIELD_NAME`设置进行全局性修改。

---

# ListSerializer

`ListSerializer`类能够序列化和一次验证多个对象。你*通常*不需要直接使用`ListSerializer`，而是应该在实例化一个序列化器时简单地传递一个`many=True`参数。

当一个序列化器在带有`many=True`选项被序列化时，将创建一个`ListSerializer`实例。该序列化器类将成为`ListSerializer`类的子类。

下面的参数也可以传递给`ListSerializer`字段或者一个带有`many=True`参数的序列化器。

### `allow_empty`

默认是`True`，但是如果你不想把空列表当作有效输入的话可以把它设置成`False`。

### 自定义`ListSerializer`行为

下面是你可能希望要定制`ListSerializer`行为的**一些**情况。例如：

* 你希望提供列表的特定验证，例如检查一个元素是否与列表中的另外一个元素冲突。
* 你要自定义多个对象的创建或更新行为。

对于这些情况，当你可以通过使用序列化器类的`Meta`类下面的`list_serializer_class`选项来修改当`many=True`时正在使用的类。

例如：

    class CustomListSerializer(serializers.ListSerializer):
        ...

    class CustomSerializer(serializers.Serializer):
        ...
        class Meta:
            list_serializer_class = CustomListSerializer

#### 自定义多个对象的创建

多个对象的创建默认实现是简单地调用列表中每个对象的`.create()`方法。如果要自定义实现，那么你需要自定义当被传递`many=True`参数时使用的`ListSerializer`类中的`.create()`方法。

例如：

    class BookListSerializer(serializers.ListSerializer):
        def create(self, validated_data):
            books = [Book(**item) for item in validated_data]
            return Book.objects.bulk_create(books)

    class BookSerializer(serializers.Serializer):
        ...
        class Meta:
            list_serializer_class = BookListSerializer

#### 自定义多对象的更新

默认情况下，`ListSerializer`类不支持多对象的更新。这是因为插入和删除的预期行为是不明确的。

要支持多对象更新的话你需要自己明确地实现。编写多个对象更新的代码时要注意以下几点：

* 如何确定数据列表中的每个元素应该对应更新哪个实例？
* 如何处理插入？它们是无效的？还是创建新对象？
* 移除应该如何处理？它们是要删除对象还是删除关联关系？它们应该被忽略还是提示无效操作？
* 排序如何处理？改变两个元素的位置是否意味着任何状态改变或者应该被忽视？

你需要向实例序列化器中显式添加一个`id`字段。默认隐式生成的`id`字段是`read_only`。这就导致它在更新时被删除。一旦你明确地声明它，它将在列表序列化器的`update`方法中可用。

下面是一个你可以选择用来做多个对象更新的示例：

    class BookListSerializer(serializers.ListSerializer):
        def update(self, instance, validated_data):
            # Maps for id->instance and id->data item.
            book_mapping = {book.id: book for book in instance}
            data_mapping = {item['id']: item for item in validated_data}

            # Perform creations and updates.
            ret = []
            for book_id, data in data_mapping.items():
                book = book_mapping.get(book_id, None)
                if book is None:
                    ret.append(self.child.create(data))
                else:
                    ret.append(self.child.update(book, data))

            # Perform deletions.
            for book_id, book in book_mapping.items():
                if book_id not in data_mapping:
                    book.delete()

            return ret

    class BookSerializer(serializers.Serializer):
        # 我们需要使用主键来识别列表中的元素，
        # 所以在这里使用可写的字段，而不是默认的只读字段。
        id = serializers.IntegerField()

        ...
        id = serializers.IntegerField(required=False)

        class Meta:
            list_serializer_class = BookListSerializer

类似于REST framework 2中`allow_add_remove`的自动支持多个对象更新操作可能会在3.1版本的第三方包中提供。

#### 自定义ListSerializer初始化

当带有`many=True`参数的序列化器被实例化时，我们需要确定哪些参数和关键字参数应该被传递给子类`Serializer`和父类`ListSerializer`的`.__init__()`方法。

默认实现是将所有参数都传递给两个类，出了`validators`和任何关键字参数。这两个参数都假定用于子序列化器类。

偶尔，你可能需要明确指定当被传递`many=True`参数时，子类和父类应该如何实例化。你可以使用`many_init`类方法来执行此操作。

        @classmethod
        def many_init(cls, *args, **kwargs):
            # 实例化子序列化器类。
            kwargs['child'] = cls()
            # 实例化列表序列化父类
            return CustomListSerializer(*args, **kwargs)

---

# BaseSerializer

`BaseSerializer` 可以很简单的用来替代序列化和反序列化的样式。

该类实现与`Serializer`类相同的基本API：

* `.data` - 返回传出的原始数据。
* `.is_valid()` - 反序列化并验证传入的数据。
* `.validated_data` - 返回经过验证后的传入数据。
* `.errors` - 返回验证期间的错误。
* `.save()` - 将验证的数据保留到对象实例中。

它还有可以覆写的四种方法，具体取决于你想要序列化类支持的功能：

* `.to_representation()` - 重写此方法来改变读取操作的序列化结果。
* `.to_internal_value()` - 重写此方法来改变写入操作的序列化结果。
* `.create()` 和 `.update()` - 重写其中一个或两个来改变保存实例时的动作。

因为此类提供与`Serializer`类相同的接口，所以你可以将它与现有的基于类的通用视图一起使用，就像使用常规`Serializer`或`ModelSerializer`一样。

这样做时你需要注意到的唯一区别是`BaseSerializer`类并不会在可浏览的API页面中生成HTML表单。

##### 只读的 `BaseSerializer` classes

要使用`BaseSerializer`类实现只读序列化程序，我们只需要覆写`.to_representation()`方法。让我们看一个简单的Django模型的示例：

    class HighScore(models.Model):
        created = models.DateTimeField(auto_now_add=True)
        player_name = models.CharField(max_length=10)
        score = models.IntegerField()

创建一个只读的序列化程序来将`HighScore`实例转换为原始数据类型非常简单。

    class HighScoreSerializer(serializers.BaseSerializer):
        def to_representation(self, obj):
            return {
                'score': obj.score,
                'player_name': obj.player_name
            }

我们现在可以使用这个类来序列化单个`HighScore`实例：

    @api_view(['GET'])
    def high_score(request, pk):
        instance = HighScore.objects.get(pk=pk)
        serializer = HighScoreSerializer(instance)
	    return Response(serializer.data)

或者使用它来序列化多个实例：

    @api_view(['GET'])
    def all_high_scores(request):
        queryset = HighScore.objects.order_by('-score')
        serializer = HighScoreSerializer(queryset, many=True)
	    return Response(serializer.data)

##### Read-write `BaseSerializer` classes

要创建一个读写都支持的序列化器，我们首先需要实现`.to_internal_value()`方法。这个方法返回用来构造对象实例的经过验证的值，如果提供的数据格式不正确，则可能引发`ValidationError`。

一旦你实现了`.to_internal_value()`方法，那些基础的验证API都会在序列化对象上可用了，你就可以使用`.is_valid()`, `.validated_data` 和 `.errors` 方法。

如果你还想支持`.save()`，你还需要实现`.create()`和`.update()`方法中的一个或两个。

下面就是完整版的，支持读、写操作的 `HighScoreSerializer` 完整示例了。

    class HighScoreSerializer(serializers.BaseSerializer):
        def to_internal_value(self, data):
            score = data.get('score')
            player_name = data.get('player_name')

            # 执行数据有效性校验
            if not score:
                raise ValidationError({
                    'score': 'This field is required.'
                })
            if not player_name:
                raise ValidationError({
                    'player_name': 'This field is required.'
                })
            if len(player_name) > 10:
                raise ValidationError({
                    'player_name': 'May not be more than 10 characters.'
                })

			# 返回通过验证的数据 这用来作为 `.validated_data` 属性的值。
            return {
                'score': int(score),
                'player_name': player_name
            }

        def to_representation(self, obj):
            return {
                'score': obj.score,
                'player_name': obj.player_name
            }

        def create(self, validated_data):
            return HighScore.objects.create(**validated_data)

#### 创建一个新的基类

`BaseSerializer`类还可以用来创建新的通用序列化程序基类来处理特定的序列化样式或者用来整合备用存储后端。

下面这个类是一个可以将任意对象强制转换为基本表示的通用序列化程序的示例。

    class ObjectSerializer(serializers.BaseSerializer):
        """
        一个只读序列化程序，它将任意复杂的对象强制转换为内置数据类型表示。
        """
        def to_representation(self, obj):
            for attribute_name in dir(obj):
                attribute = getattr(obj, attribute_name)
                if attribute_name('_'):
                    # 忽略私有属性
                    pass
                elif hasattr(attribute, '__call__'):
                    # 忽略方法和其他可调用对象
                    pass
                elif isinstance(attribute, (str, int, bool, float, type(None))):
                    # 内置的原始数据类型不做修改
                    output[attribute_name] = attribute
                elif isinstance(attribute, list):
                    # 递归处理列表中的对象
                    output[attribute_name] = [
                        self.to_representation(item) for item in attribute
                    ]
                elif isinstance(attribute, dict):
                    # 递归处理字典中的对象
                    output[attribute_name] = {
                        str(key): self.to_representation(value)
                        for key, value in attribute.items()
                    }
                else:
                    # 将其他数据类型强制转换为字符串表示
                    output[attribute_name] = str(attribute)

---

# serializer进阶用法

## 重写序列化和反序列化行为

如果你需要更改序列化程序类的序列化，反序列化或验证的行为，可以通过重写`.to_representation()`或`.to_internal_value()`方法来完成。

我们可能这么做的一些原因包括......

* 为新的序列化程序基类添加新行为。
* 对现有的类稍作修改。
* 提高频繁访问返回大量数据的API端点的序列化性能。

这些方法的签名如下：

#### `.to_representation(self, obj)`

接收一个需要被序列化的对象实例并且返回一个序列化之后的表示。通常，这意味着返回内置Python数据类型的结构。可以处理的确切类型取决于你为API配置的渲染类。

#### ``.to_internal_value(self, data)``

将未经验证的传入数据作为输入，返回可以通过`serializer.validated_data`来访问的已验证的数据。如果在序列化程序类上调用`.save()`，则该返回值也将传递给`.create()`或`.update()`方法。

如果任何验证条件失败，那么该方法会引发一个`serializers.ValidationError(errors)`。通常，此处的`errors`参数将是错误消息字典的一个映射字段。

传递给此方法的`data`参数通常是`request.data`的值，因此它提供的数据类型将取决于你为API配置的解析器类。

## Serializer 继承

与Django表单类似，你可以通过继承扩展和重用序列化程序。这允许你在父类上声明一组公共字段或方法，然后可以在许多序列化程序中使用它们。举个例子，

    class MyBaseSerializer(Serializer):
        my_field = serializers.CharField()

        def validate_my_field(self):
            ...

    class MySerializer(MyBaseSerializer):
        ...

像Django的`Model`和`ModelForm`类一样，序列化器上的内部`Meta`类不会隐式地继承它的父元素内部的`Meta`类。如果你希望从父类继承`Meta`类，则必须明确地这样做。比如：

    class AccountSerializer(MyBaseSerializer):
        class Meta(MyBaseSerializer.Meta):
            model = Account

通常我们建议*不要*在内部Meta类上使用继承，而是明确声明所有选项。

此外，以下警告适用于序列化程序继承：

* 正常的Python名称解析规则适用。如果你有多个声明了`Meta`类的基类，则只使用第一个类。这意味要么是孩子的`Meta`（如果存在），否则就是第一个父母的`Meta`等。
* 通过在子类上将名称设置为`None`，可以声明性地删除从父类继承的`Field`。

        class MyBaseSerializer(ModelSerializer):
            my_field = serializers.CharField()

        class MySerializer(MyBaseSerializer):
            my_field = None

    但是，你只能使用此技术去掉父类声明性定义的字段；它不会阻止`ModelSerializer`生成默认字段。想要从默认字段中选择去掉某个字段，请参阅 [指定要包括的字段](#specifying-which-fields-to-include)。

## Dynamically modifying fields

Once a serializer has been initialized, the dictionary of fields that are set on the serializer may be accessed using the `.fields` attribute.  Accessing and modifying this attribute allows you to dynamically modify the serializer.

Modifying the `fields` argument directly allows you to do interesting things such as changing the arguments on serializer fields at runtime, rather than at the point of declaring the serializer.

### Example

For example, if you wanted to be able to set which fields should be used by a serializer at the point of initializing it, you could create a serializer class like so:

    class DynamicFieldsModelSerializer(serializers.ModelSerializer):
        """
        A ModelSerializer that takes an additional `fields` argument that
        controls which fields should be displayed.
        """

        def __init__(self, *args, **kwargs):
            # Don't pass the 'fields' arg up to the superclass
            fields = kwargs.pop('fields', None)

            # Instantiate the superclass normally
            super(DynamicFieldsModelSerializer, self).__init__(*args, **kwargs)

            if fields is not None:
                # Drop any fields that are not specified in the `fields` argument.
                allowed = set(fields)
                existing = set(self.fields.keys())
                for field_name in existing - allowed:
                    self.fields.pop(field_name)

This would then allow you to do the following:

    >>> class UserSerializer(DynamicFieldsModelSerializer):
    >>>     class Meta:
    >>>         model = User
    >>>         fields = ('id', 'username', 'email')
    >>>
    >>> print UserSerializer(user)
    {'id': 2, 'username': 'jonwatts', 'email': 'jon@example.com'}
    >>>
    >>> print UserSerializer(user, fields=('id', 'email'))
    {'id': 2, 'email': 'jon@example.com'}

## Customizing the default fields

REST framework 2 provided an API to allow developers to override how a `ModelSerializer` class would automatically generate the default set of fields.

This API included the `.get_field()`, `.get_pk_field()` and other methods.

Because the serializers have been fundamentally redesigned with 3.0 this API no longer exists. You can still modify the fields that get created but you'll need to refer to the source code, and be aware that if the changes you make are against private bits of API then they may be subject to change.

---

# Third party packages

The following third party packages are also available.

## Django REST marshmallow

The [django-rest-marshmallow][django-rest-marshmallow] package provides an alternative implementation for serializers, using the python [marshmallow][marshmallow] library. It exposes the same API as the REST framework serializers, and can be used as a drop-in replacement in some use-cases.

## Serpy

The [serpy][serpy] package is an alternative implementation for serializers that is built for speed. [Serpy][serpy] serializes complex datatypes to simple native types. The native types can be easily converted to JSON or any other format needed.

## MongoengineModelSerializer

The [django-rest-framework-mongoengine][mongoengine] package provides a `MongoEngineModelSerializer` serializer class that supports using MongoDB as the storage layer for Django REST framework.

## GeoFeatureModelSerializer

The [django-rest-framework-gis][django-rest-framework-gis] package provides a `GeoFeatureModelSerializer` serializer class that supports GeoJSON both for read and write operations.

## HStoreSerializer

The [django-rest-framework-hstore][django-rest-framework-hstore] package provides an `HStoreSerializer` to support [django-hstore][django-hstore] `DictionaryField` model field and its `schema-mode` feature.

## Dynamic REST

The [dynamic-rest][dynamic-rest] package extends the ModelSerializer and ModelViewSet interfaces, adding API query parameters for filtering, sorting, and including / excluding all fields and relationships defined by your serializers.

## Dynamic Fields Mixin

The [drf-dynamic-fields][drf-dynamic-fields] package provides a mixin to dynamically limit the fields per serializer to a subset specified by an URL parameter.

## DRF FlexFields

The [drf-flex-fields][drf-flex-fields] package extends the ModelSerializer and ModelViewSet to provide commonly used functionality for dynamically setting fields and expanding primitive fields to nested models, both from URL parameters and your serializer class definitions.

## Serializer Extensions

The [django-rest-framework-serializer-extensions][drf-serializer-extensions]
package provides a collection of tools to DRY up your serializers, by allowing
fields to be defined on a per-view/request basis. Fields can be whitelisted,
blacklisted and child serializers can be optionally expanded.

## HTML JSON Forms

The [html-json-forms][html-json-forms] package provides an algorithm and serializer for processing `<form>` submissions per the (inactive) [HTML JSON Form specification][json-form-spec].  The serializer facilitates processing of arbitrarily nested JSON structures within HTML.  For example, `<input name="items[0][id]" value="5">` will be interpreted as `{"items": [{"id": "5"}]}`.

## DRF-Base64

[DRF-Base64][drf-base64] provides a set of field and model serializers that handles the upload of base64-encoded files.

## QueryFields

[djangorestframework-queryfields][djangorestframework-queryfields] allows API clients to specify which fields will be sent in the response via inclusion/exclusion query parameters.  

## DRF Writable Nested

The [drf-writable-nested][drf-writable-nested] package provides writable nested model serializer which allows to create/update models with nested related data.

[cite]: https://groups.google.com/d/topic/django-users/sVFaOfQi4wY/discussion
[relations]: relations.md
[model-managers]: https://docs.djangoproject.com/en/stable/topics/db/managers/
[encapsulation-blogpost]: http://www.dabapps.com/blog/django-models-and-encapsulation/
[django-rest-marshmallow]: http://tomchristie.github.io/django-rest-marshmallow/
[marshmallow]: https://marshmallow.readthedocs.io/en/latest/
[serpy]: https://github.com/clarkduvall/serpy
[mongoengine]: https://github.com/umutbozkurt/django-rest-framework-mongoengine
[django-rest-framework-gis]: https://github.com/djangonauts/django-rest-framework-gis
[django-rest-framework-hstore]: https://github.com/djangonauts/django-rest-framework-hstore
[django-hstore]: https://github.com/djangonauts/django-hstore
[dynamic-rest]: https://github.com/AltSchool/dynamic-rest
[html-json-forms]: https://github.com/wq/html-json-forms
[drf-flex-fields]: https://github.com/rsinger86/drf-flex-fields
[json-form-spec]: https://www.w3.org/TR/html-json-forms/
[drf-dynamic-fields]: https://github.com/dbrgn/drf-dynamic-fields
[drf-base64]: https://bitbucket.org/levit_scs/drf_base64
[drf-serializer-extensions]: https://github.com/evenicoulddoit/django-rest-framework-serializer-extensions
[djangorestframework-queryfields]: http://djangorestframework-queryfields.readthedocs.io/
[drf-writable-nested]: http://github.com/Brogency/drf-writable-nested
