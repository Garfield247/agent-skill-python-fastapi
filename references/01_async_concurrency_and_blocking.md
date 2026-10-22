# Async/Await 事件循环调度与防阻塞黄金法则 (Async Concurrency Rules)

## 1. FastAPI 协程调度本质差异与分发矩阵

FastAPI 在 `async def`（主事件循环单线程运行）与普通 `def`（后台线程池 AnyIO Worker 派发）之间具有完全不同的调度行为：

| 业务场景 | 路由函数声明 | 核心处置规范与底层机理 |
| :--- | :---: | :--- |
| **纯异步非阻塞 I/O** | `async def` | 必须使用全异步库：`httpx.AsyncClient`、`asyncpg`、`aiofiles`、`redis.asyncio`、`asyncio.sleep`。 |
| **同步阻塞第三方库/SDK** | **普通 `def`**<br/>或 `anyio` 包装 | 遇无法异步化的旧 SDK 或文件操作，**必须声明为普通 `def`**（由 FastAPI 自动派发至线程池调度，不卡死事件循环），或使用 `await anyio.to_thread.run_sync(...)`。 |
| **CPU 密集型重计算** | **普通 `def`**<br/>或进程池 | 密码散列 (`bcrypt`)、图像音视频处理、复杂矩阵计算**严禁在 `async def` 中直接计算**，必须使用普通 `def` 或后台 `ProcessPoolExecutor`。 |
| **纯内存计算/无 I/O** | 普通 `def` / 同步函数 | 简单的字典取值组装，无需强加 `async def`，避免无意义的协程上下文切换开销。 |

---

## 2. 典型正反代码对比

```python
# ❌ 绝命死穴：在 async def 中调用同步阻塞 I/O，全局并发在此停摆
@router.get("/user/info")
async def get_user_info():
    time.sleep(2)                        # 绝对禁止！整个主事件循环冻结 2 秒
    res = requests.get("https://api...") # 绝对禁止！网络阻塞事件循环
    return res.json()

# ✅ 正确方案 1：纯异步驱动非阻塞调度
@router.get("/user/info")
async def get_user_info(client: httpx.AsyncClient = Depends(get_http_client)):
    await asyncio.sleep(2)
    res = await client.get("https://api...")
    return res.json()

# ✅ 正确方案 2：无法避免同步阻塞库时，声明为普通 def 路由
@router.post("/legacy/export")
def export_excel(payload: ExportRequest):
    # FastAPI 会自动将该请求丢入后台线程池，不会阻塞主事件循环
    return run_heavy_sync_task(payload)
```

---

## 3. 事件循环卡顿检测探针 (Event Loop Blocking Detector)

当接口并发吞吐量骤降或出现大量客户端超时时，前置开启环境变量定位卡死事件循环的违规调用：

```bash
PYTHONASYNCIODEBUG=1 uvicorn main:app --reload
```

终端将自动捕获并打印耗时超过 100ms 的违规调用堆栈：
`Executing <Handle ...> took 1.250 seconds` -> 精确锁定是哪个同步函数卡死了主线程！
