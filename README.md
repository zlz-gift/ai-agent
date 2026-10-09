# 通用 AI 对话助手（Agent 工具调用 + 分层记忆）

一个基于 **LangChain + FastAPI + SQLite** 的通用对话助手，重点实现两件事：**Agent 自主工具调用** 与 **分层记忆**。

- 模型侧：`bind_tools` + `ToolMessage` 手写 ReAct 式循环，工具由模型自主选择、结果回填后继续推理
- 记忆侧：短期滑动窗口 + 长期摘要压缩，把上下文从"随对话轮数线性增长"压成"常量级波动"
- 交付侧：FastAPI 服务化 + 自研移动端 Web 界面 + 一键启动脚本（uvicorn + cpolar 隧道）

---

## 功能

| 模块 | 说明 |
|---|---|
| 工具调用 | `@tool` 定义时间查询 / 安全计算器 / 本地笔记检索，模型自主决定是否调用、调用哪个 |
| 工具安全 | 计算器用 **AST 白名单求值**替代 `eval`，只放行 `+ - * / % **`，拦截 `__import__("os").system(...)` 一类注入；不提供写文件、执行命令、访问网络的能力 |
| 短期记忆 | SQLite 滑动窗口保留最近 **20** 条消息（约 10 轮），按 `session_id` 建索引 |
| 长期记忆 | 累计消息超过 **30** 条时自动调用 LLM 把早期对话压缩为结构化摘要并持久化 |
| 降级兜底 | 模型或网关不支持工具调用时，整轮降级为纯对话链，服务不中断 |
| 会话隔离 | 全部消息与摘要按 `session_id` 分表存储，重启服务不丢记忆 |

---

## 快速开始

```bash
# 1. 安装依赖
python -m venv .venv
.venv\Scripts\activate            # Windows；Linux/Mac: source .venv/bin/activate
pip install -r requirements.txt

# 2. 配置模型（兼容 OpenAI 协议的任意网关）
#    新建 .env，写入：
#    OPENAI_API_KEY=你的密钥
#    OPENAI_BASE_URL=https://api.deepseek.com/v1      # 按你的网关填写
#    （已在 .gitignore 中，不会被提交）

# 3. 启动
python web_app.py
# 浏览器打开 http://localhost:8000
```

命令行版本：`python cli.py`

### 手机访问（可选）

`一键启动.bat` 会同时拉起 uvicorn 和 [cpolar](https://www.cpolar.com/) 内网穿透，并从日志里正则抓取公网地址打印出来，手机扫码即可对话。

> cpolar 需自行下载（脚本默认读取上一级目录 `../cpolar/cpolar.exe`）。**不需要公网访问时直接用 `python web_app.py` 即可**，脚本不是必需的。

---

## 接口

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/` | 单文件移动端聊天界面（含打字动画、回车发送、清空记忆二次确认） |
| POST | `/api/chat` | 入参 `{"session_id": "...", "text": "..."}`，返回 `{"reply": "..."}` |
| POST | `/api/clear` | 清空指定会话的消息缓冲与长期摘要 |

```bash
curl -X POST http://localhost:8000/api/chat \
  -H "Content-Type: application/json" \
  -d '{"session_id":"demo","text":"现在几点？顺便算一下 (45+5)/2"}'
```

---

## 目录结构

```
.
├── assistant.py        # 对话大脑：Agent 循环 + 分层记忆 + SQLite 持久化
├── tools.py            # 工具集合（@tool 定义）与 AST 安全求值
├── web_app.py          # FastAPI 服务 + 单文件前端
├── cli.py              # 命令行入口
├── launcher.py         # 一键启动：uvicorn + cpolar 隧道
├── notes/              # 笔记检索工具的素材目录（sample.md 为示例）
├── requirements.txt
└── 一键启动.bat
```

---

## 关键设计取舍

**1. 为什么用"摘要"而不是保留全量历史？**
全量历史会让 token 成本随轮数线性上涨，而且早期对话对当前回答的边际价值很低。这里采用「短期原文 + 长期摘要」双层结构：

- **保留量 20 条**：压缩后仍留在上下文里的原文；
- **触发量 30 条**：累计超过才触发一次压缩（多一次 LLM 调用）；
- 中间的 10 条是**缓冲带**——如果触发量等于保留量，就会每轮都触发压缩，成本翻倍。

结果是上下文总量在 20~30 条之间有界波动，不随对话轮数增长。

**2. 压缩失败怎么办？**
压缩在单独的 `try/except` 里执行：失败时只打日志、保留原文、跳过本轮，下一轮再试。**压缩失败不会让本次回复失败，也不会让缓冲区无限增长**（另设 200 条加载上限兜底）。

**3. 为什么只写入最终问答？**
工具调用的中间步骤（`AIMessage(tool_calls)` / `ToolMessage`）只存在于当轮上下文，不进历史、不进摘要——否则摘要会被大量工具原始返回污染。

**4. 降级策略为什么用 try/except 而不是能力探测？**
不维护"哪些模型支持工具调用"的名单，而是把整轮调用包进 `try/except`：`bind_tools` / `invoke` 抛异常就立刻切纯对话链重试。代价是多一次调用，好处是兼容任意网关。

---

## 已知限制

- 降级为纯对话模式时**前端无感知**（仅服务端日志），应把 `mode` 回传给前端提示；
- 没有单次调用的 token / 耗时统计，无法做成本观测；
- 记忆存储是单个 SQLite 文件，多进程并发写入场景需要换成 Redis / PostgreSQL；
- 模型名目前写在 `assistant.py` 里（`deepseek-chat`），可改为从环境变量读取。

> 上面几条是明确知道、但本轮未做的取舍，列出来是为了说明边界，而不是宣称已经解决。

---

## 技术栈

Python · LangChain（`@tool` / `bind_tools` / `ToolMessage`）· FastAPI · Uvicorn · SQLite · python-dotenv
