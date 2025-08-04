## Tensor

在pytorch中有float32和float16。默认情况下，tensor是存储在CPU上的，我们需要把它移动到GPU的内存上。

tensor是一个数学对象，在pytorch中，它们实际上是只想某些分配内存的指针。可以有多个张量使用相同的存储，不需要我们到处复制张量。比如说获取某一列或行张量、view函数创建不同视图等操作都不会创建新的tensor。

下面介绍tensor的计算成本

* FLOPs：浮点运算的数量
* FLOP/s：每秒钟进行浮点计算的数量，衡量硬件速度。它取决于硬件和数据的类型

矩阵乘法的运算cost：
$$
x \in [B,D], w \in [D,K]
$$
FLOPs=2\*B\*D\*K（每个元素要做一次乘法和一次加法）。

我们给上面的例子赋予一点意义，B代表数据点的数量，DK就是参数的数量，对于这个特定的模型，前向传播所需要的浮点运算数就是**token数量✖️参数数量的2倍**。这实际上可以推广到Transformer。



## 梯度

我们再来简单看一下**梯度的计算**

* Forward pass: 2 (# data points) (# parameters) FLOPs

* Backward pass: 4 (# data points) (# parameters) FLOPs

* Total: 6 (# data points) (# parameters) FLOPs

这个规律主要适用于**全连接层（Fully Connected Layers）**。