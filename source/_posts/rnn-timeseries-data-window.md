---
title: RNN 时间序列预测里，数据窗口到底该怎么做？
date: 2026-04-10 20:30:00
tags: [深度学习, RNN, TensorFlow, Keras, 时间序列]
---

最近在看《Hands-On Machine Learning with Scikit-Learn, Keras & TensorFlow》里关于 RNN 做时间序列预测的内容。我本来以为重点会落在 `SimpleRNN` 本身，结果实际做下来才发现，真正麻烦的地方往往不是模型，而是**怎么把时间序列整理成监督学习的数据集**。

这篇文章我就专门梳理这个问题：当我们要用过去 56 天的数据预测未来的地铁客流时，数据窗口到底应该怎么切？

<!-- more -->

## 一、两个做滑动窗口的工具

在这个问题里，我主要遇到两个工具：

- `keras.utils.timeseries_dataset_from_array`
- `tf.data.Dataset.window`

它们都能做滑动窗口，但思路并不一样。

`timeseries_dataset_from_array()` 更像一个高层封装。你给它数组，它就帮你把数组切成一段段时间序列，再和目标值配起来。对于单步预测或者结构比较规则的任务，它非常顺手。

`window()` 则更底层一点。它不是直接吐出 Tensor，而是先吐出一个“数据集的集合”，也就是 nested dataset。所以后面通常还要接一个 `flat_map()`，把窗口重新拍平成普通的数据集。

```python
import tensorflow as tf

dataset = tf.data.Dataset.range(6).window(4, shift=1, drop_remainder=True)
dataset = dataset.flat_map(lambda window_ds: window_ds.batch(4))
```

为了少写一点样板代码，我后面统一用了一个小工具函数：

```python
def to_window(dataset, length):
    dataset = dataset.window(length, shift=1, drop_remainder=True)
    return dataset.flat_map(lambda window_ds: window_ds.batch(length))
```

这个函数很好用，但有一个坑要先记住：`to_window()` 处理的是数据集一条一条吐出来的样本，它并不理解 batch 的语义。也就是说，如果你的输入已经是批处理后的数据集，就不能把整个 batch 当成一个普通样本再去套 `to_window()`。

另外还有一个边界要讲清楚：`keras.utils.timeseries_dataset_from_array()` 的输入和 `targets` 都应该是数组类对象，例如 NumPy array、tensor 或者 list；它**不接受 `tf.data.Dataset` 直接作为 `targets`**。这个限制会直接影响我们后面“多步预测”的写法。

## 二、先看数据和一个朴素基线

这份数据集是芝加哥公共交通系统的日客流数据，字段并不复杂，大概就是：

- 日期
- 日期类型
- 公交乘客量
- 地铁乘客量

如果把 2019 年春季那段数据画出来，会比较明显地看到 7 天左右的周期性。也正因为这样，一个非常自然的基线模型就是：**直接拿 7 天前的值作为今天的预测**。

这个基线虽然简单，但很有意义。因为它提醒我一件事：时间序列建模别一上来就盯着网络深度，先看看序列本身有没有周期性，很多时候已经能解释一大半现象。

## 三、四种任务，数据集到底怎么做

这篇文章里，重点不是比较哪种模型最终效果最好，而是梳理四种常见任务的数据形状。

这里先提前说一下：下面反复出现的 `(batch, 56, 5)` 里面，最后那个 `5` 指的是**每个时间步有 5 个特征**。这 5 个特征分别是：

- 公交乘客量
- 地铁乘客量
- 下一天日期类型的 one-hot 编码，共 3 维

所以 `5 = 2 + 3`。先把这件事说清楚，后面看到这些 shape 就不会觉得那个 `5` 来得很突然了。

四种任务分别是：

- 单变量单步预测：`(batch, 56) -> (batch,)`
- 多变量单步预测：`(batch, 56, 5) -> (batch,)`
- 多变量多步预测：`(batch, 56, 5) -> (batch, 14)`
- 序列到序列预测：`(batch, 56, 5) -> (batch, 56, 14)`

下面我只写训练集的构造。验证集和测试集按同样的窗口规则处理即可，没必要重复贴一遍。

### 1、单变量单步预测

这是最直接的一种情形：输入是前 56 天的地铁乘客量，输出是第 57 天的地铁乘客量。

```python
sequence_length = 56

rails_train_ds = keras.utils.timeseries_dataset_from_array(
    rail_train.to_numpy(),
    targets=rail_train[sequence_length:],
    sequence_length=sequence_length,
    batch_size=32,
    shuffle=True,
    seed=42,
)
```

训练数据集直接打印出来时，输入输出形状是：

```python
(batch, 56) -> (batch,)
```

这里直接看到的输入就是一个长度为 56 的序列，目标则是后面的那个标量。这种写法理解起来也很直接：拿长度为 56 的窗口做输入，再拿紧接着的下一个标量做目标。

