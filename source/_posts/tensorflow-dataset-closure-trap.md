---
title: TensorFlow Dataset from_generator 的闭包陷阱
date: 2026-03-14 16:30:00
tags: [TensorFlow, 深度学习, 数据处理, Python, 闭包]
---

今天在处理一个文本解析任务时，遇到了一个让人头疼的问题。本来是一个简单的需求：读取目录下的多个文本文件，每个文件里包含若干个用 `<doc>` 标签包裹的文档，想要把它们解析成一个 `tf.data.Dataset`，每个元素是一个文档字符串。
    
看起来很简单对吧？但就是这个"简单"的需求，让我陷入了 TensorFlow 图执行模式的深坑，也让我重新理解了 Python 闭包和 TensorFlow 的 SymbolicTensor。

<!-- more -->

## 一、问题场景

先看一下原始代码：

```python
import tensorflow as tf
from pathlib import Path


def parse_doc(file_path_tensor: tf.Tensor):
    """解析单个文件并产出文档"""
    current_doc = []
    in_doc = False
    
    # 尝试将 tensor 转为字符串路径
    file_path = file_path_tensor.numpy().decode('utf-8')
    
    with open(file_path, 'r', encoding='utf-8') as f:
        for line in f:
            line = line.strip()
            
            if line == "<doc>":
                in_doc = True
                current_doc = []
            elif line == "</doc>":
                in_doc = False
                yield " ".join(current_doc)
            elif in_doc:
                current_doc.append(line)


def build_doc_ds(x: tf.Tensor):
    """为单个文件路径构建 dataset"""
    return tf.data.Dataset.from_generator(
        lambda: parse_doc(x),  # 注意这里的闭包
        output_signature=tf.TensorSpec(shape=(), dtype=tf.string)
    )


def create_dataset(data_dir="test_data"):
    files = [str(p) for p in Path(data_dir).glob("*.txt")]
    file_ds = tf.data.Dataset.from_tensor_slices(files)
    
    # 使用 flat_map 展平所有文件的文档
    return file_ds.flat_map(build_doc_ds)
```

运行时报错：

```
AttributeError: 'SymbolicTensor' object has no attribute 'numpy'
```

错误发生在 `file_path_tensor.numpy().decode('utf-8')` 这一行。但奇怪的是，我是在 `flat_map` 里调用的，按理说这时候应该已经有实际的文件路径值了，为什么会有 SymbolicTensor？

## 二、核心问题：Python 闭包的延迟绑定

问题的根源在于 **Python 闭包的延迟绑定（Late Binding）** 机制。

### 1. 什么是延迟绑定？

Python 的闭包捕获的是**变量名**，而不是**变量的值**。看这个例子：

```python
def outer():
    x = "original"
    inner = lambda: print(x)  # 捕获的是变量 x，不是值 "original"
    x = "changed"             # 修改 x
    return inner

f = outer()
f()  # 输出 "changed"，不是 "original"
```

内层函数在**定义时**并没有捕获 `x` 的值，而是在**调用时**才去查找变量 `x`。这就是延迟绑定。

### 2. 在我们的代码中发生了什么？

回到问题代码：

```python
def build_doc_ds(x: tf.Tensor):
    return tf.data.Dataset.from_generator(
        lambda: parse_doc(x),  # 这里的 x 是闭包变量
        output_signature=...
    )

dataset = file_ds.flat_map(build_doc_ds)
```

**时间线分析：**

```
建图阶段 (Graph Building)
─────────────────────────────
flat_map 调用 build_doc_ds
    ↓
传入的 x 是 SymbolicTensor（图节点）
    ↓
lambda 定义: lambda: parse_doc(x)
    ↓
闭包捕获了变量 x（不是值！）
    ↓
返回 Dataset 定义（此时还未执行）


运行阶段 (Runtime)
─────────────────────────────
迭代 Dataset
    ↓
调用 lambda()
    ↓
解析闭包，查找变量 x
    ↓
x 还是那个 SymbolicTensor！
    ↓
parse_doc(x) 传入 SymbolicTensor
    ↓
报错: 'SymbolicTensor' object has no attribute 'numpy'
```

关键点：**`lambda` 定义时 `x` 是 SymbolicTensor，运行时它依然指向那个 SymbolicTensor**，因为闭包捕获的是变量引用。

## 三、为什么 Dataset.map() 没问题？

作为对比，看看 `map` 操作：

```python
def multiply(x):
    return x * 2

dataset = tf.data.Dataset.from_tensor_slices([1, 2, 3])
dataset = dataset.map(multiply)  # 正常工作
```

为什么这里的 `multiply` 可以正常工作？

**区别：**

| 特性 | `map` | `from_generator` + `flat_map` |
|------|-------|------------------------------|
| 执行方式 | TensorFlow trace 整个函数 | Python 生成器在运行时执行 |
| 参数处理 | 自动将 Python 值转为图常量 | 依赖闭包捕获的变量 |
| 闭包问题 | 无（函数内部不依赖外部状态） | 有（闭包捕获 SymbolicTensor） |

