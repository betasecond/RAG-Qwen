# API 接口文档 for RAGdemo

## 文档概述

本技术文档为 `RAGdemo` 项目提供了完整的 API 接口规范，旨在为开发者提供清晰、准确的接口调用指南。

## 接口概览

| 功能模块 | 接口地址 | 方法 | 描述 |
| :--- | :--- | :--- | :--- |
| **知识库检索** | `/search` | `POST` | 根据输入问题在知识库中进行语义搜索 |
| **精排搜索** | `/search/reranked` | `POST` | 使用Qwen3-Reranker cross-encoder进行精排检索 |
| **监视页面** | `/monitor` | `GET` | 知识库监视页面 |
| **统计信息** | `/monitor/stats` | `GET` | 获取知识库统计信息 |
| **数据样本** | `/monitor/samples` | `GET` | 获取知识库数据样本 |

---

## 核心接口详解

### `POST /search`

此端点是 `RAGdemo` 服务的核心功能，用于接收用户的自然语言问题，并通过 RAG（检索增强生成）模型在向量知识库中执行语义搜索，返回最相关的知识片段。

#### 请求 (Request)

**请求体 (Body)**

请求体必须是一个 JSON 对象，包含以下字段：

| 字段名 | 类型 | 是否必需 | 描述 | 默认值 |
| :--- | :--- | :--- | :--- | :--- |
| `query` | `string` | 是 | 需要在知识库中搜索的问题或关键词。 | |
| `top_k` | `integer`| 否 | 指定需要返回的最相关结果的数量。 | `5` |

**请求示例**

```json
{
  "query": "在Python中如何定义一个函数？",
  "top_k": 3
}
```

#### 响应 (Response)

**成功响应 (200 OK)**

成功处理请求后，API 将返回一个 JSON 对象，其中包含了检索结果和相关的元数据。

| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| `provider` | `string` | 当前使用的 Embedding 服务提供商（例如 "ollama" 或 "dashscope"）。|
| `query` | `string` | 用户原始的查询内容。 |
| `results`| `array` | 一个包含多个搜索结果对象的数组。 |

**`result` 对象结构**

数组中的每个对象代表一个从知识库中检索到的相关条目。

| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| `id` | `string` | 该知识条目在数据库中的唯一标识符。 |
| `content` | `string` | 检索到的知识原文，通常是"问题+选项"的组合。 |
| `metadata` | `object` | 包含与该条目相关的详细元数据。 |
| `distance`| `number` | 语义相似度距离。值越小表示相关性越高。 |

**`metadata` 对象结构**

| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| `question_id` | `string` | 原始问题的唯一 ID。 |
| `question_text`| `string` | 原始问题的完整文本。 |
| `option_key`| `string` | 选项的标识符（如 "A", "B"）。 |
| `option_text`| `string` | 选项的完整文本。 |
| `is_correct`| `boolean`| 标记该选项是否为正确答案。 |

**成功响应示例**

```json
{
  "provider": "ollama",
  "query": "在Python中如何定义一个函数？",
  "results": [
    {
      "id": "q1_B",
      "content": "题目：在Python中，哪个关键字用于定义一个函数？ 选项B：def",
      "metadata": {
        "question_id": "1",
        "question_text": "在Python中，哪个关键字用于定义一个函数？",
        "option_key": "B",
        "option_text": "def",
        "is_correct": true
      },
      "distance": 0.2185
    },
    {
      "id": "q1_A",
      "content": "题目：在Python中，哪个关键字用于定义一个函数？ 选项A：func",
      "metadata": {
        "question_id": "1",
        "question_text": "在Python中，哪个关键字用于定义一个函数？",
        "option_key": "A",
        "option_text": "func",
        "is_correct": false
      },
      "distance": 0.4331
    }
  ]
}
```

#### 错误处理

| 状态码 | 错误详情 | 描述 |
| :--- | :--- | :--- |
| `500 Internal Server Error` | `"搜索过程中发生内部错误: {e}"` | 在搜索过程中发生未预期的服务器端错误。 |
| `503 Service Unavailable` | `"服务不可用: RAGManager 初始化失败..."` | RAG 核心服务在应用启动时未能成功初始化。 |
| `503 Service Unavailable` | `"服务尚未完全初始化，请稍后再试。"` | 服务正在启动，但核心 RAG 管理器尚未准备就绪。 |

---

## 精排API说明（Qwen3-Reranker Cross-Encoder）

### `POST /search/reranked`

对输入查询进行知识库检索，并用Qwen3-Reranker cross-encoder模型对候选切片进行高质量相关性精排。

