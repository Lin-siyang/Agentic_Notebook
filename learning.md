# RAG Notebook Project Learning Plan

这份清单面向已经学过 LangChain 和 RAG 基础流程的人：你不需要从“什么是向量库/检索器/Prompt”重新开始，而是把重点放在这个项目如何把 RAG 做成一个完整产品。

项目可以按三条主线学习：

- FastAPI RAG 后端：文档处理、向量库、检索、重排、Agent、笔记 AI 能力。
- Django 用户服务：注册登录、JWT、用户信息、文件上传。
- Vue 前端：移动端页面、接口调用、SSE 流式展示、笔记编辑体验。

推荐顺序是先跑通服务，再读请求链路，最后拆 RAG 和业务模块。

## 0. 先建立项目地图

目标：知道每个目录负责什么，避免一开始陷进细节。

重点文件：

- `README.md`：项目定位、功能说明、启动方式。
- `backend/main.py`：FastAPI 应用入口，路由注册、CORS、启动初始化。
- `front/src/router/index.js`：前端页面路由。
- `front/src/config/api.js`：前端接口路径集中配置。
- `DjangoUserService/DjangoUserService/urls.py`：Django 总路由。
- `DjangoUserService/DjangoUserService/settings.py`：Django 配置、数据库、JWT、CORS。

学习任务：

- 画出三服务关系：Vue `3000`、FastAPI `8000`、Django `8001`。
- 标出哪些接口走 FastAPI，哪些接口走 Django。
- 理解 JWT 为什么由 Django 生成，却由 FastAPI 校验。

## 1. 跑通本地环境

目标：先让项目能启动，再学习代码。这个项目依赖 MySQL、Redis、ChromaDB、LLM/Ollama 或阿里云模型。

重点文件：

- `backend/pyproject.toml`
- `backend/requirements.txt`
- `backend/app/config/chroma.yaml`
- `backend/app/config/prompt.yaml`
- `DjangoUserService/pyproject.toml`
- `front/package.json`
- `front/vite.config.js`

学习任务：

- 分别安装 `backend`、`DjangoUserService`、`front` 的依赖。
- 准备两个 `.env`：一个给 FastAPI，一个给 Django。
- 确认 FastAPI 和 Django 的 JWT 密钥一致：
  - FastAPI 读取 `SECRET_KEY` 和 `ALGORITHM`。
  - Django 读取 `JWT_SECRET_KEY`。
- 启动 MySQL、Redis、Django、FastAPI、Vue。
- 访问：
  - `http://127.0.0.1:8000/docs`
  - `http://127.0.0.1:8001/docs/`
  - `http://127.0.0.1:3000`

建议先用 Ollama 跑通最小链路，再切到阿里云模型。

## 2. 学认证链路：Django 发 Token，FastAPI 验 Token

目标：理解整个项目的用户隔离基础。

重点文件：

- `DjangoUserService/apps/user/views.py`
- `DjangoUserService/apps/user/serializers.py`
- `DjangoUserService/apps/user/authentications.py`
- `DjangoUserService/apps/user/models.py`
- `backend/app/utils/auth_utils.py`
- `front/src/store/user.js`

学习任务：

- 从前端登录页面开始追踪：登录请求发到哪里，返回 token 后存在哪里。
- 阅读 Django 的 `LoginView`、`RegisterView`、`UserDetailView`。
- 阅读 `JWTTokenGenerator` 如何生成 token。
- 阅读 FastAPI 的 `get_current_user_id` 如何解析 Django token。
- 理解 `user_id` 如何贯穿笔记、知识库、聊天会话。

检查点：

- 能说清楚“为什么 FastAPI 不自己登录，而是信任 Django 的 JWT”。
- 能找到每个需要登录接口里 `Depends(get_current_user_id)` 的作用。

## 3. 学 FastAPI 总入口和生命周期

目标：知道后端启动时初始化了哪些东西。

重点文件：

