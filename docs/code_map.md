# MoneyPrinterTurbo 代码地图（Code Map）

> 面向二次开发的代码导航文档。基于 v1.3.8 源码（2026-10 通读）整理。
> 配套阅读：根目录 `AGENTS.md`（开发规范）、`README.md`（功能与部署）。

## 1. 项目概览

- **定位**：一站式 AI 短视频生成工具。输入主题 → LLM 生成文案/关键词 → 获取素材 → TTS 配音 → 字幕/BGM → FFmpeg 合成 mp4 → 可选发布。
- **四个入口**：Streamlit WebUI、FastAPI API 服务、CLI（`cli.py`）、Agent Skill（`docs/skill/`），全部汇入同一套 `app/` 核心代码。
- **运行形态**：单进程本地服务为主；任务状态默认存本地文件，可选 Redis。

## 2. 目录结构树

```text
MoneyPrinterTurbo/
├── main.py                     # FastAPI 服务启动入口（uvicorn 启动 app.asgi:app，端口来自 config.listen_port=8080）
├── cli.py                      # CLI 入口：创建任务并执行流水线（python cli.py --help）
├── webui.bat / webui.sh        # WebUI 启动脚本（优先使用项目 .venv，回退 uv run streamlit）
├── config.example.toml         # 配置模板；首次运行自动复制为 config.toml（敏感信息，禁止提交）
├── pyproject.toml              # 依赖主定义（moviepy/fastapi/streamlit/edge-tts 等，uv 管理）
├── uv.lock                     # 依赖锁文件（uv sync --frozen 安装）
├── requirements.txt            # 仅兼容旧 pip 安装方式
├── Dockerfile*                 # 标准 / claude / gpu 三种镜像
├── docker-compose*.yml         # 对应三种部署编排
├── app/                        # ★ 核心应用（所有入口共享）
│   ├── asgi.py                 # FastAPI 应用组装：中间件、CORS、lifespan、API 鉴权开关
│   ├── router.py               # 路由汇总：注册 ping / video / llm 等 v1 模块
│   ├── config/
│   │   └── config.py           # config.toml 加载 → 全局单例 config；含 WebUI 运行期非阻塞配置回写（待更新队列 + 配置锁）
│   ├── controllers/            # 路由层（薄层，参数校验后调 services）
│   │   ├── base.py / ping.py   # 基础设施与 /ping 健康检查
│   │   ├── manager/            # 任务队列管理器：base_manager 抽象 + memory_manager（默认）+ redis_manager（enable_redis 时）
│   │   └── v1/
│   │       ├── video.py        # POST /api/v1/videos 等视频任务端点（create_video）
│   │       ├── llm.py          # /api/v1/scripts、/terms 等文案/关键词端点
│   │       └── base.py         # v1 通用逻辑（task_id 生成、任务创建/回滚）
│   ├── models/
│   │   ├── schema.py           # pydantic 模型：VideoParams（L91-165 全量视频参数）、视频比例/分辨率/拼接/转场枚举
│   │   ├── llm_provider.py     # LLM 服务商注册表（20+ 家的默认 base_url 与模型名）
│   │   ├── const.py            # 常量：任务状态（TASK_STATE_*）、目录名等
│   │   └── exception.py        # 业务异常定义
│   ├── services/               # ★ 核心业务层（改动最频繁）
│   │   ├── task.py             # ★★ 任务流水线编排（约 1800 行，详见第 4 节）
│   │   ├── llm.py              # LLM 调用：generate_script / generate_terms，服务商路由与失败重试
│   │   ├── video.py            # FFmpeg/moviepy 合成：combine_videos（拼接素材）、generate_video（烧字幕出片，L1538+）
│   │   ├── voice.py            # TTS：edge-tts / Azure / 云 TTS 统一封装，返回 sub_maker 供字幕用
│   │   ├── subtitle.py         # 字幕生成与解析（edge 时间轴 / faster-whisper）
│   │   ├── bgm.py              # 背景音乐：内置曲库（resource/songs）扫描、下载、音量混合
│   │   ├── material.py         # 素材搜索与下载：Pexels / Pixabay / Coverr / 本地
│   │   ├── material_cache.py   # 素材缓存（避免重复下载）
│   │   ├── material_upload.py  # 本地素材上传处理
│   │   ├── video_project.py    # "视频项目"批量生成管理（多任务编排）
│   │   ├── webui_task.py       # WebUI 专用任务包装：线程内调用 tm.start，收集日志回传界面
│   │   ├── state.py            # 任务状态读写：update_task / patch_task（本地文件或 Redis）
│   │   ├── task_artifacts.py   # 任务产物（script/terms/params 等）的落盘
│   │   ├── cache_manager.py    # storage 目录清理/缓存管理
│   │   ├── loomloom.py         # LoomLoom SkillBot 文案候选后端
│   │   ├── ofox.py / muapi.py / metaso_minimax.py / volcengine_seedance.py  # 付费文生视频素材源适配
│   │   ├── twelvelabs.py       # TwelveLabs 语义排序/视频理解（可选依赖）
│   │   ├── sonilo.py / elevenlabs_music.py  # AI 配乐服务适配
│   │   ├── kimi_code_oauth.py  # Kimi Code OAuth 设备授权登录
│   │   ├── upload_post.py      # Upload-Post 一键发布（TikTok/IG/YouTube）
│   │   ├── version_checker.py  # 版本检查
│   │   ├── data/azure_voices.json        # Azure TTS 音色目录
│   │   └── utils/video_effects.py        # 视频特效（缩放/转场等）
│   └── utils/                  # 通用工具
│       ├── file_security.py    # 路径安全校验（防目录穿越，供 /download、/stream 用）
│       ├── subtitle_reader.py / subtitle_writer.py   # srt 读写
│       ├── logging_utils.py    # loguru 配置
│       └── utils.py            # 杂项工具（时间格式化、路径等）
├── webui/
│   ├── Main.py                 # ★ Streamlit 页面主体（约 8000 行：参数面板、生成按钮 L7981+、设置弹窗）
│   ├── kimi_login.py           # Kimi Code 浏览器登录辅助
│   ├── styles.css              # 界面样式
│   ├── .streamlit/config.toml  # Streamlit 服务配置（端口等）
│   └── i18n/*.json             # 14 种语言翻译（zh/en/ja/ko…）
├── test/
│   ├── test_main.py            # 入口测试
│   └── services/               # 171 个 pytest 用例（按模块命名，如 test_llm.py、test_bgm.py、test_task*）
├── docs/
│   ├── skill/                  # Agent Skill（SKILL.md + mpt_agent.py，AI Agent 调用入口）
│   ├── loomloom/               # LoomLoom 集成 JSON 模板
│   ├── sponsors/               # 赞助方 logo
│   └── *.jpg / *.ipynb         # 界面截图、Colab 笔记本、音色清单
└── resource/songs/*.mp3        # 内置 BGM 曲库（约 30 首，随机/固定选取）
```

