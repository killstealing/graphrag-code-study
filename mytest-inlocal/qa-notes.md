# GraphRAG 学习笔记

> 基于实际探索和调试过程整理的 Q&A。

---

## 1. `create_base_text_units` 和 `create_final_documents` 是做什么的？

这两个步骤在 GraphRAG 索引管道中紧密衔接：

```
load_input_documents  →  create_base_text_units  →  create_final_documents  →  后续图提取...
```

### `create_base_text_units` — 文档分块

**实现文件:** `packages/graphrag/graphrag/index/workflows/create_base_text_units.py`

将原始大文档切割成小块文本单元（chunk）：

| 步骤 | 说明 |
|---|---|
| 读取 | 遍历 `documents` 表，逐行获取每个文档的 `text` 字段 |
| 元数据前置 | 如果配置了 `prepend_metadata`（如 `["title"]`），会把标题等元数据拼到每个 chunk 前面 |
| 分块 | 调用 `chunker.chunk()` 按 token 数量将文档文本切分成小块（由 `chunking` 配置控制 size/overlap） |
| 计算 token 数 | 对每个 chunk 用 tokenizer 编码，记录 `n_tokens` |
| 生成 ID | 对 chunk 文本做 SHA-512 哈希，作为去重 ID |
| 写入 | 输出到 `text_units` 表，每行包含 `id`、`document_id`、`text`、`n_tokens` |

**输入:** `documents` 表（由 `load_input_documents` 产生）

**输出:** `text_units` 表（一个文档 → 多个 text_unit）

---

### `create_final_documents` — 建立文档与分块的关系

**实现文件:** `packages/graphrag/graphrag/index/workflows/create_final_documents.py`

将 text_unit 的 ID 反向关联回文档，并规范化文档的最终字段：

| 步骤 | 说明 |
|---|---|
| 第一遍扫描 | 遍历 `text_units` 表，按 `document_id` 分组，建立映射 `{document_id → [text_unit_id, ...]}` |
| 第二遍扫描 | 遍历 `documents` 表，为每个文档填充 `text_unit_ids` 字段（它包含的所有 chunk 的 ID 列表） |
| 字段裁剪 | 将文档投影到固定的输出 schema：`id`、`short_id`、`title`、`text`、`text_unit_ids`、`creation_date`、`raw_data` |
| 写入 | 覆写 `documents` 表 |

**输入:** `documents` 表 + `text_units` 表

**输出:** 规范化后的 `documents` 表（新增了 `text_unit_ids` 列表字段）

**一句话总结：**
- **`create_base_text_units`**：把长文档 **切碎**（一大段文本 → 多个小 text_unit）
- **`create_final_documents`**：把切碎的关系 **缝合回去**（文档知道自己的 text_unit 们是谁）

---

## 2. `tabulate()` 参数解析

```python
print(tabulate(df_documents, headers='keys', tablefmt='pretty',
               showindex=False, stralign='left',
               maxcolwidths=[20, 20, 20, 20]))
```

| 参数 | 值 | 含义 |
|---|---|---|
| 第一个位置参数 | `df_documents` | 要打印的 DataFrame |
| `headers` | `'keys'` | 使用 DataFrame 的列名作为表头 |
| `tablefmt` | `'pretty'` | 表格样式用 Pretty（Unicode 画线框） |
| `showindex` | `False` | 不显示行索引 |
| `stralign` | `'left'` | 字符串列左对齐 |
| `maxcolwidths` | `[20, 20, 20, 20]` | 每列最大显示宽度 20 字符 |

> **建议:** `maxcolwidths` 写死了 4 列，更稳健的写法是 `[20] * len(df.columns)`。

---

## 3. `tabulate` + DataFrame 报 `ValueError` 的修复

### 报错信息

```
ValueError: The truth value of an array with more than one element is ambiguous.
Use a.any() or a.all()
```

### 根因

`tabulate` 内部会对每个单元格调用 `line.strip()` 来做文本换行处理，但 `documents` 表的 `text_unit_ids` 列存储的是 **list 类型**（如 `['id1', 'id2']`），不是字符串。list 没有 `.strip()` 方法，pandas 回退到 numpy 的数组比较逻辑，抛出 ambiguous truth value 错误。

