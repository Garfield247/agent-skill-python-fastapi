# 统一响应包装与全局业务异常拦截 (Unified Response & Exceptions)

## 1. 统一外层标准响应包装

所有 HTTP 接口响应报文统一使用标准三段式结构：

```json
{
  "code": 0,
  "msg": "ok",
  "data": { ... }
}
```

Pydantic 泛型响应基类：
```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")

class BaseResponse(BaseModel, Generic[T]):
    code: int = 0
    msg: str = "ok"
    data: T | None = None
```

---

## 2. 全局业务异常拦截 (拒绝裸 500)

严禁把可预见的业务逻辑失败直接抛出给客户端 HTTP 500：

```python
# pkg/exceptions.py
class BaseBusinessException(Exception):
    """业务异常基类"""
    def __init__(self, code: int = 400, msg: str = "业务处理失败", data: any = None):
        self.code = code
        self.msg = msg
        self.data = data
        super().__init__(msg)

class ParamValidateException(BaseBusinessException):
    def __init__(self, msg: str = "请求参数不合法"):
        super().__init__(code=422, msg=msg)

class NotFoundException(BaseBusinessException):
    def __init__(self, msg: str = "请求资源不存在"):
        super().__init__(code=404, msg=msg)
```

### 全局异常处理器注册 (`main.py`)
```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
import logging

logger = logging.getLogger(__name__)
app = FastAPI()

@app.exception_handler(BaseBusinessException)
async def business_exception_handler(request: Request, exc: BaseBusinessException):
    """捕获业务异常，由统一业务 code 承载"""
    return JSONResponse(
        status_code=200,
        content={"code": exc.code, "msg": exc.msg, "data": exc.data}
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    """捕获 Pydantic 入参校验失败，提取首个错误"""
    err_msg = exc.errors()[0].get("msg") if exc.errors() else "参数格式错误"
    return JSONResponse(
        status_code=200,
        content={"code": 422, "msg": f"参数校验失败: {err_msg}", "data": exc.errors()}
    )

@app.exception_handler(Exception)
async def global_unhandled_exception_handler(request: Request, exc: Exception):
    """兜底未预料的系统异常，写全堆栈日志，对外脱敏返回通用提示"""
    logger.error(f"Unhandled Exception: {request.method} {request.url} - {str(exc)}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={"code": 500, "msg": "系统繁忙，请稍后重试", "data": None}
    )
```