运行时生成（不入库）：`storage/`（任务目录与缓存）、`.venv/`、`config.toml`、`models/`（whisper 模型）。

## 3. 核心模块职责速查

| 模块 | 职责 | 二开关注点 |
|---|---|---|
| `main.py` | 启动 uvicorn 服务 | 改端口/日志去 config.toml，不改这里 |
| `app/config/config.py` | 配置加载单例 + WebUI 运行期回写队列 | **配置启动时读取**，改 config.toml 必须重启进程 |
| `app/controllers/v1/video.py` | 视频任务 API：`create_video()` → `create_task(stop_at="video")` | 新增 API 端点从这里抄结构 |
| `app/controllers/manager/` | 任务队列：内存版 / Redis 版可切换 | 并发上限来自 `max_concurrent_tasks` |
| `app/models/schema.py` | `VideoParams` 全量参数模型 + 枚举 | 加新参数先改这里，再改 config/WebUI |
| `app/models/llm_provider.py` | LLM 注册表（默认 base_url/模型名） | 接新 LLM 服务商只需在此注册 |
| `app/services/task.py` | 流水线编排（6 个阶段函数） | 一切生成逻辑的调度中心 |
| `app/services/llm.py` | 文案/关键词生成，服务商路由 | 改提示词/换模型行为在这 |
| `app/services/video.py` | moviepy/FFmpeg 合成 | 剪辑逻辑（拼接、转场、字幕烧制） |
| `app/services/voice.py` / `subtitle.py` | TTS 与字幕 | `sub_maker` 是两者之间的关键中间对象 |
| `app/services/state.py` | 任务状态持久化 | 状态流转与断点恢复 |
| `webui/Main.py` | Streamlit 全部界面 | 参数默认值同步改 `[ui]` 段 |

## 4. 视频生成流水线调用链

### 4.1 三条入口 → 汇聚到 tm.start

| 入口 | 调用链 |
|---|---|
| **API** | `POST /api/v1/videos` → `app/controllers/v1/video.py::create_video()`（L208）→ `create_task(..., stop_at="video")`：生成 task_id、写初始任务状态 → `task_manager.add_task(tm.start, ...)` 后台执行（L228-264）；失败回滚状态并返回 429/400 |
| **WebUI** | `webui/Main.py::"生成视频"按钮`（L7981-7999）→ `_prepare_generation_task()`（校验主题/脚本非空）→ `app/services/webui_task.py::_run_generation()`（L55-98，线程中调用 `tm.start(...)`，日志回传界面）——**进程内直调，不走 HTTP** |
| **CLI** | `cli.py`（L1739-1768）：解析参数生成 task_id → 调 `tm.start(...)` → 成功/失败输出 JSON 结果与退出码 |

