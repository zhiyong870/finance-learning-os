# finance-learning-os

> 个人金融学习操作系统 —— 用结构化方法建立投资认知体系

## 项目目标

通过六大核心组件，系统化地积累投资知识、分析案例、追踪认知盲区，最终形成可复用的投资决策框架。

## 目录结构

```
finance-learning-os/
├── .claude/skills/      # 领域 Skills 和方法论 Skills
├── tools/               # LLM review、invariants checks 等辅助工具
├── components/          # 六大核心组件（每个一个子目录）
├── reports/             # 爬取的年报和研报归档
├── concepts/            # 概念库
├── cases/               # 投资案例库
├── journal/             # 投资日志
├── knowledge_gaps.md    # 知识盲区清单
├── retrospectives/      # 每周复盘
└── tests/               # 测试
```

## 开发规范

- Python 依赖通过 `uv` 管理，见 `pyproject.toml`
- 敏感信息（API key、.env）绝不提交 git
- 每次新增概念/案例后更新 `knowledge_gaps.md`
- 每周五写复盘到 `retrospectives/`

## 常用命令

```bash
# 激活环境
source .venv/bin/activate

# 安装依赖
uv pip install -r requirements.txt

# 运行测试
pytest tests/
```
