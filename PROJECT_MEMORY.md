# MoneyPrinterTurbo 项目记忆

## 文档用途

本文档用于帮助后续开发者和 AI 助手快速恢复项目上下文。

- 记录长期稳定的项目事实，不记录一次性任务进度。
- 代码、配置和测试结果优先级高于本文档。
- 架构或运行方式发生变化时，应同步更新本文档。

## 项目定位

MoneyPrinterTurbo 是一个基于 Python 的短视频自动生成工具。用户提供视频主题、关键词或自定义文案后，系统可以自动生成文案、配音、字幕、视频素材和背景音乐，并合成最终视频。

项目提供三种入口：

| 入口 | 文件 | 运行模型 |
| --- | --- | --- |
| WebUI | `webui/Main.py` | Streamlit 直接调用业务服务 |
| REST API | `main.py`、`app/asgi.py` | FastAPI 通过任务管理器异步执行 |
| CLI | `cli.py` | 命令行同步调用核心流水线 |

WebUI 不通过 REST API 调用后端。三种入口最终共享 `app/services` 中的业务逻辑。

## 技术栈

- Python `>=3.11,<3.13`
- FastAPI + Uvicorn
- Streamlit
- Pydantic
- MoviePy 2.x + FFmpeg
- Edge TTS、Azure Speech、Gemini、MiMo、SiliconFlow
- faster-whisper
- Redis（可选）
- `uv` + `uv.lock`

主依赖定义在 `pyproject.toml`，`requirements.txt` 仅用于兼容旧的 pip 安装方式。

## 目录职责

```text
MoneyPrinterTurbo/
├── app/
│   ├── config/          # TOML 配置加载、保存和环境适配
│   ├── controllers/     # FastAPI 路由及任务队列管理
│   ├── models/          # Pydantic 请求模型、枚举和异常
│   ├── services/        # 核心业务能力
│   └── utils/           # 路径、响应、文本和文件安全工具
├── webui/               # Streamlit 页面和多语言资源
├── resource/
│   ├── fonts/           # 字幕字体
│   ├── songs/           # 默认背景音乐
│   └── public/          # API 根路径静态页面
├── storage/             # 运行时生成，存放任务和素材缓存
├── test/                # unittest 测试
├── docs/                # 截图、部署说明和辅助文档
├── main.py              # API 启动入口
├── cli.py               # CLI 入口
└── config.example.toml  # 配置模板
```

## 核心流水线

核心编排函数是 `app/services/task.py::start()`：

1. `generate_script`：使用自定义文案，或调用 LLM 生成文案。
2. `generate_terms`：为在线素材源生成搜索关键词。
3. `generate_audio`：使用自定义音频，或调用 TTS 生成音频。
4. `generate_subtitle`：通过 Edge TTS 时间戳或 whisper 生成字幕。
5. `get_video_materials`：处理本地素材，或下载在线素材。
6. `generate_final_videos`：拼接素材、混合音频、渲染字幕和背景音乐。
7. 可选调用 Upload Post，将成品发布到 TikTok 或 Instagram。

CLI 可以通过 `--stop-at` 停在以下阶段：

```text
script -> terms -> audio -> subtitle -> materials -> video
```

本地素材模式不需要生成搜索关键词。

## 关键模块

| 模块 | 职责 |
| --- | --- |
| `app/services/task.py` | 完整视频生成流程编排 |
| `app/services/llm.py` | 多 LLM 适配、文案、关键词和社交元数据生成 |
| `app/services/voice.py` | TTS、音频时长和 Edge 字幕时间戳 |
| `app/services/subtitle.py` | whisper 转录、SRT 解析和字幕校正 |
| `app/services/material.py` | Pexels、Pixabay、Coverr 搜索与下载 |
| `app/services/video.py` | 素材预处理、拼接、字幕和最终视频渲染 |
| `app/services/state.py` | 内存或 Redis 任务状态 |
| `app/models/schema.py` | 视频参数和 API 数据模型 |
| `app/controllers/v1/video.py` | 视频任务、上传、查询、删除和流式播放 API |
| `app/controllers/manager/` | 并发任务和有界等待队列 |

## API 模型

所有业务接口前缀为：

```text
/api/v1
```

主要接口：

- `POST /api/v1/videos`：创建完整视频任务。
- `POST /api/v1/audio`：只生成音频。
- `POST /api/v1/subtitle`：生成音频和字幕。
- `GET /api/v1/tasks/{task_id}`：查询任务状态。
- `GET /api/v1/tasks`：分页查询任务。
- `DELETE /api/v1/tasks/{task_id}`：删除任务及产物。
- `POST /api/v1/scripts`：生成文案。
- `POST /api/v1/terms`：生成素材关键词。
- `POST /api/v1/social-metadata`：生成发布元数据。
- `GET/POST /api/v1/musics`：查询或上传背景音乐。
- `GET/POST /api/v1/video_materials`：查询或上传本地素材。

