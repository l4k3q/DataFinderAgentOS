# DataFinderAgentOS

> 智能瞭望与问数系统（v1.0.0）—— 数据采集 · 模型引擎 · 数字员工 · 自然语言问数 · 舆情风控 · 可视化大屏一体化全栈平台

一个**轻依赖、开箱即用**的全栈项目：`requirements.txt` 仅声明 4 个第三方包（tornado / requests / tiktoken / fpdf2），数据层由 Python 标准库 `sqlite3` 直连，前端资源（Bootstrap 5 + LayUI + ECharts 5 / echarts-gl）全部本地化，无 npm 构建链。启动时自动建表、自动迁移、自动灌入种子数据（角色权限树 / 瞭源 / 数字员工 / 敏感词库 / 系统设置），`python app.py` 一条命令即可跑通全链路。

**一键部署：**

```powershell
python -m venv venv
venv\Scripts\pip install -r requirements.txt
venv\Scripts\pip install numpy opencv-python    # 人脸登录模块为顶层导入，启动必需（未列入 requirements.txt）
# 可选：venv\Scripts\pip install edge-tts crawl4ai   # 语音合成 / 浏览器级深度采集
venv\Scripts\python app.py
# 前台:  http://127.0.0.1:10010/           （注册普通用户进入）
# 后台:  http://127.0.0.1:10010/admin/login  （默认 admin / admin888）
```

> LLM 能力（问数 / 意图识别 / 分析研判）需登录后台「模型引擎」，将预置 DeepSeek 模型的占位 API Key 替换为真实密钥；未配置时系统自动降级为关键词意图识别 + 预定义查询 + 兜底话术，核心流程不中断。

---

## 技术栈总览

| 层次 | 技术选型 | 依赖数 |
|---|---|---|
| Web 框架 | Tornado ≥ 6.5（单进程 + IOLoop，HTTP + WebSocket 双协议） | 1 |
| 数据库 | SQLite（`database/finderos.db` 单文件，20 张表） | 0（标准库 sqlite3） |
| 模板引擎 | Tornado Template（模板继承 + 自动转义） | 0（内置） |
| 出站 HTTP | requests（LLM 调用 / 数字员工 API / 瞭源采集，支持 SSE 流式） | 1 |
| Token 估算 | tiktoken（cl100k_base 兜底） | 1 |
| PDF 导出 | fpdf2（对话记录导出，中文字体动态探测） | 1 |
| 人脸登录 | numpy + opencv-python（像素均值比对，简化实现） | 2（顶层导入，启动必需） |
| 语音合成 | edge-tts（zh-CN-XiaoxiaoNeural） | 软依赖 |
| 深度采集 | crawl4ai（headless 浏览器，60s 超时线程封装） | 软依赖 |
| 前端 | 原生 HTML/CSS/JS + Bootstrap 5 + LayUI + ECharts（本地静态资源） | 0 |

---

## 系统架构

```
app.py                          路由表（46 条）+ 服务器启动（端口 10010）
└── app/controllers/            C 层：鉴权 → 取参 → 调 M 层 → 选模板
    ├── BaseHandler             前台基类（secure cookie 会话）
    └── AdminBaseHandler        后台基类（RBAC 路由拦截 + 动态菜单注入）→ 各业务 Handler
└── app/models/                 M 层：
    ├── *Repository             数据访问（一个类对应一张表，参数化 SQL）
    └── *Service                业务编排（意图路由 / NL2SQL / 数字员工 / 深度采集 / 敏感词三级匹配）
└── app/templates/ + static/    V 层：Tornado 模板（前台 4 + 后台 25）+ 本地前端资源
└── database/finderos.db        SQLite 单文件（20 张表，启动自动迁移 + 种子）
```

**智能问数核心链路**（每条消息经过：敏感词扫描 → 意图识别 → 七路分发）：

```
用户消息 ──敏感词三级匹配──┬─命中→ 拦截 + 预警入库
                          └─安全→ 意图识别（@员工正则快路径 / LLM / 关键词兜底）
       ├─ digital_employee     → 数字员工（LLM 型：Prompt+MD知识库+crawl4ai / API 型：可视化 HTTP 配置）
       ├─ database_query       → NL2SQL：预定义查询 → LLM 生成 SQL → 安全校验 → 表格回传
       ├─ report_generation    → 查询 + 智能选型（bar/line/pie/scatter）→ ECharts option 回传
       ├─ data_search          → 数据仓库全文检索
       ├─ data_analysis        → 查询 + LLM 深度分析（[INSIGHTS] 洞察解析）→ 分析卡片回传
       ├─ relationship_mining  → 数据摘要 + LLM 关系图谱描述
       └─ general_chat         → 流式对话（30 条历史上下文，失败自动降级非流式）
```

---

## 功能模块

