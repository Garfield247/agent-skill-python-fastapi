---
name: python-fastapi
description: >-
  Python 3.10+ 与 FastAPI 现代异步高并发 Web API 架构、防阻塞黄金法则、Pydantic v2 与统一业务异常工程规范技能。
  涵盖 Async/Await 事件循环防阻塞调度、统一响应封装 ({"code": 0, "msg": "ok", "data": ...})、
  全局业务异常拦截 (BaseBusinessException, 拒绝笼统 500)、Pydantic v2 ConfigDict 校验规范、
  依赖注入 (Annotated[..., Depends])、环境变量强绑定 (pydantic-settings) 及异步数据库 Session 生命周期管理。
---

# Python 3.10+ & FastAPI 异步高并发 Web 开发规范技能 (FastAPI Mastery Skill)

## 概述 (Overview)

本技能定义了研发工程师与 AI 编码助手在基于 **Python 3.10+** 与 **FastAPI** 框架构建高性能、高可用、类型安全的异步 Web API 服务时的通用架构分层标准与工程红线。

### 核心设计原则

1. **Async/Await 防阻塞黄金法则**：深刻理解单线程事件循环（Event Loop）与线程池调度的本质区别，坚决禁止在 `async def` 中调用同步阻塞 I/O 或执行 CPU 密集型长耗时计算。
2. **统一响应与精确异常拦截体系**：全局统一外层包装 `{"code": 0, "msg": "ok", "data": ...}`，通过全局 `exception_handler` 自定义业务异常，**坚决杜绝向前端直接抛出笼统的 HTTP 500**。
3. **强类型安全与 Pydantic v2 标准**：强依赖 Python 3.10+ 现代类型提示，统一使用 Pydantic v2 `model_config = ConfigDict(...)` 声明，环境变量读取强依赖 `pydantic-settings`。
4. **分层清晰与依赖注入 (Dependency Injection)**：Router 控制层 $\to$ Service 业务编排层 $\to$ Repository / ORM 持有层，统一使用 `Annotated[..., Depends(...)]` 进行生命周期管理与上下文解耦。

---

# 1. Async/Await 核心红线与防阻塞黄金法则 (Core Concurrency Rules)

FastAPI 在 `async def`（主事件循环单线程）与普通 `def`（后台线程池 AnyIO Worker）之间的调度机制存在本质差异。**严禁滥用 `async` 导致事件循环卡死**：

| 场景分类 | 声明方式 | 处置准则与技术方案 |
| :--- | :---: | :--- |
| **纯异步 I/O 驱动** | `async def` | 必须使用纯异步库：`httpx.AsyncClient`、`asyncpg`、`aiofiles`、`redis.asyncio`、`asyncio.sleep`。 |
| **同步阻塞第三方库** | **普通 `def`**<br/>或 `anyio` 包装 | 若无纯异步驱动（如第三方同步 SDK、旧版驱动、文件操作），**必须声明为普通 `def` 路由**（由 FastAPI 自动派发至线程池执行），或在 `async def` 中使用 `await anyio.to_thread.run_sync(...)` 包装。 |
| **CPU 密集型计算** | **普通 `def`**<br/>或进程池 | 密码哈希（`bcrypt`）、图像/音视频处理、复杂数学运算**严禁在 `async def` 中直接计算**，必须使用普通 `def` 或后台 `ProcessPoolExecutor`。 |
| **纯内存数据计算** | 普通 `def` / 同步函数 | 简单的字典组装、内存遍历严禁无脑使用 `async def`（避免微小伪异步开销）。 |

### 典型正反代码对比
```python
# ❌ 致命错误：在 async def 中调用同步阻塞 I/O，卡死全局并发事件循环
@router.get("/user/info")
async def get_user_info():
    time.sleep(2)                        # 绝对禁止！整个进程在此停摆 2 秒
    res = requests.get("https://api...") # 绝对禁止！网络阻塞事件循环
    return res.json()

# ✅ 正确方案 1：纯异步驱动
@router.get("/user/info")
async def get_user_info(client: httpx.AsyncClient = Depends(get_http_client)):
    await asyncio.sleep(2)
    res = await client.get("https://api...")
    return res.json()

# ✅ 正确方案 2：无法避免同步阻塞库时，声明为普通 def 路由让出事件循环
@router.post("/legacy/export")
def export_excel(payload: ExportRequest):
    # FastAPI 会自动将该请求丢入线程池调度，不会卡死主事件循环
    return run_heavy_sync_task(payload)
```

---

# 2. 统一 API 响应与业务异常拦截体系 (Unified Response & Exceptions)

