source: status.py

# Status Codes (状态码)

> 418 I'm a teapot - Any attempt to brew coffee with a teapot should result in the error code "418 I'm a teapot".  The resulting entity body MAY be short and stout.
>
> &mdash; [RFC 2324][rfc2324], Hyper Text Coffee Pot Control Protocol

不建议在响应中使用裸露的状态码。REST框架包含一组可以使用以使代码更加明显和可读的命名常量。

    from rest_framework import status
    from rest_framework.response import Response

    def empty_view(self):
        content = {'please move along': 'nothing to see here'}
        return Response(content, status=status.HTTP_404_NOT_FOUND)

下面列出了 `state` 模块中包含的全套HTTP状态码。

该模块还包括一组辅助函数，用于测试状态码是否在给定范围内。

    from rest_framework import status
	from rest_framework.test import APITestCase

	class ExampleTestCase(APITestCase):
	    def test_url_root(self):
	        url = reverse('index')
	        response = self.client.get(url)
	        self.assertTrue(status.is_success(response.status_code))

有关正确使用HTTP状态代码的更多信息 [RFC 2616][rfc2616] 和 [RFC 6585][rfc6585].

## Informational - 1xx

此类状态代码表示临时响应。默认情况下，REST框架中没有使用1xx状态代码。

    HTTP_100_CONTINUE
    HTTP_101_SWITCHING_PROTOCOLS

## Successful - 2xx

此类状态码表示已成功接收、理解和接受客户端的请求。

    HTTP_200_OK
    HTTP_201_CREATED
    HTTP_202_ACCEPTED
    HTTP_203_NON_AUTHORITATIVE_INFORMATION
    HTTP_204_NO_CONTENT
    HTTP_205_RESET_CONTENT
    HTTP_206_PARTIAL_CONTENT
    HTTP_207_MULTI_STATUS

## Redirection - 3xx

此类状态码表示用户代理需要采取进一步的操作才能完成请求。

    HTTP_300_MULTIPLE_CHOICES
    HTTP_301_MOVED_PERMANENTLY
    HTTP_302_FOUND
    HTTP_303_SEE_OTHER
    HTTP_304_NOT_MODIFIED
    HTTP_305_USE_PROXY
    HTTP_306_RESERVED
    HTTP_307_TEMPORARY_REDIRECT

## Client Error - 4xx

此类状态码用于客户端似乎出错的情况。除了响应HEAD请求时，服务器应该包含一个实体对象，其中包含对错误情况的解释，以及它是临时的还是持续的。

    HTTP_400_BAD_REQUEST
    HTTP_401_UNAUTHORIZED
    HTTP_402_PAYMENT_REQUIRED
    HTTP_403_FORBIDDEN
    HTTP_404_NOT_FOUND
    HTTP_405_METHOD_NOT_ALLOWED
    HTTP_406_NOT_ACCEPTABLE
    HTTP_407_PROXY_AUTHENTICATION_REQUIRED
    HTTP_408_REQUEST_TIMEOUT
    HTTP_409_CONFLICT
    HTTP_410_GONE
    HTTP_411_LENGTH_REQUIRED
    HTTP_412_PRECONDITION_FAILED
    HTTP_413_REQUEST_ENTITY_TOO_LARGE
    HTTP_414_REQUEST_URI_TOO_LONG
    HTTP_415_UNSUPPORTED_MEDIA_TYPE
    HTTP_416_REQUESTED_RANGE_NOT_SATISFIABLE
    HTTP_417_EXPECTATION_FAILED
    HTTP_422_UNPROCESSABLE_ENTITY
    HTTP_423_LOCKED
    HTTP_424_FAILED_DEPENDENCY
    HTTP_428_PRECONDITION_REQUIRED
    HTTP_429_TOO_MANY_REQUESTS
    HTTP_431_REQUEST_HEADER_FIELDS_TOO_LARGE
    HTTP_451_UNAVAILABLE_FOR_LEGAL_REASONS

## Server Error - 5xx

此类响应状态码表示服务器意识到自身出错或无法执行请求的情况。除了响应HEAD请求的时候，服务器应该包含一个实体，其中包含对错误情况的解释，以及它是临时的还是持续的。

    HTTP_500_INTERNAL_SERVER_ERROR
    HTTP_501_NOT_IMPLEMENTED
    HTTP_502_BAD_GATEWAY
    HTTP_503_SERVICE_UNAVAILABLE
    HTTP_504_GATEWAY_TIMEOUT
    HTTP_505_HTTP_VERSION_NOT_SUPPORTED
    HTTP_507_INSUFFICIENT_STORAGE
    HTTP_511_NETWORK_AUTHENTICATION_REQUIRED

## Helper functions （辅助函数）

下列helper函数可用于标识响应代码的类别。

    is_informational()  # 1xx
    is_success()        # 2xx
    is_redirect()       # 3xx
    is_client_error()   # 4xx
    is_server_error()   # 5xx

[rfc2324]: http://www.ietf.org/rfc/rfc2324.txt
[rfc2616]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html
[rfc6585]: http://tools.ietf.org/html/rfc6585