`map` 会 trace 函数体，把 Python 值转换为图中的常量节点。而 `from_generator` 是在 Python 运行时执行生成器，此时闭包变量早已固定为建图时的 SymbolicTensor。

## 四、解决方案

**以下方案是我在网上搜索的结果，没有真实实践，有些方案还并不好用。后期如果有时间，我打算专门出一期解决方案的文章。**

### 方案一：使用默认参数实现早绑定

利用 Python 默认参数在**定义时**就求值的特性：

```python
def build_doc_ds(x: tf.Tensor):
    return tf.data.Dataset.from_generator(
        lambda path=x.numpy().decode('utf-8'): parse_doc(path),
        output_signature=tf.TensorSpec(shape=(), dtype=tf.string)
    )
```

这里的 `path=x.numpy().decode('utf-8')` 在 lambda **定义时**就执行了，此时 `x` 虽然是 SymbolicTensor，但我们可以用 `.numpy()` 获取它的值（因为 `from_tensor_slices` 传入的是 Python 字符串列表，这时候 x 实际上是 EagerTensor）。

**等等，还是有问题！**

如果在 `flat_map` 中使用，TensorFlow 在 trace 时会把函数转成图模式，此时 `x` 仍然是 SymbolicTensor，无法调用 `.numpy()`。

### 方案二：避免嵌套结构

最稳妥的方案是在 Python 层面处理，完全避开闭包问题：

```python
def create_dataset(data_dir="test_data"):
    files = [str(p) for p in Path(data_dir).glob("*.txt")]
    
    if not files:
        raise ValueError(f"No .txt files found in {data_dir}")
    
    # 为每个文件单独创建 dataset
    datasets = [
        tf.data.Dataset.from_generator(
            lambda f=f: parse_doc(f),  # 立即绑定文件路径
            output_signature=tf.TensorSpec(shape=(), dtype=tf.string)
        )
        for f in files
    ]
    
    # 合并所有 datasets
    return datasets[0].concatenate(*datasets[1:]) if len(datasets) > 1 else datasets[0]
```

关键点：`lambda f=f: parse_doc(f)`，这里的 `f=f` 使用默认参数立即绑定当前循环的值。

### 方案三：使用 tf.py_function

如果必须在 `flat_map` 中使用，可以用 `tf.py_function` 包装：

```python
def build_doc_ds(x):
    # 使用 py_function 在 eager 模式下执行
    def parse_file(path_tensor):
        path = path_tensor.numpy().decode('utf-8')
        docs = list(parse_doc(path))
        return tf.constant(docs, dtype=tf.string)
    
    docs = tf.py_function(
        parse_file,
        [x],
        tf.string
    )
    return tf.data.Dataset.from_tensor_slices(docs)
```

## 五、深入理解：为什么执行阶段不重新调用 build_doc_ds？

有读者可能会问：执行阶段不应该重新调用 `build_doc_ds` 吗？为什么传入的还是 SymbolicTensor？

这是一个触及 TensorFlow Dataset 执行机制核心的好问题。

### 关键概念：flat_map 的执行过程

**常见误区**：以为执行阶段会**重新调用** `build_doc_ds`

**实际情况**：`flat_map` 在建图阶段**只 trace 一次**，执行阶段运行的是**编译后的 graph**，不再调用 Python 函数。

### 详细流程

#### 1. 建图阶段（调用 flat_map 时）

```python
dataset = file_ds.flat_map(build_doc_ds)
#           ↑ 此时发生！
```

TensorFlow 内部：
1. 需要确定 `build_doc_ds` 的输出类型
2. **Trace `build_doc_ds`**：传入一个 SymbolicTensor `x`
3. `build_doc_ds(x)` 执行，返回 `Dataset`
4. TensorFlow 记录：`build_doc_ds` 返回 `Dataset<string>`
5. 编译成 graph，**不再保留 Python 函数**

#### 2. 执行阶段（迭代时）

```python
for item in dataset:
    print(item)
```

TensorFlow 内部：
1. 执行已编译的 graph
2. 对于每个输入文件路径，填充到 graph 的 placeholder
3. 运行 graph 中的操作（`from_generator` 节点）
4. **不再调用 `build_doc_ds`**！

### 对比：map vs flat_map

| 操作 | 建图阶段 | 执行阶段 | 是否重新调用 Python 函数 |
|------|---------|---------|----------------------|
| `map(func)` | Trace `func` 推断类型 | 对每个元素调用 `func` | ✅ 是 |
| `flat_map(func)` | Trace `func` 推断类型 | 运行编译后的 graph | ❌ 否 |

### 为什么 from_generator 的生成器能执行？

因为 `from_generator` 是特殊的：它在 graph 中注册了一个 PyFunc 节点，执行阶段会调用 Python 生成器。但这时候闭包变量 `x` 早就被固定为 SymbolicTensor 了。