## 2.1 统一外层标准响应包装
所有 HTTP 接口响应结构统一为：

```json
{
  "code": 0,
  "msg": "ok",
  "data": { ... }
}
```

通用 Pydantic 响应包装模型：
```python
from typing import Generic, TypeVar
from pydantic import BaseModel

T = TypeVar("T")

class BaseResponse(BaseModel, Generic[T]):
    code: int = 0
    msg: str = "ok"
    data: T | None = None
```

## 2.2 全局业务异常拦截 (拒绝笼统 500)
- **核心红线**：严禁把可预见的业务逻辑失败、参数校验错误直接抛出笼统的 HTTP 500；
- **业务异常定义 (`pkg/exceptions.py`)**：
  ```python
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

- **全局异常拦截器注册 (`main.py`)**：
  ```python
  from fastapi import FastAPI, Request
  from fastapi.responses import JSONResponse
  from fastapi.exceptions import RequestValidationError
  import logging

  logger = logging.getLogger(__name__)

  app = FastAPI()

  @app.exception_handler(BaseBusinessException)
  async def business_exception_handler(request: Request, exc: BaseBusinessException):
      """捕获可预见的业务异常，返回精准错误码"""
      return JSONResponse(
          status_code=200, # 保持 HTTP 200，由业务层 code 承载错误
          content={"code": exc.code, "msg": exc.msg, "data": exc.data}
      )

  @app.exception_handler(RequestValidationError)
  async def validation_exception_handler(request: Request, exc: RequestValidationError):
      """捕获 Pydantic 入参校验失败"""
      err_msg = exc.errors()[0].get("msg") if exc.errors() else "参数格式错误"
      return JSONResponse(
          status_code=200,
          content={"code": 422, "msg": f"参数校验失败: {err_msg}", "data": exc.errors()}
      )

  @app.exception_handler(Exception)
  async def global_unhandled_exception_handler(request: Request, exc: Exception):
      """兜底未预料的系统级异常，记录堆栈并脱敏返回"""
      logger.error(f"Unhandled Exception: {request.method} {request.url} - {str(exc)}", exc_info=True)
      return JSONResponse(
          status_code=500,
          content={"code": 500, "msg": "系统繁忙，请稍后重试", "data": None}
      )
  ```

---

# 3. 类型安全与 Pydantic v2 规范 (Type Safety & Pydantic v2)

1. **全面采用 Python 3.10+ 原生类型语法**：
   - 联合类型：使用 `str | None`（严禁旧版 `Optional[str]`）；
   - 容器集合：使用原生 `list[str]`、`dict[str, Any]`、`set[int]`（严禁从 `typing` 导入 `List`、`Dict`、`Set`）。
2. **Pydantic v2 配置标准**：
   - 模型配置统一使用 `model_config = ConfigDict(...)` 类属性（禁止使用旧版内部 `class Config:`）。
   - ORM 模型与 Request/Response Schema 必须添加详尽的 `Field(description="...")` 中文注释。
   ```python
   from pydantic import BaseModel, ConfigDict, Field

   class UserCreateRequest(BaseModel):
       model_config = ConfigDict(
           populate_by_name=True,
           str_strip_whitespace=True, # 自动去除首尾空格
           extra="forbid"             # 严格拦截未知入参
       )

       username: str = Field(..., min_length=3, max_length=50, description="用户登录名")
       email: str | None = Field(default=None, description="电子邮箱地址")
       phone: str | None = Field(default=None, pattern=r"^1[3-9]\d{9}$", description="国内手机号")
   ```

3. **配置中心与环境变量管理**：
   - 强依赖 `pydantic-settings`：
   ```python
   from pydantic_settings import BaseSettings, SettingsConfigDict

   class AppSettings(BaseSettings):
       model_config = SettingsConfigDict(
           env_file=".env",
           env_file_encoding="utf-8",
           extra="ignore"
       )

       APP_ENV: str = "local"
       DATABASE_URL: str
       REDIS_HOST: str = "127.0.0.1"
       REDIS_PORT: int = 6379

   settings = AppSettings()
   ```

---

# 4. 依赖注入与数据库 Session 管理 (Dependency Injection & DB Session)

推荐使用现代 `typing.Annotated` 语法配合 FastAPI `Depends`，保证代码高度清晰可测：

```python
from typing import Annotated
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from core.database import get_db_session

# 定义强类型别名
DBSessionDep = Annotated[AsyncSession, Depends(get_db_session)]
CurrentUserDep = Annotated[User, Depends(get_current_active_user)]

router = APIRouter(prefix="/users", tags=["用户模块"])

