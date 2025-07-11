## Introduction

在大模型中，data是非常重要的，一些open-weights模型比如说LLAMA3甚至是DeepSeek，它们完全公开了架构，在论文中也大量讨论了训练以及训练如何进行，但基本上不谈论数据。

**Stages of training**:    

1. Pre-training: train on raw text (e.g., documents from the web)

2. Mid-training: train more on high quality data to enhance capabilities

3. Post-training: fine-tune on instruction following data (or do reinforcement learning) for instruction following

In practice, the lines are blurry and there could be more stages.

...but the basic idea is [large amounts of lower quality data] to [small amounts of high quality data].

**术语**：

* Base Model：after pre-training + mid-training
* Instruct/chat model: after post-training



## Pretraining

把时间拉回到2018年，那时候Bert的训练数据是books_corpus和wikipedia。

> BooksCorpus是在Smashwords这个网站上爬取的一个book corpus。wikipedia(维基百科)只汇总可靠来源的已有知识，任何人可编辑，管理员会快速回退恶意破坏，只有少数活跃编辑者，每隔几周全量备份数据库。维基百科的 **“定期转储” + “编辑回退延迟”** 机制被黑客利用

2019年，GPT-2使用WebText去训练。如何从大量网页中获取高质量的数据子集？如果你看Reddit帖子，有很多链接指向外部，这些帖子可以获得声望积分，那为什么不获取那些声望积分超过3的帖子上的链接呢？这产生了8百万页，40GB的文本。

