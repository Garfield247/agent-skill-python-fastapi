# python-fastapi

> Python 3.10+ 与 FastAPI 现代异步高并发 Web API 架构、防阻塞黄金法则、Pydantic v2 与统一业务异常工程规范技能库。

## 🌟 核心特性 (Features)

- **Async/Await 防阻塞调度规范**：彻底理清单线程事件循环与 AnyIO 线程池调度差异，严禁同步 I/O 阻塞。
- **统一响应与业务异常体系**：标准 `{"code": 0, "msg": "ok", "data": ...}` 响应，拒绝笼统 HTTP 500。
- **现代类型安全 & Pydantic v2**：采用 Python 3.10+ 原生类型与 `ConfigDict` 声明，强类型环境变量绑定。
- **依赖注入与分层治理**：`Annotated[..., Depends]` 强解耦，严格管理异步数据库 Session 生命周期。

## 📦 安装与多 Agent 使用指南 (Installation & Multi-Agent Usage)

本项目遵循开放 Agent 规范，支持在 **Gemini / Antigravity**、**Anthropic Claude**、**Cursor / Codex** 等各类主流 Agent 环境中一键安装与激活：

### 1. Google Antigravity / Gemini Code Assist
- **全局安装（推荐）**：
  ```bash
  git clone git@github.com:Garfield247/agent-skill-python-fastapi.git ~/.gemini/config/skills/python-fastapi
  ```
- **项目工作区局部引入**：
  ```bash
  mkdir -p .agents/skills
  git clone git@github.com:Garfield247/agent-skill-python-fastapi.git .agents/skills/python-fastapi
  ```

### 2. Anthropic Claude (Claude Code / Claude Projects)
- **Claude Code (CLI 终端智能体)**：
  克隆至 Claude 全局技能库：
  ```bash
  mkdir -p ~/.claude/skills
  git clone git@github.com:Garfield247/agent-skill-python-fastapi.git ~/.claude/skills/python-fastapi
  ```
  *或者在项目根目录的 `CLAUDE.md` 中追加引入：*
  ```markdown
  See detailed engineering specifications in: ~/.claude/skills/python-fastapi/SKILL.md
  ```
- **Claude Projects (Web / 桌面端)**：
  直接将仓库中的 `SKILL.md` 内容复制并粘贴至 Project 的 **Project Knowledge (项目知识库)** 或 **Custom Instructions (自定义指令)** 中。

### 3. Cursor / GitHub Copilot / OpenAI Codex
- **Cursor (现代 MDC 规则体系)**：
  在项目根目录创建或链接规则：
  ```bash
  mkdir -p .cursor/rules
  # 克隆或软链接为 Cursor 专有规则文件
  git clone git@github.com:Garfield247/agent-skill-python-fastapi.git .cursor/rules/python-fastapi
  ```
- **GitHub Copilot / Codex**：
  将本技能规范注入 Copilot 指令集：
  ```bash
  mkdir -p .github
  cat << 'EOF' >> .github/copilot-instructions.md
  # 引入本技能核心规则
  EOF
  cat path/to/SKILL.md >> .github/copilot-instructions.md
  ```

## 📄 开源协议 (License)
本项目采用 [MIT License](LICENSE) 授权。
