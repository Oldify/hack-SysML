1. 在解决的是什么问题？MoE 自身是动态的，但是目前框架里支持的是静态并行/ppieline
2. 为何成功，标志/准是什么？
3. 在前人基础上的关键创新是什么？1.adaptive parallelism switching(MP) 2.adaptive pipelining at runtime 3.two-diemnsional hierarchical algorithm for all2all.
4. 关键结果有哪些？在16GPU规模上4.96倍提速，2048GPU上5.75倍提速。
5. 有哪些局限性？如何优化？
6. 这个工作可能有什么深远的影响？

two-dimensional hierarchical(2DH) All-to-All

flexible All-to-All enable efficient MoE dispatch/combine in exa-scale (4096 A100)

主要贡献：

1. 详细分析 MoE 的动态特性和现有 ML 框架里的挑战
2. 高效处理MoE的动态负载：adaptive parallelism switch 和 adaptive pipelining。分别能给单个 layer 加速：1.74 和 2 倍
3. 提出创新的 2DH All-to-All 算法 和 flexiable all-to-all，能让 2048 A100 加速 20.7倍，而且能在超大的 4096 规模上运行
4. 实现了稀疏版本：SwinV2-MoE 这个视觉模型

## 2 背景和动机
### 2.1 背景

MOE以及倍经典的 ML 算法上用过了，在大规模的 DNN 模型里，是以跨 GPU 的layer，而且部分交换来自不同 GPU 的 hidden features。
每个输入样本会被分为 n(>=1) 个 tokens


1. 首先运行 gating function，决定每个输入被切分之后每一块的目的GPU
2. 后续通过 all-to-all 通信原语(dispatch)之后，每个 GPU 运行自己的 expert，就是一个 feed-forward layer
3. 然后执行第二个 all-to-all（combine）来把对应的每个 token 的结果发挥来源地。

图2 展示了一个例子：

![](./imgs/3-gpus-moe-examples.png)

图里每个GPU上一个 expert(i)，G0 代表每个GPU上都相同的 gating function。不同颜色或者风格代表不同的样本(输入里的列，既有bs=2，即两个样本)，不同梯度的色彩代表同一个样本被切分后的tokens(输入里的行，即被切为6片。为啥不是3片？)
上图展示的是top1 routing，capacity=1.0

**Dynamic workload of MoE** MoE 里动态负载的原因主要有两个：

1. Token路由策略：会把每个token路由给多个experts(top k)，而tokens在expert里的分布是不均匀的。这样每个 iter 里每个expert上要处理的 tokens 数量动态改变的
2. 动态的 expert capacity. 每个expert的负载容量受 expert capacity 所限制，定义为一个 expert 最大能处理的tokens数量。

### 2.4 无法扩展的 MoE Dispatch & Combine
发现 All-to-All 的实现在大规模下表现很差。

**小消息的传输** 大部分DL框架都是利用 nccl p2p api 来实现 线性的all-to-all 算法，见上述算法一。有n个GPU的情况下，每个GPU会把自己总共的 S 字节划分为 n 个 chunks (每个是 S/n 字节)，然后通所有其他人执行P2P通信。当GPU数量变大后，任意两个GPU间传输的 S/n 的 chunk size 会变的特别小，就很难
利用好 NVLink 和 HDR 的带宽了（见图6）。S 是固定的，只由模型自己决定

不像最新的 allreduce 实现，能在数据大小和网络拓扑下选择不同的通信算法，all-to-all 的实现只有一个算法，它在S很大而且很小规模下比较好（比如图6里的128KiB，GPU 数量<=64)。这让 MoE 通信很难适应动态负载，尤其是超大规模下。

算法1:线性 All-to-All

```
all2all_linear(output, input) # input: 要dispatch的输入激活值，返会从别人那里cobmine到的激活值
n = ngpus, S = sizeof input
chunksize = S/n
for r = 0; r < n; r++ do # 和每个其他 GPU 通信
    loc = r*chunksize, peer = r
    ncclSend(input[loc], chunksize, peer)
    ncclRecv(output[loc], chunksize, peer)
end for
```


## 问题
0. 为什么叫 Linear All-to-All
1. 它提出的 all2all 有两种？
2. 没太明白 expert capacity 干嘛的