# Lifespan 生命周期治理与 SSE 流式响应 (Lifespan & Streaming)

## 1. 现代 Lifespan 优雅停机全面取代 On-Event

全面废弃旧版 `@app.on_event("startup")`，统一采用新版基于 `@asynccontextmanager` 的标准生命周期管理器：

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

---

## 2. Server-Sent Events (SSE) 流式响应标准

针对大模型输出与长耗时任务，采用 `StreamingResponse` 配合异步生成器让出事件循环：

```python
import json
import asyncio
from fastapi.responses import StreamingResponse

async def event_generator():
    for item in fetch_large_stream():
        yield f"data: {json.dumps(item)}\n\n"
        await asyncio.sleep(0.01) # 适时让出事件循环，保障系统并发响应性

@router.get("/stream/events")
async def stream_events():
    return StreamingResponse(event_generator(), media_type="text/event-stream")
```
