# TalkTrace

TalkTrace 将技术演讲转化为有证据支持的 Engineering Notes。

它不是简单压缩 transcript，而是从演讲中提取核心观点、技术原理、
示例、实践建议和限制，并让重要内容能够回到原始时间点或字幕片段。

## 基本流程

```text
导入 transcript / 字幕 / slides
        |
        v
Understand
理解主题和知识结构
        |
        v
Compose
选择笔记模块并生成草稿
        |
        v
Validate ----------------------> Engineering Note
检查结构、证据和遗漏     通过       候选笔记
        |
        | 发现问题
        v
Revise
修订草稿
        |
        +----------------------> Validate
```

AI 负责生成候选笔记，用户负责检查、修改、保存或导出。

## Engineering Note Protocol v0.1

Protocol 定义生成什么内容以及如何判断内容合格，不负责模型调用和数据存储。

核心思想：

> 固定骨架 + 类型模块 + 实际内容与证据过滤

### 核心字段

所有 Engineering Notes 共享以下结构：

| 字段 | 内容规则 | 缺失处理 |
| --- | --- | --- |
| `Executive Summary` | 3–5 句话；说明主题、适合的读者和关注价值 | 必须生成 |
| `Problem` | 解决什么问题、谁会遇到、现有方案为何不足 | `not_discussed` |
| `Approach` | 演讲者提出的方案、核心机制及主要差异 | `not_discussed` |
| `Key Takeaways` | 3–7 条可复用结论，每条必须有证据 | 必须生成 |
| `Limitations` | 只记录演讲明确提到的局限、条件和代价 | `not_discussed` |
| `Further Investigation` | 值得验证的问题和需要继续学习的概念 | 无内容时保留空列表 |

`Further Investigation` 是编辑性建议，不得伪装成演讲者观点。
所有核心字段都必须存在；`not_discussed` 表示字段存在但演讲没有提供相关内容。

### 演讲类型与可选模块

一场演讲可以有一个主类型和多个次类型。

| 演讲类型 | 候选模块 |
| --- | --- |
| `Architecture Talk` | Components、Data Flow、Deployment、Observability |
| `Case Study` | Context、Implementation、Metrics、Lessons |
| `Research Talk` | Hypothesis、Method、Evaluation、Results |
| `Tool Introduction` | Capabilities、Integration、Alternatives、Adoption Cost |
| `Opinion / Trend Talk` | Claims、Supporting Arguments、Counterarguments |

演讲类型只决定“应该检查哪些模块”，不能强制生成模块。

模块只有同时满足以下条件才启用：

1. 符合演讲类型；
2. 演讲确实包含有价值的相关内容；
3. 能找到支持该内容的证据。

例如，Architecture Talk 没有讨论 Deployment，就不生成 Deployment
内容，也不推测可能的部署方案。

### 内容状态

重要内容使用以下状态表达其证据关系：

- `supported`：有直接证据支持；
- `inferred`：由多段内容归纳得出，必须明确标记；
- `not_discussed`：演讲没有讨论；
- `unclear`：演讲提到但无法确定；
- `conflicting`：演讲中存在相互冲突的说法。

不能把 `inferred` 内容写成演讲者的直接结论，也不能用常识填补
`not_discussed`。

### Evidence

Evidence 表示笔记内容来自演讲的哪个位置，可以指向 transcript 片段、
时间范围或 slide。

```text
source: transcript
start: 00:12:30
end: 00:13:18
```

证据规则：

- 每条 `Key Takeaway` 必须至少有一条证据；
- 重要数据、技术判断、结果和局限必须有证据；
- 一条笔记内容可以关联多个证据；
- 证据必须实际支持内容，只有主题相关还不够；
- Evidence 证明内容忠实于演讲，不代表演讲者的说法一定正确。

### 组合规则

```text
核心字段
+ 演讲类型的候选模块
+ 演讲实际包含的内容
+ 可验证的证据
= Engineering Note
```

简而言之：

> 类型决定检查什么，证据决定写什么。

### 验证规则

Pipeline 的 `Validate` 阶段至少检查：

1. 核心字段是否存在；
2. Summary 是否为 3–5 句话；
3. Takeaways 是否为 3–7 条；
4. 每条 Takeaway 是否有有效证据；
5. 证据是否真正支持对应内容；
6. 是否把未讨论或推断内容写成事实；
7. 是否遗漏演讲中的重要主题；
8. 可选模块是否满足启用条件；
9. 演讲中的条件、限制和不确定性是否得到保留。

Validate 发现问题后进入 `Revise`，再重新检查。循环次数由 Pipeline
设计决定，不属于 Protocol。

## 核心对象

- `Technical Talk`：一场技术演讲；
- `Source`：transcript、字幕或 slides；
- `Engineering Note`：从演讲整理出的结构化笔记；
- `Evidence`：笔记内容与原始材料的关联；
- `Revision`：用户保存或重新生成的笔记版本；
- `Pipeline Run`：AI 执行一次笔记生成的过程和结果。

## MVP

第一版只做：