### 修复方法

在调用 `tabulate` 之前，把 list/dict/numpy array 类型的单元格转成字符串：

```python
import numpy as np
from tabulate import tabulate

df = pd.read_parquet("../output/documents.parquet")
df_display = df.copy()
for col in df_display.columns:
    df_display[col] = df_display[col].apply(
        lambda x: str(x) if isinstance(x, (list, dict, np.ndarray)) else x
    )

print(tabulate(df_display, headers='keys', tablefmt='pretty',
               showindex=False, stralign='left',
               maxcolwidths=[20] * len(df_display.columns)))
```

---

## 4. `uv` 是什么？

**uv** 是一个用 Rust 编写的极速 Python 包管理器和项目管理工具，由 Astral 公司（ruff 的开发者）开发。

### 核心功能

| 功能 | 命令 | 替代谁 |
|---|---|---|
| 安装包 | `uv pip install` | `pip install`（快 10-100x） |
| 创建虚拟环境 | `uv venv` | `python -m venv` / `virtualenv` |
| 管理 Python 版本 | `uv python install 3.12` | `pyenv` |
| 项目初始化 | `uv init` | `poetry init` |
| 依赖解析/锁 | `uv lock` | `pip-tools` / `poetry lock` |
| 运行脚本 | `uv run script.py` | `pipx run` |

### 为什么快

- **Rust 实现**：核心逻辑编译为原生代码
- **全局缓存**：所有包缓存在 `~/.cache/uv`，避免重复下载
- **并行下载/解析**：多任务并发执行
- **共享虚拟环境**：相同依赖的多个项目可复用 venv

---

## 5. 为什么 GraphRAG 中使用 `async for` 迭代表

工作流代码里大量出现 `async for row in table`，而不是普通的 `for row in table`。

### 调用链

```
async for row in table
    → __aiter__ → _aiter_impl()
        → await self._storage.get(file_key, as_bytes=True)   ← 异步 I/O
        → pd.read_parquet(BytesIO(data))
        → for _, row in self._df.iterrows(): yield row_dict
```

### 根因：底层 Storage 是异步的

`Table` 抽象类只定义了 `__aiter__`（异步迭代器），不存在同步的 `__iter__`：

```python
class Table(ABC):
    @abstractmethod
    def __aiter__(self) -> AsyncIterator[Any]:
        ...
```

| 存储后端 | 异步原因 |
|---|---|
| 本地文件 | 接口统一需要，实际是同步的但包装成 `async` |
| Azure Blob Storage | 真正的异步 HTTP I/O |
| Cosmos DB | 真正的异步数据库查询 |

### 设计动机：统一接口

- 如果有些后端提供同步接口、有些提供异步，上游代码就要写两套
- **设计成全部异步**后，所有 workflow 代码写一套就够了，不关心底层存储是什么
- 这是一个常见模式：**"在抽象层选最高要求的，能力弱的上层模拟"** — 远程存储必须是异步的，本地存储去模拟它

### 不用异步会怎样

写普通 `for row in table` 会直接报错：
```
TypeError: 'ParquetTable' object is not iterable
```

---

## 6. `create_final_documents` 详细代码分析

**实现文件:** `packages/graphrag/graphrag/index/workflows/create_final_documents.py`

### 核心逻辑（仅 23 行）

```python
async def create_final_documents(
    text_units_table: Table,
    documents_table: Table,
    output_table: Table,
) -> list[dict[str, Any]]:
    # 阶段 1：构建映射 {document_id → [text_unit_id, ...]}
    mapping: dict[str, list[str]] = {}
    async for row in text_units_table:
        document_id = row.get("document_id", "")
        if document_id:
            mapping.setdefault(document_id, []).append(row["id"])

    # 阶段 2：填充并写入
    sample_rows: list[dict[str, Any]] = []
    async for row in documents_table:
        row["text_unit_ids"] = mapping.get(row["id"], [])     # 用映射表填充
        out = {c: row.get(c) for c in DOCUMENTS_FINAL_COLUMNS} # 字段裁剪到 7 列
        await output_table.write(out)
        if len(sample_rows) < 5:
            sample_rows.append(out)

    return sample_rows
```