- `backend/main.py`
- `backend/app/core/background_init.py`
- `backend/app/db/db_config.py`
- `backend/app/db/redis_config.py`
- `backend/app/core/failed_response_register.py`
- `backend/app/core/success_response.py`
- `backend/app/core/rate_limit.py`

学习任务：

- 看 `main.py` 中注册了哪些 router。
- 理解启动事件里做了什么：
  - 初始化 MySQL 表。
  - 初始化会话管理器。
  - 连接 Redis。
  - 后台加载模型、NoteService、Reranker。
- 理解 `background_init.py` 为什么用 `asyncio.create_task` 后台加载重资源。
- 阅读统一响应格式 `success_response`。
- 阅读限流实现，理解它如何基于 Redis 和客户端 IP 计数。

注意点：

- `init_manager.note_service` 是后台初始化完成后才可用，启动初期请求可能遇到未就绪状态。
- `redis_config.py` 当前 Redis 地址是硬编码，学习时可以先接受，后续优化时改成环境变量。

## 4. 学数据库模型和会话持久化

目标：搞清楚 MySQL 里存什么，ChromaDB 里存什么。

重点文件：

- `backend/app/models/chat_history.py`
- `backend/app/models/note.py`
- `backend/app/models/review_record.py`
- `backend/app/db/db_config.py`
- `backend/app/services/database_session_manager.py`
- `backend/app/router/chat_service.py`

学习任务：

- 阅读 SQLAlchemy 的 `Base`、`ChatSession`、`ChatMessage`。
- 阅读 `Note` 和 `ReviewRecord` 的字段设计。
- 理解会话历史为什么存 MySQL，而知识库内容为什么进 ChromaDB。
- 对比关系型数据和向量数据的职责：
  - MySQL：强结构、用户数据、会话、复习记录。
  - ChromaDB：语义检索、文档 chunk、笔记向量。

检查点：

- 能画出 `users -> notes -> review_records` 的业务关系。
- 能说清楚聊天历史如何按 `session_id` 保存和读取。

## 5. 学知识库上传和文档处理

目标：掌握“文件上传 -> 文档解析 -> 切片 -> 向量入库”的完整链路。

重点文件：

- `backend/app/router/knowledge_router.py`
- `backend/app/router/knowledge_service.py`
- `backend/app/rag/vector_store.py`
- `backend/app/rag/text_spliter.py`
- `backend/app/rag/document_handler/processor.py`
- `backend/app/utils/file_handler.py`
- `backend/app/utils/pdf_multimodal_loader.py`
- `backend/app/utils/image_extractor.py`
- `backend/app/utils/vision_service.py`
- `backend/app/rag/md5_manager/md5_store.py`
- `backend/app/config/chroma.yaml`

学习任务：

- 从 `/knowledge/add/multiple/stream` 入口追踪上传流程。
- 理解支持的文件类型：`txt`、`pdf`、`md`、`pptx`、`docx`。
- 阅读文件去重逻辑：MD5 如何记录，重复上传如何处理。
- 阅读文档切片配置：
  - `chunk_size`
  - `chunk_overlap`
  - `separators`
- 阅读 ChromaDB collection 如何创建、写入、查询、删除。
- 如果项目启用了多模态 PDF，重点看图片提取和视觉模型摘要如何参与知识库构建。

检查点：

- 能从上传一个 PDF 讲到它最后如何变成 ChromaDB 里的多个 chunk。
- 能解释 metadata 里为什么必须保存 `user_id`。

## 6. 学 RAG 查询链路

目标：把你学过的 RAG 基础流程对照到项目实现里。

重点文件：

- `backend/app/router/chat.py`
- `backend/app/router/chat_service.py`
- `backend/app/rag/rag_service.py`
- `backend/app/rag/vector_store.py`
- `backend/app/rag/retrievers/hybrid_retriever.py`
- `backend/app/rag/retrievers/empty_retriever.py`
- `backend/app/rag/reorder_service.py`
- `backend/app/prompt/rag_summarize.txt`
- `backend/app/prompt/reorder_prompt.txt`