@router.get("/me", response_model=BaseResponse[UserDetailResponse])
async def get_my_profile(
    db: DBSessionDep,
    current_user: CurrentUserDep
):
    service = UserService(db)
    result = await service.get_user_detail(current_user.id)
    return BaseResponse(data=result)
```

### 数据库 Session 生命周期黄金法则
- Session 必须通过 `async_generator` 在 `finally` 块中自动归还连接池，严禁手动 `close` 遗漏；
- 事务边界清晰控制，推荐只读请求禁用事务（`autocommit` 或独立引擎），写操作显式 `await session.commit()`；
- 严禁把持久化 ORM 模型直接作为接口对外返回值，必须转换为独立的 Pydantic Schema。

---

# 5. 快速排查 Checklist (Pre-Merge Inspection)

- [ ] 是否存在 `async def` 内部调用 `time.sleep`、`requests` 或第三方同步阻塞库的情况？
- [ ] 所有接口返回是否遵循统一响应体 `{"code": 0, "msg": "ok", "data": ...}`？
- [ ] 业务异常是否继承自 `BaseBusinessException` 并在全局拦截，杜绝向客户端抛出 500？
- [ ] 是否已全部采用 Python 3.10+ 类型语法（`str | None`、`list[...]`）？
- [ ] Pydantic 模型是否使用 `model_config = ConfigDict(...)` 标准规范？
- [ ] 环境变量与敏感配置是否全部通过 `pydantic-settings` 安全接管？

---

# 6. Bug 分析、排查与调试武器库 (Troubleshooting & Debugging Guide)

在排查 FastAPI 异步 Web 服务 Bug 时，必须严格遵循 `systematic-debugging` 根因分析 SOP，并使用以下专属工具诊断：

### 6.1 事件循环防阻塞检测 (Event Loop Blocking Detector)
- **开启 Asyncio 调试模式**：
  当接口吞吐量骤降或出现大量并发超时（Client Timeout）时，极大可能是某处调用了同步阻塞代码卡死了主线程事件循环。
  通过在启动命令前注入环境变量开启阻塞检测（默认告警阈值 100ms）：
  ```bash
  PYTHONASYNCIODEBUG=1 uvicorn main:app --reload
  ```
  终端将自动打印导致事件循环挂起的函数调用堆栈及阻塞耗时：
  `Executing <Handle ...> took 1.250 seconds` -> 直接定位违规同步阻塞函数！

### 6.2 异步数据库连接池泄漏与长事务排查
- **现象**：高并发下服务报错 `Timeout context manager should be used with async with` 或 `QueuePool limit of size 5 overflow 10 reached, connection timed out`；
- **排查手段**：
  1. 检查是否存在只开启事务但未在 `finally` 块中执行 `await session.close()` 的遗漏；
  2. 严格使用 `AsyncSession` 上下文生成器依赖注入，严禁在全局生命周期中共享单个 Session 实例。

### 6.3 Pydantic 入参校验失败与 422 诊断
- **现象**：客户端提示 `422 Unprocessable Entity` 但前端未能明确知道哪个字段出错；
- **排查手段**：
  在全局 `validation_exception_handler` 中打印 `exc.errors()` 结构，定位具体的定位路径（`loc`）与校验类型（`type`）。

---

# 7. 现代 Web API 架构进阶：Lifespan 治理与流式响应 (Lifespan & SSE Standards)

### 7.1 现代 Lifespan 生命周期全面取代 On-Event
全面淘汰已过时的 `@app.on_event("startup")`，统一采用新版基于 `@asynccontextmanager` 的标准生命周期管理器：
```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
import httpx
from core.database import init_db_pool, close_db_pool

@asynccontextmanager
async def lifespan(app: FastAPI):
    # 【启动阶段】初始化长连接池、连接 Redis、预热缓存
    http_client = httpx.AsyncClient(timeout=15.0)
    app.state.http_client = http_client
    await init_db_pool()
    yield
    # 【优雅停机阶段】释放连接池、关闭后台异步队列
    await http_client.aclose()
    await close_db_pool()

app = FastAPI(lifespan=lifespan)
```

### 7.2 流式响应与 SSE (Server-Sent Events) 标准
针对大模型输出、长耗时报表等场景，统一规范流式推送与背压控制：
```python
from fastapi.responses import StreamingResponse
import asyncio

async def event_generator():
    for item in fetch_large_stream():
        yield f"data: {json.dumps(item)}\n\n"
        await asyncio.sleep(0.01) # 适时让出事件循环，保障系统响应性

@router.get("/stream/events")
async def stream_events():
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```