### 数据流图

```
阶段 1：扫描 text_units → 构建 mapping
  text_units 表                        mapping
  ┌──────────────────────┐             ┌─────────────────────┐
  │ id="tu_aaa"          │──┐          │ "doc_01": [         │
  │ document_id="doc_01" │  │          │   "tu_aaa",         │
  │ id="tu_bbb"          │  │ ──→      │   "tu_bbb",         │
  │ document_id="doc_01" │  │          │   "tu_ccc"          │
  │ id="tu_ccc"          │──┘          │ ],                  │
  │ document_id="doc_01" │             │ "doc_02": ["tu_ddd"]│
  │ id="tu_ddd"          │             └─────────┬───────────┘
  │ document_id="doc_02" │                       │
  └──────────────────────┘                       │
                                                 ▼
阶段 2：扫描 documents → 查 mapping → 填充 → 写入
  documents 表                          documents 表（覆写）
  ┌──────────────────┐                 ┌──────────────────────────┐
  │ id="doc_01"      │                 │ id="doc_01"              │
  │ title="..."      │ ──────────→     │ title="..."              │
  │ text="..."       │  row["id"]      │ text="..."               │
  │ ...              │  去 mapping 查  │ text_unit_ids=[tu_aaa,..]│ ← 新增
  └──────────────────┘                 │ creation_date=...        │
                                       └──────────────────────────┘
```

### 设计要点

- **两次扫描而非 pandas merge**：内存友好（只有 `mapping` 字典常驻内存），兼容远程存储流式读取
- `DOCUMENTS_FINAL_COLUMNS` = `["id", "short_id", "title", "text", "text_unit_ids", "creation_date", "raw_data"]`
- 读 documents 时传了 `transformer=transform_document_row`，会把 Parquet 里的 numpy array 转成普通 Python list

---

## 7. `sample_rows` 代码块 Python 语法解析

```python
sample_rows: list[dict[str, Any]] = []
async for row in documents_table:
    row["text_unit_ids"] = mapping.get(row["id"], [])
    out = {c: row.get(c) for c in DOCUMENTS_FINAL_COLUMNS}
    await output_table.write(out)
    if len(sample_rows) < 5:
        sample_rows.append(out)
```

| 语法 | 说明 |
|---|---|
| `list[dict[str, Any]]` | 类型注解，表示 list 里每个元素是 `{字符串键: 任意值}` 的字典。不影响运行，仅供 IDE 和读者参考 |
| `async for` | 异步迭代。底层可能涉及网络 I/O，用 `await` 等待时不阻塞整个程序。初学可当成普通 `for` 理解 |
| `mapping.get(row["id"], [])` | 安全取值。key 不存在返回 `[]` 而不抛 `KeyError` |
| `{c: row.get(c) for c in COLUMNS}` | 字典推导式。遍历标准字段列表，从 row 中提取对应值生成新字典。等价于字段裁剪 |
| `row.get(c)` | 同样安全取值，字段缺失返回 `None` 而不报错 |
| `if len(sample_rows) < 5` | 只保留前 5 行作为样本，用于后续进度报告/调试 |
| `await output_table.write(out)` | 异步写入一行 |

---

## 8. `filter_orphan_relationships` — 清理孤儿关系

**实现文件:** `packages/graphrag/graphrag/index/operations/extract_graph/utils.py`

### 一句话概括

**删除那些 source 或 target 在 entities 表中不存在的 relationship**，防止下游构建图时遇到悬空边。

### 原因

LLM 在抽取关系时可能"幻觉"——引用了一个实体名，但这个实体并没有被抽取成为独立的实体行：

```
entities: ["张三", "李四"]
relationships: [("张三","李四"), ("张三","王五")]
                                         ↑ "王五" 不在 entities 里 → 孤儿关系
```

