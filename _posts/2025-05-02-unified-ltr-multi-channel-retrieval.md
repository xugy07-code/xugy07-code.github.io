---
layout: post
title: "Multi-Channel Retrieval Fusion: From Heuristic Weights to Unified Ranking"
date: 2025-05-02
tags: [Search, Learning-to-Rank, E-Commerce, Recommendation]
description: "E-commerce search pulls candidates from text, semantic, and behavioral channels — then merges them with hand-tuned weights. We replace that with a unified LTR model trained end-to-end, deployed at Target.com with +2.85% conversion lift in A/B testing."
bilingual: true
---

<div class="lang-block" data-lang="zh" markdown="1">

> 本文是对论文 [Unified Learning-to-Rank for Multi-Channel Retrieval in Large-Scale E-Commerce Search](https://arxiv.org/abs/2602.23530)（arXiv 2602.23530）的解读。该论文来自 Target.com 搜索团队，已部署于生产环境，在线 A/B 实验中用户转化率提升 **+2.85%**。

---

## 背景：电商搜索为什么需要多通道检索？

大规模电商搜索面临一个根本矛盾：商品库有数百万 SKU，但每次查询必须在极低延迟下返回结果。单一检索通道无法同时满足所有需求——

- **词法匹配通道**（BM25 类）：精确匹配关键词，但召回率有限
- **语义检索通道**（向量检索）：理解语义，但可能漏掉精确匹配
- **热销/新品/季节性通道**：满足业务目标，但与相关性无直接关联

因此，现代电商搜索系统普遍采用**多通道检索**：每个通道独立召回 top-$n_k$ 个候选，最终合并成一个候选池，再交给排序模型处理。

$$
\mathcal{R}(q) = \bigcup_{k=1}^{K} \mathcal{R}_k^{(n_k)}(q)
$$

**问题来了：如何把来自不同通道、分布完全不同的候选合并成一个最优排序？**

---

## 现有方法的局限

### Reciprocal Rank Fusion（RRF）

RRF 是最常见的无监督融合方法，每个候选的得分由其在各通道中的排名倒数求和：

$$
\text{score}_\text{RRF}(i) = \sum_{k=1}^{K} \frac{1}{r_k(i) + c}
$$

其中 $r_k(i)$ 是候选 $i$ 在通道 $k$ 中的排名，$c$ 是平滑常数（通常取 60）。

**缺陷**：权重固定，不感知 query。同一个通道对不同 query 的贡献差异极大，但 RRF 无法区分。

### Weighted Interleaving（WI）

WI 按预设的通道权重概率性地交叉采样各通道的结果。比 RRF 灵活一点，但权重仍然是全局固定的，无法根据 query 动态调整，也无法建模通道间的交互。

**核心问题**：这两种方法都把通道当作独立的黑盒，忽略了：
1. 不同 query 对不同通道的依赖程度不同
2. 通道之间存在候选重叠和互补关系
3. 业务目标（转化率）与排名位置之间的复杂关系

---

## 论文方案：统一排序模型（Unified Ranking）

### 核心思路

把多通道融合重新定义为一个**有监督的 Learning-to-Rank 问题**：

> 不再给通道分配固定权重，而是把通道信息作为特征，训练一个统一的排序函数 $f(q, i; \theta)$，直接优化业务目标。

模型选择 **GBDT（梯度提升决策树）**，原因是：在有大量结构化特征的工业场景下，GBDT 在延迟和效果上仍然与深度模型持平甚至更优，且推理速度快，满足 p95 < 50ms 的生产要求。

---

## 三个关键设计

### 1. 数据表示：query–item–week 三元组

训练样本以**周**为粒度聚合，而不是单次 session。周级聚合平衡了时效性与统计稳定性，同时能捕捉季节性和趋势变化。模型同时使用长期（历史销量）和短期（近期点击速度）特征，平衡稳定商品和新兴趋势商品。

### 2. 标签构造：转化加权的参与度标签

用户行为存在一个转化漏斗：曝光 → 点击 → 加购 → 购买。论文定义了一个加权标签：

$$
L(q, i, w) = a \cdot P_{q,i,w} + b \cdot A_{q,i,w} + c \cdot C_{q,i,w}
$$

权重按语料库级别的转化率自动校准：

$$
a = 1, \quad b = \frac{|\mathcal{P}|}{|\mathcal{A}|}, \quad c = \frac{|\mathcal{P}|}{|\mathcal{C}|}
$$

**购买的权重最高，点击的权重最低**，直接对齐业务目标。最后做 per-query max 归一化，把标签映射到 $[0, 4]$。

### 3. 特征工程：三类特征

| 特征类别 | 内容 | 作用 |
|---------|------|------|
| **商品特征** | 价格、类目、多时间窗口销量 | 建模商品长期受欢迎程度 |
| **通道感知的 query-item 特征** | 每个通道的检索得分 | 让模型学习 query 对各通道的依赖 |
| **用户参与度特征** | 点击、加购、购买的短期加权聚合 | 捕捉短期用户意图变化 |

**通道感知特征**是核心创新：把所有通道的检索得分都作为特征输入，让模型自己学习在不同 query 下应该信任哪个通道。

---

## 实验结果

| 模型变体 | NDCG@8 | CTR 提升 | 加购提升 | 转化提升 |
|---------|--------|---------|---------|---------|
| WI（基线） | 0.6620 | — | — | — |
| UR | 0.7169 | +0.26% | +1.21%* | +1.28%* |
| UR + 参与度特征 | 0.7799 | +1.52%* | +2.72%* | +2.38%* |
| UR + 参与度特征 + 转化标签 | 0.7994 | +1.46%* | +2.81%* | **+2.85%*** |

*表示 95% 置信水平下统计显著。

三个关键结论：
1. **统一排序 vs. 加权交叉**：仅把融合问题换成有监督 LTR，转化率就提升 +1.28%，说明通道间交互信息有价值
2. **参与度特征贡献最大**：短期用户行为信号比商品长期历史销量更能预测当前搜索的转化意图
3. **转化加权标签**：让模型更关注购买行为而不是点击行为，进一步提升 +0.47%

---

## 总结

这篇论文的核心贡献：**把多通道融合从"给通道分配固定权重"变成"让模型学习 query 对通道的依赖"**。技术上没有特别复杂的创新，但正确的问题定义 + 正确的特征工程，在真实生产环境中取得了显著的业务提升。

> **论文链接**：[arXiv:2602.23530](https://arxiv.org/abs/2602.23530)

</div>

<div class="lang-block" data-lang="en" markdown="1">

> This post is a reading note on [Unified Learning-to-Rank for Multi-Channel Retrieval in Large-Scale E-Commerce Search](https://arxiv.org/abs/2602.23530) (arXiv 2602.23530), from the Target.com search team. The system is deployed in production and achieved a **+2.85% improvement in user conversion** in online A/B experiments.

---

## Background: Why Multi-Channel Retrieval?

Large-scale e-commerce search faces a fundamental tension: the catalog contains millions of SKUs, yet every query must return results under strict latency budgets. No single retrieval channel can satisfy all needs simultaneously:

- **Lexical matching** (BM25-style): precise keyword matching, but limited recall
- **Semantic retrieval** (dense vectors): understands intent, but may miss exact matches
- **Popularity / freshness / seasonal channels**: serve business objectives, but not directly tied to relevance

Modern systems therefore use **multi-channel retrieval**: each channel independently retrieves its top-$n_k$ candidates, which are merged into a single candidate pool for re-ranking:

$$
\mathcal{R}(q) = \bigcup_{k=1}^{K} \mathcal{R}_k^{(n_k)}(q)
$$

**The challenge: how do you merge candidates from heterogeneous channels with completely different score distributions into a single optimal ranking?**

---

## Limitations of Existing Methods

### Reciprocal Rank Fusion (RRF)

RRF is the most common unsupervised fusion method. Each candidate's score is the sum of reciprocal ranks across channels:

$$
\text{score}_\text{RRF}(i) = \sum_{k=1}^{K} \frac{1}{r_k(i) + c}
$$

**Limitation**: fixed global weights, query-agnostic. The same channel may be highly relevant for some queries and irrelevant for others — RRF cannot distinguish.

### Weighted Interleaving (WI)

WI probabilistically samples from channels according to predefined weights. Slightly more flexible than RRF, but weights are still globally fixed and cannot model cross-channel interactions.

**Core problem**: both methods treat channels as independent black boxes, ignoring:
1. Different queries depend on different channels to different degrees
2. Candidates overlap and complement each other across channels
3. The complex relationship between ranking position and business KPIs like conversion

---

## Proposed Approach: Unified Ranking Model

### Core Idea

Reformulate multi-channel fusion as a **supervised learning-to-rank problem**:

> Instead of assigning fixed weights to channels, treat channel information as features and train a unified scoring function $f(q, i; \theta)$ that directly optimizes business objectives.

The model uses **GBDT (Gradient Boosted Decision Trees)** because in industrial settings with large structured feature sets, GBDT remains competitive with deep neural models under realistic latency constraints, and its inference speed satisfies the p95 < 50ms production requirement.

---

## Three Key Design Choices

### 1. Data Representation: Query–Item–Week Triples

Training instances are aggregated at **weekly granularity** rather than per-session. Weekly aggregation balances temporal responsiveness and statistical stability, and captures seasonality and trend shifts. The model uses both long-term (historical sales) and short-term (recent click velocity) features to balance established products with newly trending items.

### 2. Label Construction: Conversion-Weighted Engagement Label

User behavior follows a conversion funnel: impression → click → add-to-cart → purchase. The paper defines a weighted label:

$$
L(q, i, w) = a \cdot P_{q,i,w} + b \cdot A_{q,i,w} + c \cdot C_{q,i,w}
$$

Weights are calibrated using corpus-level conversion statistics:

$$
a = 1, \quad b = \frac{|\mathcal{P}|}{|\mathcal{A}|}, \quad c = \frac{|\mathcal{P}|}{|\mathcal{C}|}
$$

**Purchases receive the highest weight, clicks the lowest**, directly aligning training with business objectives. Labels are then per-query max-normalized to $[0, 4]$.

### 3. Feature Engineering: Three Feature Groups

| Feature Group | Content | Purpose |
|--------------|---------|---------|
| **Item features** | Price, category, multi-window sales | Model long-term item popularity |
| **Channel-aware query–item features** | Retrieval scores from all channels | Learn query-dependent channel utility |
| **User engagement features** | Short-term weighted clicks/ATC/purchases | Capture short-term intent shifts |

**Channel-aware features** are the core innovation: all channel retrieval scores are fed as features, letting the model learn which channel to trust for each query. Missing channel scores are treated as NA — GBDT handles sparse features natively.

---

## Results

| Model Variant | NDCG@8 | CTR Lift | ATC Lift | Conversion Lift |
|--------------|--------|---------|---------|----------------|
| WI (Baseline) | 0.6620 | — | — | — |
| UR | 0.7169 | +0.26% | +1.21%* | +1.28%* |
| UR + Engagement Features | 0.7799 | +1.52%* | +2.72%* | +2.38%* |
| UR + EF + Conversion Label | 0.7994 | +1.46%* | +2.81%* | **+2.85%*** |

*Statistically significant at 95% confidence level.

Three key takeaways:
1. **Unified ranking vs. weighted interleaving**: simply replacing heuristic fusion with supervised LTR yields +1.28% conversion, confirming that cross-channel interaction signals are valuable
2. **Engagement features have the largest impact**: short-term user behavior signals are more predictive of current-session conversion intent than long-term item history
3. **Conversion-weighted labels**: orienting the model toward purchase behavior rather than clicks adds a further +0.47% conversion lift

---

## Takeaways

The core contribution of this paper: **shifting multi-channel fusion from "assign fixed weights to channels" to "let the model learn query-dependent channel utility"**. There is no particularly novel algorithm — the value lies in the correct problem formulation and the right feature engineering, which together produce significant business impact in a real production environment.

> **Paper link**: [arXiv:2602.23530](https://arxiv.org/abs/2602.23530)

</div>
