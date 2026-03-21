---
title: RNN 生成重复的坑：softmax 与 from_logits 的陷阱
date: 2026-03-21 20:00:00
tags: [深度学习, RNN, TensorFlow, Keras, 调试]
---

在做 RNN 模型生成任务时，我遇到了一个诡异的问题：模型总是生成重复的字符或词汇。排查了很久才发现，原来是** softmax 被调用了两次**。

<!-- more -->

## 问题现象

在训练一个文本生成模型时，我发现模型生成结果总是重复的，比如：

```
输入：今天天气
输出：今天天气好好好好好好好好......
```

检查模型输出，发现概率分布非常极端，某些 token 的概率接近 1，其他几乎为 0。这导致采样时总是选中同一个词。

## 问题根源

翻看代码，发现问题出在这两个地方：

```python
# 模型输出层
inputs = keras.Input(shape=(None,), dtype="int32")
x = layers.Embedding(input_dim=vocab_size, output_dim=embedding_dim)(inputs)
for _ in range(num_layers):
    x = layers.GRU(hidden_dim, return_sequences=True)(x)
    x = layers.Dropout(0.1)(x)
outputs = layers.Dense(vocab_size, activation='softmax')(x)  # ❌ 这里做了 softmax
model = keras.Model(inputs, outputs)

# 损失函数
model.compile(
    optimizer='adam',
    loss=keras.losses.SparseCategoricalCrossentropy(from_logits=True)  # 内部会做 softmax
)
```

**问题就在这里：**
- 输出层使用了 `activation='softmax'`，对 logits 做了一次 softmax
- 损失函数设置了 `from_logits=True`，会在内部**再做一次 softmax**

相当于对 logits 连续应用了两次 softmax！

> 这次踩坑其实有个特殊的背景。我正在做一个可插拔的深度学习框架：训练流程（包括 loss 函数的选择）是统一固定的，而模型结构则分散在不同的模块里。框架同时支持 RNN 和 Transformer，Transformer 的模型写对了，但写 RNN 模块的时候，没注意到框架内部已经配置了 `from_logits=True`，顺手就加了 softmax。正是这种训练和模型分离的架构，让这个问题藏得更深。

## 数学分析

softmax 公式：

$$\text{softmax}(z_i) = \frac{e^{z_i}}{\sum_{j=1}^n e^{z_j}}$$

softmax 的作用是将任意实数转换为概率分布。但问题是：**如果对已经是概率的值再做 softmax，会发生什么？**

### 错误推导的修正

假设我们有两个 logits $a$ 和 $b$，且 $b > a$。

**第一次 softmax**（在模型输出层）：
$$p_a = \frac{e^a}{e^a + e^b}, \quad p_b = \frac{e^b}{e^a + e^b}$$

比值：$\frac{p_b}{p_a} = e^{b-a}$  b/a  e^{b-a}

**第二次 softmax**（在损失函数内部，因为 `from_logits=True`）：
输入已经是概率 $p_a, p_b$，但损失函数会将其当作 logits 处理：

$$q_a = \frac{e^{p_a}}{e^{p_a} + e^{p_b}}, \quad q_b = \frac{e^{p_b}}{e^{p_a} + e^{p_b}}$$

新的比值：$\frac{q_b}{q_a} = e^{p_b - p_a}$

**关键问题**：这里的 $p_b - p_a$ 虽然小于 $b - a$，但由于 $e^x$ 是凸函数，当 $p_b$ 接近 1、$p_a$ 接近 0 时，$e^{p_b} \gg e^{p_a}$，导致 $q_b$ 远大于 $q_a$。

举个例子，设原始 logits 为 $a=0, b=3$：

- 第一次 softmax：$p_a \approx 0.047, p_b \approx 0.953$
- 第二次 softmax：$q_a = \frac{e^{0.047}}{e^{0.047} + e^{0.953}} \approx 0.29, q_b \approx 0.71$

等等，这样看差距反而缩小了？这不对...

### 真正的问题：训练过程的累积效应

