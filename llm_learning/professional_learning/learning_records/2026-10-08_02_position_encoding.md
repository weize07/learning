# 位置表示：相加、RoPE 概览与梯度

- 学习日期：2026-10-08。
- 每日概览：[topics_by_date](topics_by_date.md)。
- 学习索引：[工作能力提升与实践手册](../index-2026-10-08.md)。

位置表示入门与当日答疑；RoPE 的详细旋转推导留待后续专题。

## 1. 为什么需要位置表示？

单靠 token embedding，模型拿到的是内容向量，没有明确编码“哪个 token 在前，哪个在后”。

例如“猫追狗”和“狗追猫”包含相同的词，但顺序不同，意义也不同。不带位置信息和顺序相关 mask 的 self-attention 对输入排列具有置换等变性，不能仅靠这套计算表达词序关系。

常见方式包括：

- **原始 Transformer 的正弦位置编码**：将位置向量加到 token embedding 上。
- **可学习绝对位置 embedding**：学习每个位置的向量，再与 token embedding 结合。
- **RoPE**：在注意力内部对 Q、K 做与位置有关的旋转，不是简单给输入加一个位置向量。

所以 [总体结构讲义](2026-10-08_01_transformer_overview.md)写的是“位置处理”，而不是假定所有模型都在 embedding 后直接加一个位置向量。causal mask 限制信息访问方向，但它与位置表示职责不同。

### 1.1. “位置向量加到 token embedding 上”是相加还是拼接？

**是同维度向量逐元素相加，不是拼接。** 这描述的是原始 Transformer 的正弦位置编码等加法式位置表示。

设第 $i$ 个位置的 token embedding 为 $e_i$，位置向量为 $p_i$：

$$
e_i,\ p_i \in \mathbb{R}^{d_{\text{model}}}
$$

$$
x_i = e_i + p_i
\in \mathbb{R}^{d_{\text{model}}}
$$

例如：

$$
\begin{aligned}
e_i &= [0.2,\,0.5,\,-0.1,\,0.3] \\
p_i &= [0.1,\,-0.2,\,0.4,\,0.0] \\
x_i &= [0.3,\,0.3,\,0.3,\,0.3]
\end{aligned}
$$

两者都是 4 维，相加后仍是 4 维。若采用拼接，则会得到：

$$
\operatorname{Concat}(e_i,p_i)
= [0.2,\,0.5,\,-0.1,\,0.3,\,0.1,\,-0.2,\,0.4,\,0.0]
\in \mathbb{R}^{8}
$$

拼接会改变维度，需要后续层适配；它不是这里所说的原始 Transformer 做法。原始论文还在相加前对 token embedding 乘以 $\sqrt{d_{\text{model}}}$，但位置与内容的结合操作仍然是加法，不是拼接。

直觉上，token embedding 提供“是什么”，位置向量提供“在哪里”。同一个 token 出现在不同位置时，通常有相同的 token embedding，但加上的位置向量不同，因此进入 Transformer 的表示也不同。相加后内容与位置混合在同一个向量中，不能理解为前半部分专门保存内容、后半部分专门保存位置。

### 1.2. 现代主流 LLM 还这样相加吗？

**在公开的现代 Decoder-only LLM 中，RoPE 及其变体是主流方案之一，不是直接在 token embedding 上加位置向量。** 例如 Llama 与 Qwen3 都采用 RoPE。不能据此推断所有 Transformer 或未公开架构的闭源模型都采用同一种方案。

| 方案 | 位置作用在哪里？ | 结合方式 |
|---|---|---|
| 正弦 / 可学习绝对位置向量 | 输入 embedding | 与 token embedding 同维度相加 |
| RoPE | 各注意力层中的 Q、K | 按位置旋转向量，通常不对 V 做同样的旋转 |
| 相对位置偏置，如 T5 的做法 | 注意力分数 | 在分数中加入与相对距离有关的偏置 |
| ALiBi | 注意力分数 | 加入按距离变化、不同头斜率不同的线性偏置 |

先把 RoPE 的位置记清楚：

```text
隐藏状态
   ↓
线性投影得到 Q、K、V
   ↓
根据各 token 的位置旋转 Q、K；V 通常不变
   ↓
用旋转后的 Q、K 计算匹配分数
   ↓
加 mask → softmax → 汇总 V
```

RoPE 常将每个头的旋转维度划分为二维配对，每对以不同频率随位置旋转。在同一频率规则下，两个位置旋转后向量的点积会包含相对位移信息。

因此可以这样区分：

> **加法式位置编码：修改输入向量；RoPE：修改注意力中的匹配方式。两者通常都不增加隐藏状态的维度。**

