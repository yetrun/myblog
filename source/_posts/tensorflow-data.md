---
title: TensorFlow 中是如何处理数据的
date: 2026-03-07 14:20:03
tags: [TensorFlow, 深度学习, 数据处理]
---

又到了一场周六了。每次就只能在周六的时候才能静下心来写文章，梳理最近的学习和心情。周六很快就来了，又很快过去了。正如这春去秋来，花落花开。

闲言少叙，这次我想梳理的知识点是 tf.data，即 TensorFlow 中如何处理数据的。这是我前面提到的书籍《Hands-On Machine Learning with Scikit-Learn, Keras & TensorFlow》中第 13 章的内容。我还是想要依靠记忆拼凑出想要的知识点。

首先，在 PyTorch 大行其道的今日，tf.data 应该是 TensorFlow 没有被侵蚀的最主要领域了。某种程度上，它独立于 TensorFlow，并作为有效地工具完成数据预处理的各项任务。

<!-- more -->

## 一、什么时候用它

如果仅仅是少量数据，可以全部加载到内存中，你可能不会考虑用它。对于 Keras 来说，直接将所有的数据一次性喂到模型的训练流程中，剩下的框架能为你处理好一切（shuffle、batch）：

```python
x_train, y_train, x_val, y_val = ...  # 一次性加载所有数据

model.fit(
    x_train, y_train, 
    epochs=10, 
    batch_size=32, 
    validation_data=(x_val, y_val)
)
```

注意以上代码的 `batch_size`，你只需要指定，训练流程会自动按照这个 batch 加载数据到模型并更新梯度。

那么，tf.data 适宜什么时候使用呢？答案是：当数据量很大或者数据流程很复杂时。

数据量很大，指的是数据不能一次性加载到内存中。（或者即使能加载到内存中，但占据了模型训练的空间也是不足取的）通常我们的数据来源是磁盘、数据库甚至是网络。

数据流程很复杂，是指 tf.data 提供了很多数据转换的工具使用，像数据流程中的 batch、shuffle、transform、并行等。本文将作为一个通识的教程，将介绍这些主要的工具。

## 二、基本操作

### 1、batch

首先，batch 不是什么魔法，只需将 batch 理解为数据格式的转换即可，即数据从 N 维变成了 N+1 维。

首先，我们准备 200 个数的数据集：

```python
range = tf.range(200)
ds = tf.data.Dataset.from_tensor_slices(range)
```

第一次调用 batch（数据流的每条数据格式从 `shape=()` 变成了 `shape=(10,)`）：

```python
data = ds.batch(10)
list(data)

>>> [<tf.Tensor: shape=(10,), dtype=int32, numpy=array([0, 1, 2, 3, 4, 5, 6, 7, 8, 9], dtype=int32)>,
>>> <tf.Tensor: shape=(10,), dtype=int32, numpy=array([10, 11, 12, 13, 14, 15, 16, 17, 18, 19], dtype=int32)>,
>>> <tf.Tensor: shape=(10,), dtype=int32, numpy=array([20, 21, 22, 23, 24, 25, 26, 27, 28, 29], dtype=int32)>,
>>> ...
```

连续调用 batch（第二次调用时，数据流的每条数据格式从 `shape=(10,)` 变成了 `shape=(10,10)`）：

```python
data = ds.batch(10).batch(10)
list(data)

>>> [<tf.Tensor: shape=(10, 10), dtype=int32, numpy=
>>> array([[ 0,  1,  2,  3,  4,  5,  6,  7,  8,  9],
>>>        [10, 11, 12, 13, 14, 15, 16, 17, 18, 19],
>>>        [20, 21, 22, 23, 24, 25, 26, 27, 28, 29],
>>> ...
```

batch 时需要元素数据是同构的，维度需要一模一样，例如以下方式调用会报错：

```python
ds.batch(150).batch(2)
```

第一次 batch 导致生成两条数据，维度分别为 `(150,)` 和 `(50,)`，再将这两条数据混合为同一个批次即报错。

### 2、shuffle

将训练数据随机打乱是训练的要义，否则你就要受其所害。在 TensorFlow 的数据流里，打乱受到参数 `buffer_size` 的制约，因为当数据量很大时，你没法实时打乱所有数据。因此，在 TensorFlow 的数据流里，打乱有两种方式：

1. 提前打乱好所有的数据再喂给数据流；
2. 接受这种制约，只做局部的打乱。

