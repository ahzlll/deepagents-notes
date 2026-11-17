# **环境准备**

本节要完成的是：在本机搭建一个可修改的 `deepagents/research` 项目，配置模型与 Tavily 搜索，启动前后端服务，并通过一个公开问题得到带来源链接的 Markdown 报告。

验收按以下三层依次进行：

| 层次   | 检查内容                      | 对应命令或证据                                                                      |
| ---- | ------------------------- | ---------------------------------------------------------------------------- |
| 本地就绪 | 工具版本、依赖目录、配置文件和环境变量齐全     | 各工具版本命令、`agentseek version`、`agentseek doctor`、`agentseek task course-check` |
| 服务可达 | 后端和前端进程正常启动               | `agentseek dev` 后用 `agentseek doctor --live` 检查 2024、5174 端口                 |
| 真实任务 | 模型调用、搜索调用和最终报告都成功，来源链接可打开 | 提交公开问题，出现 `tavily_search`，报告含可访问的来源链接                                        |

## 具体实践步骤

### 一、准备本地工具

本机走课程给出的原生 Windows 路线。已具备：Python 3.13、Node 22、npm 10、uv 0.12.7、Git。

先确认工具可用：

```bash
python --version
uv --version
node --version
npm --version
git --version
```

安装 AgentSeek CLI 并确认五个核心命令存在：

```bash
# 把 CLI 装进 uv 的独立工具环境；--upgrade 保证拿到当前最新版
uv tool install --upgrade agentseek
agentseek version
# 帮助信息里应能看到 create / info / task / doctor / dev
agentseek --help
```

课程文章基于 AgentSeek v0.1.2 验证，本文使用 v0.1.6，命令输出细节可能略有差异。

### 二、创建项目并查看生命周期声明

查看可用模板（`--list-templates` 读 CLI 内置目录，不需联网）：

```bash
agentseek create --list-templates
```

课程原文用 `--checkout main` 指定从模板仓库 main 分支取模板：

```bash
# --checkout main 表示读取模板仓库当前 main 分支，而不是 CLI 内置的锁定目录
agentseek create deepagents/research --checkout main --no-input
```

在 v0.1.6 上执行这条命令会报 `Explicit template catalog is invalid: catalog repository fetch failed`。原因是一旦带上 `--checkout`，CLI 就改为拉取远端模板目录，这一步在当前网络下失败。因此改为先把模板仓库取到本地，再指向其中的模板目录，生成结果相同：

```bash
# 只取最新一次提交
git clone --depth 1 https://github.com/agentseek-ai/agentseek-templates.git
# 直接指向本地模板目录，绕开远端模板目录拉取；--no-input 使用模板默认变量
agentseek create /绝对路径/agentseek-templates/templates/deepagents/research --no-input
# 生成的项目目录名取自模板，本例为 research_deepagent
cd research_deepagent
```

查看生命周期配置：

```bash
agentseek info
agentseek task --list
```

关键文件是 `.agentseek/lifecycle.toml`：它声明必需工具、环境变量、一次性任务和本地服务。模板版本变化后，以它为准。

### 三、安装依赖

```bash
# 运行模板声明的后端依赖任务，当前对应 uv sync --frozen
agentseek task sync
# 运行模板声明的前端依赖任务，当前对应 npm install --prefix frontend
agentseek task frontend
```

两条任务分别对应后端 `uv sync --frozen` 和前端 `npm install --prefix frontend`。如果前端安装失败，先看报错里的包名和版本：本次是默认镜像源缺少 `@langchain/langgraph-sdk@1.8.13`，改用官方 registry 即可，不必重装其他工具：

```bash
cd frontend
# --registry 指向官方源补齐镜像缺失的版本；--legacy-peer-deps 跳过严格的 peer 依赖校验
npm install --registry=https://registry.npmjs.org --legacy-peer-deps
cd ..
```

项目里与 Agent 定义相关的三个文件，后续要改模型、工具或提示词都从这里入手：

- `src/research_deepagent/agent.py`：`create_deep_agent` 的组装入口，模型、中间件、工具、子 Agent 都在这里指定。
- `src/research_deepagent/tools.py`：`tavily_search`、`think_tool` 等具体工具的实现。
- `src/research_deepagent/prompts.py`：主 Agent 与子 Agent 的系统提示词。

### 四、填写配置

先从模板复制环境文件：

```bash
cp .env.example .env
cp frontend/.env.example frontend/.env
```

在 `.env` 中只保留一组实际使用的模型配置。本机采用硅基流动的 OpenAI 兼容接口和免费模型：

