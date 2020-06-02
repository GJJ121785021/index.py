## 路由

在`views`里创建任意合法名称的`.py`文件，并在其中创建名为 `HTTP` 的类，即可使此文件能够处理对应其相对于 `views` 的路径的 HTTP 请求。

但较为特殊的是名为 `index.py` 的文件，它能够处理以 `/` 作为最后一个字符的 URI。

!!! tip
    由于 Python 规定，模块名称必须由字母、数字与下划线组成，但这种 URI 不友好，所以 Index 会将 URI 中的 `_` 全部替换成 `-` 并做 301 跳转，你可以通过设置 [ALLOW_UNDERLINE](/config/list/#allow_underline) 为真去关闭此功能。

一些例子|文件相对路径|文件能处理的URI
---|---|---
|views/index.py|/
|views/about.py|/about
|views/api/create_article.py|/api/create-article
|views/article/index.py|/article/

`HTTP` 的类应从 `indexpy.http.HTTPView` 继承而来，你可以定义如下方法去处理对应的 HTTP 请求。

- get
- post
- put
- patch
- delete
- head
- options
- trace

这些函数默认不接受任何参数，但你可以使用 `self.request` 去获取此次请求的一些信息。

!!! notice
    注意：这些被用于实际处理 HTTP 请求的函数，无论你以何种方式定义，都会在加载时被改造成异步函数，但为了减少不必要的损耗，尽量使用 `async def` 去定义它们。

## 响应

对于任何正常处理的 HTTP 请求都必须返回一个 `indexpy.http.responses.Response` 对象或者是它的子类对象。

在 `index.http.repsonses` 里内置的可用对象如下：

* [Response](https://www.starlette.io/responses/#response)
* [HTMLResponse](https://www.starlette.io/responses/#htmlresponse)
* [PlainTextResponse](https://www.starlette.io/responses/#plaintextresponse)
* [JSONResponse](https://www.starlette.io/responses/#jsonresponse)
* [RedirectResponse](https://www.starlette.io/responses/#redirectresponse)
* [StreamingResponse](https://www.starlette.io/responses/#streamingresponse)
* [FileResponse](https://www.starlette.io/responses/#fileresponse)

* TemplateResponse

Index 提供了使用 Jinja2 的方法。如下代码将会自动在项目下的 `templates` 目录里寻找对应的模板进行渲染。

```python
from indexpy.http import HTTPView
from indexpy.http.responses import TemplateResponse


class HTTP(HTTPView):
    def get(self):
        return TemplateResponse("chat.html", {"request": self.request})
```

* YAMLResponse

由于 YAML 与 JSON 的等价性，YAMLResponse 与 JSONResponse 的使用方法相同。

唯一不同的是，一个返回 YAML 格式，一个返回 JSON 格式。

### 自定义返回类型

为了方便使用，Index 允许自定义一些函数来处理 `HTTP` 内的处理方法返回的非 `Response` 对象。它的原理是拦截响应，通过响应值的类型来自动选择处理函数，把非 `Response` 对象转换为 `Response` 对象。

Index 内置了三个处理函数用于处理六种类型：

```python
@automatic.register(type(None))
def _none(ret: typing.Type[None]) -> typing.NoReturn:
    raise TypeError(
        "Get 'None'. Maybe you need to add a return statement to the function."
    )


@automatic.register(tuple)
@automatic.register(list)
@automatic.register(dict)
def _json(
    body: typing.Tuple[tuple, list, dict],
    status: int = 200,
    headers: dict = None
) -> Response:
    return JSONResponse(body, status, headers)


@automatic.register(str)
@automatic.register(bytes)
def _plain_text(
    body: typing.Union[str, bytes], status: int = 200, headers: dict = None
) -> Response:
    return PlainTextResponse(body, status, headers)
```

## 中间件

在 `views` 中任意 `__init__.py` 中定义名为 `Middleware` 的类, 它将能处理所有通过该路径的 HTTP 请求。

譬如在 `views/__init__.py` 中定义的中间件，能处理所有 URI 的 HTTP 请求；在 `views/api/__init__.py` 则只能处理 URI 为 `/api/*` 的请求。

`Middleware` 需要继承 `indexpy.http.MiddlewareMixin`，有以下两个方法可以重写。

1. `process_request(request)`

    此方法在请求被层层传递时调用，可用于修改 `request` 对象以供后续处理使用。必须返回 `None`，否则返回值将作为最终结果并直接终止此次请求。

2. `process_response(request, response)`

    此方法在请求被正常处理、已经返回响应对象后调用，它必须返回一个可用的响应对象（一般来说直接返回 `response` 即可）。

!!! notice
    以上函数无论你以何种方式定义，都会在加载时被改造成异步函数，但为了减少不必要的损耗，尽量使用 `async def` 去定义它们。

### 子中间件

很多时候，对于同一个父 URI，需要有多个中间件去处理。通过指定 `Middleware` 中的 `mounts` 属性，可以为中间件指定子中间件。执行时会先执行父中间件，再执行子中间件。

!!! notice
    子中间件的执行顺序是从左到右。

```python
from indexpy.http import MiddlewareMixin


class ExampleChildMiddleware(MiddlewareMixin):
    async def process_request(self, request):
        print("enter first process request")

    async def process_response(self, request, response):
        print("enter last process response")
        return response


class Middleware(MiddlewareMixin):
    mounts = (ExampleChildMiddleware,)

    async def process_request(self, request):
        print("example base middleware request")

    async def process_response(self, request, response):
        print("example base middleware response")
        return response
```

对于一些故意抛出的异常或者特定的 HTTP 状态码，Index 提供了方法进行统一处理。

以下为样例：

```python
from indexpy import Index
from indexpy.types import Request, Response
from indexpy.http.responses import PlainTextResponse
from starlette.exceptions import HTTPException

app = Index()


@app.exception_handler(404)
def not_found(request: Request, exc: HTTPException) -> Response:
    return PlainTextResponse("what do you want to do?", status_code=404)


@app.exception_handler(ValueError)
def value_error(request: Request, exc: ValueError) -> Response:
    return PlainTextResponse("Something went wrong with the server.", status_code=500)
```

!!!notice
    如果是捕捉 HTTP 状态码，则处理函数的 `exc` 类型是 `starlette.exceptions.HTTPException`。否则，捕捉什么异常，则 `exc` 就是什么类型的异常。

## 后台任务

### After Response

参考 starlette 的 `background` 设计，Index 提供了更简单可用的使用方法。

```python
from indexpy.http import HTTPView
from indexpy.http import after_response


@after_response
def only_print(message: str) -> None:
    print(message)


class HTTP(HTTPView):
    async def get(self):
        """
        welcome page
        """
        only_print("world")
        print("hello")
        return ""
```

得益于 [contextvars](https://docs.python.org/zh-cn/3.7/library/contextvars.html)，你可以在整个 HTTP 请求的周期内的任何位置去调用函数，它们都将在响应成功完成后开始执行。

### Finished Response

Index 提供了另一个装饰器 `finished_response`，它的使用与 `after_response` 完全相同。不同的是，`finished_response` 的执行时间节点在此次响应结束后（包括 `after_response` 任务执行完成），无论在此过程中是否引发了错误导致流程提前结束，`finished_response` 都将执行。

粗浅的理解，`after_response` 用于请求被正常处理完成后执行一些任务，一旦处理请求的过程中抛出错误，`after_response` 将不会执行。而 `finished_response` 充当了 `finally` 的角色，无论如何，它都会执行（除非 Index 服务终止）。