> 后面在构造 RNN 模型时，输入格式定义为 (batch, 56, 1)，这不是一种失误，RNN 模型可以很好地将 (batch, 56) 的格式 reshape 为 (batch, 56, 1)，这个过程是自动的。但是为了清晰可见，最好在明确定义输入格式的时候，让数据集吐出来的格式与之匹配，以避免潜在的问题和理解上的歧义。
> 
> ```python
> (batch, 56, 1) -> (batch, 1)
> ```

### 2、多变量单步预测

这次输入序列里，每个时间步不再只有一个数，而是一个五维向量，也就是前面说的那 5 个特征：

所以输入输出形状会变成：

```python
(batch, 56, 5) -> (batch,)
```

这里先把包含这 5 个特征的训练数据记为 `mulvar_train`，然后直接交给 `timeseries_dataset_from_array()`：

```python
mulvar_train_ds = keras.utils.timeseries_dataset_from_array(
    mulvar_train.to_numpy(dtype="float32"),
    targets=mulvar_train["rail"][sequence_length:],
    sequence_length=sequence_length,
    batch_size=32,
    shuffle=True,
    seed=42,
)
```

这种方法也不难理解：输入窗口里保留更多上下文特征，但目标仍然只是窗口之后的那个单点值。

### 3、多变量多步预测

这里开始变得有意思了。输入仍然是前 56 天、每个时间步 5 个特征，但目标不再是单个标量，而是**未来 14 天的地铁乘客量**：

```python
(batch, 56, 5) -> (batch, 14)
```

这部分其实有两种构造思路，我觉得都值得写出来。

#### 方法一：先做目标窗口数组

第一种思路是先把目标单独整理好。因为 `timeseries_dataset_from_array()` 的 `targets` 参数只接受数组类对象，所以这里不能直接把 `Dataset` 扔进去，而是要先把目标窗口做成数组。

```python
mulvar_train_rail = mulvar_train["rail"][sequence_length:]

mulvar_rail_target = tf.data.Dataset.from_tensor_slices(
    mulvar_train_rail.to_numpy()
)
mulvar_rail_target = to_window(mulvar_rail_target, 14)
mulvar_rail_target = list(mulvar_rail_target.as_numpy_iterator())

train_ahead_ds = keras.utils.timeseries_dataset_from_array(
    mulvar_train.to_numpy(dtype="float32"),
    targets=mulvar_rail_target,
    sequence_length=sequence_length,
    batch_size=32,
    shuffle=True,
    seed=42,
)
```

这种方法可以这样理解：输入窗口照旧切，目标则提前单独做成“未来 14 天”的窗口数组，再把这组数组塞给 `targets`。

它的好处是直观，坏处是你会明显感觉到，输入窗口和目标窗口是分两步手工对齐出来的。

#### 方法二：先做总窗口再拆分

第二种思路是把输入和目标先拼成一个更长的大窗口，然后再拆开。我后来会更偏向这种写法，因为输入输出的对应关系更清楚。

这里的大窗口长度是 `56 + 14`。

```python
def split_to_input_target(batch_windows):
    inputs = batch_windows[:, :-14]
    targets = batch_windows[:, -14:, 1]  # 第 1 列是 rail
    return inputs, targets

train_ahead_ds = keras.utils.timeseries_dataset_from_array(
    mulvar_train.to_numpy(dtype="float32"),
    targets=None,
    sequence_length=sequence_length + 14,
    batch_size=32,
    shuffle=True,
    seed=42,
).map(split_to_input_target)
```

这种方法可以理解为：先切出一个完整样本，再在 batch 里把“前 56 步输入”和“后 14 步目标”拆开。

如果只是做普通的多步预测，我会更偏向这一种。因为当输入和目标都来自同一个总窗口时，对齐关系不太容易出错。

### 4、序列到序列预测

这是最绕的一部分。这里的目标不再是“对整个输入序列只预测一次未来 14 天”，而是**对输入序列中的每一个时间步，都预测从该时间步往后的 14 天**。

所以目标形状不再是 `(batch, 14)`，而是：

```python
(batch, 56, 14)
```

这意味着输入序列的每个位置，都要对应一个 14 步的未来窗口。

#### 方法一：先做总窗口，再在张量里 frame

第一种办法是：先像前面一样做一个 `56 + 14` 的总窗口，然后再拆成输入和目标两部分。输入部分就是窗口里的前 56 条数据；目标数据从第 2 条开始取直到最后，然后使用 `tf.signal.frame()` 将其改造为窗口格式的数据。

> 这里要专门说一下 `tf.signal.frame()`。 它是在底层操作，直接在 Tensor 上做滑动窗口。输入是 `Tensor`，输出也还是 `Tensor`。
>
> 也许你认为能直接借助 to_window 将目标数据做成滑动格式的数据。不！你不能！千万别这么做。

