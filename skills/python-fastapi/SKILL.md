---
name: python-fastapi
description: >-
  Python 3.10+ 与 FastAPI 现代异步高并发 Web API 架构、防阻塞黄金法则、Pydantic v2 与统一业务异常工程规范技能。
  涵盖 Async/Await 事件循环防阻塞调度、统一响应封装 ({"code": 0, "msg": "ok", "data": ...})、
  全局业务异常拦截 (BaseBusinessException, 拒绝笼统 500)、Pydantic v2 ConfigDict 校验规范、
  依赖注入 (Annotated[..., Depends])、环境变量强绑定 (pydantic-settings) 及异步数据库 Session 生命周期管理。
---

# FastAPI 现代异步高并发 Web 开发规范 (FastAPI Mastery Skill)

本技能定义了基于 **Python 3.10+、FastAPI、Pydantic v2 与 SQLAlchemy 2.0 Async** 构建现代 Web API 服务时的架构分层准则、防阻塞红线与渐进式手册索引。

---

## 1. 核心铁律与绝对红线 (Core Redlines)

1. 🚨 **Async/Await 事件循环防阻塞黄金法则**：
   - `async def` 路由中**严禁调用任何同步阻塞 I/O**（如 `time.sleep`、`requests`、旧版阻塞 SDK）；
   - 遇同步阻塞库必须声明为普通 `def` 路由（由 FastAPI 自动调度至后台线程池），或通过 `await anyio.to_thread.run_sync(...)` 包装；
   - CPU 密集型运算严禁在主协程直接运行，必须派发至进程池。
2. 🚨 **统一三段式响应与拒绝裸 500**：
   - 所有接口统一返回 `{"code": 0, "msg": "ok", "data": ...}`；
   - 业务逻辑错误必须继承 `BaseBusinessException` 并在全局统一拦截，严禁直接向客户端抛出 HTTP 500 堆栈。
3. 🚨 **Pydantic v2 与配置强接管**：
   - 统一采用 `model_config = ConfigDict(extra="forbid")` 拦截未知参数；
   - 严禁在代码中直接散落 `os.getenv`，所有敏感配置统一经由 `pydantic-settings` 接管。
4. 🚨 **依赖注入与 DB Session 生命周期防泄漏**：
   - 统一采用 `Annotated[..., Depends]` 强类型注入；
   - 数据库 Session 必须通过异步生成器 `yield` 驱动并在 `finally` 彻底回收，严禁跨请求全局持有单个 Session。

---

## 2. 标准开发工作流 (Standard Workflow)

1. **配置与生命周期**：在 `core/config.py` 与 `lifespan` 中初始化配置与连接池；
2. **Schema 契约**：在 `schemas/` 中使用 Pydantic v2 定义请求与响应模型；
3. **路由分发**：根据 I/O 属性明确声明 `async def`（纯异步）或 `def`（同步 SDK）；
4. **业务注入**：通过 `Depends` 注入 DB Session 与当前认证上下文；
5. **诊断闭环**：通过 `PYTHONASYNCIODEBUG=1` 验证无事件循环卡死隐患。

---

## 3. 渐进式专业参考手册 (Progressive Disclosure References)

根据具体涉及的业务场景，**按需查阅 `references/` 目录下的专门规约**，严禁将全量子文档一次性无脑全部展开：

| 场景模块 | 参考子文档 | 核心涵盖规范 |
| :--- | :--- | :--- |
| **异步并发与防阻塞** | [01_async_concurrency_and_blocking.md](references/01_async_concurrency_and_blocking.md) | • `async def` vs 普通 `def` 调度矩阵<br>• 同步 SDK 线程池分发与正反范例<br>• `PYTHONASYNCIODEBUG=1` 阻塞探针诊断 |
| **统一响应与异常拦截** | [02_unified_response_and_exceptions.md](references/02_unified_response_and_exceptions.md) | • 统一三段式外层响应包装<br>• `BaseBusinessException` 全局异常处理器<br>• 422 参数校验错误格式化与脱敏 |
| **Pydantic v2 与配置** | [03_pydantic_v2_and_settings.md](references/03_pydantic_v2_and_settings.md) | • Pydantic v2 ConfigDict 与 v1 降级对照<br>• Request Schema 规范与 extra="forbid"<br>• `pydantic-settings` 环境变量强治理 |
| **依赖注入与 DB Session** | [04_dependency_injection_and_db_session.md](references/04_dependency_injection_and_db_session.md) | • `Annotated[..., Depends]` 强类型注入范式<br>• SQLAlchemy 2.0 AsyncSession 生命周期<br>• 读写事务边界与防泄漏连接池规约 |
| **Lifespan 与流式响应** | [05_lifespan_and_streaming.md](references/05_lifespan_and_streaming.md) | • `@asynccontextmanager` Lifespan 优雅停机<br>• HTTP Client 与长连接池全局预热与销毁<br>• Server-Sent Events (SSE) 流式响应标准 |