```python
ds.shuffle(buffer_size=32)  # buffer_size 参数是必须要给的
```

我们现在想要弄懂的是，批处理和打乱的先后关系。

首先，我们准备 100 个数：

```python
range = tf.range(100)
ds = tf.data.Dataset.from_tensor_slices(range)
```

然后使用先打乱后批处理的方式：

```python
data = ds.shuffle(buffer_size=100).batch(10)
print(list(data))

>>> [<tf.Tensor: shape=(10,), dtype=int32, numpy=array([84, 89, 91, 79, 90,  3, 99, 66, 78,  7], dtype=int32)>,
>>> <tf.Tensor: shape=(10,), dtype=int32, numpy=array([64, 15, 65, 28, 97, 41,  2, 62, 10, 52], dtype=int32)>,
>>> <tf.Tensor: shape=(10,), dtype=int32, numpy=array([76, 92, 40, 43, 31, 21, 73, 88, 38, 20], dtype=int32)>,
>>> ...
```

再使用先批处理后打乱的方式：

```python
data = ds.batch(10).shuffle(buffer_size=10)
print(list(data))

>>> [<tf.Tensor: shape=(10,), dtype=int32, numpy=array([90, 91, 92, 93, 94, 95, 96, 97, 98, 99], dtype=int32)>,
>>> <tf.Tensor: shape=(10,), dtype=int32, numpy=array([0, 1, 2, 3, 4, 5, 6, 7, 8, 9], dtype=int32)>,
>>> <tf.Tensor: shape=(10,), dtype=int32, numpy=array([50, 51, 52, 53, 54, 55, 56, 57, 58, 59], dtype=int32)>,
>>> ...
```

解释：答案就是一种很自然的方式，你用什么样的顺序调用，数据流就按什么顺序处理。

### 3、map

map 是将数据集从一种格式转换为另一种格式的语法。

```python
ds = tf.data.Dataset.from_tensor_slices(tf.range(5))
data = ds.map(lambda x: x*2)
list(data)

>>> [<tf.Tensor: shape=(), dtype=int32, numpy=0>,
>>> <tf.Tensor: shape=(), dtype=int32, numpy=2>,
>>> <tf.Tensor: shape=(), dtype=int32, numpy=4>,
>>> <tf.Tensor: shape=(), dtype=int32, numpy=6>,
>>> <tf.Tensor: shape=(), dtype=int32, numpy=8>]
```

map 时支持并行，所以一定不要忘了加 `num_parallel_calls` 参数：

```python
ds.map(num_parallel_calls=8)
```

或者让 TensorFlow 帮你决定：

```python
ds.map(num_parallel_calls=tf.data.AUTOTUNE)
```

### 4、prefetch

prefetch 是预取的意思。顾名思义，prefetch 始终让数据读取流程领先数据处理流程一个单位。当使用 prefetch 时，记得将 prefetch 调用放在最后：

```python
ds.map(...).shuffle(64).batch(64).prefetch(1)
```

（Note：主要这里的 `buffer_size` 和 `batch_size` 都设置为 64 了，如何评估这时的随机程度呢？这里甚至可以发明一种数学）

### 5、cache

cache 是一种不太常用的流程，它是将前面的处理结果在内存下缓存下来，不用消耗资源重复计算了。试想一下 tf.data 处理的数据集通常内存下无法全部存下，这样缓存的结果也无法存下来。

凡事有例外，我设想了以下的场景，适用于 cache：

```python
urls = [...]
ds = tf.data.Dataset.from_tensor_slices(urls)
ds = ds.map(download_file_from_url).cache().map(read_file_from_disk)...
```

这里的场景是，先要通过网络下载文件到本地磁盘，返回本地磁盘的路径，后续再从本地磁盘读取数据。由于网络访问非常消耗资源，因此缓存下来后续直接提供。

### 6、interleave

interleave 是交错的意思，意味着交错地提取数据。交错不是为了打乱，它通常是为了并行。

设想一下我们有 50 个文件，每个文件的每行是一个文档。然后我们要每 5 个文件交错地读取。所谓交错，指的是我们同时读取第 1 到 第 5 个文件，第一次读取这 5 个文件的第一行、第二次读取这 5 个文件的第二行、……

```python
filenames = [...]
ds = tf.data.Dataset.from_tensor_slices(filenames)
ds.interleave(
    lambda f: tf.data.TextLineDataset(f),
    cycle_length=5,
    num_parallel_calls=5
)
```