学习任务：

- 从 `/chat/rag/query` 入口开始读。
- 理解 `RagService` 的几个关键步骤：
  - 初始化 retriever。
  - HyDE 生成假设性回答。
  - 向量库检索知识库文档。
  - 同时检索笔记库。
  - 合并 note 和 knowledge_base 来源。
  - reranker 重排。
  - 分批摘要，再生成最终回答。
- 阅读 `hybrid_retriever.py`，理解向量检索和 BM25 的混合检索思路。
- 阅读 `reorder_service.py`，理解重排模型在 RAG 中的位置。

检查点：

- 能把基础 RAG 的 `load -> split -> embed -> retrieve -> generate` 对应到项目脚本。
- 能解释这个项目相比基础 RAG 增加了什么：HyDE、混合检索、重排、笔记库融合、SSE thinking。

## 7. 学 Agent 对话

目标：理解项目如何把 RAG 能力包装成 Agent 工具。

重点文件：

- `backend/app/router/chat.py`
- `backend/app/agent/agent.py`
- `backend/app/agent/agent_tools.py`
- `backend/app/agent/agent_middleware.py`
- `backend/app/prompt/main_prompt.txt`
- `backend/app/schemas/models.py`
- `front/src/views/AIChat.vue`
- `front/src/store/session.js`

学习任务：

- 从 `/chat/agent/query/stream` 开始追踪。
- 阅读 `get_agent_stream_response` 如何流式返回。
- 阅读 `agent_tools.py` 里注册了哪些工具：
  - RAG 查询。
  - 笔记搜索。
  - 笔记创建。
  - 分类统计。
  - 相关笔记。
- 理解 Agent 如何拿到 `session_id` 和 `user_id`。
- 阅读前端 `AIChat.vue` 如何处理 SSE 消息和 thinking 过程。

检查点：

- 能区分 `/chat/rag/query` 和 `/chat/agent/query/stream` 的用途。
- 能说明 Agent 工具和普通 RAG 查询的关系。

## 8. 学笔记系统：CRUD + 向量双写 + AI 标签

目标：理解这个项目从“RAG demo”变成“AI 笔记产品”的关键部分。

重点文件：

- `backend/app/router/note_router.py`
- `backend/app/services/note_service.py`
- `backend/app/models/note.py`
- `backend/app/models/review_record.py`
- `backend/app/prompt/auto_tag_prompt.txt`
- `backend/app/prompt/autocomplete_prompt.txt`
- `backend/app/prompt/write_assistant_prompt.txt`
- `front/src/views/NoteList.vue`
- `front/src/views/NoteEditor.vue`
- `front/src/components/MarkdownEditor.vue`
- `front/src/components/InlineCompletion.vue`
- `front/src/components/RelatedNotes.vue`

学习任务：

- 从 `/note/create` 看创建笔记流程：
  - MySQL 写入。
  - ChromaDB 写入笔记向量。
  - 后台生成标签和复习记录。
- 阅读更新笔记时如何同步更新向量。
- 阅读删除笔记时如何删除 MySQL 和 ChromaDB 数据。
- 阅读自动标签 `_auto_tag_and_review`。
- 阅读联想补全 `/note/autocomplete`。
- 阅读写作助手 `/note/assist/stream`。
- 阅读相关笔记 `/note/{note_id}/related`。

注意点：

- `search_notes` 使用了 `filter={"user_id": user_id, "doc_type": "note"}`。
- `get_related_notes` 的笔记相似检索目前没有加 `user_id` filter，后续优化时应补上。

检查点：

- 能说明为什么笔记既要进 MySQL，也要进 ChromaDB。
- 能说明自动标签为什么适合做后台异步任务。

## 9. 学每日复习

目标：理解间隔重复如何和笔记系统结合。

重点文件：