```python
def split_to_input_target_seq2seq(batch_windows):
    inputs = batch_windows[:, :56]
    rail_targets = batch_windows[:, 1:, 1]  # 从第 2 条开始取 rail 列

    target_sequences = tf.map_fn(
        lambda x: tf.signal.frame(x, frame_length=14, frame_step=1),
        rail_targets
    )
    return inputs, target_sequences

seq2seq_train_ds = keras.utils.timeseries_dataset_from_array(
    mulvar_train.to_numpy(dtype="float32"),
    targets=None,
    sequence_length=56 + 14,
    batch_size=32,
    shuffle=True,
    seed=42,
).map(split_to_input_target_seq2seq)
```

这种方法可以理解为：先把一整段样本切出来，其中前 56 条直接作为输入；然后只对目标列继续滑一次窗口，把目标加工成 `(56, 14)`。

它的优点是逻辑集中在一个拆分函数里，缺点是你得先在脑子里想清楚 `frame()` 之后维度会怎么变化。

#### 方法二：两次 `to_window()`

第二种办法更像是在数据流里一层层加工。第一次 `to_window()` 先把每一个时间步扩成“当前输入 + 未来 14 步目标”；第二次 `to_window()` 再把 56 个时间步拼成一个完整样本。

```python
def split_to_input_target_seq2seq(sample_windows):
    inputs = sample_windows[:, 0]
    targets = sample_windows[:, 1:, 1]  # 第 1 列是 rail
    return inputs, targets

mulvar_train_ds = tf.data.Dataset.from_tensor_slices(
    mulvar_train.to_numpy(dtype="float32")
)

seq2seq_train_ds = to_window(to_window(mulvar_train_ds, length=15), length=56)
seq2seq_train_ds = seq2seq_train_ds.map(split_to_input_target_seq2seq).batch(32)
```

这种方法可以理解为：先构造“单步监督单元”，再把 56 个单步监督单元组装成一个样本。

这套写法我觉得很漂亮，因为完全是在数据集这一层连续变换；但它也更绕，尤其是一旦没有完全搞清楚 `to_window()` 是按“逐元素样本”而不是按 batch 工作的，就很容易把自己绕进去。

如果做一个很粗的比较：

- `tf.signal.frame()` 的版本更像在张量上做后处理
- 双 `to_window()` 的版本更像在数据流层面做连续加工

## 四、模型结构其实改得不大

把数据集这一层想清楚之后，模型本身反而没那么复杂。

### 1、单变量单步

输入输出形状是 `(batch, 56, 1) -> (batch, 1)`：

```python
model = keras.Sequential([
    keras.Input(shape=(None, 1)),
    keras.layers.SimpleRNN(32),
    keras.layers.Dense(1)
])
```

### 2、多变量单步

输入输出形状是 `(batch, 56, 5) -> (batch, 1)`：

```python
model = keras.Sequential([
    keras.Input(shape=(None, 5)),
    keras.layers.SimpleRNN(32),
    keras.layers.Dense(1)
])
```

### 3、多变量多步

输入输出形状是 `(batch, 56, 5) -> (batch, 14)`：

```python
model = keras.Sequential([
    keras.Input(shape=(None, 5)),
    keras.layers.SimpleRNN(32),
    keras.layers.Dense(14)
])
```

这里其实只是把最后一层 `Dense` 的输出维度从 1 改成了 14。也就是说，模型最后一次输出，不再是一个标量，而是一整个未来两周的向量。

### 4、序列到序列

输入输出形状是 `(batch, 56, 5) -> (batch, 56, 14)`：

```python
model = keras.Sequential([
    keras.Input(shape=(56, 5)),
    keras.layers.SimpleRNN(32, return_sequences=True),
    keras.layers.Dense(14)
])
```

这里真正关键的是 `return_sequences=True`。因为我们不再只需要最后一个时间步的输出，而是需要 56 个时间步全部保留下来，再让后面的 `Dense(14)` 对每个时间步各自产生一个 14 维预测向量。

所以这四种模型看下来，变化其实没有想象中那么大。真正复杂的部分，并不是 `SimpleRNN` 要怎么堆，而是**目标张量到底长什么样，以及数据窗口要怎么和它对齐**。

## 五、总结

回头看这个问题，我觉得最值得记住的不是“哪种模型更强”，而是下面这句话：

> 先想清楚目标张量长什么样，再决定窗口应该怎么切。

如果目标是标量，那通常直接用 `timeseries_dataset_from_array()` 就够了；如果目标开始变成 14 步向量，或者甚至变成 `(56, 14)` 这样的序列集合，那么你就要认真想一想，到底是：

- 先把目标单独做成窗口数组；
- 还是先把输入和目标拼成一个总窗口，再统一拆分；
- 还是干脆在 `tf.data` 这一层连续做两次窗口变换。

从这个角度看，`SimpleRNN` 在这里更像一个教学模型。它真正帮我理解的，不只是循环网络本身，而是时间序列问题到底怎样才能被组织成标准的监督学习数据。