第一个参数 `map_func` 接受一个 tf.data.Dataset，返回一个 tf.data.Dataset；`cycle_length` 指定交错数；`num_parallel_calls` 指定并行数。

> 个人建议：
> 1. 当使用交错时，不设置并行数是没有意义的。
> 2. 内部交错的数据集，其批次数应当相等，否则会造成资源的浪费。

### 7、unbatch

这小节将 batch 的理念统一梳理清楚。

tf.data 在处理数据时，总会将数据的第一维视为 batch，这与你是否调用了 batch 方法无关的。当调用 batch 时，实际上是将数据升维；当调用 unbatch 方法时，是将数据降维。

这次我们将 shape 为 (5,4,3) 的数据直接构建为数据集：

```python
tensor = tf.range(4*3*2)
tensor2 =tf.reshape(tensor, [4, 3, 2])
ds = tf.data.Dataset.from_tensor_slices(tensor2)
```

此时第一个维度 4 是 batch_size，内部数据的维度为 (3,2)：

```python
for elem in ds.take(1):
    print(elem)
>>> tf.Tensor(
>>> [[0 1]
>>> [2 3]
>>> [4 5]], shape=(3, 2), dtype=int32)

print(len(list(ds)))
>>> 12
```

调用 unbatch 之后，将数据维度从 (3,2) 退化为 (2,)，数据数量从 4 上升为 12：

```python
ds = ds.unbatch()

for elem in ds.take(1):
    print(elem)
>>> tf.Tensor([0 1], shape=(2,), dtype=int32)

print(len(list(ds)))
>>> 12
```

这时候可以重新 batch(4)，数据维度从 (2,) 上升为 (4,2)，数据数量从 12 合并为 3：

```python
ds = ds.batch(4)

for elem in ds.take(1):
    print(elem)
>>> tf.Tensor(
>>> [[0 1]
>>> [2 3]
>>> [4 5]
>>> [6 7]], shape=(4, 2), dtype=int32)

print(len(list(ds)))
>>> 3
```

### 8、rebatch

所谓 rebatch，是 unbatch 和 batch 的简写，下面两行代码等价：

```python
ds.unbatch().batch(4)
ds.rebatch(4)
```

### 9、repeat

数据集从头到尾遍历一次称作一轮，repeat 可以将这个轮次继续重复下去。`.repeat(3)` 一共遍历 3 轮，`.repeat()` 可以无限遍历下去。

shuffle 和 repeat 如果不同顺序结果会不同。repeat 是针对前面数据集的重复迭代，理解这一点就能理解先 shuffle 和 repeat 的顺序不同的结果了。

我们先构造一个轻量的数据集：

```python
range = tf.range(4)
ds = tf.data.Dataset.from_tensor_slices(range)
```

先 shuffle 后 repeat，不同迭代轮次的数据不会串，因此每轮数据都会是 0、1、2、3，只是每一轮的数据会有所不同。

```python
ds = ds.shuffle(buffer_size=4).repeat(3)
for i in ds:
    print(i)
>>> tf.Tensor(1, shape=(), dtype=int32)
>>> tf.Tensor(3, shape=(), dtype=int32)
>>> tf.Tensor(2, shape=(), dtype=int32)
>>> tf.Tensor(0, shape=(), dtype=int32)
>>> tf.Tensor(3, shape=(), dtype=int32)
>>> tf.Tensor(1, shape=(), dtype=int32)
>>> tf.Tensor(2, shape=(), dtype=int32)
>>> tf.Tensor(0, shape=(), dtype=int32)
>>> tf.Tensor(0, shape=(), dtype=int32)
>>> tf.Tensor(2, shape=(), dtype=int32)
>>> tf.Tensor(3, shape=(), dtype=int32)
>>> tf.Tensor(1, shape=(), dtype=int32)
```

先 repeat 后 shuffle，就是先将数据 0、1、2、3 重复 3 遍得到 12 个数，然后将这 12 数整体打乱。

```python
ds = ds.repeat(3).shuffle(buffer_size=12)
for i in ds:
    print(i)
>>> tf.Tensor(3, shape=(), dtype=int32)
>>> tf.Tensor(2, shape=(), dtype=int32)
>>> tf.Tensor(1, shape=(), dtype=int32)
>>> tf.Tensor(2, shape=(), dtype=int32)
>>> tf.Tensor(2, shape=(), dtype=int32)
>>> tf.Tensor(0, shape=(), dtype=int32)
>>> tf.Tensor(3, shape=(), dtype=int32)
>>> tf.Tensor(0, shape=(), dtype=int32)
>>> tf.Tensor(1, shape=(), dtype=int32)
>>> tf.Tensor(1, shape=(), dtype=int32)
>>> tf.Tensor(3, shape=(), dtype=int32)
>>> tf.Tensor(0, shape=(), dtype=int32)
```

