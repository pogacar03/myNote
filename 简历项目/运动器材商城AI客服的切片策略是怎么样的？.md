# 运动器材商城 AI 客服的切片策略是怎么样的？

### 面试回答

我们的知识库主要是纯文本商品说明和配置文档，没有图片和表格，所以采用的是**结构优先 + 语义切片 + metadata 过滤**的方案。

先按照标题、章节和段落进行切分，尽量保证一个 chunk 内语义完整；对于过长章节，再按大约 300～500 中文字进一步切分，并保留 50～100 字左右的 overlap。每个 chunk 会携带 `productId`、`category`、`brand`、`section`、`docType` 等 metadata。

检索时先根据商品、类目等信息做 metadata filter，再进行向量召回，最后通过 Rerank 进一步排序。对于价格、库存、SKU、尺寸等强结构化和精确数值信息，不走纯 RAG，而是直接查数据库或业务接口，避免向量召回导致数值错误。

### 核心切片策略

不是全文按固定长度硬切，而是：

```text
文档结构切分
↓
语义切分
↓
超长 chunk 再按长度兜底
```

优先按这些 section 切：

```text
商品介绍
规格参数
使用方法
适用场景
维护保养
售后规则
FAQ
```

例如一篇商品文档：

```text
商品：速干跑步T恤

适用场景：
适合春夏季跑步、健身、日常训练。

材质：
88% 聚酯纤维，12% 氨纶。

尺码：
S...
M...
L...

洗涤说明：
建议30℃以下水洗，不建议烘干。

退换货：
未拆吊牌且不影响二次销售，7天内可退换。
```

可以切成：

```text
chunk1：商品介绍 + 适用场景 + 材质
chunk2：尺码说明
chunk3：洗涤说明
chunk4：退换货规则
```

核心原则是：

> 宁可稍微长一点，也不要把一个完整语义拆碎。

### Chunk 大小

可先使用：

```text
300～500 中文字 / chunk
50～100 字 overlap
```

但长度只是兜底参数，真正的优先级是：

```text
标题 / Section
↓
段落
↓
语义完整性
↓
长度限制
```

### Metadata 设计

每个 chunk 都携带商品相关 metadata，例如：

```json
{
  "product_id": "P10086",
  "product_name": "速干跑步T恤",
  "category": "运动服饰",
  "brand": "XX",
  "section": "洗涤说明",
  "doc_type": "product_manual"
}
```

这样用户问：

```text
这件跑步T恤能不能烘干？
```

可以先根据上下文识别商品：

```text
product_id = P10086
```

然后：

```text
metadata filter
↓
product_id = P10086
↓
向量召回
↓
Rerank
```

避免召回其他商品的说明。

### 完整检索链路

```text
用户问题
↓
识别商品 / 类目 / 意图
↓
metadata filter
↓
Embedding 向量召回
↓
Top-K
↓
Rerank
↓
LLM 生成答案
```

如果使用 Pinecone，则可以设计为：

```text
vector = embedding(chunk_text)

metadata =
product_id
category
brand
section
doc_type
```

### 哪些信息不应该完全依赖 RAG

像下面这种精确参数：

```text
价格
库存
SKU
重量
尺寸
承重
实时促销信息
```

不建议只依赖向量搜索。

如果这些数据已经存在业务数据库中，优先：

```text
结构化信息
→ MySQL / ES / 业务 API

非结构化知识
→ RAG
```

例如：

```text
价格 / 库存 / SKU / 尺寸
→ 查业务数据库

使用方法 / 材质说明 / 售后说明 / 适用场景
→ RAG
```

### 一句话速记

**纯文本文档先按标题和 section 做语义切片，过长再按 300～500 字兜底；每个 chunk 带商品 metadata，检索时先 metadata filter，再向量召回 + Rerank；价格、库存、SKU 等精确结构化数据直接查数据库，不走纯 RAG。**