> `tm` 即 `app/controllers/manager/` 里的任务管理器（默认 memory_manager，`enable_redis=true` 时为 redis_manager）。

### 4.2 流水线六阶段（app/services/task.py::start，L1510）

`start()`（L1510-1526）初始化：任务状态置 `TASK_STATE_PROCESSING`、进度归零。之后按阶段推进，每步都通过 `state.update_task()` / `patch_task()`（`state.py` L89-126）持久化状态与进度：

| 阶段 | task.py 函数 | 调用的 service | 产物 |
|---|---|---|---|
| 1. 生成文案 | `generate_script()`（L1649-1679，细节 L352-424） | `llm.generate_script()`（llm.py，服务商由 `llm_provider` 决定） | 文案文本 |
| 2. 生成关键词 | 同上，非 local 素材源时执行 | `llm.generate_terms()` | 关键词列表 |
| — 落盘 | `save_script_data()` | `task_artifacts.py` | `task.json`（script/terms/params） |
| 3. TTS 配音 | `generate_audio()`（L1696，细节 L545-640） | `voice.tts()`（voice.py，支持自定义音频直传） | `audio.mp3` + 时长 + `sub_maker` |
| 4. 生成字幕 | `generate_subtitle()`（细节 L643-714） | `subtitle_provider = "edge"`：用 TTS 的 `sub_maker` 时间轴；`"whisper"`：faster-whisper 本地转写 | `subtitle.srt`（校验时间轴） |
| 5. 获取素材 | `get_video_materials()`（L1772，细节 L717-902） | 按 `video_source` 分支：local 直用 / loomloom / pexels·pixabay·coverr 走 `material.download_videos()`（material.py）/ 付费文生视频源（ofox、muapi 等，各独立 service） | 素材片段（下载到素材目录或任务目录） |
| 6. 合成出片 | `generate_final_videos()`（细节 L971-1136） | ① `video.combine_videos()`（video.py）→ `combined-*.mp4`（素材拼接/裁剪/变速）；② 可选 `bgm.py` 混入背景音乐；③ `video.generate_video()`（video.py L1538-1560）烧录字幕、合并音轨 → **`final-*.mp4`** | 最终视频 |

### 4.3 任务目录与产物（storage/tasks/<task_id>/）

- `task.json` — 任务参数与文案/关键词元数据
- `audio.mp3` — 配音
- `subtitle.srt` — 字幕
- 素材片段 — 素材目录由 `material_directory` 控制（默认 `storage/cache_videos` 共享缓存）
- `combined-*.mp4` — 中间拼接产物
- `final-*.mp4` — **最终成片**，通过 `/api/v1/download/{file_path}` 下载（经 `utils/file_security.py` 防穿越校验）

### 4.4 状态流转

`const.py` 定义任务状态常量；`state.py::update_task()/patch_task()` 写入状态、进度（%）、当前失败阶段（`failed_stage`）与错误信息；每阶段失败都会标记后抛出，由入口层决定回滚（API 返回 4xx）或界面提示。

## 5. config.toml 字段说明

> 模板见 `config.example.toml`（含双语注释）。首次运行自动生成 `config.toml`；改完需重启服务生效。

### 5.1 顶层（API 服务）

| 字段 | 含义 |
|---|---|
| `log_level` | 日志级别：DEBUG / INFO / WARNING / ERROR |
| `listen_host` | API 监听地址；`127.0.0.1` 仅本机，`0.0.0.0` 对局域网开放 |
| `listen_port` | API 端口（默认 8080，文档在 `/docs`） |

### 5.2 `[app]` 通用

| 字段 | 含义 |
|---|---|
| `api_key` | API 路由与任务产物下载保护；留空=本地免认证，配置后客户端需带 `x-api-key` 请求头 |
| `hide_config` | 隐藏 WebUI 基础配置面板 |
| `script_generation_backend` | 文案后端：`"local"`（默认 LLM 流程）或 `"loomloom"`（SkillBot 多候选） |
| `loomloom_*` | LoomLoom 集成：base_url、token、各超时/轮询间隔 |
| `edge_tts_timeout` | 单次 Edge TTS 流式请求超时秒数（0=不超时） |
| `tls_verify` | 外部 API / 素材下载是否校验 TLS 证书（默认 true） |

### 5.3 素材源（`[app]` 段内）

