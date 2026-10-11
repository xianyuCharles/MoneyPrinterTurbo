# AGENTS.md

> 本文件供 AI 编码助手（Trae / Claude Code / Cursor 等）快速建立项目上下文。修改项目结构或技术栈时请同步更新本文件。

## 项目简介

- **MoneyPrinterTurbo v1.3.8**：一站式 AI 短视频生成工具（MIT 协议）。
- 核心流程：输入主题 → LLM 生成文案与关键词 → 获取素材（Pexels/Pixabay 图库或文生视频服务）→ TTS 配音 → 自动生成字幕/背景音乐 → FFmpeg 合成成片 → 可选一键发布（TikTok / YouTube / Instagram）。
- 四种使用入口：Streamlit WebUI、FastAPI 服务、CLI（`cli.py`）、Agent Skill（`docs/skill/`）。
- 本仓库是 `xianyuCharles/MoneyPrinterTurbo`（fork，二次开发工作区），上游为 `harry0703/MoneyPrinterTurbo`；同步上游用 `git fetch upstream`。

## 技术栈

| 层 | 技术 |
|---|---|
| 语言 | Python 3.11（`.python-version` 固定），包管理 **uv**（`uv.lock` 锁定版本） |
| API 服务 | FastAPI + Uvicorn，入口 `main.py` → `app/asgi.py`，默认 `0.0.0.0:8080`，文档 `/docs` |
| WebUI | Streamlit，入口 `webui/Main.py`，Windows 用 `webui.bat` 启动，默认端口 8501 |
| 视频处理 | moviepy 2.2.1 + FFmpeg（自动检测，缺失可自动下载） |
| LLM | 统一注册表 `app/models/llm_provider.py`，支持 moonshot/deepseek/openai/qwen 等 20+ 服务商，litellm 兜底 |
| TTS / 字幕 | edge-tts（默认）、Azure Speech、faster-whisper（可选）、MiniMax/ElevenLabs/Fish 等云服务 |
| 任务状态 | 本地文件为主，可选 Redis（`enable_redis`） |
| 配置 / 日志 | TOML（`config.toml`，敏感信息不提交）+ loguru |

## 关键目录

```text
├── main.py                 # FastAPI 服务启动入口（端口来自 config.toml listen_port=8080）
├── cli.py                  # CLI 生成入口（python cli.py --help）
├── config.example.toml     # 配置模板；首次运行自动复制为 config.toml（含 API Key，禁止提交）
├── app/
│   ├── asgi.py             # FastAPI 应用组装：CORS、lifespan、鉴权开关
│   ├── router.py           # 路由汇总
│   ├── config/             # 配置加载（config.toml → 全局单例，启动时读取，改配置需重启）
│   ├── controllers/        # 路由处理层（薄层，参数校验后调 services）
│   ├── models/             # pydantic schema（schema.py）、常量（const.py）、LLM 注册表（llm_provider.py）、异常
│   ├── services/           # 核心业务逻辑（改动最频繁的目录）
│   │   ├── task.py         # ★ 视频生成任务流水线（编排素材/配音/字幕/合成，FFmpeg 探测在这里）
│   │   ├── llm.py          # LLM 调用与文案/关键词生成
│   │   ├── video.py        # 视频剪辑合成
│   │   ├── voice.py        # TTS 配音
│   │   ├── subtitle.py     # 字幕
│   │   ├── material.py     # 素材搜索与下载
│   │   ├── bgm.py / state.py / video_project.py / cache_manager.py ...
│   ├── utils/              # 通用工具
│   └── __init__.py         # 版本号
├── webui/Main.py           # Streamlit 页面主体（视频参数、生成按钮、设置面板）
├── test/                   # pytest 测试（test_main.py）
├── docs/skill/             # Agent Skill（SKILL.md + mpt_agent.py）
├── storage/                # 运行时产物（任务目录、缓存，不入库）
└── webui.bat               # Windows 一键启动 WebUI（优先用 .venv）
```

## 开发规范

### 环境与依赖
- 一律用 uv：安装 `uv sync --frozen`；新增依赖用 `uv add <pkg>`（同时更新 `pyproject.toml` 和 `uv.lock`），**不要用裸 pip**。
- 运行 Python 用项目 venv：`uv run python ...` 或先 `.venv\Scripts\Activate.ps1`。
- PowerShell 5 不支持 `&&`，命令链用 `;`。

### 代码质量
- Lint / 格式化：`uv run ruff check` / `uv run ruff format`（规则见 `pyproject.toml`）。
- 测试：`uv run pytest`；分支覆盖率底线 70%（`fail_under = 70`，统计范围 app/cli/webui/main/docs/skill）。
- 改 `app/services/` 下模块时，同步补/改 `test/` 中的对应用例。

### 提交与分支
- 提交信息用 Conventional Commits，scope 取模块名：`feat(llm):`、`fix(task):`、`fix(video):`、`chore:`（参考 git log 现有风格）。
- 二次开发在 feature 分支进行，不要直接改 main；同步上游：`git fetch upstream` → `git merge upstream/main`。

### 安全红线
- `config.toml`、`storage/`、`.venv/`、API Key 一律不进 git。
- 新增配置项：在 `config.example.toml` 补注释模板 + 在 `app/config/` 补字段。

### 常用命令

```powershell
uv sync --frozen                # 安装/同步依赖（清华镜像见 UV_DEFAULT_INDEX）
.\webui.bat                     # 启动 WebUI（http://127.0.0.1:8501）
uv run python main.py           # 启动 API（http://127.0.0.1:8080/docs）
uv run pytest                   # 跑测试
uv run ruff check               # Lint
.venv\Scripts\python.exe -c "import moviepy, fastapi, streamlit; print('OK')"   # 依赖自检
```

### 给 AI 的特别提示
- 配置是**启动时加载**的：改了 `config.toml` 后必须重启 WebUI/API 进程才生效。
- API 请求体/响应以 `app/models/schema.py` 为准，可直接查 `http://127.0.0.1:8080/openapi.json`。
- 本项目注释与文档以中文为主，保持一致。
