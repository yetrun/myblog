---
title: Keras 下的自定义组件
date: 2026-03-01 17:43:00
tags: [Keras, TensorFlow, 深度学习, 自定义层]
---

## 导言

首先分享一下最近看的两本书：

- 《Deep Learning with Python》
- 《Hands-On Machine Learning with Scikit-Learn, Keras & TensorFlow》

两本书都到第三版了，阅读时请认准英文原版。

其中第一本书 DLWP 我已经基本读完，第二本 HOML 正在阅读。你可以理解为前者主要在讲 Keras，而后者可以深入到 TensorFlow。这样的阅读顺序我觉得挺好。

言归正传，这篇文章我想谈谈 Keras 的**自定义组件**，其主要思想来源于 HOML 的第 12 章。具体来讲，涵盖：

- 自定义 Initializer
- 自定义 Regularizer
- 自定义 Constraint
- 自定义 Activation Function
- 自定义 Loss
- 自定义 Metrics
- 自定义 Layer
- 自定义 Model

在这里面，有一些可以归纳出的点：
1. 大多数情况下，可以只传递一个函数进去，Keras 会自动帮你处理这一切；
2. 或者可以继承对应的类，每种组件都有其对应的类，一般要实现其 `__call__` 或者 `call` 方法；
3. 在模型保存和加载的阶段，我们还要为每个组件实现 `get_config` 方法；
4. Metrics、Layer、Model 这三者除了 `call` 方法之外，还有其他的方法需要实现。

总之，这里面有许多东西需要处理和记忆，初看之下还会有些混乱。这篇文章就是在梳理这些内容。

*由于篇幅有限，我只捡重要的讲。除了宣导之外，还能帮助我记忆。*

<!-- more -->

## 正文

### 1. 支持函数作为自定义组件

在大多数情况下，我们可以直接使用函数作为自定义组件。例如，常见的 MSE Loss 我们可以直接实现为一个函数：

```python
def mse_loss(y_true, y_pred):
    return tf.reduce_mean((y_pred - y_true) ** 2)
```

*最近被 TensorFlow 种草了，所以这里直接使用 TensorFlow 语法，没有使用 `keras.ops`.*

支持函数作为自定义组件的有：

- 自定义 Initializer
- 自定义 Regularizer
- 自定义 Constraint
- 自定义 Activation Function
- 自定义 Loss
- 自定义 Metrics

除了 Layer、Model，其余的组件都支持传递函数作为自定义组件，Keras 会自动帮你处理好剩下的工作。

### 2. 继承特定的基类实现自定义组件

几乎所有的组件都支持继承某个基类实现自定义组件（而且这极有可能就是内部统一实现的方式）。

> 目的：当使用类作为自定义组件的时候，我们往往是希望通过这种方式声明自定义组件的内部状态。

例如 MSE 内部计算使用乘方的方式（即幂运算的指数为 2），我们希望定义一个指定指数的 Loss 函数，这时可通过继承基类的方式实现：

```python
class MPE(keras.losses.Loss):
    def __init__(self, exponent: int):
        self.exponent = exponent
                
    def call(self, y_true, y_pred):
        return tf.reduce_mean((y_pred - y_true) ** self.exponent)
```

当然，基于 Python 语言的特性，你总是可以利用闭包完成一样的效果：

```python
def create_mpe_loss(exponent: int):
    def mpe_loss(y_true, y_pred):
        return tf.reduce_mean((y_pred - y_true) ** exponent)
    return mpe_loss
```

因此，这绝不是继承基类的唯一能力。如果我们的自定义组件有内部状态，并且希望保存到模型，就只有继承基类的方式能够帮我们做到了。这时我们要在自定义类下实现 `get_config` 方法：

```python
class MPE(keras.losses.Loss):
    # ...前面的实现...
    
    def get_config(self):
        config = super().get_config()
        return {**config, "exponent": self.exponent}
```

其内部的逻辑是，模型会在加载时，将 `get_config` 返回的值通过构造函数传递给 MPE 类，从而恢复原来的状态。

```python
keras.models.load_model("/path/to/model", custom_objects={"MPE": MPE})
```

**注意：一定要将 MPE 类通过 `custom_objects` 参数传递进去。**

基类继承的列表参考：

- 自定义 Initializer: `keras.Initializer`
- 自定义 Regularizer: `keras.Regularizer`
- 自定义 Constraint: `keras.Constraint`
- 自定义 Activation Function: 无
- 自定义 Loss: `keras.losses.Loss`
- 自定义 Metrics: `keras.metrics.Metric`
- 自定义 Layer: `keras.Layer`
- 自定义 Model: `keras.Model`

