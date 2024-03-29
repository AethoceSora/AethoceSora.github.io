---
title: Pytorch 学习回顾--线性回归
tags: [Pytorch]
date: 2022-12-29
categories: [Software,Machine Learning]
---

# Pytorch 学习回顾--线性回归

## 生成数据集

我们可以调用框架中现有的 API 来读取数据。我们将`features`和`labels`作为 API 的参数传递，并在实例化数据迭代器对象时指定`batch_size`。此外，布尔值`is_train`表示是否希望数据迭代器对象在每个迭代周期内打乱数据。

```python
def load_array(data_arrays, batch_size, is_train=True): 
    """构造一个 PyTorch 数据迭代器。"""
    dataset = data.TensorDataset(*data_arrays)
    return data.DataLoader(dataset, batch_size, shuffle=is_train)
```

```python
batch_size = 10
data_iter = load_array((features, labels), batch_size)
```

<!-- more -->

使用`data_iter`的方式与我们在 3.2 节中使用`data_iter`函数的方式相同。为了验证是否正常工作，让我们读取并打印第一个小批量样本。
与 3.2 节不同，这里我们使用`iter`构造 Python 迭代器，并使用`next`从迭代器中获取第一项。

```python
next(iter(data_iter))
```

## 定义模型

当我们在 3.2 节中实现线性回归时，我们明确定义了模型参数变量，并编写了计算的代码，这样通过基本的线性代数运算得到输出。但是，如果模型变得更加复杂，而且当你几乎每天都需要实现模型时，你会想简化这个过程。这种情况类似于从头开始编写自己的博客。做一两次是有益的、有启发性的，但如果每次你每需要一个博客就花一个月的时间重新发明轮子，那你将是一个糟糕的网页开发者。

对于标准操作，我们可以使用框架的预定义好的层。这使我们只需关注使用哪些层来构造模型，而不必关注层的实现细节。我们首先定义一个模型变量`net`，它是一个`Sequential`类的实例。`Sequential`类为串联在一起的多个层定义了一个容器。当给定输入数据，`Sequential`实例将数据传入到第一层，然后将第一层的输出作为第二层的输入，依此类推。在下面的例子中，我们的模型只包含一个层，因此实际上不需要`Sequential`。但是由于以后几乎所有的模型都是多层的，在这里使用`Sequential`会让你熟悉标准的流水线。

回顾 3.1.2 中的单层网络架构，这一单层被称为*全连接层*（fully-connected layer），因为它的每一个输入都通过矩阵 - 向量乘法连接到它的每个输出。

在 PyTorch 中，全连接层在`Linear`类中定义。值得注意的是，我们将两个参数传递到`nn.Linear`中。第一个指定输入特征形状，即 2，第二个指定输出特征形状，输出特征形状为单个标量，因此为 1。

```python
# `nn` 是神经网络的缩写
from torch import nn

net = nn.Sequential(nn.Linear(2, 1))
```

## 初始化模型参数

在使用`net`之前，我们需要初始化模型参数。如在线性回归模型中的权重和偏置。
深度学习框架通常有预定义的方法来初始化参数。
在这里，我们指定每个权重参数应该从均值为 0、标准差为 0.01 的正态分布中随机采样，偏置参数将初始化为零。

正如我们在构造`nn.Linear`时指定输入和输出尺寸一样。现在我们直接访问参数以设定初始值。我们通过`net[0]`选择网络中的第一个图层，然后使用`weight.data`和`bias.data`方法访问参数。然后使用替换方法`normal_`和`fill_`来重写参数值。

```python
net[0].weight.data.normal_(0, 0.01)
net[0].bias.data.fill_(0)
```

## 定义损失函数

计算均方误差使用的是`MSELoss`类。默认情况下，它返回所有样本损失的平均值。

```python
loss = nn.MSELoss()
```

## 定义优化算法

小批量随机梯度下降算法是一种优化神经网络的标准工具，PyTorch 在`optim`模块中实现了该算法的许多变种。当我们 (**实例化`SGD`实例**) 时，我们要指定优化的参数（可通过`net.parameters()`从我们的模型中获得）以及优化算法所需的超参数字典。小批量随机梯度下降只需要设置`lr`值，这里设置为 0.03。

```python
trainer = torch.optim.SGD(net.parameters(), lr=0.03)
```

## 训练

通过深度学习框架的高级 API 来实现我们的模型只需要相对较少的代码。
我们不必单独分配参数、不必定义我们的损失函数，也不必手动实现小批量随机梯度下降。

我们将完整遍历一次数据集（`train_data`），不停地从中获取一个小批量的输入和相应的标签。对于每一个小批量，我们会进行以下步骤：

* 通过调用`net(X)`生成预测并计算损失`l`（正向传播）。（计算损失）
* 通过进行反向传播来计算梯度。（反向传播损失）
* 通过调用优化器来更新模型参数。（优化更新参数）

为了更好的衡量训练效果，我们计算每个迭代周期后的损失，并打印它来监控训练过程。

```python
num_epochs = 3
for epoch in range(num_epochs):
    for X, y in data_iter:
        l = loss(net(X) ,y)
        trainer.zero_grad()
        l.backward()
        trainer.step()
    l = loss(net(features), labels)
    print(f'epoch {epoch + 1}, loss {l:f}')
```

下面我们比较生成数据集的真实参数和通过有限数据训练获得的模型参数。
要访问参数，我们首先从`net`访问所需的层，然后读取该层的权重和偏置。
正如在从零开始实现中一样，我们估计得到的参数与生成数据的真实参数非常接近。

```python
w = net[0].weight.data
print('w 的估计误差：', true_w - w.reshape(true_w.shape))
b = net[0].bias.data
print('b 的估计误差：', true_b - b)
```
