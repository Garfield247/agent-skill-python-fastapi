# 依赖注入与异步数据库 Session 生命周期 (DI & DB Session)

## 1. `Annotated[..., Depends]` 强类型依赖注入

```python
from typing import Annotated
from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from core.database import get_db_session
from core.auth import get_current_active_user
from models.user import User

# 定义清晰、可组合的类型别名
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

---

## 2. 数据库 Session 生命周期黄金法则

- **自动生成与回收**：Session 必须使用异步生成器 `async def get_db_session()`，并在 `finally` 块中通过上下文管理器彻底关闭，严禁跨请求泄露；
- **事务边界明确**：写操作显式 `await session.commit()`，读操作禁止开启事务；
- **禁止直接返回 ORM 对象**：持久化 ORM 模型严禁直接透传给客户端，必须经由独立 Pydantic Response Schema 映射解耦。