- `video_source` — 素材来源，可选：`pexels`（默认）/ `pixabay` / `coverr` / `wavespeed` / `volcengine_seedance` / `ofox` / `metaso_minimax` / `muapi` / `loomloom` / `openai_image` / `local`
- 图库类：`pexels_api_keys` / `pixabay_api_keys` / `coverr_api_keys`（数组，支持多 key 轮换；Pexels 注册：pexels.com/api）
- 文生视频类（按次计费）：`wavespeed_api_keys`（含 model 与时长范围）、`volcengine_seedance_*`（火山 Ark：key/base_url/model/resolution/时长/轮询/超时）、`ofox_*`（key/model/resolution/provider/时长）、`muapi_*`（key/endpoint/resolution）、`metaso_minimax_*`（MiniMax H3 专用 key，勿与 LLM key 混用）
- `openai_image_*` — 任意 OpenAI 兼容 `/images/generations` 端点生图（key/base_url/model/size/prompt 模板）
- `twelvelabs_api_keys` — 语义排序/视频 QA（需 `uv sync --extra twelvelabs`）；`twelvelabs_rerank_terms` 开关
- `match_materials_to_script` — 素材按文案顺序匹配（默认 false）
- `sonilo_api_key` — AI 原创配乐（Sonilo）

### 5.4 LLM 服务商

- `llm_provider` — 当前服务商（如 `moonshot`、`deepseek`、`openai`、`qwen`…）
- 每家服务商均为"四件套"字段：`<provider>_api_key`（必填）、`<provider>_base_url` / `<provider>_model_name`（**留空跟随 `app/models/llm_provider.py` 注册表默认**，仅中转时显式覆盖）
- 特殊项：`claude_code_*`（用本地 claude CLI，无需 API key）、`kimi_code_*`（OAuth 登录后自动存 token）、`litellm_model_name`（litellm 走环境变量凭证）、`ollama_*`（本地模型）

### 5.5 语音 / 字幕 / 视频

| 字段 | 含义 |
|---|---|
| `subtitle_provider` | 字幕方式：`"edge"`（TTS 时间轴，默认）或 `"whisper"`（本地转写，首次自动下模型） |
| `ffmpeg_path` | FFmpeg 可执行文件路径；留空自动检测/下载 |
| `ffmpeg_concat_timeout_seconds` | 拼接卡死超时（默认 3600s），防任务卡在 50% |
| `ffmpeg_tts_timeout_seconds` | 单次 FFmpeg 音频转换超时（默认 600s） |
| `material_concurrency` | 素材搜索下载并发（1/2/4/6/8，默认 1，过高易触发限流） |
| `video_clip_concurrency` | 剪辑/编码并发（同上，默认 1） |
| `video_codec` | 编码器：libx264（默认）/ h264_nvenc / h264_amf 等硬件编码，不支持自动回退 |

### 5.6 存储与任务运行

| 字段 | 含义 |
|---|---|
| `endpoint` | 生成视频下载链接的基础 URL；留空用当前 API 地址 |
| `material_directory` | 素材存储：空=共享缓存 `storage/cache_videos`；`"task"`=存任务目录；绝对路径=自定义 |
| `enable_redis` + `redis_host/port/db/password` | 启用 Redis 任务状态（默认关闭，纯本地文件） |
| `max_concurrent_tasks` / `max_queued_tasks` | API 任务并发上限（默认 5）/ 排队上限（默认 100） |

### 5.7 跨平台发布（Upload-Post）

`upload_post_enabled`（总开关）、`upload_post_api_key` / `upload_post_username`、`upload_post_platforms`（tiktok/instagram/youtube）、`upload_post_auto_upload`（成片后自动发布）、`upload_post_youtube_privacy_status`、`upload_post_youtube_made_for_kids`、`upload_post_max_pending_tasks`。

### 5.8 其他分区

- `[whisper]`：`model_size`（默认 large-v3）、`device`（cpu/cuda）、`compute_type`（int8/float16）、`initial_prompt`（术语偏置）
- `[proxy]`：素材请求代理 `http` / `https`（格式 `http://user:pass@host:port`）
- TTS 服务商段：`[azure]`（speech_key/region，与 Azure OpenAI 无关）、`[siliconflow]`、`[minimax_tts]`（key/model/voice/采样率）、`[elevenlabs]`（含配乐 music_* 字段）、`[chatterbox]` / `[kokoro]`（本地 OpenAI 兼容 TTS 服务）、`[fish_audio]`、`[voxcpm]`
- `[ui]`：WebUI 偏好（`hide_log`、`open_task_folder_on_completion`，以及语言、字幕字体/颜色/位置、TTS 音色、BGM 音量等默认值——多数由界面自动写回）

---

*本文档由 AI 通读源码生成（2026-10-11），结构与行号对应 v1.3.8（commit 4204d0f）。大改模块后请同步更新。*