- `backend/app/router/review_router.py`
- `backend/app/services/review_service.py`
- `backend/app/models/review_record.py`
- `backend/app/prompt/review_question_prompt.txt`
- `front/src/views/DailyReview.vue`
- `front/src/components/ReviewCard.vue`

学习任务：

- 阅读今日待复习接口。
- 阅读复习完成后如何更新：
  - `review_count`
  - `interval_days`
  - `next_review_at`
  - `last_reviewed_at`
- 阅读 AI 如何基于笔记生成复习问题。
- 对照 `INTERVALS = [1, 2, 4, 7, 15, 30]` 理解间隔重复策略。

检查点：

- 能讲清楚一篇新笔记从创建到第一次复习提醒的过程。
- 能说明复习记录和笔记记录为什么分表。

## 10. 学前端请求与页面组织

目标：看懂前端如何把后端能力组织成移动端产品。

重点文件：

- `front/src/main.js`
- `front/src/App.vue`
- `front/src/router/index.js`
- `front/src/config/api.js`
- `front/src/store/user.js`
- `front/src/store/session.js`
- `front/src/store/theme.js`
- `front/src/store/language.js`
- `front/src/views/Login.vue`
- `front/src/views/Register.vue`
- `front/src/views/AIChat.vue`
- `front/src/views/KnowledgeBase.vue`
- `front/src/views/NoteList.vue`
- `front/src/views/NoteEditor.vue`
- `front/src/views/DailyReview.vue`
- `front/src/components/TabBar.vue`

学习任务：

- 阅读页面路由，理解主页面结构。
- 阅读 Pinia store，理解 token、用户、会话状态如何保存。
- 阅读 `vite.config.js`，理解代理如何把请求转到 `8000` 或 `8001`。
- 阅读 `AIChat.vue`，学习 SSE 前端处理。
- 阅读 `KnowledgeBase.vue`，学习文件上传和处理进度展示。
- 阅读 `NoteEditor.vue`，学习 Markdown 编辑、自动保存、AI 补全、相关笔记展示。

检查点：

- 能从点击“上传文档”追踪到后端 `knowledge_router`。
- 能从发送聊天消息追踪到后端 Agent SSE。
- 能从保存笔记追踪到 MySQL + ChromaDB 双写。

## 11. 学配置、Prompt 和模型工厂

目标：理解项目如何切换 LLM、Embedding、Vision、Reranker。

重点文件：

- `backend/app/utils/factory.py`
- `backend/app/utils/prompt_loader.py`
- `backend/app/utils/config.py`
- `backend/app/utils/config_handler.py`
- `backend/app/config/prompt.yaml`
- `backend/app/config/chroma.yaml`
- `backend/app/prompt/main_prompt.txt`
- `backend/app/prompt/rag_summarize.txt`
- `backend/app/prompt/auto_tag_prompt.txt`
- `backend/app/prompt/autocomplete_prompt.txt`
- `backend/app/prompt/write_assistant_prompt.txt`
- `backend/app/prompt/review_question_prompt.txt`

学习任务：

- 阅读 `ChatModelFactory`，理解 `LLM_TYPE=ALIYUN/OLLAMA` 如何切换。
- 阅读 `EmbedModelFactory`，理解 embedding 模型如何切换。
- 阅读 `VisionModelFactory`，理解多模态 PDF 处理为什么需要单独模型。
- 阅读 Prompt 配置如何从 `prompt.yaml` 映射到具体 txt 文件。
- 修改一个 Prompt，观察接口返回变化。

检查点：

- 能独立把模型从阿里云切换到 Ollama。
- 能独立新增一个 Prompt 文件并通过 `prompt.yaml` 加载。

## 12. 后续优化练习

完成阅读后，可以用这些任务检验你是否真正理解项目。

优先级 1：稳定性

