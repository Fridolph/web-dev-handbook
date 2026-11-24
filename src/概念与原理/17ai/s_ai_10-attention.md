# Attention（注意力机制）

## 概念

注意力机制（Attention Mechanism）是 Transformer 的核心组件，它让模型在处理每个 token 时能够**动态地关注输入序列中的相关部分**，而不是平等对待所有 token。

类比来说：人类阅读时不会逐字逐句地平等处理，而是对关键信息"pay attention"。Attention 机制赋予 AI 同样的能力。

## 核心原理

### Scaled Dot-Product Attention

Transformer 的自注意力计算公式：

```
Attention(Q, K, V) = softmax(QK^T / √d_k) · V
```

**三个核心矩阵**：

- **Q（Query，查询）**：当前 token 想"查询"什么——我要找什么信息
- **K（Key，键）**：每个 token 的"索引"标签——我有什么信息可被查询
- **V（Value，值）**：每个 token 的"内容"——我实际携带的信息

**计算步骤**：

1. 计算 Q 和所有 K 的点积 → 得到注意力分数矩阵
2. 除以 √d_k（缩放因子，防止梯度过小）
3. 通过 Softmax 归一化为概率分布（注意力权重）
4. 用权重对 V 做加权求和 → 得到当前 token 的上下文表示

### Multi-Head Attention（多头注意力）

将 Q、K、V 通过不同的线性投影映射到多个子空间，在每个子空间独立计算 Attention，最后拼接。

**为什么需要多头**：
- 每个头可以学习**不同类型的注意力模式**
- 有的头关注语法结构，有的关注语义关联，有的关注位置关系
- 多个头 = 从不同角度理解输入

### Cross-Attention vs Self-Attention

- **Self-Attention**：Q、K、V 来自同一个序列，让序列内互相注意
- **Cross-Attention**：Q 来自当前序列，K、V 来自另一个序列（如 Encoder 的输出）

## 注意力机制的类型

| 类型 | 计算范围 | 复杂度 | 用途 |
|------|---------|--------|------|
| 全局注意力 | 全部 token | O(n²) | 标准 Transformer |
| 因果/掩码注意力 | 只看左边 | O(n²) | GPT 等自回归模型 |
| 稀疏注意力 | 部分 token | O(n·log n) | 长文本优化 |
| 滑动窗口注意力 | 固定窗口 | O(n·w) | Mistral 等 |
| Flash Attention | 算法优化 | O(n²) 更快 | 主流 LLM 标配 |

## 注意力权重不是「解释」

一个注意力权重只描述：**某一层、某一个头、某个查询位置**上，汇总信息时分配给某个可见位置的比例。

它不能直接说明：

- 某个词在整段文本里有多重要；
- 模型得出最终结论的完整原因；
- 模型是否正确执行了某条业务规则。

最终输出还受其他注意力头、前馈网络、残差路径和后续层影响。所以：

- **不要把单张注意力热力图当因果解释**；
- 不要把多头命名成「金额头」「订单头」——没有证据支持这种固定分工；
- 真要分析，也得看多层多头的聚合结果，而且仍不等于因果证明。

## 长上下文为什么更慢、更贵

对**标准密集自注意力**，处理长度为 n 的已知序列时，位置之间的关系数量随 n² 增长。但这条结论有边界：

- 只适用于标准密集注意力，稀疏 / 滑动窗口 / FlashAttention 等实现不同；
- **不能直接外推**「上下文翻倍 → 延迟 4 倍、显存 4 倍」，实际还受硬件、批处理、实现影响；
- 生成阶段是逐 Token 的 Decode，KV Cache 省掉了重复计算，但缓存本身占内存，且随长度与并发增长。

费用也不等于计算量：API 计费看实际输入/输出 Token、计费规则与缓存命中。

> **窗口上限变大 ≠ 每次请求更贵；实际发送的上下文变长，才通常增加输入处理量和费用。**

## 对 Agent 的启发

1. **信息必须可用，但可见 ≠ 用对**：规则没进上下文模型就用不上；进了上下文也不保证它注意到、理解对。关键结论仍要业务校验。
2. **Context Budget 仍然重要**：能处理长上下文，不等于该把历史、文档、工具结果全塞进去，见 [Context Budget](./s_ai_36-context_budget)。
3. **注意力不是权限**：模型「关注」了退款规则，不代表它有查订单、确认状态或发起退款的授权。
4. **长上下文质量要评估**：能否找到关键事实、区分有效与过时规则、抗无关文本干扰、引用是否准确、延迟与成本是否可接受。

想亲手跑一遍 Q/K/V → Softmax → 加权汇总，可看示例：[examples/attention-demo](https://github.com/Fridolph/AI-Journey-Fighting/tree/main/examples/attention-demo)。

## 面试常问

- Self-Attention 为什么需要 Q、K、V 三个矩阵？一个不行吗？
- 注意力分数的 softmax 缩放因子 √d_k 的作用是什么？
- 注意力机制的计算复杂度是多少？如何处理长文本？
- 注意力权重能直接当作模型解释吗？
- 上下文翻倍，延迟一定翻 4 倍吗？

## 参考

- [Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- [The Illustrated Attention](https://jalammar.github.io/visualizing-neural-machine-translation-mechanics-of-seq2seq-models-with-attention/)
