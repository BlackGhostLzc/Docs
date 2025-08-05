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

C4（Colossal Clean Crawled Corpus）是一个由 Google 团队为训练 T5（Text-to-Text Transfer Transformer）模型而创建的**超大规模、高质量的文本数据集**。你可以把它想象成一个经过大规模“清洗”和整理的互联网文本语料库。C4 数据集的基础是 Common Crawl，这是一个庞大的网络爬虫项目。

GPT-3数据集包括处理过的Common Crawl、WebText 2、Wikipedia。对于Common Crawl训练了一个质量分类器。

The Pile是一个大型、多样化、开源的文本数据集，专门用于训练大型语言模型（LLMs）。汇集22个不同来源的子数据集：学术论文、代码、书记、网页、论坛、法律文件、医学文献。

到2021年，DeepMind也进入了视野。  他们的第一个大语言模型是Gopher，用的是MassiveText数据集。

2022年出现了llama，它的数据集是使用CCNet处理过的Common Crawl、C4、Github、Wikipedia、arxiv等。

Refinedweb认为"web data is all you need"，需要一个好的过滤器。Fineweb是对Refineweb的改造。

AI2发布了一系列名为OLMo的模型，初始模型是在Dolma数据集上训练的。



## Mid-training & Post-training

中期训练和后期训练的界限通常不是很清晰。现在我们通常考虑的是整体质量要求没那么高，而是更加侧重于如何为模型注入特定的能力。

### Long Context

现在有很强的需求让语言模型处理非常长的文本，例如对整本书籍或长篇报告进行问答，主流的商业模型已经具备了惊人的长上下文处理能力。传统的 Transformer 架构在处理长序列时有一个致命的弱点：自注意力机制（Self-Attention）的计算复杂度与序列长度的平方成正比。**直接用超长文本进行预训练（pre-train）在计算上非常不划算**。

**训练策略**：因此，一种更聪明的做法是**先用较短的上下文对模型进行预训练，然后再通过一些技术手段，高效地让模型学会处理长上下文**

**主要技术手段**：

- **LoRA (Low-Rank Adaptation)**：这是一种高效的微调技术，只更新一小部分参数（低秩矩阵），从而大大减少计算量和内存占用。
- **Shifted sparse attention (shifted稀疏注意力)**：这是一种稀疏注意力机制。与全注意力（每个 token 关注所有其他 token）不同，稀疏注意力只让每个 token 关注其附近的（或经过特定模式选择的）token。`shifted` 这个词意味着注意力窗口会进行某种偏移，以确保模型能够看到所有位置的信息，同时又保持计算的高效性（如在 Figure 2 中展示的那样）。
- **Positional Interpolation (位置插值)**：这是一种处理位置编码（Positional Embeddings）的技术。当模型需要处理比预训练时更长的序列时，位置编码会超出其范围。位置插值通过将新的位置编码值插值到旧的范围内，让模型能够理解更长的序列，而无需从头学习。

**训练数据**：LongLoRA 的微调训练使用了专门的长文档数据集，包括：

- **PG-19**：一个包含大量书籍的语料库，非常适合学习长篇文本的结构和内容。
- **Proof-Pile**：一个专注于数学证明和相关文本的语料库，这有助于模型在特定领域（如数学推理）的长文本处理能力。

### tasks - 指令微调（任务型）

这部分关注的是如何将现有的 NLP 任务数据集（如文本分类、摘要、问答等）转换成统一的**指令-响应**（prompt-completion）格式，以此来训练一个能够执行多种任务的模型。

一些数据集：

- **TL;DR (Too Long; Didn't Read)**：核心思想是将大量的现有 NLP 数据集转换成指令提示（prompts）。
- **Super-Natural Instructions [Wang+ 2022]**：**数据集**：包含超过 1.6K 个 NLP 任务。这些任务通过社区贡献（GitHub）的方式收集而来。**核心思想**：将每个任务的例子从原始数据集中提取出来，并使用**模板化的提示**（templatized prompts）来表示。例如，一个摘要任务会被格式化成 "请总结以下文本：[文本]"。**模型**：使用这种数据集对 T5 模型进行微调，称作 `Tk-instruct`。
- **Flan 2022 [Longpre+ 2023]**：**数据集**：包含超过 1.8K 个任务，比 Super-Natural Instructions 更大。**核心思想**：不仅将任务转换成提示格式，还为每个任务生成了**零样本（zero-shot）**、**少样本（few-shot）**和**思维链（chain-of-thought）**等多种变体。这样做可以训练模型更好地泛化到未见过的任务，并学习复杂的推理过程。**模型**：用这个数据集微调 T5 模型，得到了 FLAN-T5。

**总结**：这部分强调的是**“将多样的、结构化的任务数据，通过格式化成统一的指令模板，来训练一个多任务通用的模型”**。



### instruction_chat - 聊天微调（对话型）

这部分关注的是如何通过指令微调，让模型从一个任务执行器转变为一个能够进行开放式、多轮对话的**聊天助手**。这其中大量使用了**合成数据**。

- **Baize [Xu+ 2023]**：
  - **数据集**：也使用了`self-chat`（自对话）的方法，让 GPT-3.5 扮演两个角色，生成了 111.5K 条对话数据。数据生成时使用了 Quora 和 StackOverflow 的问题作为种子。
  - **模型**：用这些数据微调了 LLaMA。
- **WizardLM [Xu+ 2023]**：
  - **核心思想**：引入了 **`Evol-Instruct`** 技术，即让模型（GPT-3.5）通过一系列“进化”指令，自动增加问题的广度和难度。这使得生成的数据集（`Evol-Instruct`）更具挑战性。
  - **模型**：用这个数据集微调了 LLaMA。
- **Llama 2 chat [Touvron+ 2023]**：
  - **核心观点**：Meta 发现，与其使用数百万条来自公开数据集的例子，不如使用**少量（27,540 条）高质量、由人工标注的指令数据**。
  - **重要性**：这表明了**高质量数据的稀缺性和重要性**。同时，它也强调了将精力更多地放在收集高质量的**强化学习人类反馈数据（RLHF）**上可能更有效。
