# Pydantic v2 规范与 pydantic-settings 配置治理 (Pydantic v2 & Settings)

## 1. Pydantic v2 现代写法与 v1 降级对照

| 模块 / 特性 | 推荐基调 (Pydantic v2) | 历史项目自适应兼容 (Pydantic v1) |
| :--- | :--- | :--- |
| **模型配置声明** | `model_config = ConfigDict(extra="forbid")` | `class Config:`<br/>`    extra = "forbid"` |
| **自定义字段校验** | `@field_validator("field_name")` | `@validator("field_name")` |
| **模型序列化导出** | `model.model_dump()` / `model.model_dump_json()` | `model.dict()` / `model.json()` |

---

## 2. 请求 Schema 标杆范式

```python
from pydantic import BaseModel, ConfigDict, Field

class UserCreateRequest(BaseModel):
    model_config = ConfigDict(
        populate_by_name=True,
        str_strip_whitespace=True, # 自动去除首尾空格
        extra="forbid"             # 严格拦截未经声明的恶意未知字段
    )

    username: str = Field(..., min_length=3, max_length=50, description="用户登录名")
    email: str | None = Field(default=None, description="电子邮箱")
    phone: str | None = Field(default=None, pattern=r"^1[3-9]\d{9}$", description="11位手机号")
```

---

## 3. `pydantic-settings` 环境变量强治理

所有数据库密码、Redis 密钥、第三方 Token 必须由 `BaseSettings` 强接管，禁止散落在代码中调用 `os.getenv`：

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class AppSettings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore"
    )

    APP_ENV: str = "production"
    DATABASE_URL: str
    REDIS_HOST: str = "127.0.0.1"
    REDIS_PORT: int = 6379

settings = AppSettings()
```
