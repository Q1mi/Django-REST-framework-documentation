
# 序列化器字段

> Form类中的每个字段不仅负责验证数据，还负责“清理”数据-将其标准化为一致的格式。
>
> &mdash; [Django文档][cite]

序列化器字段处理原始值和内部数据类型之间的转换。它们还处理验证输入值，以及从其父对象检索和设置值。

---

**注意** 序列化器字段是在`fields.py`中声明的, 但是按照惯例你应该使用 `from rest_framework import serializers`导入它们并将字段称为`serializers.<FieldName>`。

---

## 核心参数

每个序列化程序字段类构造函数都至少接受这些参数。某些Field类采用其他特定于字段的参数，但应始终接受以下内容：

### `read_only`

只读字段包含在API输出中，但在创建或更新操作期间不应包含在输入中。错误地包含在串行器输入中的所有`read_only`字段都将被忽略。

进行设置`True`以确保序列化表示形式时使用该字段，而在反序列化期间创建或更新实例时不使用该字段。

默认为`False`

### `write_only`

进行设置`True`以确保在更新或创建实例时可以使用该字段，但在序列化表示形式时不包括该字段。

默认为`False`

### `required`

如果反序列化期间未提供字段，通常会引发错误。如果反序列化过程中不需要此字段，则设置为false。

将此设置为`False`还可以在序列化实例时从输出中省略对象属性或字典键。如果不存在该密钥，则它将不会直接包含在输出表示中。

默认为`True`。

### `allow_null`

如果`None`传递给序列化器字段，通常会引发错误。将此关键字参数设置为`True`if `None`应该被视为有效值。

默认为`False`

### `default`

如果设置，则给出默认值，如果没有提供输入值，该默认值将用于该字段。如果未设置，则默认行为是根本不填充该属性。

该`default`过程中部分更新操作不适用。在部分更新的情况下，仅传入数据中提供的字段将返回经过验证的值。

