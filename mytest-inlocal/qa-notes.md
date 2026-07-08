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