```bash
# openai 是适配器名；SiliconFlow 的 OpenAI 兼容接口也用这个值
AGENTSEEK_MODEL_PROVIDER=openai
# 完整模型 ID，需确认该服务商当前提供
AGENTSEEK_MODEL=Qwen/Qwen3-8B
# 模型服务地址，不是 HTTP 代理
OPENAI_API_BASE=https://api.siliconflow.cn/v1
# 填入该服务的 Key
OPENAI_API_KEY=<your-siliconflow-api-key>
# 请求模型顺序调用工具，便于兼容网关稳定工作
AGENTSEEK_PARALLEL_TOOL_CALLS=false
# 关闭 LangSmith 追踪，不影响本地运行
LANGSMITH_TRACING=false
```

再把 Tavily 搜索 Key 填进同一文件：

```bash
TAVILY_API_KEY=<your-tavily-api-key>
```

原生 Windows 上还要把持久化改成本地 SQLite：

```bash
# 嵌入式 SeekDB 的底层依赖仅支持 Linux/macOS，原生 Windows 需关闭
SEEKDB_EMBED=false
# 元数据与 checkpoint 改用 SQLite
METADATA_DB_BACKEND=sqlite
# 本地数据库文件位置，换成自己的项目目录
METADATA_DB_URL=sqlite:///D:/<项目路径>/.agentseek/agentseek.db
# 任务在当前进程中执行，不依赖 Redis worker
EXECUTOR_BACKEND=inline
```

**模型 Key 用于调用模型服务，搜索 Key 用于调用 Tavily，二者不能混用。**

### 五、检查并启动服务

静态检查：

```bash
# 检查文件、工具、依赖目录与环境变量
agentseek doctor
```

不调用模型的自动检查，共 6 项，应全部通过：

```bash
# 用 pytest 跑模板自带的 model-free 检查，不消耗模型额度
agentseek task course-check
```

预览启动计划，不启动服务：

```bash
agentseek dev --dry-run
```

应看到两个进程：后端 `http://127.0.0.1:2024`，前端 `http://127.0.0.1:5174`。

启动前后端：

```bash
# 按 lifecycle.toml 启动声明的两个进程
agentseek dev
```

保持这个终端运行，另开一个终端进入同一项目目录，检查服务：

```bash
# dev 运行时检查声明的服务端点是否可达
agentseek doctor --live
```

两个地址都应显示为可访问。用一个范围小、能核验的问题做首次验收，例如：

```text
查找 Python 官方文档，说明 list 与 tuple 的两个主要区别。请用中文回答，附上来源链接，并尽量把回答控制在 300 字以内。
```

界面提交或直接调后端接口都可以。界面方式是在 `http://127.0.0.1:5174` 输入问题；接口方式分两步，先取 `assistant_id`，再提交一次运行：

```bash
# 取当前图的 assistant_id
curl -s -X POST http://127.0.0.1:2024/assistants/search -H "Content-Type: application/json" -d '{}'
# 提交一次运行并等待最终结果，input 里放上面的问题
curl -s -X POST http://127.0.0.1:2024/runs/wait -H "Content-Type: application/json" \
  -d '{"assistant_id":"<上一步返回的 id>","input":{"messages":[{"role":"user","content":"<问题>"}]}}'
```

先只提交一次。确认有搜索调用、报告完整、来源链接能打开。结束后回到运行 `agentseek dev` 的终端按 `Ctrl+C` 停止服务。

### 六、安装编码助手开发技能

这一步需要安装 Node.js 和 npm，`npx` 依赖它们。

要装的两项技能：

- `langchain-dev-guide`：模型接入、中间件、流式输出、多 Agent 编排等开发问题的处理指南。
- `langsmith-trace`：查询和分析 LangSmith Trace 的操作流程。

先列出仓库里有哪些技能：

```bash
npx skills add ob-labs/agentseek --list
```

再把这两项装进当前项目：

```bash
# --skill 指定要装的技能
# --agent 指定给哪个编码助手用
# --yes 跳过交互确认
npx skills add ob-labs/agentseek --skill langchain-dev-guide --skill langsmith-trace --agent codex --yes
```

装到当前项目，不加 `--global`。确认结果：

```bash
npx skills list
```

Skills CLI 会把技能放进编码助手会自动读取的目录。我们装的是项目级，目录就在当前项目里；Codex 和 Cursor 读取 `.agents/skills/`，Claude Code 读取 `.claude/skills/`。这里应能看到：

```text
.agents/skills/langchain-dev-guide/SKILL.md
.agents/skills/langsmith-trace/SKILL.md
```

也就是说，这个文件夹只在当前项目里生效；换成全局安装才会写到用户目录，对所有项目生效。本笔记用的是项目级安装。

## 遇到的问题与处理

### 1. 依赖下载超时与镜像源缺包