### 核心代码

```python
def filter_orphan_relationships(
    relationships: pd.DataFrame,
    entities: pd.DataFrame,
) -> pd.DataFrame:
    if relationships.empty or entities.empty:
        return relationships.iloc[0:0].reset_index(drop=True)  # 返回空 DataFrame

    entity_titles = set(entities["title"])           # set → O(1) 查找
    mask = (relationships["source"].isin(entity_titles) &
            relationships["target"].isin(entity_titles))  # 两端都必须存在
    filtered = relationships[mask].reset_index(drop=True)

    if len(relationships) - len(filtered) > 0:
        logger.warning("Dropped %d relationship(s)...", dropped)
    return filtered
```

### 关键细节

- **`set` 而非 `list`**：`x in set` 是 O(1) 哈希查找，`x in list` 是 O(n) 线性扫描
- **`&` 而非 `and`**：对 pandas Series 必须用按位与 `&`，不能用 Python 逻辑与 `and`
- **`iloc[0:0]`**：取空切片，返回结构相同但 0 行的 DataFrame，而不是 `None`

### 调用位置

| 位置 | 场景 |
|---|---|
| `extract_graph.py` | 首次索引：LLM 抽取完后立即过滤 |
| `update_entities_relationships.py` | 增量更新：新旧关系合并后再次过滤 |

---

## 9. `extract_covariates` — 从文本中提取"声明/主张"

**实现文件:** `packages/graphrag/graphrag/index/workflows/extract_covariates.py`

### 一句话概括

**用 LLM 从 text_units 中提取"声明（claim）"**——文本中关于实体的事实性陈述，如"张三于2023年入职某公司"。

### Covariate 数据结构

```python
@dataclass
class Covariate:
    covariate_type: str    # 类型，如 "claim"
    subject_id: str        # 主体（谁）
    object_id: str         # 客体（对谁/什么）
    type: str              # 声明的类别
    status: str            # 状态（如 "true" / "false"）
    start_date: str        # 生效时间
    end_date: str          # 失效时间
    description: str       # 描述文本
    source_text: list[str] # 原文证据
    doc_id: str            # 来源文档
    record_id: int         # 记录编号
    id: str                # 唯一 ID
```

### 核心流程

```
text_units 表
     │
     ▼
ClaimExtractor 逐条处理
     │
     ├── 构造 prompt：填入文本 + 实体类型 + 声明描述
     ├── LLM 返回结构化元组：(主体<|>客体<|>类型<|>状态<|>日期<|>描述<|>原文)
     ├── Gleaning 循环：max_gleanings > 0 时追问"还有更多吗？"直到 LLM 回答 N
     ├── _parse_claim_tuples() 解析 <|> 分隔的响应
     │
     ▼
covariates 表
```

### LLM 响应格式

```
(张三<|>某公司<|>入职<|>true<|>2023-01-01<|><|>张三于2023年加入某公司<|>原文段落)
```

### 和 entity/relationship 提取的对比

| 维度 | extract_graph | extract_covariates |
|---|---|---|
| 提取内容 | 实体 + 关系 | 声明 / 主张 |
| 粒度 | 粗（谁认识谁） | 细（谁做了什么事、有什么属性） |
| 例子 | `(张三) -[同事]-> (李四)` | `张三 于 2023年 获得 博士学位` |
| 下游用途 | 构建知识图谱、社区检测 | 增强问答、事实核查 |
| 默认开启 | ✅ | ❌（需要 `config.extract_claims.enabled = true`） |

---

## 10. `create_final_text_units` 详细代码分析

**实现文件:** `packages/graphrag/graphrag/index/workflows/create_final_text_units.py`

### 核心任务

**把 entity、relationship、covariate 的 ID 反向关联回 text_units**。告诉每个 text_unit："你的文本里抽出了哪些实体、哪些关系、哪些声明"。

与 `create_final_documents` 的对比：