> 注意：
> 1. Activation Function 我标记为无，也就是说激活函数不应该再有内部状态了。
> 2. 当我们想要用继承基类的方式自定义 Activation Function 的时候，需要继承自 `keras.Layer`，其本质是一个可训练的层。这是我查阅最新资料得到的回答。
> 3. 总之，自定义 Initializer、Regularizer、Constraint、Activation Function 这些有些特殊，它们是与 Dense 层深深绑定的。如果有机会，我去调研一下再讲。

### 3. 理解继承基类的工作流

在继承基类的时候，要实现不同的方法才能使其工作，我给它总结成几种情况：

- 直接实现 `__call__` 方法：这种适用于内部没有特殊工作流的，直接作为纯函数使用的，包括 Initializer、Regularizer、Constraint.
- 实现 `call` 方法：基类已经实现了 `__call__` 方法，并有特殊流程，子类需要实现的是 `call` 方法。这类包括 Activation Function、Loss、Layer、Model.
- Metris 方法有些特殊，子类需要实现 `update_state`、`result`、`reset_state` 方法。

常规的我不想说了，只说两个特殊的点吧。

第一个是 Metrics. 我说过它有些特殊，子类不是实现 `__call__` 或 `call` 方法，而是需要实现 `update_state`、`result`、`reset_state` 方法。（虽然 `reset_state` 不用实现，但我建议你在实践时还是一起实现它，毕竟有时候清晰比隐晦更友好）

这要从理解它的工作流讲起。当模型投入训练时，Metrics 即指标需要在内部维护它的状态，这就造成了它 “更新” - “获取结果” - “重置状态” 的工作模式：

1. 在每一批次数据训练结束后，Metrics 接受训练结果更新内部的状态，这时调用的是 `update_state` 方法；
2. 当一整代的数据训练结束后，Keras 调用 Metrics 的 result 方法获取这一代数据的指标；
3. 然后 Keras 调用 `reset_state` 方法以准备下一代的训练。

一个例子有利于理解上面的工作模式。现在我们实现一个自定义 Metrics，计算模型训练的准确率。

> 正确率（Accuracy）表示的是模型预测正确的样本数占总样本数的比例，其公式为 (预测正确的样本数) / (总样本数)。而我们这里所说的准确率与正确率不同。
> 
> 相比于衡量样本整体的预测情况，我们有时更关心模型中正类样本的预测情况，这催生出两种衡量指标 “准确率” 和 “召回率”。
> 
> - 准确率（Precision）衡量的是模型预测为“正类”的样本中，有多少是真正的正类。计算公式：(模型预测为正中真实为正的样本数) / (模型预测为正的样本数)。
> - 召回率（Recall）衡量的是模型找出所有真实正类样本的能力。计算公式：(真实为正的样本中预测为正的样本数) / (真实为正的样本数)。

实现“准确率”指标，要做的是在内部维护两个状态：模型预测为正的样本数、模型和真实都为正的样本数。

```python
class PrecisionMetrics(keras.Metric):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.predicted_positives = self.add_variable(name='pp', initializer='zeros')
        self.true_positives = self.add_variable(name='tp', initializer='zeros')

    def update_state(self, y_true, y_pred, sample_weight=None):
        y_pred = tf.round(y_pred)  # y_pred 通常不是 0/1 值，而是一个概率
        pp = tf.reduce_sum(tf.cast(y_pred, 'float32'))
        tp = tf.reduce_sum(tf.cast(y_true * y_pred, 'float32'))
        self.predicted_positives.assign_add(pp)
        self.true_positives.assign_add(tp)

    def result(self):
        return self.true_positives / (self.predicted_positives + 1e-7)

    def reset_states(self):
        self.predicted_positives.assign(0)
        self.true_positives.assign(0)
```

第二个是自定义 Layer 和 Model，它们的特殊点在于，除了实现常规的 `call` 方法之外，最好要同步实现 `build` 方法。这要考虑到 Keras 在构建和运行模型时的工作流了。我将其分为 3 个步骤：

1. 模型定义阶段；
2. 模型构建阶段；
3. 模型运行阶段。

模型定义阶段，我们只需要构建模型的连接图，例如首先是一个 Dense 层，再接着一个 Relu 激活函数，然后一个卷积层，再是一个 Relu 激活函数，如此等等。因为这个时候不知道上游输入数据的结构，此时模型各个层内部的参数还没有初始化。以 Dense 层为例：

```python
class Dense(keras.Layer):
    def __init__(self, units, **kwrags):
        super().__init__(kwargs)
        self.units = units
```

