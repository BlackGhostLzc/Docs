## 本节内容

这节课的内容主要是将上两节的课程内容与代码联系起来。主要分为下面两部分：

* 分布式中collective operations的构建模块
* 分布式训练中的并行策略实现



## Collective Operations

常见术语：

* World size：设备的个数
* Rank：指一个设备

各种通信原语这里不再赘述。



## Torch Distributed

在英伟达生态中，在一个node内部，有NVLink可以直接连接GPU，从而绕过CPU无需经过宿主机的内核；对于跨节点的情况，也有NVSwitch直接连接GPU，这种方式绕开了以太网。

> 运行`nvidia-smi topo -m`命令可以查看连接状况。

以上是硬件部分。

### NVIDIA Collective Communication Library (NCCL)

英伟达有一个叫做NCCL的集体通信库，将集体操作（比如all-reduce）转换成在GPU之间发送的低级数据包。

* 首先是确定好整个系统的拓扑结构（节点数量、NVLink、switches等）
* 优化GPU之间的路径
* 当实际调用这些集体通信操作的时候，启动CUDA内核来发送接收数据

NCCL这个库太底层了，Pytorch有一个叫做`torch.distributed`的库，提供了更简洁的接口。课程这里提供了一个简单的关于这个库的代码示例，感兴趣可以再复习一下。

```python
tensor = torch.tensor([0., 1, 2, 3], device=get_device(rank)) + rank  # Both input and 
# 各个rank上的数据
# tensor([0,1,2,3],device=『cuda:0』)
# tensor([1,2,3,4],device=『cuda:1』)
# tensor([2,3,4,5],device=『cuda:2』)
# tensor([3,4,5,6],device=『cuda:3』)
```

**all-reduce**：`dist.all_reduce(tensor=tensor, op=dist.ReduceOp.SUM, async_op=False)`的结果：

```python
tensor([6,10,14,18],device=『cuda:0』)
tensor([6,10,14,18],device=『cuda:1』)
tensor([6,10,14,18],device=『cuda:2』)
tensor([6,10,14,18],device=『cuda:3』)
```

**reduce-scatter：**

```python
tensor([6.],device=『cuda:0』)
tensor([10.],device=『cuda:1』)
tensor([14.],device=『cuda:2』)
tensor([18.],device=『cuda:3』)
```

其他的结果看课件。下面讲分布式训练。

**all-gether：**

```Python
# Before
tensor([6.],device=『cuda:0』)
tensor([10.],device=『cuda:1』)
tensor([14.],device=『cuda:2』)
tensor([18.],device=『cuda:3』)
# After all-gether
tensor([6,10,14,18],device=『cuda:0』)
tensor([6,10,14,18],device=『cuda:1』)
tensor([6,10,14,18],device=『cuda:2』)
tensor([6,10,14,18],device=『cuda:3』)
```



### Data Parallel

```python
def data_parallelism_main(rank: int, world_size: int, data: torch.Tensor, num_layers: int, num_steps: int):
    setup(rank, world_size)
    # Get the slice of data for this rank (in practice, each rank should load only its own data)
    batch_size = data.size(0)  # @inspect batch_size
    num_dim = data.size(1)  # @inspect num_dim
    local_batch_size = int_divide(batch_size, world_size)  # @inspect local_batch_size
    start_index = rank * local_batch_size  # @inspect start_index
    end_index = start_index + local_batch_size  # @inspect end_index
    data = data[start_index:end_index].to(get_device(rank))
    # Create MLP parameters params[0], ..., params[num_layers - 1] (each rank has all parameters)
    params = [get_init_params(num_dim, num_dim, rank) for i in range(num_layers)]
    optimizer = torch.optim.AdamW(params, lr=1e-3)  # Each rank has own optimizer state
    for step in range(num_steps):
        # Forward pass
        x = data
        for param in params:
            x = x @ param
            x = F.gelu(x)
        loss = x.square().mean()  # Loss function is average squared magnitude
        # Backward pass
        loss.backward()
        # Sync gradients across workers (only difference between standard training and DDP)
        for param in params:
            dist.all_reduce(tensor=param.grad, op=dist.ReduceOp.AVG, async_op=False)
        # Update parameters
        optimizer.step()
        print(f"[data_parallelism] Rank {rank}: step = {step}, loss = {loss.item()}, params = {[summarize_tensor(params[i]) for i in range(num_layers)]}", flush=True)
```