| 模块 | 入口 | 说明 |
|---|---|---|
| 前台工作台 | `/index` | 多会话智能问数、七路意图路由、@数字员工、表格/图表/分析卡片、流式输出、模型切换、TTS、摄像头手势控制 |
| 用户认证 | `/` 与 `/register` | 账密登录注册（PBKDF2-HMAC-SHA256，10 万次迭代 + 随机盐）+ 人脸注册/登录 |
| 对话导出 | `/export` | 会话记录 PDF 导出（SimHei/微软雅黑/宋体自适应） |
| 瞭望采集 | `/admin/watch` | 关键词 + 多瞭源勾选采集，四套解析器（百度/新浪/搜狗/360），橱窗勾选入库（自动敏感词扫描） |
| 瞭源管理 | `/admin/watch/source` | 采集源 CRUD、collect_config 参数映射（关键词/翻页参数名/解析器）、连通性测试 |
| 数据仓库 | `/admin/data` | 仓库检索、单条/批量深度采集（数字员工调度 + crawl4ai 兜底） |
| 采集管理 | `/admin/collection` | 深度采集任务列表，状态/进度/日志追踪 |
| 模型引擎 | `/admin/model` | 生文/生图/生视频模型注册、默认模型、SSE 对话测试、Token 用量统计 |
| 数字员工 | `/admin/digital_employee` | LLM/API 双形态员工 CRUD、Prompt 模板、MD 知识库、结果卡片模板 |
| 接口管理 | `/admin/api_interface` | HTTP 接口注册复用（headers/params/body/响应映射） |
| 技能管理 | `/admin/skills` | 技能 CRUD + 增强管道配置（数据清洗/上下文增强/可视化预处理） |
| 用户与权限 | `/admin/users` 等 | 用户 CRUD、角色授权树、功能/菜单管理、动态菜单（RBAC，路由级拦截） |
| 数智大屏 | `/admin/screen` | 来源统计、分类占比、关键词词云、3D 地球飞线、实时运营日志 |
| 舆情大屏 | `/admin/sentiment` | 预警统计/趋势/来源/风险分布、TOP 敏感词、AI 综合研判 |
| 敏感词管理 | `/admin/sensitive-word` | 三级匹配词库（全词 / 2-gram 子串 / 邻近模式）、批量导入、5 分钟缓存 |
| 预警记录 | `/admin/security-alert` | 预警检索、处置流转、AI 单条风险分析 |
| 会话/对话管理 | `/admin/conversations`、`/admin/messages` | 全站会话与消息审计 |
| 系统设置 | `/admin/setting` | 站点/LLM/安全参数 KV 配置 |

---

## 配置体系

| 配置位置 | 是否生效 | 说明 |
|---|---|---|
| `config/*.yaml`（app / database / llm / security） | 未加载 | 设计声明式快照，代码不读取（含 `${OPENAI_API_KEY}` 占位） |
| `system_settings` 表（后台「系统设置」） | 运行时 | 站点信息、LLM 默认参数、登录/限速安全参数 |
| `ai_models` 表（后台「模型引擎」） | 运行时 | LLM 实际调用凭据：base_url / api_key / model_id / 采样参数 |
| `watch_sources.collect_config` 字段 | 运行时 | 每个采集源的参数名、分页步长、解析器选择 |
| `app.py`（硬编码） | 运行时 | 监听端口 10010、cookie_secret、debug 开关——生产前必须修改源码 |

---

## 测试

```powershell
venv\Scripts\python -m compileall app app.py          # 编译检查
venv\Scripts\python test_chat_improvements.py          # 对话链路结构自检（流式/路由注册/模型切换/历史上下文）
venv\Scripts\python check_sources.py                   # 瞭源种子数据与启用状态检查
```

更多后台权限模型的实现说明见 `docs/admin_console_permissions_model.md`。

---

## 注意事项

本项目为教学/演示产物，以下为刻意简化：

1. **同步阻塞 I/O**：LLM 调用（requests，timeout 120s）在 WebSocket `on_message` 中同步执行，单条慢请求会阻塞整个 IOLoop，所有在线会话排队（生产应换 `AsyncHTTPClient` 或 `run_on_executor`）。
2. **SQLite 单文件**：每次操作新建连接、无连接池；消息同时写入 `messages` 表与 `conversations.messages` JSON 冗余字段，长会话下冗余膨胀，高并发写会锁表。
3. **默认凭据与密钥**：`admin/admin888`、cookie_secret 硬编码、`debug=True` 三项均在 `app.py`，生产部署前必须修改。
4. **NL2SQL 白名单未强制**：`ALLOWED_TABLES` 仅用于向 LLM 提供表结构，生成的 SQL 未做表白名单校验，仅靠关键词黑名单 + 单语句截断兜底。
5. **大屏地理数据为规则模拟**：关键词→城市映射表 + 轮询兜底生成 3D 地球点位，非真实地理解析。
6. **人脸登录为简化实现**：中心裁剪 100×100 像素矩阵做均值差比对（阈值 45），非人脸特征模型，仅作演示。
7. **数字员工存在 code_name 特判**：weather/music 的城市提取、结果改写硬编码在通用服务层，新增同类员工需改核心代码。
8. **WebSocket 会话越权**：`load_conversation` 未校验会话归属，任意登录用户可拉取他人会话（生产需补归属校验）。