### 执行流程图示

```
建图阶段
─────────────────────────────────────────
flat_map(build_doc_ds)
    ↓
调用 build_doc_ds(SymbolicTensor)  ← 只执行一次！
    ↓
返回 Dataset 定义
    ↓
编译成 graph（build_doc_ds 被"遗忘"）

执行阶段
─────────────────────────────────────────
迭代 dataset
    ↓
执行编译好的 graph
    ↓
不再调用 build_doc_ds！
    ↓
直接运行 from_generator 节点
    ↓
调用生成器（闭包里的 x 还是 SymbolicTensor）
```

这就是为什么即使到了执行阶段，你拿到的还是 SymbolicTensor，而不是实际的文件路径。

## 六、对比实验：为什么 TextLineDataset 没有闭包问题？

为了进一步验证，我们做一个简单的对比实验：将 `from_generator` 替换为 `TextLineDataset`. 在这个实验中，我们不追求解析 `<doc>` 文档，而是简单的将多个文件的每行组合成单个数据集。

```python
import tensorflow as tf
from pathlib import Path


def build_doc_ds(x: tf.Tensor):
    """最简单的：用 TextLineDataset 读取文件"""
    print(f"build_doc_ds called with: {x}")
    return tf.data.TextLineDataset(x)


def create_dataset(data_dir="test_data"):
    files = [str(p) for p in Path(data_dir).glob("*.txt")]
    file_ds = tf.data.Dataset.from_tensor_slices(files)
    
    print("Before flat_map")
    dataset = file_ds.flat_map(build_doc_ds)
    print("After flat_map")
    
    return dataset


if __name__ == "__main__":
    dataset = create_dataset()
    print(f"\nDataset created, element_spec: {dataset.element_spec}")
    
    print("\n--- Start iterating ---")
    for i, line in enumerate(dataset):
        if i < 10:
            print(f"Line {i}: {line.numpy().decode('utf-8')}")
```

### 运行结果

```
Before flat_map
build_doc_ds called with: Tensor("args_0:0", shape=(), dtype=string)
After flat_map

Dataset created, element_spec: TensorSpec(shape=(), dtype=tf.string, name=None)

--- Start iterating ---
Line 0: <doc>
Line 1: 文件2的第一个文档。
Line 2: 记录一些测试数据。
...
```

### 关键观察

**`build_doc_ds` 只在建图阶段调用一次**，传入的是 SymbolicTensor。但在执行阶段，代码**完美运行**，没有报错！

### 为什么 TextLineDataset 可以工作？

核心区别在于：**闭包问题只发生在 Python 生成器中**。

| 特性 | `from_generator` | `TextLineDataset` |
|------|-----------------|-------------------|
| 实现方式 | Python 生成器 + PyFunc | 纯 TensorFlow Op |
| 执行时机 | 运行时才执行生成器代码 | 完全在 Graph 中执行 |
| 是否依赖 Python 运行时 | ✅ 是 | ❌ 否 |
| 闭包问题 | ✅ 有 | ❌ 无 |

`TextLineDataset` 是纯 TensorFlow 操作（C++ 实现），它完全接受 SymbolicTensor 作为输入，整个过程都在 graph 中执行，不需要在运行时回调 Python 代码。因此不存在闭包捕获问题。

### 本质区别

```python
# ❌ 有问题：Python 生成器在运行时执行，闭包已固定
return tf.data.Dataset.from_generator(
    lambda: parse_doc(x),  # x 是闭包变量，运行时查找
    ...
)

# ✅ 没问题：纯 TF 操作，在 graph 中执行
return tf.data.TextLineDataset(x)  # x 直接传给 TF Op
```

这说明：**`flat_map` 本身没有问题，问题出在 `from_generator` 的 Python 生成器机制上**。

## 七、总结

这个问题的本质是对 TensorFlow **图执行模式**和 Python **闭包机制**的理解不够深入。

**关键要点：**

1. **Python 闭包是延迟绑定的**：内层函数捕获的是变量名，不是值
2. **TensorFlow 有两种执行模式**：
   - Eager 模式：立即执行，可以调用 `.numpy()`
   - Graph 模式：构建计算图，参数是 SymbolicTensor
3. **`flat_map`、`map` 等操作在建图时 trace 函数**：此时传入的是 SymbolicTensor
4. **`from_generator` 的生成器在运行时执行**：但闭包变量在建图时就已绑定

**最佳实践：**

- 避免在 `flat_map`、`filter` 等操作中使用复杂的闭包
- 如需使用，考虑用默认参数实现**早绑定**
- 对于文件读取等 IO 操作，优先考虑在 Python 层面预处理，或用 `tf.data.TextLineDataset` 等内置方法

调试这个问题的过程中，我对 TensorFlow 的数据流有了更深的理解。希望这篇文章能帮助到遇到同样问题的你。