RoPE 能表达相对位置关系，但不保证模型自动支持任意长上下文。长上下文通常还需要位置缩放、训练适配与实际效果评测；“可以计算更大的位置编号”不等于“在更长输入上仍然准确”。

资料：

- [原始 Transformer 论文：Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- [RoPE 论文：RoFormer](https://arxiv.org/abs/2104.09864)
- [Transformers 的 Llama 文档：说明采用 RoPE](https://huggingface.co/docs/transformers/model_doc/llama)
- [Transformers 的 Qwen3 文档：RoPE 配置](https://huggingface.co/docs/transformers/model_doc/qwen3)

### 1.3. 相加时某个维度抵消，会丢信息或影响反向传播吗？

**先区分两个问题：前向相加可能改变、抵消某个坐标；但相加这个操作本身不会因此阻断梯度。**

**问题一：如果某个维度恰好抵消怎么办？**

例如：

$$
e_i=[1,\,2],\qquad p_i=[-1,\,0.5]
$$

$$
x_i=e_i+p_i=[0,\,2.5]
$$

第一个维度确实变成了零。模型看到的是相加结果，不会自动保留这一维原来的两个数。

但这不等于“token 信息全没了”：

- 隐藏表示通常分布在多个维度上，并不是每一维都固定对应一个独立概念。
- 后续线性投影会组合多个维度，因此不能只靠某个坐标是否为零判断整个表示是否有效。
- token embedding 是可训练的，训练会根据任务损失调整它；但这不是“保证永远不会抵消”。

还要明确一个限制：**加法不是对两个任意向量的无损拼接。** 如果只给出 $x_i$，却不知道 $e_i$ 和 $p_i$，一般无法唯一恢复它们：

$$
[1,\,2]+[-1,\,0.5]
= [2,\,1]+[-2,\,1.5]
= [0,\,2.5]
$$

因此不能声称模型一定能把内容与位置“完美拆回来”。加法式位置编码的目的，是提供适合任务学习的联合表示，不是分别保存两份可逆的数据。

如果位置 $i$ 已知，且 $p_i$ 是已知的固定位置向量，那么从数学上可以用 $e_i=x_i-p_i$ 恢复 embedding。对某个固定位置，加上固定向量本身是可逆的；这不代表 Transformer 实际会执行这个减法，也不排除不同 token 与位置组合之间出现表示碰撞。

**问题二：如果前向结果为零，反向传播是不是也为零？**

不是。梯度取决于导数，不是单看前向输出数值。

设损失为 $\mathcal{L}$，相加前的向量维度为 $d$：

$$
x_i=e_i+p_i
$$

$$
\frac{\partial x_i}{\partial e_i}=I_d,
\qquad
\frac{\partial x_i}{\partial p_i}=I_d
$$

这里 $I_d$ 是单位矩阵。由链式法则：

$$
\boxed{
\frac{\partial\mathcal{L}}{\partial e_i}
=\frac{\partial\mathcal{L}}{\partial x_i},
\qquad
\frac{\partial\mathcal{L}}{\partial p_i}
=\frac{\partial\mathcal{L}}{\partial x_i}
}
$$

假设上游梯度是 $[3,\,-2]$，则加法传给两条输入支路的梯度都是 $[3,\,-2]$，不会因为 $x_i$ 的第一个坐标为零，就把对应梯度变成零；也不会把梯度“各分一半”。

可以用一个标量例子直接验证：

$$
e=1,\quad p=-1,\quad x=e+p=0,\quad
\mathcal{L}=\frac{1}{2}(x-1)^2
$$

$$
\frac{\partial\mathcal{L}}{\partial e}
=\frac{\partial\mathcal{L}}{\partial x}
=x-1=-1
$$

前向完全抵消，但 embedding 的梯度仍然非零。

需要区分：

- **固定正弦位置编码**：位置向量不是训练参数，通常不需要为它计算或保存梯度；embedding 分支仍能正常得到梯度。
- **可学习位置 embedding**：token embedding 和位置 embedding 都可以通过各自分支获得梯度，并在各自参数表中累计更新。
- **原始 Transformer 的 embedding 缩放**：若输入写成 $x_i=\sqrt{d_{\text{model}}}\,e_i+p_i$，则 embedding 分支的梯度也乘以 $\sqrt{d_{\text{model}}}$。

最后，**不阻断梯度只针对加法本身**。前向混合会改变后续网络的输入，进而可能改变损失和梯度；后续激活函数、注意力或损失也可能产生小梯度或零梯度，不能由加法导数为单位矩阵推断整个网络永远没有梯度问题。

一句话总结：

> **坐标抵消是前向表示的问题；梯度传播看操作的导数。加法可能混合或抵消信息，但它本身的导数不会因结果为零而变成零。**