先 shuffle 后 repeat，中间可以加个 cache，这样每轮迭代的随机打乱效果一致了：

```python
ds = ds.shuffle(buffer_size=4).cache().repeat(3)
for i in ds:
    print(i)
>>> tf.Tensor(1, shape=(), dtype=int32)
>>> tf.Tensor(3, shape=(), dtype=int32)
>>> tf.Tensor(2, shape=(), dtype=int32)
>>> tf.Tensor(0, shape=(), dtype=int32)
>>> tf.Tensor(1, shape=(), dtype=int32)
>>> tf.Tensor(3, shape=(), dtype=int32)
>>> tf.Tensor(2, shape=(), dtype=int32)
>>> tf.Tensor(0, shape=(), dtype=int32)
>>> tf.Tensor(1, shape=(), dtype=int32)
>>> tf.Tensor(3, shape=(), dtype=int32)
>>> tf.Tensor(2, shape=(), dtype=int32)
>>> tf.Tensor(0, shape=(), dtype=int32)
```

## 三、直接将 tensors 构建为数据集

### 1、再看 `from_tensor_slices` 方法

我们已经见过 `tf.data.Dataset.from_tensor_slices` 方法可以接受 tensors，它将 tensors 的第一维视为 batch_size，剩下的是数据维度。因此，理论上它是不能够接受 `shape=()` 的数据的：

```python
a = tf.constant(5)
tf.data.Dataset.from_tensor_slices(a)
>>> ValueError: Unbatching a tensor is only supported for rank >= 1
```

这很合理，接受。

实际上，`tf.data.Dataset.from_tensor_slices` 还可以接受元组、字典甚至是元组内嵌字典、字典内嵌元组等格式。唯一且必须的要求是，数据的第一个维度即批次应当是相等的。

```python
a = tf.reshape(tf.range(9), (3, 3))
b = tf.reshape(tf.range(6), (3, 2))
c = tf.reshape(tf.range(3), (3,))
ds = tf.data.Dataset.from_tensor_slices(({"a": a, "b": b}, c))

for elem in ds.take(1):
    print(elem)

>>> ({'a': <tf.Tensor: shape=(3,), dtype=int32, numpy=array([0, 1, 2], dtype=int32)>,
>>>   'b': <tf.Tensor: shape=(2,), dtype=int32, numpy=array([0, 1], dtype=int32)>},
>>>  <tf.Tensor: shape=(), dtype=int32, numpy=0>)
```

另外，数组和 numpy 也接受，tf.data 会将这两者首先转换为 Tensor 类型，再后续处理。因此，下面三者是等价的：

```python
tf.data.Dataset.from_tensor_slices([1,2,3,4])
tf.data.Dataset.from_tensor_slices(np.array([1,2,3,4]))
tf.data.Dataset.from_tensor_slices(tf.constant([1,2,3,4]))
```

结果都是：

```python
>>> [<tf.Tensor: shape=(), dtype=int32, numpy=1>,
>>>  <tf.Tensor: shape=(), dtype=int32, numpy=2>,
>>>  <tf.Tensor: shape=(), dtype=int32, numpy=3>,
>>>  <tf.Tensor: shape=(), dtype=int32, numpy=4>]
```

### 2、看一下 `from_tensors` 方法

为了避免混淆，介绍一下 `tf.data.Dataset` 的另一个方法 `from_tensors`。它接受与 `from_tensor_slices` 方法同样的结构，包括 tensors、元组、字典、数组和 numpy. 它们的区别是针对 batch_size 的处理，`from_tensor_slices` 将参数第一个维度视为 batch 的维度，而 `from_tensors` 方法参数整体视为单个数据，并自动赋予批次的数量为 1.

因此，下面的调用不会报错：

```
a = tf.constant(5)
ds = tf.data.Dataset.from_tensors(a)
list(ds)

>>> [<tf.Tensor: shape=(), dtype=int32, numpy=5>]
```

`from_tensors` 我几乎看不到用法。

四、总结

本文只是简单地介绍了 TensorFlow 数据集的基本操作以及构造方法。