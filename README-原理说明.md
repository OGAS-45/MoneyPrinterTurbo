# MoneyPrinterTurbo 原理说明

这份说明不是安装手册，而是给新人和新 Agent 看的“项目工作原理地图”。

如果你只记住一句话：

> 这个项目本质上是一个“把文本需求变成短视频”的流水线系统，WebUI、API、CLI 只是不同入口，真正干活的是同一套服务层。

## 这个项目到底在做什么

用户输入一个视频主题、关键词或完整脚本后，项目会按顺序完成这些事：

1. 让大模型生成脚本或素材检索词
2. 去本地或在线素材源拿视频片段
3. 生成配音或直接使用用户提供的音频
4. 生成字幕
5. 把素材、音频、字幕、背景音乐合成最终视频
6. 可选地把结果上传到外部平台

所以它不是单点工具，而是一条完整的视频生产线。

## 它是怎么跑起来的

### 1. 启动入口

项目有三个常见入口：

- `main.py`：启动 API 服务
- `webui/Main.py`：启动 Streamlit 界面
- `cli.py`：命令行生成视频

API 的启动逻辑很薄，只负责把 `app.asgi:app` 交给 Uvicorn：`main.py`

### 2. FastAPI 应用是怎么组装的

真正的 API 应用在 `app/asgi.py` 里创建：

- 构造 `FastAPI`
- 挂载路由
- 注册异常处理器
- 开启 CORS
- 把静态目录挂出来
- 在启动时恢复未完成的跨平台发布任务

这意味着：

- `/docs` 是自动生成的接口文档
- `/tasks` 对应任务产物的静态访问
- `/` 会挂载前端静态资源

### 3. 路由层负责什么

路由层只做“请求分发”，不做重计算。

#### 视频相关接口

`app/controllers/v1/video.py` 里包含最核心的接口：

- `/videos`：生成完整视频
- `/subtitle`：只生成字幕
- `/audio`：只生成音频
- `/tasks`：查询任务列表和单个任务
- `/musics`：上传/查询背景音乐
- `/video_materials`：上传/查询本地素材
- `/stream` 和 `/download`：播放或下载最终视频

#### 文本生成相关接口

`app/controllers/v1/llm.py` 主要负责：

- 生成脚本
- 生成素材关键词
- 生成社媒发布文案

### 4. 真正的“流水线”在哪里

核心编排在 `app/services/task.py`。

可以把它理解成一条工作流：

```text
请求进入 -> 建任务 -> 排队/执行 -> 生成脚本
-> 找素材 -> 合成音频 -> 生成字幕 -> 合成视频 -> 保存结果 -> 更新状态
```

这里的关键设计是：

- HTTP 请求不会一直阻塞到视频做完
- 任务会先写入状态，再进入后台线程/队列
- 任务状态会持续更新，前端或 API 可以随时查

这就是它能“做长任务”的原因。

## 底层原理：为什么它能稳定工作

### 1. 入口和业务逻辑分离

入口只负责接收请求，真正的工作放到服务层：

- `controllers` 负责 API 形状
- `services` 负责业务流水线
- `models` 负责数据结构
- `utils` 负责通用工具

这样做的好处是：

- 改界面不一定动核心视频逻辑
- 改视频流程不一定动 API 协议
- CLI、WebUI、API 能复用同一套服务

### 2. 任务状态是独立保存的

任务状态在 `app/services/state.py` 中管理，有两种实现：

- `MemoryState`：进程内字典
- `RedisState`：Redis 持久化

默认可以先用内存，想支持更稳的多进程或更长任务，就切 Redis。

这也是为什么项目能做：

- 任务列表
- 任务进度
- 失败状态追踪
- 任务删除控制

### 3. 产物按任务目录隔离

每个视频任务会有自己的任务目录，最终产物、字幕、音频、视频片段都放在对应目录里。

这样做的目的很直接：

- 不同任务不会互相覆盖
- 任务删除可以按目录清理
- `/stream` 和 `/download` 只需要根据任务目录拿文件

### 4. 配置是“先有模板，再复制到本地”

项目第一次运行时会把 `config.example.toml` 复制成 `config.toml`。

这意味着：

- 示例配置可以安全提交到仓库
- 真实 API Key 留在本地
- 不同机器只需要改自己的 `config.toml`

### 5. WebUI 和 API 用的是同一套底层能力

WebUI 并不是另一套逻辑，它只是把同样的服务层包装成页面操作。

`webui/Main.py` 里做的主要事情是：

- 读配置
- 初始化页面状态
- 组装表单
- 把用户输入转成 `VideoParams`
- 调用同一个任务流水线

所以你看到的是两个入口，实际上底层只是一条主线。

## 一张简图

```text
用户输入
  -> WebUI / API / CLI
  -> 参数校验
  -> 生成任务 ID
  -> 写入任务状态
  -> task.py 调度流水线
  -> llm / material / voice / subtitle / video
  -> 保存到 storage/tasks/<task_id>/
  -> 状态更新为成功或失败
```

## 新人最该先看哪些文件

如果你想最快看懂项目，不要从所有代码里乱翻，先按这个顺序看：

1. `README.md`：知道项目目标和启动方式
2. `app/asgi.py`：知道 FastAPI 怎么挂起来
3. `app/controllers/v1/video.py`：知道请求怎么进入任务系统
4. `app/services/task.py`：知道视频是怎么一步步做出来的
5. `app/services/state.py`：知道任务状态怎么保存
6. `app/models/schema.py`：知道请求参数和数据结构长什么样
7. `webui/Main.py`：知道页面是怎么映射到服务层的

## 如果你要改造，这里是最常改的地方

### 改“文案/脚本生成”

看：

- `app/services/llm.py`
- `app/controllers/v1/llm.py`
- `app/models/schema.py`

### 改“素材获取/筛选”

看：

- `app/services/material.py`
- `config.example.toml`
- `app/services/task.py`

### 改“语音合成”

看：

- `app/services/voice.py`
- `config.example.toml`
- `webui/Main.py`

### 改“字幕样式/字幕逻辑”

看：

- `app/services/subtitle.py`
- `app/models/schema.py`
- `webui/Main.py`

### 改“视频合成方式”

看：

- `app/services/video.py`
- `app/services/task.py`

### 改“任务状态/队列/并发”

看：

- `app/services/state.py`
- `app/controllers/manager/*`
- `app/services/task.py`

### 改“WebUI 界面”

看：

- `webui/Main.py`
- `webui/styles.css`
- `webui/i18n/*`

## 本地运行的最短路径

```bash
cd "/Users/wangpingchuan/ComateProjects/终身学习者/MoneyPrinterTurbo"
uv sync --frozen
uv run python main.py
```

如果要开 WebUI：

```bash
sh webui.sh
```

如果要走命令行：

```bash
uv run python cli.py --help
```

## 给新 Agent 的操作原则

- 先读 `config.example.toml`，再看业务代码
- 先找入口，再找服务层，不要直接在工具函数里乱改
- 先确认任务状态和目录约定，再碰删除、下载、流式播放
- 改动尽量只落在一个层次，不要同时改 UI、API 和底层算法
- 如果你要验证功能，优先跑最短闭环，而不是一上来全量重构

## 一句话总结

这个项目的核心是“把生成短视频这件事拆成可配置、可追踪、可替换的流水线”，所以真正重要的不是某一个页面，而是入口、任务状态、服务编排和产物目录这四件事。