- 给 `init_manager.note_service`、`chat_model`、`reorder_service` 加 ready 检查。
- 如果服务未就绪，接口返回明确的 `503`，而不是 `NoneType` 异常。
- 涉及文件：
  - `backend/app/core/background_init.py`
  - `backend/app/router/note_router.py`
  - `backend/app/router/chat.py`
  - `backend/app/router/chat_service.py`

优先级 2：配置统一

- 把 Redis 配置从硬编码改成环境变量。
- 统一 README 和代码里的阿里云 API Key 名称。
- 明确 FastAPI `SECRET_KEY` 与 Django `JWT_SECRET_KEY` 的关系。
- 涉及文件：
  - `backend/app/db/redis_config.py`
  - `backend/app/utils/factory.py`
  - `backend/app/utils/auth_utils.py`
  - `README.md`

优先级 3：用户隔离

- 给相关笔记检索补充 `user_id` filter。
- 检查所有 ChromaDB 查询是否都带用户隔离条件。
- 涉及文件：
  - `backend/app/services/note_service.py`
  - `backend/app/rag/vector_store.py`
  - `backend/app/rag/rag_service.py`

优先级 4：前端体验

- 给后端初始化中、模型加载中、RAG 处理中增加更清晰的前端状态。
- 给上传失败、token 过期、Redis/MySQL 未连接做友好提示。
- 涉及文件：
  - `front/src/views/AIChat.vue`
  - `front/src/views/KnowledgeBase.vue`
  - `front/src/views/NoteEditor.vue`
  - `front/src/store/user.js`

优先级 5：测试

- 为认证、笔记 CRUD、知识库上传、RAG 查询写最小测试。
- 涉及文件：
  - `backend/app/router/*`
  - `backend/app/services/*`
  - `DjangoUserService/apps/user/tests.py`
  - `DjangoUserService/apps/file/tests.py`

## 建议学习节奏

### 第 1 天：跑通和画图

- 跑通三个服务。
- 注册/登录一个用户。
- 上传一个简单 txt 文档。
- 发起一次 RAG 查询。
- 画出前端、FastAPI、Django、MySQL、Redis、ChromaDB 的关系图。

### 第 2 天：认证和接口

- 读 Django 登录注册。
- 读 FastAPI JWT 校验。
- 用浏览器 DevTools 看请求 header 里的 token。
- 手动调用 `/chat/rag/query`，确认未登录会失败。

### 第 3 天：知识库 RAG

- 读上传、切片、向量入库。
- 读 RAG 查询、HyDE、重排、摘要。
- 对照你学过的基础 RAG 流程做一张映射表。

### 第 4 天：Agent 和会话

- 读 Agent 工具注册。
- 读 SSE 返回。
- 读会话持久化。
- 理解普通 RAG 查询和 Agent 查询的区别。

### 第 5 天：笔记系统

- 读笔记 CRUD。
- 读笔记向量双写。
- 读自动标签和写作助手。
- 修复 `get_related_notes` 缺少用户 filter 的问题。

### 第 6 天：复习系统和前端

- 读每日复习后端。
- 读 `DailyReview.vue`。
- 读 `NoteEditor.vue` 和 `AIChat.vue`。
- 理解前端如何展示 AI 流式响应。

### 第 7 天：做一个小改造

任选一个：

- 新增一个“只搜索笔记”的 Agent 工具。
- 新增一个“导出全部笔记”的接口。
- 给 RAG 查询返回来源文档列表。
- 给知识库上传页面增加失败原因展示。
- 把 Redis 配置改成 `.env` 驱动。

## 最终掌握目标

学完这个项目后，你应该能做到：

- 独立解释一个完整 RAG 产品的工程结构。
- 独立追踪一次请求从前端到后端再到数据库/向量库的全过程。
- 独立替换 LLM 和 embedding 模型。
- 独立新增一个 FastAPI router 和一个前端页面。
- 独立修复用户隔离、配置、初始化时序这类工程问题。
- 基于这个项目裁剪出自己的 RAG 知识库、AI 笔记或企业知识问答系统。

