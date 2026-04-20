# tests/ 目录工作规则

本目录存放项目的所有测试代码。在本目录下工作时,本文件规则**叠加**在根 CLAUDE.md 之上。

---

## 1. 测试哲学

本项目的测试**不是**追求高 coverage,而是追求"**真正验证行为正确**"。
特别是涉及 AI 交互的部分,传统单元测试不够用,需要 LLM-as-Judge 辅助。

## 2. 测试目录结构

```
tests/
├── CLAUDE.md              # 本文件
├── conftest.py            # pytest fixtures 和配置
├── fixtures/              # 测试用的固定数据
│   ├── annual_reports/    # 年报样例 PDF
│   ├── expected_outputs/  # 期望输出样例
│   └── invariants/        # invariants 检查的违规/合规样例
├── unit/                  # 单元测试(纯逻辑,无 AI 调用)
│   ├── tools/
│   └── components/
├── integration/           # 集成测试(含 AI 调用,用 mock 或录制)
│   └── components/
├── e2e/                   # 端到端测试(真实 AI 调用,慢,不每次跑)
│   └── components/
└── invariants/            # invariants 脚本自身的测试
```

## 3. 测试分层与运行时机

### 3.1 Unit 测试
- **范围**:纯逻辑,不碰 AI,不碰文件系统(或只碰临时目录)
- **运行**:每次 pre-commit,必须全通过
- **速度**:整套 unit 测试 30 秒内跑完

### 3.2 Integration 测试
- **范围**:覆盖模块间协作,AI 调用用 mock 或录制回放
- **工具**:对 `call_llm()` 用 monkeypatch,返回预录的 response
- **运行**:每次 pre-commit,必须全通过
- **速度**:1-2 分钟内跑完

### 3.3 E2E 测试
- **范围**:真实调用 Claude API,跑完整组件流程
- **运行**:手动触发,或每周一次 CI。**不要求**每次 commit 跑
- **成本**:一次完整 E2E 跑下来可能 $1-5,用户可接受范围内控制频率

### 3.4 Invariants 测试
- 单独目录 `tests/invariants/`,验证 `tools/invariants/` 下的检查脚本本身正确
- 每个检查脚本对应一个测试文件
- 测试用 fixture:`tests/fixtures/invariants/` 下准备合规和违规样例

## 4. 组件测试的必备项

每个 `components/<name>/` 组件必须有对应的 `tests/integration/components/<name>_test.py`,至少覆盖:

### 4.1 功能性测试
- 正常路径:给定合法输入,产出符合 schema
- 边界路径:空输入、超长输入、非 UTF-8 等
- 错误路径:API 失败时的降级、部分数据缺失时的处理

### 4.2 学习目的测试(关键)

这是本项目特有的测试类别,验证组件是否真的符合"逼用户思考"原则。

**用 LLM-as-Judge 验证交互流程**,判别依据:

```python
def test_earnings_reader_does_not_replace_thinking():
    # 模拟用户只输入"我不知道,你帮我分析"
    session = EarningsReaderSession(chapter="...")
    response = session.interact(user_input="帮我分析一下这段")

    # 用 LLM judge 判断 response 是否违反核心原则
    judgment = llm_judge(
        response=response,
        criteria="""
        response 必须:
        1. 不直接给出这段年报的分析结论
        2. 引导用户先自己写出理解
        3. 符合 teaching-style skill 规范
        """
    )
    assert judgment.passes, f"违反学习目的原则: {judgment.reasons}"
```

**每个涉及 AI 交互的组件,至少有 3 个这类测试**,覆盖典型的"用户偷懒"场景。

### 4.3 沉淀产出测试
- 验证会话结束时生成了沉淀文件
- 验证沉淀文件的 schema 符合 components/CLAUDE.md 第 4 节规范
- 验证沉淀内容与会话实际发生的事匹配(可选,用 LLM judge)

## 5. LLM-as-Judge 测试模式

### 5.1 何时使用

传统 assert 无法判断的场景:
- AI 输出的教学质量
- 沉淀文件的内容合理性
- 组件输出是否符合 skill 规范
- prompt 修改后行为是否仍合规

### 5.2 使用方式

`tests/conftest.py` 提供 `llm_judge` fixture:

```python
def llm_judge(response, criteria, strict=True) -> JudgmentResult:
    """
    调用 Opus 作为 judge,用 criteria 评估 response。
    返回 JudgmentResult(passes: bool, reasons: List[str], severity: str)
    """
```

### 5.3 判官 prompt 规范

所有判官 prompt 放在 `tools/prompts/_system/judge/`。判官必须:
- 明确说明"你是严格的评审,宁可错杀",避免过度宽容
- 输出结构化 JSON,不输出自由文本
- 对每个不通过的点给出具体原因和违反的规则引用

### 5.4 Judge 测试的稳定性

LLM-as-Judge 有随机性。解决:
- 使用 `temperature=0`
- 每个 judge 调用使用相同的 seed(如支持)
- 判断标准写得足够明确,减少 judge 理解偏差
- 关键测试跑 3 次,取多数结果

## 6. Fixtures 使用规范

### 6.1 年报等大文件
- 放在 `tests/fixtures/annual_reports/`
- 用小样本(10-20 页的片段),不放完整年报
- 文件命名:`<ticker>_<year>_<chapter>.pdf`(如 `600519_2023_chapter3.pdf`)

### 6.2 期望输出
- 放在 `tests/fixtures/expected_outputs/`
- 用 Markdown 或 JSON 格式
- 不追求字符级精确匹配,关键字段匹配 + LLM judge 辅助

### 6.3 预录的 LLM response
- 放在 `tests/fixtures/llm_responses/`
- 文件名含 prompt_name 和 input hash
- 用于 integration 测试,避免真实调用

## 7. 测试相关的反模式

**以下做法禁止**:

- ❌ 测试中硬编码 API key
- ❌ 测试中直接调用 `anthropic.Anthropic()`(应通过 `call_llm` 的 mock)
- ❌ 测试名不说明测试内容(如 `test_1`、`test_basic`)
- ❌ 跳过测试但不说明原因(`@pytest.skip` 必须带 reason)
- ❌ 测试失败时随意改 fixture 让它通过
- ❌ Integration 测试每次跑都调用真实 API(应用 mock)
- ❌ 提交代码前没跑 `pytest tests/unit tests/integration`

## 8. 运行测试的命令约定

项目根目录下运行:

```bash
# 日常开发(每次改动后)
pytest tests/unit tests/integration

# 提交前
pytest tests/unit tests/integration && python tools/invariants/run_all.py

# 手动跑 E2E(消耗真实 API)
pytest tests/e2e --run-live

# 只跑某个组件的测试
pytest tests/ -k earnings_reader
```

## 9. 测试的学习价值

**反直觉提醒**:本项目的测试本身也是用户"学金融"的辅助。
测试用例里的 fixture(年报样例、概念定义、投资案例)是用户接触真实金融材料的渠道。
选 fixture 时优先选**教学价值高**的样本(比如有争议的财报、有重述的公司),
而不是最简单的 toy example。