可以设置为一个函数或其他可调用的，在这种情况下，每次使用该值时都会对其求值。调用时，它将不接收任何参数。如果callable具有`set_context`方法，则每次在将字段实例作为唯一参数获取值之前都会被调用。这与[验证器](validators.md#using-set_context)的工作方式相同。

请注意，设置`default`值意味着该字段不是必需的。同时包含`default`和`required`关键字参数都是无效的，并且会引发错误。

### `source`

将用于填充字段的属性的名称。可以是仅接受self参数的方法，例如`URLField(source='get_absolute_url')`，也可以使用点分符号遍历属性，例如`EmailField(source='user.email')`。

该值`source='*'`具有特殊含义，用于指示应将整个对象传递给该字段。这对于创建嵌套表示或对于需要访问完整对象才能确定输出表示的字段很有用。

默认为字段名称。

### `validators`

验证器功能列表，应将其应用于输入字段输入，并引发验证错误或简单地返回。验证器函数通常应该提高`serializers.ValidationError`，但是`ValidationError`还支持Django的内置函数，以便与Django代码库或第三方Django软件包中定义的验证器兼容。

### `error_messages`

错误代码到错误消息的字典。

### `label`

一个简短的文本字符串，可用作HTML表单字段或其他描述性元素中的字段名称。

### `help_text`

可用作在HTML表单字段或其他描述性元素中对该字段进行描述的文本字符串。

### `initial`

该值应用于预先填充HTML表单字段的值。您可以将callable传递给它，就像处理任何常规Django一样`Field`：

    import datetime
    from rest_framework import serializers
    class ExampleSerializer(serializers.Serializer):
        day = serializers.DateField(initial=datetime.date.today)

### `style`

键值对字典，可用于控制渲染器应如何渲染字段。

这里的两个例子是`'input_type'`和`'base_template'`：

    # Use <input type="password"> for the input.
    password = serializers.CharField(
        style={'input_type': 'password'}
    )

    # Use a radio input instead of a select input.
    color_channel = serializers.ChoiceField(
        choices=['red', 'green', 'blue'],
        style={'base_template': 'radio.html'}
    )

有关更多详细信息，请参见[HTML & Forms][html-and-forms]文档。

---

# 布尔字段

## BooleanField

表示布尔值

使用HTML编码的表单输入时，请注意，即使省略了值，也将始终将其设置为`False`，即使该字段`default=True`指定了选项。这是因为HTML复选框输入通过忽略该值来表示未选中状态，因此REST框架将忽略视为其为空复选框输入。

对应于`django.db.models.fields.BooleanField`.

**签名：** `BooleanField()`

## NullBooleanField

表示布尔值，也接受`None`为有效值。

对应于`django.db.models.fields.NullBooleanField`.

**签名：** `NullBooleanField()`

---

# 字符串字段

## CharField

表示文本。（可选）验证文本是否短于`max_length`和长于`min_length`。

对应于`django.db.models.fields.CharField`或`django.db.models.fields.TextField`.

**签名：** `CharField(max_length=None, min_length=None, allow_blank=False, trim_whitespace=True)`

- `max_length` - 验证输入的字符数不超过此数量。
- `min_length` - 验证输入的字符数不少于此。
- `allow_blank` - 如果设置为`True`，则应将空字符串视为有效值。如果设置为，`False`则认为空字符串无效，并将引发验证错误。默认为`False`。
- `trim_whitespace` - 如果设置为，`True`则修剪前后的空白。默认为`True`。

该`allow_null`选项也可用于字符串字段，尽管不建议使用`allow_blank`。设置`allow_blank=True`和`allow_null=True`都是有效的，但是这样做意味着对于字符串表示而言，将存在两种不同类型的空值，这可能导致数据不一致和细微的应用程序错误。

## EmailField

表示文本形式，将文本验证为有效的电子邮件地址。

对应于`django.db.models.fields.EmailField`

**签名：** `EmailField(max_length=None, min_length=None, allow_blank=False)`

## RegexField

表示文本形式，用于验证给定值是否与某个正则表达式匹配。

对应于`django.forms.fields.RegexField`.

**签名：** `RegexField(regex, max_length=None, min_length=None, allow_blank=False)`

强制`regex`参数可以是字符串，也可以是已编译的python正则表达式对象。

使用Django的`django.core.validators.RegexValidator`进行验证。

## SlugField

一个验证输入满足表达式`[a-zA-Z0-9_-]+`的`RegexField`字段。

对应于`django.db.models.fields.SlugField`.

**签名：** `SlugField(max_length=50, min_length=None, allow_blank=False)`

## URLField

一个验证输入满足URL格式的`RegexField`字段。要求使用以下格式的完全限定网址`http://<host>/<path>`.

对应于 `django.db.models.fields.URLField`。 使用Django的 `django.core.validators.URLValidator`进行验证。

**签名：** `URLField(max_length=200, min_length=None, allow_blank=False)`

## UUIDField

确保输入为有效UUID字符串的字段。该`to_internal_value`方法将返回一个`uuid.UUID`实例。在输出时，该字段将返回标准连字符格式的字符串，例如：

    "de305d54-75b4-431b-adb2-eb6b9e546013"

**签名：** `UUIDField(format='hex_verbose')`

- `format`: 确定uuid值的表示格式
    - `'hex_verbose'` - 规范的十六进制表示形式，包括连字符： `"5ce0e9a5-5ffa-654b-cee0-1238041fb31a"`
    - `'hex'` - UUID的紧凑十六进制表示形式，不包括连字符： `"5ce0e9a55ffa654bcee01238041fb31a"`
    - `'int'` - UUID的128位整数表示： `"123456789012312313134124512351145145114"`
    - `'urn'` - UUID的RFC 4122 URN表示形式： `"urn:uuid:5ce0e9a5-5ffa-654b-cee0-1238041fb31a"`
  更改`format`参数仅影响表示形式值。 所有格式均被`to_internal_value`接受。

## FilePathField

一个字段，其选择仅限于文件系统上某个目录中的文件名

对应于 `django.forms.fields.FilePathField`.

**签名：** `FilePathField(path, match=None, recursive=False, allow_files=True, allow_folders=False, required=None, **kwargs)`

- `path` - 目录的绝对文件系统路径，应从中选择此FilePathField。
- `match` - FilePathField将用于过滤文件名的正则表达式（作为字符串）。
- `recursive` - 指定是否应包含路径的所有子目录。默认值为`False`。
- `allow_files` - 指定是否应包含指定位置的文件。默认值为`True`。这个参数和下面的 `allow_folders` 必须有一个为 `True`。
- `allow_folders` - 指定是否应包含指定位置的文件夹。默认值为`False`。 这个参数和上面的 `allow_files` 必须有一个为 `True`.

## IPAddressField

确保输入为有效IPv4或IPv6字符串的字段。

对应于 `django.forms.fields.IPAddressField` 和 `django.forms.fields.GenericIPAddressField`.

**签名：**: `IPAddressField(protocol='both', unpack_ipv4=False, **options)`

- `protocol` 将有效输入限制为指定的协议。接受的值是“两个”（默认），“ IPv4”或“ IPv6”。匹配不区分大小写。
- `unpack_ipv4` 解压缩IPv4映射的地址，如:: ffff：192.0.2.1。如果启用此选项，则该地址将解压缩为192.0.2.1。默认设置为禁用。只能在协议设置为“ both”时使用。

---

# Numeric fields

## IntegerField

表示整数

对应于 `django.db.models.fields.IntegerField`, `django.db.models.fields.SmallIntegerField`, `django.db.models.fields.PositiveIntegerField` 和 `django.db.models.fields.PositiveSmallIntegerField`。

**签名**: `IntegerField(max_value=None, min_value=None)`

- `max_value` 验证提供的数字不大于此值。
- `min_value` 验证提供的数字不小于此值。

## FloatField

表示浮点

对应于 `django.db.models.fields.FloatField`.

**Signature**: `FloatField(max_value=None, min_value=None)`

- `max_value` 验证提供的数字不大于此值。
- `min_value` 验证提供的数字不小于此值。

## DecimalField

表示十进制形式，在Python中由Decimal实例表示。

对应于 `django.db.models.fields.DecimalField`。

**签名**: `DecimalField(max_digits, decimal_places, coerce_to_string=None, max_value=None, min_value=None)`

- `max_digits` 数字中允许的最大位数。它必须是`None`或大于或等于的整数`decimal_places`。
- `decimal_places` 与数字一起存储的小数位数。
- `coerce_to_string` 如果表示形式应该返回字符串值就设置为`True`, 或者应该返回`Decimal`对象的话就设置为 `False`。除非覆盖，否则默认为设置中`COERCE_DECIMAL_TO_STRING`设置键相同值的`True`值。 如果`Decimal`对象由序列化程序返回，则最终输出格式将由渲染器确定。请注意，设置`localize`会将值强制设为`True`。
- `max_value` 验证提供的数字不大于此值。
- `min_value` 验证提供的数字不小于此值。
- `localize` 设置为`True`启用以基于当前语言环境本地化输入和输出。这也将迫使`coerce_to_string`到`True`。默认为`False`。请注意，如果你`USE_L10N=True`在设置文件中进行了设置，则会启用数据格式设置。

#### 用法示例

要验证最大为999且分辨率为2位小数的数字，请使用：

    serializers.DecimalField(max_digits=5, decimal_places=2)

并使用十进制小数位数来验证小于十亿的数字：

    serializers.DecimalField(max_digits=19, decimal_places=10)

该字段还带有一个可选参数`coerce_to_string`。如果设置为`True`表示将作为字符串输出。如果设置为`False`，表示将作为`Decimal`实例保留，最终表示将由渲染器确定。

如果未设置，则默认为与`COERCE_DECIMAL_TO_STRING`设置相同的值，`True`除非另行设置。

*此处原与文无关，2019.12.14日q1mi翻译于北京。*

---

# Date and time fields

## DateTimeField

表示日期和时间。

对应于 `django.db.models.fields.DateTimeField`.

**签名：** `DateTimeField(format=api_settings.DATETIME_FORMAT, input_formats=None)`

* `format` - 代表输出格式的字符串。如果未指定，则默认为与`DATETIME_FORMAT`设置键相同的值，`'iso-8601'`除非设置，否则为默认值。设置为格式字符串表示`to_representation`应将返回值强制为字符串输出。格式字符串如下所述。将此值设置为`None`表示应由`to_representation`返回Python `datetime`对象。在这种情况下，日期时间编码将由渲染器确定。
* `input_formats` - 表示可用于解析日期的输入格式的字符串列表。如果未指定，`DATETIME_INPUT_FORMATS`将使用该设置，默认为`['iso-8601']`。

#### `DateTimeField` format strings.

格式字符串可以是显式指定格式的[Python strftime formats][strftime]，也可以是特殊字符串`'iso-8601'`，它指示应使用[ISO 8601][iso8601]样式的日期时间。（例如`'2013-01-29T12:34:56.000000Z'`）

当将值`None`用于格式时，`datetime`对象将由`to_representation`返回，并且最终输出表示形式将由renderer类确定。

#### `auto_now` and `auto_now_add` model fields.

使用`ModelSerializer`或`HyperlinkedModelSerializer`，请注意，默认情况下带有`auto_now=True`或任何模型字段`auto_now_add=True`都将使用`read_only=True`的序列化器字段。

如果要覆盖此行为，则需要`DateTimeField`在序列化程序上显式声明。例如：

    class CommentSerializer(serializers.ModelSerializer):
        created = serializers.DateTimeField()

        class Meta:
            model = Comment

## DateField

表示日期

对应于 `django.db.models.fields.DateField`

**签名：** `DateField(format=api_settings.DATE_FORMAT, input_formats=None)`

* `format` - 代表输出格式的字符串。如果未指定，则默认为与`DATE_FORMAT`设置键相同的值，除非设置，否则默认值为`'iso-8601'`。设置为格式字符串表示`to_representation`应将返回值强制为字符串输出。格式字符串如下所述。将此值设置为`None`表示Python的`date`对象应由`to_representation`返回。在这种情况下，时间编码将由渲染器确定。
* `input_formats` - 表示可用于解析日期的输入格式的字符串列表。如果未指定，`DATE_INPUT_FORMATS`将使用该设置，默认为`['iso-8601']`。

#### `DateField` 格式化字符串

格式字符串可以是显式指定格式的[Python strftime formats][strftime]，也可以是特殊字符串`'iso-8601'`，它指示应使用[ISO 8601][iso8601]样式时间。（例如`'2013-01-29'`）

## TimeField

表示时间

对应于 `django.db.models.fields.TimeField`

**签名：** `TimeField(format=api_settings.TIME_FORMAT, input_formats=None)`

* `format` - 代表输出格式的字符串。如果未指定，则默认为与`TIME_FORMAT`设置键相同的值，`'iso-8601'`除非设置，否则为默认值。设置为格式字符串表示`to_representation`应将返回值强制为字符串输出。格式字符串如下所述。将此值设置为`None`表示Python `time`对象应由`to_representation`返回。在这种情况下，时间编码将由渲染器确定。
* `input_formats` - 表示可用于解析日期的输入格式的字符串列表。如果未指定，`TIME_INPUT_FORMATS`将使用该设置，默认为`['iso-8601']`

#### `TimeField` format strings

格式字符串可以是显式指定格式的[Python strftime 格式][strftime]，也可以是特殊字符串`'iso-8601'`，它指示应使用[ISO 8601][iso8601]样式时间。（例如`'12:34:56.000000'`）

## DurationField

表示时间间隔
对应于 `django.db.models.fields.DurationField`

在这些字段的`validated_data`将包含一个`datetime.timedelta`实例。该表示形式是遵循此格式的字符串`'[DD] [HH:[MM:]]ss[.uuuuuu]'`。

**注意：** 此字段仅适用于Django版本> = 1.8。

**签名：** `DurationField()`

---

# 选择字段

## ChoiceField

可以接受有限选择集中的值的字段。

`ModelSerializer`如果相应的模型字段包含`choices=…`参数，则用于自动生成字段。

**签名：** `ChoiceField(choices)`

- `choices` - 有效值列表或`(key, display_name)`元组列表。
- `allow_blank` - 如果设置为`True`，则应将空字符串视为有效值。如果设置为，`False`则认为空字符串无效，并将引发验证错误。默认为`False`。
- `html_cutoff` - 如果设置，则将是HTML select下拉列表将显示的最大选择数。可以用来确保自动生成的ChoiceFields具有非常大的选择范围，不会阻止模板的呈现。默认为`None`。
- `html_cutoff_text` - 如果设置了此选项，则如果在HTML选择下拉列表中已截断最大数量的项目，则它将显示文本指示器。默认为`"More than {count} items…"`。

无论是`allow_blank`与`allow_null`上有效的选项`ChoiceField`，但我们强烈建议您只使用一个，而不是两个。`allow_blank`应该首选用于文本选择，并且`allow_null`应该首选用于数字或其他非文本选择。

## MultipleChoiceField

一个可以接受一组零个，一个或多个值的字段，这些值是从一组有限的选择中选择的。接受一个强制性参数。`to_internal_value`返回`set`包含所选值的。

**签名：** `MultipleChoiceField(choices)`

- `choices` - 有效值列表或`(key, display_name)`元组列表。
- `allow_blank` - 如果设置为`True`，则应将空字符串视为有效值。如果设置为，`False`则认为空字符串无效，并将引发验证错误。默认为`False`。
- `html_cutoff` - 如果设置，则将是HTML select下拉列表将显示的最大选择数。可以用来确保自动生成的ChoiceFields具有非常大的选择范围，不会阻止模板的呈现。默认为`None`。
- `html_cutoff_text` - 如果设置了此选项，则如果在HTML选择下拉列表中已截断最大数量的项目，则它将显示文本指示器。默认为`"More than {count} items…"`。

与`ChoiceField`一样，`allow_blank`和`allow_null`选项都有效，尽管强烈建议您仅使用一个，而不要同时使用。`allow_blank`应该首选用于文本选择，并且`allow_null`应该首选用于数字或其他非文本选择。

---

# 文件上传字段

#### Parsers and file uploads.

`FileField` 和 `ImageField`类是仅适用于使用`MultiPartParser`或`FileUploadParser`的。大多数解析器（例如JSON）不支持文件上传。Django的常规[FILE_UPLOAD_HANDLERS]用于处理上传的文件。

## FileField

表示文件。执行Django的标准FileField验证。

对应于 `django.forms.fields.FileField`.

**签名：** `FileField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL)`

 - `max_length` - 指定文件名的最大长度。
 - `allow_empty_file` - 指定是否允许空文件。
- `use_url` - 如果设置为`True`则URL字符串值将用于输出表示。如果设置为`False`则文件名字符串值将用于输出表示。默认为`UPLOADED_FILES_USE_URL`设置键的值，除非另有设置，否则为默认值`True`。

## ImageField

表示图片。 验证上传的文件内容是否与已知图像格式匹配。

对应于 `django.forms.fields.ImageField`.

**签名：** `ImageField(max_length=None, allow_empty_file=False, use_url=UPLOADED_FILES_USE_URL)`

 - `max_length` - 指定文件名的最大长度。
 - `allow_empty_file` - 指定是否允许空文件。
- `use_url` - 如果设置为`True`则URL字符串值将用于输出表示。如果设置为`False`则文件名字符串值将用于输出表示。默认为`UPLOADED_FILES_USE_URL`设置键的值，除非另有设置，否则为默认值`True`。

需要安装`Pillow`包或`PIL`包。推荐使用`Pillow`包，因为`PIL`包已经不积极维护。

---

# 复合字段

## ListField

验证对象列表的字段类。

**签名**: `ListField(child, min_length=None, max_length=None)`

- `child` - 应该用于验证列表中对象的字段实例。如果未提供此参数，则将不验证列表中的对象。
- `min_length` - 验证列表中包含的元素不少于此数量。
- `max_length` - 验证列表中所包含的元素数量不超过此数量。

例如，要验证整数列表，可以使用如下所示的内容：

    scores = serializers.ListField(
       child=serializers.IntegerField(min_value=0, max_value=100)
    )

`ListField`类还支持声明式样式，该样式允许你编写可重用的列表字段类。

    class StringListField(serializers.ListField):
        child = serializers.CharField()

现在，我们可以在整个应用程序中重用自定义的`StringListField`类，而不必为其提供`child`参数。

## DictField

一个验证对象字典的字段类。 `DictField` 中的key都总是假定为字符串值。

**签名**: `DictField(child)`

- `child` - 应该用于验证字典中值的字段实例。如果未提供此参数，则将不验证映射中的值。

例如，要创建一个验证字符串到字符串映射的字段，你可以编写如下代码：

    document = DictField(child=CharField())

你也可以像使用一样使用声明式样式`ListField`。例如：

    class DocumentField(DictField):
        child = CharField()

## JSONField

一个字段类，用于验证传入的数据结构是否包含有效的JSON原语。在其备用二进制模式下，它将表示并验证JSON编码的二进制字符串。

**签名**: `JSONField(binary)`

- `binary` - 如果设置为`True`则该字段将输出并验证JSON编码的字符串，而不是原始数据结构。默认为`False`。

---

# Miscellaneous fields

## ReadOnlyField

一个字段类，仅返回该字段的值而无需修改。

`ModelSerializer`当包含与属性而不是模型字段相关的字段名称时，默认情况下使用此字段。

**签名**: `ReadOnlyField()`

例如，如果`has_expired`是`Account`模型的属性，则以下序列化器将自动将其生成为`ReadOnlyField`：

    class AccountSerializer(serializers.ModelSerializer):
        class Meta:
            model = Account
            fields = ('id', 'account_name', 'has_expired')

## HiddenField

一个字段类，它不基于用户输入获取值，而是从默认值或可调用对象获取其值。

**签名**: `HiddenField()`

例如，要包括一个始终提供当前时间的字段作为序列化程序验证数据的一部分，则可以使用以下内容：

    modified = serializers.HiddenField(default=timezone.now)

`HiddenField`通常只有在你需要基于某些预先提供的字段值来运行某些验证，而又不想将所有这些字段公开给最终用户时，才需要使用该类。

有关 `HiddenField` 的更多示例，请参照[validators](validators.md) 文档。

## ModelField

可以绑定到任意模型字段的通用字段。 `ModelField`类将序列化/反序列化任务委托给与其关联的model字段。 此字段可用于为自定义模型字段创建序列化程序字段，而不必创建新的自定义序列化程序字段。

该字段用于`ModelSerializer`对应于自定义模型字段类。

**签名:** `ModelField(model_field=<Django ModelField instance>)`


`ModelField`一般用于内部使用，但如果需要的话可以通过你的API使用。为了正确地实例化`ModelField`，必须传递一个附加到实例化模型的字段。例如：`ModelField(model_field=MyModel()._meta.get_field('custom_field'))`。

## SerializerMethodField

这是一个只读字段。它通过在附加的序列化器类上调用一个方法来获取其值。它可以用于将任何类型的数据添加到对象的序列化表示中。

**签名**: `SerializerMethodField(method_name=None)`

- `method_name` - 要调用的序列化程序上的方法的名称。如果未包括，则默认为`get_<field_name>`。

`method_name`参数引用的序列化程序方法应接受单个参数（除了之外`self`），该参数是要序列化的对象。它应该返回要包含在对象的序列化表示中的任何内容。例如：

    from django.contrib.auth.models import User
    from django.utils.timezone import now
    from rest_framework import serializers

    class UserSerializer(serializers.ModelSerializer):
        days_since_joined = serializers.SerializerMethodField()

        class Meta:
            model = User

        def get_days_since_joined(self, obj):
            return (now() - obj.date_joined).days

---

# 自定义字段

如果要创建自定义字段，则需要继承`Field`类，然后重写`.to_representation()`和`.to_internal_value()`方法之一或二者。这两种方法用于在初始数据类型和原始的可序列化数据类型之间进行转换。基本数据类型通常是任何数字，字符串，布尔，`date/time/datetime`或`None`。它们也可以是仅包含其他原始对象的任何列表或字典之类的对象。根据您使用的渲染器，可能支持其他类型。

`.to_representation()` 调用该方法可将初始数据类型转换为原始的可序列化数据类型。

`to_internal_value()`调用该方法可将原始数据类型恢复为其内部python表示形式。如果数据无效，则此方法应抛出一个 `serializers.ValidationError`。

请注意，`WritableField`版本2.x中存在的类不再存在。如果该字段支持数据输入，则应继承`Field`并重写`to_internal_value()`。

## 示例

让我们看一个序列化代表RGB颜色值的类的示例：

    class Color(object):
        """
        A color represented in the RGB colorspace.
        """
        def __init__(self, red, green, blue):
            assert(red >= 0 and green >= 0 and blue >= 0)
            assert(red < 256 and green < 256 and blue < 256)
            self.red, self.green, self.blue = red, green, blue

    class ColorField(serializers.Field):
        """
        Color objects are serialized into 'rgb(#, #, #)' notation.
        """
        def to_representation(self, obj):
            return "rgb(%d, %d, %d)" % (obj.red, obj.green, obj.blue)

        def to_internal_value(self, data):
            data = data.strip('rgb(').rstrip(')')
            red, green, blue = [int(col) for col in data.split(',')]
            return Color(red, green, blue)

默认情况下，字段值被视为映射到对象上的属性。如果需要自定义如何访问和设置字段值，则需要覆盖`.get_attribute()`和/或`.get_value()`。

例如，让我们创建一个字段，该字段可用来表示要序列化的对象的类名：

    class ClassNameField(serializers.Field):
        def get_attribute(self, obj):
            # We pass the object instance onto `to_representation`,
            # not just the field attribute.
            return obj

        def to_representation(self, obj):
            """
            Serialize the object's class name.
            """
            return obj.__class__.__name__

#### Raising validation errors

我们上面的`ColorField`类目前不执行任何数据验证。
为了指示无效数据，我们应该引发一个`serializers.ValidationError`，如下所示：

    def to_internal_value(self, data):
        if not isinstance(data, six.text_type):
            msg = 'Incorrect type. Expected a string, but got %s'
            raise ValidationError(msg % type(data).__name__)

        if not re.match(r'^rgb\([0-9]+,[0-9]+,[0-9]+\)$', data):
            raise ValidationError('Incorrect format. Expected `rgb(#,#,#)`.')

        data = data.strip('rgb(').rstrip(')')
        red, green, blue = [int(col) for col in data.split(',')]

        if any([col > 255 or col < 0 for col in (red, green, blue)]):
            raise ValidationError('Value out of range. Must be between 0 and 255.')

        return Color(red, green, blue)

`.fail()`方法是引发的快捷方式`ValidationError`，它从`error_messages`字典中获取消息字符串。例如：

    default_error_messages = {
        'incorrect_type': 'Incorrect type. Expected a string, but got {input_type}',
        'incorrect_format': 'Incorrect format. Expected `rgb(#,#,#)`.',
        'out_of_range': 'Value out of range. Must be between 0 and 255.'
    }

    def to_internal_value(self, data):
        if not isinstance(data, six.text_type):
            self.fail('incorrect_type', input_type=type(data).__name__)

        if not re.match(r'^rgb\([0-9]+,[0-9]+,[0-9]+\)$', data):
            self.fail('incorrect_format')

        data = data.strip('rgb(').rstrip(')')
        red, green, blue = [int(col) for col in data.split(',')]

        if any([col > 255 or col < 0 for col in (red, green, blue)]):
            self.fail('out_of_range')

        return Color(red, green, blue)

这种样式使您的错误消息与代码更清晰地分开，因此应首选。

# 第三方包

以下第三方软件包也可使用。

## DRF Compound Fields

[drf-compound-fields][drf-compound-fields]包提供了“复合”序列化器字段，例如简单值列表，可以由其他字段来描述，而不是带有`many = True`选项的序列化器。 还提供了用于键入字典和值的字段，这些字段可以是特定类型，也可以是该类型的项目的列表。

## DRF Extra Fields

[drf-extra-fields][drf-extra-fields] 包为REST框架提供了额外的序列化器字段，包括`Base64ImageField`和`PointField`类。

## djangrestframework-recursive

[djangorestframework-recursive][djangorestframework-recursive] 包提供了一个`RecursiveField`，用于序列化和反序列化递归结构

## django-rest-framework-gis

[django-rest-framework-gis][django-rest-framework-gis] 包为django rest框架提供了地理插件，例如`GeometryField`字段和GeoJSON序列化器。

## django-rest-framework-hstore

The [django-rest-framework-hstore][django-rest-framework-hstore] 包提供了一个`HStoreField`来支持 [django-hstore][django-hstore] `DictionaryField` 模型字段。

[cite]: https://docs.djangoproject.com/en/stable/ref/forms/api/#django.forms.Form.cleaned_data
[html-and-forms]: ../topics/html-and-forms.md
[FILE_UPLOAD_HANDLERS]: https://docs.djangoproject.com/en/stable/ref/settings/#std:setting-FILE_UPLOAD_HANDLERS
[ecma262]: http://ecma-international.org/ecma-262/5.1/#sec-15.9.1.15
[strftime]: https://docs.python.org/3/library/datetime.html#strftime-and-strptime-behavior
[django-widgets]: https://docs.djangoproject.com/en/stable/ref/forms/widgets/
[iso8601]: http://www.w3.org/TR/NOTE-datetime
[drf-compound-fields]: https://drf-compound-fields.readthedocs.io
[drf-extra-fields]: https://github.com/Hipo/drf-extra-fields
[djangorestframework-recursive]: https://github.com/heywbj/django-rest-framework-recursive
[django-rest-framework-gis]: https://github.com/djangonauts/django-rest-framework-gis
[django-rest-framework-hstore]: https://github.com/djangonauts/django-rest-framework-hstore
[django-hstore]: https://github.com/djangonauts/django-hstore