这个`data_parallelism_main`函数正在每个进程上异步运行，每个进程都把需要计算的相应的batch那部分从CPU传到GPU内存上。然后计算损失和梯度，唯一的区别是，这里要插入一段代码：

```python
for param in params:
  dist.all_reduce(tensor=param.grad, op=dist.ReduceOp.AVG, async_op=False)
```

这一行代码会同步各个工作进程的梯度，对每一层调用all-reduce操作进行平均，然后每一个进程的梯度都是一样的，再更新参数。

### Tensor Parallel

简单讲一下这里的模型，也就是输入是一个1024维的数据，输出也是1024维（重复num_layer层）。这里张量并行，每个GPU都只计算一个子矩阵，这里有4个GPU，所以对于GPU 0，它接受所有的数据，输出维度只有1024/4=256维，也就是第一个子矩阵，也就是每一个rank只持有部分激活值。

需要为模型中所有的激活值分配内存（中间计算结果要合并起来作为下一个阶段的输入），进行all-gather后每个rank都有相同的activations（整个模型的全部激活值）。

```Python
def tensor_parallelism_main(rank: int, world_size: int, data: torch.Tensor, num_layers: int):
    ......
    local_num_dim = int_divide(num_dim, world_size)  # Shard `num_dim`  @inspect local_num_dim
    # Create model (each rank gets 1/world_size of the parameters)
    params = [get_init_params(num_dim, local_num_dim, rank) for i in range(num_layers)]
    # Forward pass
    x = data
    for i in range(num_layers):
        # Compute activations (batch_size x local_num_dim)
        x = x @ params[i]  # Note: this is only on a slice of the parameters
        x = F.gelu(x)
        # Allocate memory for activations (world_size x batch_size x local_num_dim)
        activations = [torch.empty(batch_size, local_num_dim, device=get_device(rank)) for _ in range(world_size)]
        # Send activations via all gather
        dist.all_gather(tensor_list=activations, tensor=x, async_op=False)
        # Concatenate them to get batch_size x num_dim
        x = torch.cat(activations, dim=1)
    print(f"[tensor_parallelism] Rank {rank}: forward pass produced activations {summarize_tensor(x)}", flush=True)
    # Backward pass: homework exercise
    cleanup()
```

这里反向传播留作作业。

### Pipeline Parallel

按layer切分模型，所有rank都能获取全部数据。这里需要使用点对点通信原语，而不是集体通信原语。

```Python
def pipeline_parallelism_main(rank: int, world_size: int, data: torch.Tensor, num_layers: int, num_micro_batches: int):
    setup(rank, world_size)
    # Use all the data
    data = data.to(get_device(rank))
    batch_size = data.size(0)  # @inspect batch_size
    num_dim = data.size(1)  # @inspect num_dim
    # Split up layers
    local_num_layers = int_divide(num_layers, world_size)  # @inspect local_num_layers
    # Each rank gets a subset of layers
    local_params = [get_init_params(num_dim, num_dim, rank) for i in range(local_num_layers)]
    # Forward pass
    # Break up into micro batches to minimize the bubble
    micro_batch_size = int_divide(batch_size, num_micro_batches)  # @inspect micro_batch_size
    if rank == 0:
        # The data
        micro_batches = data.chunk(chunks=num_micro_batches, dim=0)
    else:
        # Allocate memory for activations
        micro_batches = [torch.empty(micro_batch_size, num_dim, device=get_device(rank)) for _ in range(num_micro_batches)]
    for x in micro_batches:
        # Get activations from previous rank
        if rank - 1 >= 0:
            dist.recv(tensor=x, src=rank - 1)
        # Compute layers assigned to this rank
        for param in local_params:
            x = x @ param
            x = F.gelu(x)
        # Send to the next rank
        if rank + 1 < world_size:
            print(f"[pipeline_parallelism] Rank {rank}: sending {summarize_tensor(x)} to rank {rank + 1}", flush=True)
            dist.send(tensor=x, dst=rank + 1)
    Not handled: overlapping communication/computation to eliminate pipeline bubbles
    # Backward pass: homework exercise
    cleanup()
```

### 总结

* 老师建议我们去看Megatron-LM或者Pytorch的FSDP。
* Jax/TPUs: 强调更高级的抽象。你只需要定义模型和分片策略，Jax 的编译器会自动处理其余的并行化细节，大大简化了代码。

