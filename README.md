# python-fastapi

> Python 3.10+ 与 FastAPI 现代异步高并发 Web API 架构、防阻塞黄金法则、Pydantic v2 与统一业务异常工程规范技能库。

## 🌟 核心特性 (Features)

- **Async/Await 防阻塞调度规范**：彻底理清单线程事件循环与 AnyIO 线程池调度差异，严禁同步 I/O 阻塞。
- **统一响应与业务异常体系**：标准 `{"code": 0, "msg": "ok", "data": ...}` 响应，拒绝笼统 HTTP 500。
- **现代类型安全 & Pydantic v2**：采用 Python 3.10+ 原生类型与 `ConfigDict` 声明，强类型环境变量绑定。
- **依赖注入与分层治理**：`Annotated[..., Depends]` 强解耦，严格管理异步数据库 Session 生命周期。

## 📦 安装与加载 (Installation)

### 方式 1: 安装至 Antigravity / Gemini 全局技能库
```bash
git clone git@github.com:Garfield247/agent-skill-python-fastapi.git ~/.gemini/config/skills/python-fastapi
```

### 方式 2: 在任意项目中作为本地工作区技能引入
```bash
mkdir -p .agents/skills
git clone git@github.com:Garfield247/agent-skill-python-fastapi.git .agents/skills/python-fastapi
```

## 📄 开源协议 (License)
本项目采用 [MIT License](LICENSE) 授权。