#### 请求体（JSON）
| 字段名   | 类型    | 必需 | 说明                 | 默认值 |
|----------|---------|------|----------------------|--------|
| query    | string  | 是   | 用户查询             |        |
| top_k    | int     | 否   | 返回结果数量         | 5      |

#### 响应体（JSON）
| 字段名         | 类型    | 说明                         |
|----------------|---------|------------------------------|
| provider       | string  | 使用的embedding服务           |
| query          | string  | 用户原始查询                  |
| rerank_strategy| string  | 精排策略（qwen3-reranker）    |
| results        | array   | 精排后的结果列表              |

**results数组结构**
| 字段名        | 类型    | 说明                       |
|---------------|---------|----------------------------|
| id            | string  | 知识片段唯一ID             |
| content       | string  | 知识片段内容               |
| metadata      | object  | 元数据                     |
| distance      | float   | embedding初步检索距离       |
| original_rank | int     | embedding初步检索排名       |
| rerank_score  | float   | Qwen3-Reranker相关性分数（越大越相关）|
| final_rank    | int     | 精排后排名                  |

#### 响应示例
```json
{
  "provider": "ollama",
  "query": "python里怎么定义一个函数",
  "rerank_strategy": "qwen3-reranker",
  "results": [
    {
      "id": "q1_B",
      "content": "题目：在Python中，哪个关键字用于定义一个函数？ 选项B：def",
      "metadata": {"question_id": "1", ...},
      "distance": 0.21,
      "original_rank": 2,
      "rerank_score": 0.92,
      "final_rank": 1
    },
    ...
  ]
}
```

#### 说明
- `rerank_score`为Qwen3-Reranker cross-encoder输出的yes概率，越大越相关。
- `final_rank`为精排后排名。
- `distance`和`original_rank`为初步embedding检索结果，仅供参考。

---

## 新增接口详解

### `GET /monitor`

知识库监视页面，提供可视化界面查看知识库数据。

#### 响应 (Response)

返回HTML页面，包含以下功能：
- 知识库统计信息展示
- 数据样本浏览（支持分页）
- 搜索功能测试
- 元数据分析

### `GET /monitor/stats`

获取知识库统计信息。

#### 响应 (Response)

**成功响应 (200 OK)**

```json
{
  "collection_name": "exam_questions",
  "total_documents": 100,
  "metadata_analysis": {
    "total_metadata_entries": 100,
    "field_statistics": {
      "question_id": {
        "unique_values": 50,
        "total_occurrences": 100,
        "most_common": ["1", 2]
      }
    },
    "available_fields": ["question_id", "question_text", "option_key", "option_text", "is_correct"]
  },
  "content_analysis": {
    "avg_length": 150,
    "min_length": 50,
    "max_length": 300,
    "length_distribution": {
      "short (0-50)": 10,
      "medium (51-200)": 60,
      "long (201-500)": 25,
      "very_long (500+)": 5
    }
  },
  "timestamp": "2024-01-01T12:00:00"
}
```

### `GET /monitor/samples`

获取知识库数据样本。

#### 查询参数 (Query Parameters)

| 参数名 | 类型 | 是否必需 | 描述 | 默认值 |
| :--- | :--- | :--- | :--- | :--- |
| `limit` | `integer` | 否 | 返回的样本数量。 | `10` |
| `offset` | `integer` | 否 | 偏移量，用于分页。 | `0` |

#### 响应 (Response)

**成功响应 (200 OK)**

```json
{
  "collection_name": "exam_questions",
  "samples": [
    {
      "id": "q1_B",
      "content": "题目：在Python中，哪个关键字用于定义一个函数？ 选项B：def",
      "metadata": {
        "question_id": "1",
        "question_text": "在Python中，哪个关键字用于定义一个函数？",
        "option_key": "B",
        "option_text": "def",
        "is_correct": true
      },
      "index": 0
    }
  ],
  "pagination": {
    "total": 100,
    "limit": 10,
    "offset": 0,
    "has_more": true
  },
  "timestamp": "2024-01-01T12:00:00"
}
```

## 注意事项

1.  **数据格式**: 所有字符串数据均使用 `UTF-8` 编码。
2.  **内容类型**: 所有 `POST` 请求的 `Content-Type` 必须是 `application/json`。
3.  **服务依赖**: API 的正常运行依赖于后端配置的 Embedding 服务（Ollama 或 DashScope）和 ChromaDB 数据库。请确保这些依赖项已正确安装、配置并正在运行。
4.  **监视功能**: 监视页面需要Jinja2模板引擎支持，确保已安装相关依赖。
5.  **精排功能**: 精排功能使用Qwen3-Reranker cross-encoder模型，首次加载模型较慢，但后续推理速度快，能显著提升检索质量。 