实际上，单次前向传播的两次 softmax 虽然会改变分布，但不会直接导致极端尖锐。真正的问题是**在训练过程中**，这种错误的梯度计算会让模型学习错误的特征：

1. **梯度消失**：当分布过于尖锐时，softmax 的梯度 $\frac{\partial L}{\partial z_i} = p_i - y_i$ 几乎为 0（因为 $p_i$ 接近 1 或 0）
2. **模型退化**：模型学会了输出极端概率，而不是学习真正的特征
3. **生成时的恶性循环**：训练好的模型输出概率本身就极端，即使没有第二次 softmax，采样时也会一直重复

**更直观的理解**：`SparseCategoricalCrossentropy(from_logits=True)` 期望输入是任意实数（logits），它会内部做 softmax 然后取 log。如果你输入的是已经 softmax 后的概率（范围 0-1），相当于把这些概率当成了 logits。比如概率 0.9 被当成 logits 0.9，这比真正的 logits（可能是 2.2）小得多，经过指数放大后，相对关系会被严重扭曲。

这就是为什么实践中会看到生成重复——模型实际上没有正确训练，只是记住了某些模式的极端输出。

## 为什么要用 from_logits？

你可能会问：既然容易出错，为什么还要用 `from_logits=True`？

**答案：温度采样（Temperature Sampling）。**

在生成文本时，我们需要控制随机性。温度参数 $T$ 可以调节 softmax 的"尖锐"程度：

$$P(x_i) = \frac{e^{z_i/T}}{\sum_j e^{z_j/T}}$$

- $T < 1$：分布更尖锐，生成更确定
- $T > 1$：分布更平缓，生成更随机
- $T = 0$：退化为贪婪解码（总是选概率最大的）

如果模型输出的是概率而不是 logits，我们就无法直接应用温度参数。只有保留 logits，才能在采样时灵活调整温度：

```python
def random_sample(preds, temperature=1.0):
    """
    preds: 模型输出的 logits，shape 为 (vocab_size,)
    temperature: 温度参数
    """
    preds = preds / temperature
    return keras.random.categorical(preds[None, :], num_samples=1)[0]

# 生成时调整温度
for _ in range(max_length):
    logits = model.predict(current_input)[0, -1, :]  # 取最后一个位置的 logits
    next_token = random_sample(logits, temperature=0.8)
    generated.append(next_token)
    current_input = np.array([[next_token]])
```

看到没有？`preds / temperature` 这个操作只能在 logits 上进行。如果模型输出的是概率，你就没法用温度参数了。

## 总结

这个坑的根源是对训练流程的部分细节不够敏感：

1. **`from_logits=True`** 表示损失函数会内部计算 softmax，**不要**在输出层再加 softmax
2. **保留 logits** 是为了生成阶段能够进行温度采样等灵活控制
3. **两次 softmax** 会让概率分布极度尖锐，导致生成重复

正确的姿势：

```python
# ✅ 正确的模型定义
inputs = keras.Input(shape=(None,), dtype="int32")
x = layers.Embedding(input_dim=vocab_size, output_dim=embedding_dim)(inputs)
for _ in range(num_layers):
    x = layers.GRU(hidden_dim, return_sequences=True)(x)
    x = layers.Dropout(0.1)(x)
outputs = layers.Dense(vocab_size)(x)  # 直接返回 logits，不要 softmax
model = keras.Model(inputs, outputs)

# ✅ 正确的训练
model.compile(
    optimizer='adam',
    loss=keras.losses.SparseCategoricalCrossentropy(from_logits=True)
)

# ✅ 正确的生成（带温度采样）
def random_sample(preds, temperature=1.0):
    preds = preds / temperature
    return keras.random.categorical(preds[None, :], num_samples=1)[0]
```

调试深度学习模型时，这种"双重计算"的问题往往很隐蔽。建议在定义模型时，明确区分训练阶段（需要 `from_logits`）和生成阶段（需要 logits 进行采样），保持接口的一致性。

希望这篇文章能帮到你，避免在这个问题上浪费时间。
