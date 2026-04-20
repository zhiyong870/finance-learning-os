# tools/ 目录工作规则

本目录存放项目的基础设施工具:LLM wrapper、观测、review 工具、invariants 检查、
prompt 模板等。在本目录下工作时,本文件规则**叠加**在根 CLAUDE.md 之上。

---

## 1. 本目录的使命

`tools/` 是整个项目的**Harness 底座**。所有 components/ 下的业务逻辑,都建立在
本目录提供的基础设施之上。本目录的代码质量和稳定性**直接决定了**整个系统的可靠性。

**本目录的代码应该是项目里最严格、最经过测试、最少依赖的部分。**

## 2. 目录结构与职责

```
tools/
├── CLAUDE.md              # 本文件
├── llm_wrapper.py         # 唯一允许的 LLM 调用入口
├── llm_review.py          # LLM-as-Judge(用更强模型 review 产出)
├── llm_stats.py           # LLM 调用统计与分析
├── invariants/            # Mechanical invariants 检查脚本
│   ├── check_llm_usage.py
│   ├── check_antipatterns.py
│   └── check_skills_reference.py
├── prompts/               # 所有 prompt 集中存储(按组件分子目录)
│   ├── _system/           # 系统级 prompt(judge、review 等)
│   ├── earnings_reader/
│   ├── concept_trainer/
│   └── ...
└── utils/                 # 通用工具函数
```

## 3. LLM 调用的唯一规范

### 3.1 调用入口

**所有对 Claude API 的调用必须通过 `tools/llm_wrapper.py` 的 `call_llm()` 函数。**

禁止的写法:
```python
# ❌ 禁止
import anthropic
client = anthropic.Anthropic()
client.messages.create(...)
```

允许的写法:
```python
# ✅ 允许
from tools.llm_wrapper import call_llm
response = call_llm(
    prompt_name="earnings_reader/chapter_intro",
    variables={"chapter_text": chapter, "user_input": user_answer},
    skills=["teaching-style", "finance-accuracy", "learning-feedback"],
    model="sonnet",  # 或 "opus"
    component="earnings_reader",
)
```

### 3.2 call_llm 必须做的事

`llm_wrapper.py` 的 `call_llm()` 函数必须:

1. 从 `tools/prompts/` 加载指定的 prompt 模板,支持变量替换
2. 从 `.claude/skills/` 加载指定的 skills,拼接到 system prompt
3. 调用 Claude API,正确处理流式/非流式、超时、重试
4. 记录调用到 `logs/llm_calls.db`(字段见下)
5. 返回结构化的 response 对象(含 text、usage、cost)

### 3.3 Observability 必须记录的字段

每次 LLM 调用记录到 SQLite,字段:

| 字段 | 说明 |
|---|---|
| `timestamp` | 调用时间 |
| `component` | 调用方组件名(earnings_reader / concept_trainer / _system 等) |
| `prompt_name` | 使用的 prompt 文件名 |
| `prompt_hash` | prompt 内容的 hash(版本追踪) |
| `skills_loaded` | 加载的 skills 列表(逗号分隔) |
| `model` | 模型名(sonnet / opus) |
| `input_tokens` | 输入 token 数 |
| `output_tokens` | 输出 token 数 |
| `cost_usd` | 本次调用成本估算 |
| `latency_ms` | 调用耗时 |
| `success` | 是否成功 |
| `error` | 失败时的错误信息 |
| `session_id` | 归属会话 ID(多轮对话用) |

## 4. Prompt 管理规范

### 4.1 所有 prompt 必须在 `tools/prompts/` 里

代码中**禁止**出现硬编码的 prompt 字符串(超过 50 字的系统指令)。

```python
# ❌ 禁止
system = "你是一个金融分析师,请分析..."
call_llm(system=system, ...)

# ✅ 允许
call_llm(prompt_name="earnings_reader/chapter_intro", ...)
```

### 4.2 Prompt 文件格式

每个 prompt 是一个 Markdown 文件,结构:

