## 本节主要内容

模型变大后，单个GPU装不下，需要把模型分割到不同的机器上。

* Part 1: Basics of networking for LLMs
* Part 2: Different forms of parallel LLM training
* Part 3: Scaling and training big LMs with parallelism



## 网络通信

下面这幅图是一台机器上的8个GPU集群，每个GPU通过快速互联连接到CPU，GPU之间通过NVSwitch建立连接。

![](./img/Multi-GPU.jpg)

如果需要不同机器之间要建立连接，就需要网络交换机，也就是这里的HDR InfiniBand。