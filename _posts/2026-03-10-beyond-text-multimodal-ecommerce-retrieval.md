---
layout: post
title: "Beyond Text: How We Built Multimodal Retrieval for E-Commerce Search"
date: 2026-03-10
categories: [research, e-commerce, multimodal]
description: "Most e-commerce search systems rely purely on text — but product images carry signals that text often misses. We propose a two-stage vision-language alignment framework for retrieval, achieving +5% Recall@100 over text-only baselines on large-scale data."
bilingual: true
---

<div class="lang-block" data-lang="en" markdown="1">

## Why Text Alone Is Not Enough

When you search for "red floral summer dress" on an e-commerce platform, what should the retrieval system do? The obvious answer is: find products whose text descriptions match your query. But here's the problem — a significant fraction of products have sparse, low-quality, or even misleading text descriptions. The image, however, rarely lies.

This is the core motivation behind our paper [**Beyond Text: Aligning Vision and Language for Multimodal E-Commerce Retrieval**](https://arxiv.org/abs/2603.04836) (arXiv:2603.04836).

---

## The Industrial Reality

Most production retrieval systems in e-commerce are **text-only two-tower models**:

```
Query encoder  →  query embedding
Product encoder →  product text embedding
Similarity = dot_product(query_emb, product_emb)
```

This works well when product text is rich and accurate. But in practice:

- **Short titles**: "Dress Women Summer" — no color, no pattern, no material
- **Missing attributes**: images show the product clearly, but the seller never wrote it down
- **Multilingual noise**: machine-translated descriptions with wrong keywords

The image contains all of this information. A model that can *see* the product should outperform one that can only *read* about it.

---

## Our Approach: Two-Stage Alignment

The key insight is that naively concatenating image and text features doesn't work well. You need **staged alignment**:

### Stage 1: Domain-Specific Fine-Tuning

General vision-language models (CLIP, BLIP-2) are pre-trained on web data. E-commerce products look very different from web images — white backgrounds, multiple angles, zoomed-in details. We fine-tune the vision encoder on large-scale product image data before any fusion.

### Stage 2: Cross-Modal Fusion

After domain adaptation, we introduce a **modality fusion network** that:

1. Takes the query text embedding $\mathbf{q}$
2. Takes the product text embedding $\mathbf{t}$ and image embedding $\mathbf{v}$
3. Computes cross-modal attention to capture complementary signals:

$$\mathbf{p}_{fused} = \text{Attention}(\mathbf{t}, \mathbf{v}) \cdot \mathbf{W}_{fusion}$$

The intuition: for a query like "red dress", the image provides color and style signals that the text might miss. For a query like "machine washable", the text is more reliable. The fusion network learns to weight these dynamically.

---

## What We Found

Experiments on large-scale e-commerce datasets show:

| Model | Recall@100 | NDCG@10 |
|-------|-----------|---------|
| Text-only baseline | 72.3 | 41.2 |
| + Image (naive concat) | 73.1 | 41.8 |
| + Domain fine-tuning | 75.6 | 43.5 |
| + Our fusion network | **77.4** | **45.1** |

The gains are especially large for **visually distinctive queries** (color, pattern, style) and **products with sparse text**.

---

## Engineering Considerations

One challenge in production: **latency**. A two-tower model is fast because you pre-compute product embeddings offline. Adding an image encoder increases the embedding dimension and offline compute, but doesn't hurt online latency — the query side remains text-only.

The product embedding is computed once and indexed. At query time:

```
query text → query encoder → query_emb
ANN search over product_emb index → top-K candidates
```

This means the multimodal enrichment is essentially **free at serving time**.

---

## Takeaways

1. **Domain-specific fine-tuning matters more than architecture** — a fine-tuned ViT-B outperforms a generic ViT-L
2. **Two-stage alignment** (domain adaptation → cross-modal fusion) is better than end-to-end training from scratch
3. **Multimodal retrieval is production-friendly** — image encoding happens offline, no latency penalty

The full paper is on arXiv: [arXiv:2603.04836](https://arxiv.org/abs/2603.04836).

</div>

<div class="lang-block" data-lang="zh" markdown="1">

## 为什么文本不够用

当你在电商平台搜索"红色碎花夏裙"时，检索系统应该做什么？直觉上的答案是：找到文字描述匹配的商品。但问题在于——相当一部分商品的文字描述稀疏、质量低，甚至有误导性。而图片，几乎不会说谎。

这就是我们论文 [**Beyond Text: Aligning Vision and Language for Multimodal E-Commerce Retrieval**](https://arxiv.org/abs/2603.04836) 的核心动机。

---

## 工业现实

大多数电商生产环境的检索系统是**纯文本双塔模型**：

```
Query encoder  →  query embedding
Product encoder →  product text embedding
相似度 = dot_product(query_emb, product_emb)
```

当商品文本丰富准确时效果不错。但实际上：

- **标题过短**："连衣裙 女 夏季" — 没有颜色、花纹、材质
- **属性缺失**：图片清晰展示了商品，但卖家从未写进描述
- **多语言噪声**：机器翻译的描述关键词错误

图片包含了所有这些信息。能"看到"商品的模型，应该优于只能"读到"商品的模型。

---

## 我们的方法：两阶段对齐

核心洞察是：简单地拼接图文特征效果不好，需要**分阶段对齐**：

### 第一阶段：领域特定微调

通用视觉语言模型（CLIP、BLIP-2）在网络数据上预训练。电商商品图片与网络图片差异很大——白色背景、多角度、细节特写。我们在大规模商品图片数据上对视觉编码器进行微调，再做融合。

### 第二阶段：跨模态融合

领域适配后，引入**模态融合网络**：

1. 输入 query 文本 embedding $\mathbf{q}$
2. 输入商品文本 embedding $\mathbf{t}$ 和图片 embedding $\mathbf{v}$
3. 通过跨模态注意力捕获互补信号：

$$\mathbf{p}_{fused} = \text{Attention}(\mathbf{t}, \mathbf{v}) \cdot \mathbf{W}_{fusion}$$

直觉：对于"红色连衣裙"这样的 query，图片提供了文字可能缺失的颜色和款式信号；对于"可机洗"这样的 query，文字更可靠。融合网络学会动态权衡。

---

## 实验结果

在大规模电商数据集上：

| 模型 | Recall@100 | NDCG@10 |
|------|-----------|---------|
| 纯文本基线 | 72.3 | 41.2 |
| + 图片（简单拼接） | 73.1 | 41.8 |
| + 领域微调 | 75.6 | 43.5 |
| + 我们的融合网络 | **77.4** | **45.1** |

对**视觉特征明显的 query**（颜色、花纹、风格）和**文本稀疏的商品**，提升尤为显著。

---

## 工程考量

生产环境的挑战之一是**延迟**。双塔模型快的原因是商品 embedding 离线预计算。加入图片编码器增加了离线计算量，但不影响在线延迟——query 侧仍然是纯文本。

商品 embedding 一次计算后建索引，query 时：

```
query text → query encoder → query_emb
ANN 检索 product_emb 索引 → top-K 候选
```

多模态增强在**服务时几乎零成本**。

---

## 总结

1. **领域微调比架构更重要** — 微调过的 ViT-B 优于通用 ViT-L
2. **两阶段对齐**（领域适配 → 跨模态融合）优于端到端从头训练
3. **多模态检索对生产友好** — 图片编码离线完成，无延迟代价

完整论文：[arXiv:2603.04836](https://arxiv.org/abs/2603.04836)

</div>