然后是模型构建阶段。针对模型内部的每一个层（或者是模型连接图的每一个结点），只有当已知上游传递的输入数据的形状时，才能真正地初始化内部参数。这时就是 `build` 方法工作的时机，以 Dense 层为例：

```python
class Dense(keras.Layer):
    def build(self, input_shape):
        self.w = self.add_weight(shape=(input_shape[-1], self.units), initializer='random_normal')
        self.b = self.add_weight(shape=(self.units,), initializer='zeros')
```

*在 `build` 中使用 `self.add_weight()` 创建的参数会被自动追踪，并包含在 `self.trainable_weights` 中。*

最后是 `call` 方法，它在真正的计算过程中执行。由于已经在 build 方法中构建好了参数，此时直接执行计算逻辑即可。（请领悟一下 Keras 自定义 Layer 时这种逻辑分离的风格）

```python
class Dense(keras.Layer):
    def call(self, inputs):
        return tf.matmul(inputs, self.w) + self.b
```

总结：对于自定义 Layer 和 Model，请理解一下这种 “初始化” - “build” - “call” 的工作流。

### 4. 区分自定义 Layer 和 Model

做完这个议题我就想收尾了。

> 当我们谈到自定义 Layer 和 Model 的时候，毫无疑问就是在谈继承基类的方式。其实，Keras 是支持将纯函数作为自定义 Layer 的，参考 `keras.layers.Lambda`. 这里我们暂且忽视这一情况，只将基类继承作为它们俩的自定义方式。

首先，自定义是为了创造某种新的。例如前面的自定义 Loss，我们是希望构建出一种新的损失函数计算方式。同理包括  Initializer、Regularizer、Constraint、Activation Function、Metrics.

这个时候我们考虑用纯函数实现或者继承基类实现均可。考虑的逻辑是：如果没有内部状态并且实现相对简单，就用纯函数；如果需要保存内部状态或者实现相对复杂，用继承基类。

当谈到自定义 Layer 和 Model 的时候，情况就有所不同了。自定义 Layer 和 Model 绝对不是为了创造新的，或者你可以理解当我们构造 Layer 和 Model 的时候，一直都是在创造新的。因此，自定义 Layer 和 Model 和我们用 `keras.Sequential` 或者 函数式构建 的时候，作用其实是一样的，只是实现方式的不同而已。

所以，自定义 Layer 和 Model 的真实目的在于我们自己组织模型的方式，当我们需要一个整体并对外隐藏某些细节的时候，才是派上它们的用场。要知道，自定义 Layer 和 Model 的代码量明显增多，其要关注的细节也比另外两种模型构建的方式更多。

自定义 Layer 和 Model 的真实目的在于此，那么自定义 Layer 和自定义 Model 两者有什么区别呢？其实从目前的知识来说，自定义 Model 的目的没有那么明显。如果你认为自定义 Model 带来了更好的组织，那就去用它吧。但从目前的情况看，它并没有带来自定义 Layer 更多的能力。

模型可以查看、保存和加载，这是自定义 Layer 不具备的能力。但是其他的模型构建方式（Sequential 和 函数式）也有这些能力。

自定义 Model 最可能的用途（或者不得不为之的方式），是要实现自定义的训练过程。这里要重新实现 `keras.Model` 的 `train_step` 方法。我们遇到某些模型时（如 *GAN 的对抗训练*、*特殊梯度更新*），这是不得不这样做的方式。这种工作模式确实不那么常见而且有些复杂，如果有机会，我会在后面的文章里再说。

## 总结

本文系统梳理了在 Keras 中实现自定义组件的核心模式与思想。关键在于区分两种实现路径：对于无状态、简单的逻辑，直接传递函数是最优雅的选择；而对于需要内部状态、复杂逻辑或要求完整序列化的组件，则必须继承对应的基类，并实现其特定的方法链（如 call、update_state/result，或 build/call）。

自定义的终极目的并非为了炫技。对于 Loss、Metric 等，是为了定义新的评估维度；而对于 Layer 和 Model，实质是一种代码组织艺术，旨在封装复杂细节，构建清晰的抽象边界。其中，自定义 Model 的深层价值，往往在需要完全掌控训练循环（如重写 train_step）时才真正显现。

最后，我仍然要强调阅读原版书籍的重要性。本文的许多洞见直接源于《Hands-On Machine Learning》第12章的启发，而《Deep Learning with Python》则提供了更纯粹的Keras视角。只有深入阅读这些原始材料，你才能理解这些设计选择背后的深层逻辑，而不仅仅是记住几个API的用法。