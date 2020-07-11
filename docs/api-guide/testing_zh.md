# 测试
## APIRequesFactory
延申django本身存在的`RequestFactory`类

### 创建测试请求
这个`APIRequestFactory`类支持与Django标准`RequestFactory`类几乎相同的API。这意味着类似`.get()`，`.post()`，`.put()`，`.patch()`，`delete()`，`.head()`和`.options()`等标准方法全部支持。
```python
from rest_framework.test import APIRequestFactory

# 使用标准的RequestFactory API去创建从POST来的请求
factory = APIRequestFactory()
request = factory.post('/notes/', {'title': 'new idea'})
```

#### 使用`format`参数
创建请求主体的方法，例如`post`，`put`和`patch`，包含`format`，这使得使用多部分表单数据以外的内容类型轻松生成请求。例如：
```python
# 创建一个JSON POST请求
factory = APIRequestFactory()
request = factory.post('/notes/', {'title': 'new idea'}, format='json')
```

默认情况下，可用格式为`multipart`和`json`。为了与Django现有的`RequestFactory`，默认格式为`multipart`。
要支持更多的请求格式集，或更改默认格式，请参见配置部分。

#### 明确编码请求主体
如果需要明确编码请求主体，可以通过设置`content_type`标志，例如：
```python
request = factory.post('/notes/'， json.dumps({'title': 'new idea'}), content_type='application/json')
```

#### 使用表格数据进行PUT和PATCH
值得注意的是Django的`RequestFactory`和REST framework的`APIRequestFactory`之间的区别是，将为除`.post()`之外的方法编码多部份表单数据。
例如，使用`APIRequestFactory`，可以像这样发送表单PUT请求：

```python
factory = APIRequestFactory()
request = factory.put('/notes/547/', {'title': 'remember to email dave'})
```
使用Django的`RequestFactory`，需要自己对数据进行显式编码：
```python
from django.test.client import encode_multipart, RequestFactory

factory = RequestFactory()
data = {'title': 'remember to email dave'}
content = encode_multipart('BoUnDaRyStRiNg', data)
content_type = 'multipart/form-data; boundary=BoUnDaRyStRiNg'
request = factory.put('/notes/547/', content, content_type=content_type)
```

### 强制验证
当测试视图直接使用请求工厂时，通常能够方便的直接对请求进行身份验证，而不必构造正确的身份验证凭据。

要强制验证请求，请使用`force_authenticate()`方法。

```python
from rest_framework.test import force_authenticate

factory = APIRequestFactory()
user = User.objects.get(username='olivia')
view = AccountDetail.as_view()

# 让一个验证请求到view中。。。
request = factory.get('/accounts/django-superstars/')
force_authenticate(request, user=user)
response = view(request)
```
这个方法为`force_authenticate(request, user=None, token=None)`。当调用时，可以设置user和token其中一个。

例如，当使用token强制进行身份验证时，你可以执行以下操作：

```python
user = User.objects.get(username='olivia')
request = factory.get('/accounts/django-superstars/')
force_authenticate(request, user=user, token=user.token)
```
***
**Note**：使用`APIRequestFactory`时，返回的对象是Django标准的`HttpRequest`，而不是REST framework的`Request`对象，后者仅在调用view后才生成。
这意味着直接在请求对象上设置属性可能并不总是具有预期的效果。 例如，直接设置`.token`将无效，而直接设置`.user`仅在使用会话身份验证时才有效。
```python
# 只有在使用SessionAuthentication时，请求才会进行身份验证。
request = factory.get('/accounts/django-superstars/')
request.user = user
response = view(request)
```
***

### 强制CSRF验证
默认情况下，使用`APIRequestFactory`创建的请求在传递到REST framework视图时不会应用CSRF验证。如果需要显式打开CSRF验证，则可以通过在实例化工厂时设置force_csrf_checks标志来实现。
```python
factory = APIRequestFactory(enforce_csrf_checks=True)
```
***
**Note**：值得注意的是，Django的标准`RequestFactory`不需要包含此选项，因为在使用常规Django时，CSRF验证在中间件中进行，而中间件在直接测试视图时不会运行。 使用REST framework时，CSRF验证在视图内部进行，因此请求工厂需要禁用视图级CSRF检查。
***

## APIClient
扩展Django现有的Client类。