```markdown
---
name: earnings_reader/chapter_intro
description: 引导用户进入年报章节精读
variables:
  - chapter_text: 章节原文
  - previous_context: 前面章节的摘要(可选)
recommended_skills:
  - teaching-style
  - finance-accuracy
model: sonnet
---

# System

[system prompt 内容]

# User

[user prompt 模板,使用 {chapter_text} 这种占位符]
```

Frontmatter 的 `variables`、`recommended_skills` 是 self-documentation,
`call_llm` 可以据此验证调用参数完整性。

### 4.3 Prompt 迭代

- Prompt 修改后版本化:文件提交到 git 即是版本
- 不要在 prompt 文件名里加 `_v2`、`_new` 这种后缀,用 git 历史追溯
- 重大修改在 commit message 里说明动机

## 5. Mechanical Invariants 规范

`tools/invariants/` 下每个脚本实现一个机械检查,集成到 pre-commit hook。

### 5.1 必须实现的 3 个检查

**check_llm_usage.py** — 扫描所有 .py 文件,违规情形:
- 直接 `import anthropic` 或 `import openai`(除 llm_wrapper.py 自身)
- 代码中出现 API key 明文(正则匹配 `sk-ant-...`、`sk-...`)
- `call_llm` 调用缺少 `component` 或 `prompt_name` 参数

**check_antipatterns.py** — 扫描代码和 prompt,违规情形:
- 函数名匹配黑名单:`auto_summarize`、`recommend_*`、`evaluate_investment`、`one_click_*` 等
- Prompt 中出现中文黑名单:「请总结」「请推荐」「请评估……是否值得投资」「自动分析」等
- 黑名单列表放在 `tools/invariants/blacklist.yaml`,可扩充

**check_skills_reference.py** — 扫描所有 `call_llm` 调用:
- `skills=[...]` 参数存在且非空(除非是 `component="_system"` 的例外)
- 引用的 skill 文件在 `.claude/skills/` 实际存在

### 5.2 失败输出规范

所有 invariants 检查失败时,输出:
- 违规文件和行号
- 违规的具体内容
- 违反的规则引用(CLAUDE.md 的章节)
- 修复建议

## 6. LLM-as-Judge(llm_review.py)规范

`llm_review.py` 提供命令行工具,对产出做严格 review。

### 6.1 基本用法

```bash
# Review 一个代码文件
python tools/llm_review.py --target components/earnings_reader/splitter.py

# Review 一份 Markdown 沉淀(用户自检用)
python tools/llm_review.py --target reports/贵州茅台/2023/chapter3_notes.md --as-user-review

# Review CLAUDE.md 修改是否冲突
python tools/llm_review.py --target CLAUDE.md --mode consistency
```

### 6.2 实现要点

- Judge 使用 Claude Opus,调用方(被 review 的产出)一般由 Sonnet 生成
- Judge 的 system prompt 明确:**职责是找问题,不是表扬,宁可严苛**
- Judge 自动加载:根 CLAUDE.md、相关子目录 CLAUDE.md、相关 skills
- 输出格式固定为三级:
  - `CRITICAL`:必须修复,违反核心原则
  - `WARNING`:建议修复,可能有问题
  - `SUGGESTION`:可选优化
- Judge 输出本身也记录到 observability

### 6.3 何时必须跑 llm_review

- 新组件合并前必跑
- CLAUDE.md 或 Skills 修改后必跑 consistency 模式
- 用户主动请求自检时

### 6.4 何时不跑

- 小 bug 修复、typo 修正
- 纯测试代码
- 配置文件修改

## 7. 测试要求

`tools/` 下的每个模块必须有测试,覆盖:

- `llm_wrapper.py`:mock LLM 调用,验证 prompt 加载、skills 拼接、observability 记录
- `llm_review.py`:用固定输入验证 review 输出结构
- `invariants/*.py`:用 fixture 文件验证能正确识别违规和合规案例

## 8. 依赖管理

本目录代码**尽可能少依赖**:
- 允许:`anthropic`、`pydantic`、`pyyaml`、`sqlite3`(标准库)、`jinja2`(prompt 模板)
- 不允许:任何 web 框架、ORM、重量级日志库

新增依赖必须问用户。