官方 PyPI 下载常超时，改用国内镜像后装上了 CLI。项目依赖里有较大的 wheel，镜像下载仍会因默认超时中断；又因为锁文件里的下载地址指向官方源，只换镜像不会生效。做法是临时用镜像重新生成锁文件、安装，再恢复原锁文件，装完不影响项目里的依赖声明：

```bash
# 备份原锁文件
cp uv.lock uv.lock.orig
# 用国内镜像重新生成锁文件，把下载地址指向镜像
UV_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple uv lock
# 按新锁文件安装
UV_INDEX_URL=https://pypi.tuna.tsinghua.edu.cn/simple uv sync --frozen
# 装完恢复原锁文件
mv uv.lock.orig uv.lock
```

npm 默认镜像源缺少 `@langchain/langgraph-sdk@1.8.13`，换回官方 registry 后成功。从完整报错里找出失败域名、命令和包名，再决定换源或加参数，不要直接重装所有工具。

Skills CLI 克隆 GitHub 仓库时可能受网络限制，可先把需要的 `skills/` 目录取到本地，再用本地路径安装，结果一致：

```bash
# 取仓库到本地
git clone --depth 1 https://github.com/ob-labs/agentseek.git
# 指向本地目录安装，参数与远程安装相同
npx skills add ./agentseek --skill langchain-dev-guide --skill langsmith-trace --agent codex --yes
```

### 2. 原生 Windows 上嵌入式 SeekDB 不可用

模板默认 `SEEKDB_EMBED=true`，但底层依赖 `pylibseekdb` 只有 Linux/macOS 版本，Windows 后端启动即失败。SQLAlchemy 的 sqlite 后端可用，因此改为 `SEEKDB_EMBED=false` 加本地 `METADATA_DB_URL`。如果能用 WSL2，建议优先在 WSL2 中操作：不少依赖默认按 Linux 环境准备。

### 3. 模型在真实运行链路中的工具调用不稳定

课程示例用 `zai-org/GLM-5.2`，本地模板预填 `deepseek-ai/DeepSeek-V3.2`，两者都需要付费。为不产生费用，先换成免费的 `THUDM/GLM-4-9B-0414`，它单次 function call 正常，但在真实运行链路中把 `write_todos` 的工具调用格式写坏，运行停在第一步。改用同样免费的 `Qwen/Qwen3-8B` 后，工具调用稳定，任务跑通。

## 总结

我把这次任务拆成两部分来做。

第一部分是环境准备和服务启动：`agentseek doctor` 除密钥配置外全部通过，`agentseek task course-check` 的 6 项检查通过，`agentseek dev --dry-run` 能正确列出后端和前端进程；启动 `agentseek dev` 后，`agentseek doctor --live` 确认 2024 和 5174 两个端口可以访问，检查完成后停止服务。

第二部分是真实任务：填入 Key 并换用免费模型后，向应用提交 list 与 tuple 的问题。Agent 调用了一次 `tavily_search`，返回带来源链接的中文 Markdown 报告，链接可正常打开。

通过本次实践，熟悉了 AgentSeek 五个命令各自的职责：`create` 从模板生成项目，`info` 查看项目入口和必需变量，`task` 执行模板声明的准备任务，`doctor` 检查本地条件，`dev` 按配置启动前后端。模板的要求集中写在 `lifecycle.toml` 中，换用其他模板时，以该文件和 `agentseek info` 的输出为准。

## 资料来源

- 课程主页：<https://datawhalechina.github.io/deepagents-in-action/>
- AgentSeek 准备篇（上）：用生命周期工作流启动 DeepAgents 模板：<https://datawhalechina.github.io/deepagents-in-action/chapters/pre01-agentseek-create/>
- AgentSeek 准备篇（下）：为 AI 编码助手安装开发技能：<https://datawhalechina.github.io/deepagents-in-action/chapters/pre02-agentseek-skills/>
- Task 1 FAQ（叶静）：<https://datawhalechina.github.io/deepagents-in-action/chapters/pre03-task1-faq/>
- 开源课程仓库：<https://github.com/datawhalechina/deepagents-in-action>
- AgentSeek CLI：<https://github.com/ob-labs/agentseek>
- AgentSeek Templates：<https://github.com/agentseek-ai/agentseek-templates>
- LangChain Deep Agents 官方文档：<https://docs.langchain.com/oss/python/deepagents/overview>
- SiliconFlow 模型广场：<https://cloud.siliconflow.cn/models>
- SiliconFlow API 文档：<https://docs.siliconflow.cn/>
- Skills CLI（编码助手技能安装工具）：<https://github.com/vercel-labs/skills>
- Tavily：<https://app.tavily.com/>