### 创建请求
`APIClient`类支持与Django的标准`Client`类相同的请求接口。 这意味着标准的`.get()`、`.post()`、`.put()`、`.patch()`、`.delete()`、`.head()`和`.options()`方法都可用。 例如：

```python
from rest_framework.test import APIClient

client = APIClient()
client.post('/notes/', {'title': 'new idea'}, format='json')
```
要支持更多的请求格式集，或更改默认格式，请参见配置部分。

### 验证

#### .login(**kwargs)
`login`方法的功能与Django常规`Client`类的功能完全相同。 这使您可以针对任何包含`SessionAuthentication`的视图对请求进行身份验证。

```python
# 在已登录会话的上下文中发出所有请求。
client = APIClient()
client.login(username='lauren', password='secret')
```
需要注销，请照常调用注销方法。
```python
# 注销
client.logout()
```
`login`方法适用于测试使用会话身份验证的API，例如包含与API进行AJAX交互的网站。

#### .credentials(**kwargs)
`credentials`方法可用于设置请求头，然后测试客户端会将其包含在所有后续请求中。
```python
from rest_framework.authtoken.models import Token
from rest_framework.test import APIClient

# 在所有请求上都包含适当的Authorization头。
token = Token.objects.get(user__username='lauren')
client = APIClient()
client.credentials(HTTP_AUTHORIZATION='Token ' + token.key)
```
请注意，第二次调用`credentials`将覆盖所有现有credentials。 您可以通过不带任何参数的方法来取消设置任何现有的credentials。

#### .force_authenticate(user=None, token=None)
有时您可能想绕过身份验证，并简单地强制将测试客户端的所有请求自动视为已身份验证。
如果您正在测试API但又不想构造有效的身份验证凭据来发出测试请求，则这可能是一个有用的快捷方式。
```python
user = User.objects.get(username='lauren')
client = APIClient()
client.force_authenticate(user=user)
```
要取消对后续请求的身份验证，请调用`force_authenticate`将用户和/或令牌设置为`None`。
```python
client.force_authenticate(user=None)
```

### CSRF验证
默认情况下，使用`APIClient`时不应用CSRF验证。 如果需要显式启用CSRF验证，则可以通过在实例化客户端时设置`force_csrf_checks`来实现。
```python
client = APIClient(enforce_csrf_checks=True)
```
与往常一样，CSRF验证仅适用于任何经过会话身份验证的视图。 这意味着仅当通过调用`login()`登录客户端后，CSRF验证才会发生。

## RequestsClient
REST framework还包括一个客户端，用于使用流行的Python库`requests`与应用程序进行交互。这在以下情况下可能有用：
- 您期望主要通过另一个Python服务与API交互，并希望在与客户端看到的相同的级别上测试该服务。
- 您希望以这样的方式编写测试，使其也可以在临时环境或实时环境中运行。 （请参见下面的“实时测试”。）
这将显示与您直接使用请求会话完全相同的界面。
```python
client = RequestsClient()
response = client.get('http://testserver/users/')
assert response.status_code == 200
```
请注意，请求客户端要求您传递完全限定的URL。

### `RequestsClient`和使用数据库
如果您要编写仅与服务接口交互的测试，则`RequestsClient`类很有用。 这比使用标准Django测试客户端要严格一些，因为这意味着所有交互都应通过API进行。
如果使用的是`RequestsClient`，则需要确保测试设置和结果断言是作为常规API调用执行的，而不是直接与数据库模型进行交互。 例如，您不必列出`Customer.objects.count（）== 3`，而是列出客户端点，并确保它包含三个记录。

### 请求头&认证
可以使用与使用标准`request.Session`实例相同的方式来提供自定义请求头和身份验证凭据。
```python
from requests.auth import HTTPBasicAuth

client.auth = HTTPBasicAuth('user', 'pass')
client.headers.update({'x-test': 'true'})
```

### CSRF
如果您使用的是`SessionAuthentication`，则需要为所有`POST`，`PUT`，`PATCH`或`DELETE`请求包括一个CSRF令牌。
您可以按照基于JavaScript的客户端将使用的相同流程进行操作。 首先发出`GET`请求以获得CRSF令牌，然后在随后的请求中显示该令牌。
例如...
```python
client = RequestsClient()

# 获取CSRF token.
response = client.get('/homepage/')
assert response.status_code == 200
csrftoken = response.cookies['csrftoken']

# 与API交互
response = client.post('/organisations/', json={
    'name': 'MegaCorp',
    'status': 'active'
}, headers={'X-CSRFToken': csrftoken})
assert response.status_code == 200
```