- 导入 transcript 或字幕；
- 生成结构化 Engineering Note；
- 展示笔记对应的演讲证据；
- 允许用户修改和保存；
- 导出 Markdown。

暂不设计 Skills、ReAct Agent、自动转写、复杂协作和知识图谱。

## 设计边界

- **Protocol**：定义笔记结构和内容组合规则；
- **Pipeline**：定义 AI 如何生成、检查和修订笔记；
- **Schema 与数据库**：以后定义数据如何保存。

当前优先验证 Protocol 是否适用于不同类型的技术演讲，再展开 Pipeline
和数据设计。

## 工程架构

采用模块化单体和 DDD 四层结构：

```text
Interfaces -> Application -> Domain
Infrastructure -> Application / Domain
```

```text
src/talktrace/
├── main.py
├── config.py
├── interfaces/
│   └── http/
│       ├── controllers.py
│       ├── schemas.py
│       └── errors.py
├── application/
│   ├── use_cases.py
│   ├── pipeline.py
│   ├── ports.py
│   └── dto.py
├── domain/
│   ├── talk.py
│   ├── note.py
│   ├── generation.py
│   ├── protocol.py
│   └── errors.py
└── infrastructure/
    ├── database.py
    ├── repositories.py
    ├── ai.py
    ├── tasks.py
    └── markdown.py
```

四层职责：

- **Interfaces**：FastAPI Controller、HTTP Schema 和错误转换；
- **Application**：Use Case、Pipeline 编排、Port 和 DTO；
- **Domain**：Aggregate、Entity、VO、Protocol 和领域校验；
- **Infrastructure**：SQLite、Repository、Pydantic AI、任务执行和 Markdown。

Pipeline 放在 Application，只编排：

```text
Understand -> Compose -> Domain Validate -> Evidence Assess -> Revise -> Persist
```

Protocol 的字段、数量、状态和 Evidence 规则只放在 Domain。

硬性依赖规则：

- Domain 不依赖 FastAPI、SQLAlchemy、SQLite 或 Pydantic AI；
- Application 不直接依赖数据库和具体模型；
- Interfaces 不直接操作 Repository；
- Infrastructure 实现 Application 定义的 Port；
- PO 属于 Infrastructure，DTO 属于 Interfaces/Application，二者不进入 Domain。

初期不为每个 Entity、VO 或 Port 单独创建文件。只有文件明显变大或概念稳定后再拆分。

## 开发计划：API First

首个交付物是可调用、可测试的 HTTP API，暂不开发前端。

建议技术栈：

- Python 3.12；
- FastAPI + Pydantic；
- Pydantic AI；
- SQLAlchemy 2 + Alembic；
- SQLite；
- pytest + Ruff + mypy；
- OpenAPI。

### MVP 接口

| 接口 | 作用 |
| --- | --- |
| `POST /v1/talks` | 创建演讲并导入 transcript |
| `GET /v1/talks/{talk_id}` | 查询演讲 |
| `POST /v1/talks/{talk_id}/runs` | 异步生成笔记 |
| `GET /v1/runs/{run_id}` | 查询生成状态、错误和结果 |
| `GET /v1/notes/{note_id}` | 查询结构化笔记和 Evidence |
| `PUT /v1/notes/{note_id}` | 保存用户修改并重新校验 |
| `GET /v1/notes/{note_id}/export` | 导出 Markdown |

Run 状态为 `queued`、`running`、`completed` 或 `failed`。

### 关键接口结构

```json
{
  "segment": {
    "id": "seg_001",
    "start_ms": 750000,
    "end_ms": 798000,
    "text": "..."
  },
  "note_item": {
    "content": "...",
    "status": "supported",
    "evidence": ["seg_001"]
  }
}
```

API 使用毫秒而不是格式化时间字符串，展示层再转换成 `00:12:30`。

### 交付阶段

| 阶段 | 交付 | 验收 |
| --- | --- | --- |
| 1. Contract | OpenAPI、数据模型、错误格式、Mock API | 全部接口可通过 Swagger 或 curl 调用 |
| 2. Protocol | 校验器、Evidence 检查、Markdown 导出 | 无证据 Takeaway 必须失败，错误位置清晰 |
| 3. Pipeline | Understand → Compose → Validate → Revise | 生成结果通过 Protocol；失败和修订次数可查询 |
| 4. Model + Storage | 模型 Provider、SQLite、版本和端到端测试 | 完整 API 流程可运行，服务重启后数据仍存在 |

模型阶段至少使用五类技术演讲样本验证：

- 每条 Takeaway 都有有效证据；
- 未讨论的 Limitations 不会被猜测；
- 更换模型不会改变 HTTP API。

每个阶段都以接口、自动化测试和可复现示例作为交付。

### 开发前确认

当前计划采用以下假设：

1. 使用 Python + FastAPI；
2. MVP 只接收带时间范围的 transcript，slides 后续支持；
3. 笔记生成使用异步 Run 接口；
4. 首版按单用户本地服务设计，不做登录和权限；
5. 先完成 Mock API 和 Protocol 校验，再选择模型 Provider。
