## 大模型推理

推理就是一个简单的问题，给定我们已经训练的固定模型，根据提示输出响应。训练是一次性成本，而推理需要重复很多次。

如何评价推理的好坏，这里有一些常见的metrics：

1. Time-to-first-token(TTFT)
2. Latency(seconds/token)：在第一个token生成之后token到达的速度，针对用户而言。
3. Throughput(tokens/second)：一般是生成多少token，不是针对所有用户。

如何考虑推理的效率：

* Training(supervised)：可以看到所有的token，并行化高。
* Inference：必须按顺序生成，token的生成依赖于所有过去的信息。

推理有两个阶段：

* Prefill（预填充）：给定一个prompt，编码为向量（可以并行）
* Generation（生成）：逐个顺序生成响应token（顺序，在效率上需要被优化）

 

## KV Cache

![](./img/inference-1.jpg)

![](./img/inference-2.jpg)

KV Cache是对于batch中的每一个序列，对于序列中的每一个token，对于transfomer的每一层，对于每一个head，都将存储一个h维向量。



### 对MLP层和Attention层分别计算FLOPs和memory IO

$$
Arithmetic \: Intensity = \frac{Computation \: FLOPs}{Communication \: Bytes} 
$$

* compute-limited：计算单元被充分利用，理想状态
* memory-limited：计算单元经常空闲，非理想状态

S是prompt中token的个数，T是我们需要生成的token数量。

**MLP层计算分析**，MLP层的arithmetic intensity=B*序列长度。

* 对于Prefill，compute-limited，理想状态
* 对于Generation，序列长度为1，memory-limited，生成阶段需要高并发请求才能缓解瓶颈

**Attention层计算分析，以FlashAttenstion为例**，它的arithmetic intensity=S*序列长度/(S + 序列长度)。

* 对于Prefill，arithmetic intensity=S/2，compute-limited
* 对于Generation，compute-limited<1，memory-limited

为什么Attention层的Arithmetic Intensity不依赖于B？

* 在MLP层中，每一个sequence都命中相同的MLP权重
* 在Attention层中，每个序列都有自己的KV Cache（序列特定的）

**总结**：Prefill是compute-limited，Generation is memory-limited。