### 现场测试
通过谨慎使用`RequestsClient`和`CoreAPIClient`都可以编写可以在开发中运行或直接在登台服务器或生产环境中运行的测试用例。
使用这种样式创建一些核心功能的基本测试是验证实时服务的有效方法。 这样做可能需要仔细注意设置和卸载，以确保测试以不直接影响客户数据的方式运行。

## CoreAPIClient
CoreAPIClient允许您使用Python `coreapi` 客户端库与API进行交互。
```python
# 提取API模式
client = CoreAPIClient()
schema = client.get('http://testserver/schema/')

# 创建一个新的organisation
params = {'name': 'MegaCorp', 'status': 'active'}
client.action(schema, ['organisations', 'create'], params)

# 确保列表中存在该组织
data = client.action(schema, ['organisations', 'list'])
assert(len(data) == 1)
assert(data == [{'name': 'MegaCorp', 'status': 'active'}])
```

### 请求头&认证
自定义请求头和身份验证可用于跟`RequestsClient`相似的`CoreAPIClient`一起使用。
```python
from requests.auth import HTTPBasicAuth

client = CoreAPIClient()
client.session.auth = HTTPBasicAuth('user', 'pass')
client.session.headers.update({'x-test': 'true'})
```

## 测试样例
REST framework包括以下测试用例类，它们反映了现有的Django测试用例类，但是使用`APIClient`而不是Django的默认`Client`。
- APISimpleTestCase
- APITransactionTestCase
- APITestCase
- APILiveServerTestCase

### 实例
您可以像使用常规Django测试用例类那样使用任何REST framework的用例类。` self.client`属性将是一个`APIClient`实例。
```python
from django.urls import reverse
from rest_framework import status
from rest_framework.test import APITestCase
from myproject.apps.core.models import Account

class AccountTests(APITestCase):
    def test_create_account(self):
        """
        确保我们可以创建一个新的帐户对象。
        """
        url = reverse('account-list')
        data = {'name': 'DabApps'}
        response = self.client.post(url, data, format='json')
        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertEqual(Account.objects.count(), 1)
        self.assertEqual(Account.objects.get().name, 'DabApps')
```

## 测试响应
### 检查响应数据
在检查测试响应的有效性时，通常更方便的方法是检查创建响应的数据，而不是检查完全呈现的响应。
例如，检查`response.data`更容易：
```python
response = self.client.get('/users/4/')
self.assertEqual(response.data, {'id': 4, 'username': 'lauren'})
```
而不是检查`response.content`的解析结果：
```python
response = self.client.get('/users/4/')
self.assertEqual(json.loads(response.content), {'id': 4, 'username': 'lauren'})
```

### 渲染响应
如果您直接使用`APIRequestFactory`测试视图，则返回的响应将不会渲染，因为模板响应的渲染是由Django的内部请求-响应周期执行的。 为了访问`response.content`，您首先需要渲染响应。
```python
view = UserDetail.as_view()
request = factory.get('/users/4')
response = view(request, pk='4')
response.render()  # Cannot access `response.content` without this.
self.assertEqual(response.content, '{"username": "lauren", "id": 4}')
```

## 配置
### 设置默认格式
可以使用`TEST_REQUEST_DEFAULT_FORMAT`设置项来设置用于发出测试请求的默认格式。 例如，要在默认情况下始终将JSON用于测试请求而不是标准的多部分表单请求，请在`settings.py`文件中设置以下内容：
```python
REST_FRAMEWORK = {
    ...
    'TEST_REQUEST_DEFAULT_FORMAT': 'json'
}
```
### 设置可用格式
如果您需要使用multipart或json请求之外的其他测试请求，则可以通过设置`TEST_REQUEST_RENDERER_CLASSES`设置来进行。
例如，要增加对在测试请求中使用`format ='html'`的支持，您的`settings.py`文件中可能会有类似的内容。

```python
REST_FRAMEWORK = {
    ...
    'TEST_REQUEST_RENDERER_CLASSES': (
        'rest_framework.renderers.MultiPartRenderer',
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.TemplateHTMLRenderer'
    )
}
```