| 维度 | create_final_documents | create_final_text_units |
|---|---|---|
| 关联方向 | text_units → documents | entities/relationships/covariates → text_units |
| 填充字段 | 1 个（`text_unit_ids`） | 3 个（`entity_ids`, `relationship_ids`, `covariate_ids`） |
| 数据来源 | text_units 表 | entities、relationships、covariates 三张表 |

### 数据流

```
entities ─────────┐
relationships ────┤
covariates ───────┤
                  ├──→ 构建反向映射 ──→ 填充 text_units ──→ 写入
text_units ───────┘
```

### 完整代码（核心 40 行）

```python
async def create_final_text_units(
    text_units_table, entities_table, relationships_table,
    output_table, covariates_table,
) -> list[dict[str, Any]]:
    # 步骤 1：构建三个反向映射
    entity_map = await _build_multi_ref_map(entities_table)
    relationship_map = await _build_multi_ref_map(relationships_table)
    covariate_map = (
        await _build_covariate_map(covariates_table)
        if covariates_table is not None else {}
    )

    # 步骤 2：遍历 text_units，填充并写入
    sample_rows = []
    human_readable_id = 0
    async for row in text_units_table:
        fields = {
            "id": row["id"],
            "human_readable_id": human_readable_id,            # 0,1,2...递增
            "text": row["text"],
            "n_tokens": row["n_tokens"],
            "document_id": row["document_id"],
            "entity_ids": entity_map.get(row["id"], []),       # ← 新增
            "relationship_ids": relationship_map.get(row["id"], []), # ← 新增
            "covariate_ids": covariate_map.get(row["id"], []),       # ← 新增
        }
        out = {c: fields[c] for c in TEXT_UNITS_FINAL_COLUMNS}
        await output_table.write(out)
        human_readable_id += 1
    return sample_rows
```

### 两个辅助函数的区别

```python
async def _build_multi_ref_map(table: Table) -> dict[str, list[str]]:
    # entity/relationship 的 text_unit_ids 是 list 字段（一对多）
    # 例：entity_1.text_unit_ids = ["tu_a", "tu_b"]
    result: dict[str, list[str]] = {}
    async for row in table:
        for tuid in row["text_unit_ids"]:       # 内层 for 遍历 list
            result.setdefault(tuid, []).append(row["id"])
    return result

async def _build_covariate_map(table: Table) -> dict[str, list[str]]:
    # covariate 的 text_unit_id 是单值字段（一对一）
    # 例：cov_1.text_unit_id = "tu_a"
    result: dict[str, list[str]] = {}
    async for row in table:
        result.setdefault(row["text_unit_id"], []).append(row["id"])
    return result
```

### 协变量可选处理

```python
has_covariates = (
    config.extract_claims.enabled
    and await context.output_table_provider.has("covariates")
)
cov_ctx = (
    context.output_table_provider.open("covariates")
    if has_covariates
    else nullcontext()        # Python 标准库，with 后返回 None
)
```

`nullcontext()` 让 async with 块统一处理，covariates 不存在时 `covariates_table` 为 `None`。

### 最终 text_units 表结构

```python
TEXT_UNITS_FINAL_COLUMNS = [
    "id",               # SHA-512 哈希
    "short_id",         # 此处未填充，为 None
    "text",             # chunk 文本
    "n_tokens",         # token 数量
    "document_id",      # 所属文档 ID
    "entity_ids",       # ← 从本 chunk 抽出的实体 ID 列表
    "relationship_ids", # ← 从本 chunk 抽出的关系 ID 列表
    "covariate_ids",    # ← 从本 chunk 抽出的声明 ID 列表
]
```

### 在管道中的位置

```
create_base_text_units   →  text_units 表（只有 id, text, n_tokens, document_id）
        │
extract_graph             →  entities 表, relationships 表
extract_covariates        →  covariates 表
        │
create_final_text_units   →  text_units 表
        │                    （增加了 entity_ids, relationship_ids, covariate_ids）
        ▼
后续社区检测、向量嵌入...
```

**一句话：** `create_final_text_units` 把 entities/relationships/covariates 与 text_units 的多对多关系完整建立起来，每条 text_unit 都知道自己"产出"了哪些图元素。