API 默认没有启用认证。CORS 未配置时允许全部来源，因此不能在没有额外访问控制的情况下直接暴露到公网。

## 任务与状态

- API 使用 `TaskManager` 创建后台线程。
- `max_concurrent_tasks` 控制同时执行数量。
- `max_queued_tasks` 限制等待队列，满载时返回 HTTP 429。
- 默认状态保存在进程内存中，进程重启后丢失。
- 启用 Redis 后，队列和状态都使用 Redis。
- 任务产物默认位于 `storage/tasks/<task_id>/`。
- 在线素材默认缓存在 `storage/cache_videos/`。

WebUI 直接同步调用 `task.start()`，不会进入 API 的任务队列。

## 配置规则

- 配置模板：`config.example.toml`
- 运行配置：`config.toml`
- 如果 `config.toml` 不存在，启动时会自动复制模板。
- WebUI 会调用 `config.save_config()`，将界面中的配置保存到 `config.toml`。
- API Key、发布凭据等敏感信息不得写入源码、日志或提交到 Git。

主要配置区：

| 配置区 | 内容 |
| --- | --- |
| `[app]` | LLM、素材源、字幕、FFmpeg、Redis、任务并发 |
| `[whisper]` | 本地转录模型和设备配置 |
| `[proxy]` | 网络代理 |
| `[azure]` | Azure Speech |
| `[siliconflow]` | SiliconFlow TTS |
| `[ui]` | WebUI 设置及可选发布配置 |

## 素材和输出

支持的在线素材源：

- Pexels
- Pixabay
- Coverr

支持本地视频和图片素材。上传文件必须限制在指定目录中，任何新增文件接口都应继续使用现有的路径校验和文件名清理逻辑，避免路径穿越。

视频比例：

- `9:16`：1080 × 1920
- `16:9`：1920 × 1080
- `1:1`：1080 × 1080

FFmpeg 编码器不可用时，视频模块会回退到 `libx264`。

## 常用命令

安装依赖：

```bash
uv python install 3.11
uv sync --frozen
```

启动 WebUI：

```bash
uv run streamlit run ./webui/Main.py --browser.gatherUsageStats=False
```

启动 API：

```bash
uv run python main.py
```

启动 CLI：

```bash
uv run python cli.py --video-subject "金钱的作用"
```

运行全部测试：

```bash
uv run python -m unittest discover -s test
```

运行单个测试：

```bash
uv run python -m unittest test.services.test_task
```

## 开发约束

- 修改业务流程前，先确认 WebUI、API 和 CLI 三个入口是否都会受到影响。
- 新参数应优先加入 `VideoParams`，并检查精简模型 `AudioRequest`、`SubtitleRequest` 是否需要同步。
- 不要让只生成音频或字幕的请求访问完整视频模型才有的字段。
- 外部 API 和 SDK 调试必须对照官方文档或参考实现，逐字段核对请求。
- 视频处理代码必须显式关闭 MoviePy clip，避免文件描述符泄漏。
- 网络、素材或非核心功能失败时，应返回可理解的错误并保持任务状态一致。
- 不要通过禁用 TLS 校验来掩盖证书问题。
- 不要把 API Key、Token、密码或用户绝对路径暴露给客户端。
- 新增素材源时，需要同时处理搜索、下载、配置、WebUI 选择和测试。
- 新增 TTS 时，需要处理声音列表、名称解析、音频生成、字幕时间戳能力和降级路径。

## 验证策略

改动后至少执行与改动范围对应的测试：

| 改动范围 | 最低验证 |
| --- | --- |
| 参数或 CLI | `test/services/test_cli.py` 及相关模型测试 |
| 流水线 | `test/services/test_task.py` |
| 视频处理 | `test/services/test_video.py` |
| 素材下载 | `test/services/test_material.py` |
| TTS | `test/services/test_voice.py` |
| 字幕 | `test/services/test_subtitle.py` |
| 状态管理 | `test/services/test_state.py` |
| WebUI 多语言 | `test/services/test_webui_i18n.py` |

涉及真实 LLM、TTS、素材下载或完整视频生成时，还需要单独说明外部凭据、网络和 FFmpeg 是否具备，不能用单元测试通过代替端到端验证。

## 已知边界

- README 所称 MVC 实际上更接近控制器、模型、服务分层。
- 默认内存状态不支持进程重启恢复，也不适合多实例部署。
- Redis 状态以字符串保存，再尝试恢复原始类型，修改状态结构时需要关注序列化兼容性。
- WebUI 是同步长任务，生成期间会占用当前 Streamlit 执行流程。
- API 使用线程执行 CPU 和 I/O 混合任务，并发数应根据机器资源控制。
- 完整生成依赖外部 LLM、TTS、素材服务及 FFmpeg，网络和配额均可能导致任务失败。
- Whisper 首次使用可能需要下载较大的模型文件。
- 自动发布属于外部写操作，必须保持默认关闭并由用户明确配置。
