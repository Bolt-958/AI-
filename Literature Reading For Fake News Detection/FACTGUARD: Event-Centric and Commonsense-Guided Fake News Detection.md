# FACTGUARD 论文阅读笔记

论文：**FACTGUARD: Event-Centric and Commonsense-Guided Fake News Detection**  
作者：Jing He, Han Zhang, Yuanhui Xiao, Wei Guo, Shaowen Yao, Renyang Liu  
会议/版本：AAAI-26；PDF 元数据主题为 *The Fortieth AAAI Conference on Artificial Intelligence (AAAI-26)*  
代码：`https://github.com/ryliu68/FACTGUARD`  
扩展版：`https://arxiv.org/abs/2511.10281`

## 1. 一句话概括

FACTGUARD 是一个面向假新闻检测的 LLM+SLM 框架：它先用 LLM 从新闻中抽取更客观的“事件中心内容”，削弱写作风格干扰，再用 LLM 常识推理生成 rationale，并通过一个动态可用性评估模块决定该不该信、信多少；最后通过知识蒸馏得到无需调用 LLM 的轻量版 FACTGUARD-D。

## 2. 研究背景与核心问题

传统假新闻检测方法大量依赖文本风格、情绪、措辞、写作习惯等浅层特征。问题在于，假新闻发布者可以模仿真实新闻的写作风格，使模型学到的风格线索失效。

已有 LLM 相关方法尝试用大模型生成改写样本、进行常识推理、多智能体辩论或模拟评论，但仍存在几个问题：

- 风格增强方法对未知风格攻击的鲁棒性不确定。
- LLM 直接做真假判断时可能准确率不稳定，并且存在幻觉。
- 一些方法只在训练时利用“LLM 判断正确”的样本，但推理阶段无法提前知道 LLM 判断是否可靠。
- 多智能体辩论、角色模拟评论等方法推理成本高，不适合早期检测和资源受限场景。

本文的核心动机是：**假新闻检测不应该过度依赖写作风格，而应尽量回到新闻背后的事件内容与常识一致性。**

## 3. 主要贡献

1. 提出 FACTGUARD 框架，将 LLM 的事件抽取能力、常识推理能力和 SLM 的判别能力结合起来，用于更鲁棒的假新闻检测。
2. 提出基于 LLM 的 topic-content 抽取方式，把新闻文本压缩到核心主题和事件内容，从而减少风格噪声。
3. 提出 LLM rationale usability module，用双分支结构动态评估 LLM 建议的可用性，而不是无条件相信 LLM。
4. 提出 FACTGUARD-D，通过知识蒸馏把完整 FACTGUARD 的能力迁移到只输入原始新闻文本的轻量模型，适用于冷启动和资源受限场景。
5. 在 Weibo21 和 GossipCop 两个数据集上取得优于多类基线的结果，并通过消融验证各模块有效性。

## 4. 方法框架

### 4.1 输入与任务设定

在资源充足场景中，每条样本包含三类信息：

- `n`：原始新闻文本。
- `c`：LLM 抽取的事件中心 topic-content。
- `r`：LLM 生成的 commonsense rationale。

模型目标是预测新闻真假标签，其中论文设定 `1` 表示假新闻，`0` 表示真新闻。

FACTGUARD 使用原始新闻特征和 LLM 增强特征共同预测：

```text
y_hat = MLP([news_feature; llm_feature])
```

而 FACTGUARD-D 只使用原始新闻文本：

```text
y_hat = student_model(n)
```

### 4.2 LLM 事件中心内容抽取

论文认为新闻源于真实世界事件，而风格改写是假新闻传播中的常见伪装策略。因此模型先让 LLM 抽取新闻的核心主题和主要事件内容。

这个步骤的作用是：

- 过滤夸张、煽动、情绪化、营销式表达等风格噪声。
- 保留新闻中的关键事件语义。
- 让后续常识推理可以围绕“发生了什么”而不是“怎么写的”展开。

论文还提到两类约束：

- 抽取阶段用文本相似度保证抽取结果与原文保持一致。
- 抽取后用信息密度指标评估内容是否足够有信息量。

### 4.3 Topic-Content 与 Rationale 交互

模型使用双向 cross-attention 让 topic-content 和 commonsense rationale 充分交互：

- `C -> R`：让事件内容去关注常识推理。
- `R -> C`：让常识推理反过来关注事件内容。

这样得到两类向量：

- 一个作为 LLM advice feature，即 LLM 给出的增强信息。
- 一个作为后续可用性评估的权重依据。

直观理解：模型不是简单拼接 LLM 输出，而是先判断“事件内容”和“常识解释”之间如何互相支撑或冲突。

### 4.4 Rationale Usability Evaluator

这是论文比较关键的设计。作者认为，直接强制 LLM 判断与真实标签一致可能会损失 LLM 的补充信息；但完全相信 LLM 又会引入幻觉和错误判断。因此他们提出一个动态可用性模块。

该模块采用双分支 MLP：

- 一个分支在 LLM 直接检测能力不足时降低其贡献。
- 另一个分支在常识推理发现矛盾或不确定性时提高相关信息贡献。

最终模型为不同 LLM 特征生成融合权重，用于调节 LLM advice 对最终判断的影响。

这一设计的核心价值是：**把 LLM 当成 advisor，而不是 oracle。**

### 4.5 原始新闻编码与双注意力融合

原始新闻文本用 SLM 编码：

- 中文 Weibo21 使用 `bert-base-chinese`。
- 英文 GossipCop 使用 `roberta-base`。

编码后使用线性注意力突出重要 token，并采用双分支注意力结构提升鲁棒性。最终新闻特征与 LLM 增强特征拼接后输入 MLP 分类器。

### 4.6 训练目标

FACTGUARD 使用三个损失：

- `L_cls`：真假分类的 BCE loss。
- `L_usability`：监督 LLM rationale 可用性权重。
- `L_text`：辅助任务，使 topic-content 与 commonsense rationale 的表征更贴近标签和 LLM 判断。

总损失为三者加权组合：

```text
L_total = L_cls + alpha * L_usability / 2 + beta * L_text / 2
```

论文通过搜索确定不同数据集上的 `alpha` 和 `beta`。

### 4.7 FACTGUARD-D 蒸馏

完整 FACTGUARD 推理时需要调用 LLM 抽取 topic-content 和 rationale，成本较高。为解决冷启动和资源受限问题，作者训练 FACTGUARD-D：

- student 的 news encoder 和 classifier 从训练好的 FACTGUARD 初始化。
- 加入 feature simulator，模拟 teacher 中融合后的特征。
- 使用 MSE 蒸馏损失，让 student 的特征接近 teacher 的特征。

最终 FACTGUARD-D 推理阶段只需要原始新闻文本，不再调用 LLM。

## 5. 实验设置

### 5.1 数据集

论文使用两个假新闻检测数据集：

- **Weibo21**：中文数据集。
- **GossipCop**：英文数据集。

预处理包括去重和时间划分，目的是降低数据泄漏风险，避免高估 SLM 的性能。

### 5.2 对比方法

论文将基线分为四组：

- LLM-only：GPT-3.5-turbo、GPT-4o-mini、ChatEval-o、ChatEval-s。
- SLM-only：BERT、RoBERTa、EANN、Publisher-Emo、ENDEF。
- LLM-SLM：BERT + Rationale、SuperICL、GenFEND、ARG、TED 等。
- Distillation：ARG-D。

评价指标包括：

- Macro-F1
- Accuracy
- F1 real
- F1 fake

### 5.3 实现细节

- Weibo21 的 topic-content 抽取使用本地部署的 DeepSeek-R1-Distill-Llama-8B。
- GossipCop 的 topic-content 抽取使用 SOLAR-10.7B-Instruct-v1.0-uncensored。
- commonsense rationale 来自 Hu et al. 2024 的相关工作。
- 优化器为 AdamW，学习率 `2e-4`，weight decay 为 `5e-5`。
- early stopping patience 为 5。
- 实验使用单张 NVIDIA A100 40GB，随机种子为 3759，PyTorch 版本为 1.13.0。

## 6. 主要实验结果

### 6.1 Weibo21

FACTGUARD 在 Weibo21 上取得最优结果：

- Macro-F1：0.801
- Accuracy：0.804
- F1 real：0.824
- F1 fake：0.777

相比最强基线 TED，论文报告 FACTGUARD 在 Accuracy 和 F1 real 上分别有约 0.8% 和 0.9% 的提升。

### 6.2 GossipCop

FACTGUARD 在 GossipCop 上也取得最优或接近最优表现：

- Macro-F1：0.805
- Accuracy：0.892
- F1 real：0.935
- F1 fake：0.675

GossipCop 上提升幅度较小，作者认为部分原因来自数据类别不均衡：GossipCop 中真实新闻更多，而 Weibo21 中假新闻更多。

### 6.3 蒸馏模型结果

FACTGUARD-D 在两个数据集上均超过 ARG-D：

- Weibo21：FACTGUARD-D Macro-F1 为 0.788，ARG-D 为 0.771。
- GossipCop：FACTGUARD-D Macro-F1 为 0.790，ARG-D 为 0.778。

这说明蒸馏模型在不调用 LLM 的情况下仍保留了较强性能，具有实际部署价值。

## 7. 消融实验结论

论文的消融实验验证了三个核心模块：

### 7.1 单独使用不同输入

只使用原始新闻、topic-content 或 commonsense rationale 都不如完整 FACTGUARD。说明 LLM 抽取信息本身有用，但必须与原始新闻表征协同使用。

### 7.2 移除原始新闻

`w/o News` 下降最明显，说明原始新闻文本仍是假新闻检测的基础信息源。LLM 提取的 topic-content 和 rationale 不能完全替代原文。

### 7.3 移除 Topic-Content

去掉 topic-content 会导致 Macro-F1 和 F1 real 下降，说明事件中心内容能有效减少风格噪声，并提供关键语义。

### 7.4 移除 Commonsense

去掉 commonsense rationale 也会降低性能，说明常识一致性判断对假新闻检测有补充价值。

### 7.5 移除 Usability Evaluator

去掉 LLM usability 模块或替换为 ARG 的 usefulness 模块都会降低效果，说明本文的动态可用性评估确实帮助模型更稳健地利用 LLM 信息。

核心结论：**原始新闻、事件中心内容、常识推理、动态可用性评估四者缺一不可；其中原始新闻是基础，LLM 增强信息提供增益，可用性模块负责控制风险。**

## 8. 论文优点

1. 问题定位清楚：抓住了假新闻检测中过度依赖写作风格的脆弱点。
2. 方法设计有现实意识：既考虑资源充足场景，也通过 FACTGUARD-D 处理冷启动和资源受限场景。
3. 对 LLM 的态度比较稳健：不是直接让 LLM 判真假，而是把 LLM 当作事件抽取器、常识解释器和可调权重的 advisor。
4. 跨语言实验有说服力：同时覆盖中文 Weibo21 和英文 GossipCop。
5. 消融较完整：验证了 topic-content、commonsense、usability evaluator 和蒸馏模块的作用。

## 9. 可能局限与批判性思考

1. LLM 抽取质量依赖 prompt 和具体模型。不同 LLM 抽取的 topic-content 可能差异较大，跨模型稳定性需要进一步验证。
2. commonsense rationale 来自既有工作，论文没有完全展开其生成质量、错误类型和人工校验情况。
3. FACTGUARD 完整版仍需要 LLM 参与预处理或推理，真实大规模平台部署时成本仍需评估。
4. 论文主要是文本假新闻检测，没有处理图像、视频、评论传播链等多模态与社交上下文信息。
5. 数据集存在潜在污染风险。作者在 future work 中也提到需要考虑 LLM 对 benchmark 数据的污染问题。
6. 可解释性仍有提升空间。Usability evaluator 能提升性能，但其权重如何对应具体推理依据，还需要更透明的分析。

## 10. 对 NLP 假新闻检测方向的启发

这篇论文对后续研究有几个值得借鉴的点：

- 从“风格检测”转向“事件语义检测”：把假新闻检测的重心从表面语言特征转移到事件事实与常识一致性。
- LLM 不一定作为最终分类器：LLM 更适合承担抽取、解释、补充知识等中间角色。
- 需要评估 LLM 输出的可用性：在事实核查任务中，LLM 信息不能无条件接入模型。
- 蒸馏是落地关键：如果方法依赖 LLM，最好同时设计无需 LLM 的轻量推理版本。
- 跨语言设计很重要：中文和英文假新闻可能有不同风格、平台和数据分布，需要分别建模或适配。

## 11. 可复现关注点

复现这篇论文时建议重点确认：

1. 论文中的 prompt 模板是否公开完整。
2. topic-content 抽取结果是否缓存，以及是否参与训练/测试泄漏。
3. commonsense rationale 的来源、格式、标签对齐方式。
4. 时间划分和去重策略是否严格复现。
5. FACTGUARD-D 的 feature simulator 结构、蒸馏损失权重和 teacher feature 保存方式。
6. 不同 LLM 替换后性能是否稳定，例如 DeepSeek、GPT、Qwen、Llama 系模型之间的差异。
7. 对未知风格攻击、跨平台迁移、突发事件早期检测的鲁棒性是否进一步验证。

## 12. 可用于汇报的总结

FACTGUARD 的核心思想是用 LLM 把新闻从“文本怎么写”还原到“事件是什么”，再用常识推理判断事件内容是否存在冲突，同时通过可用性评估模块避免盲目信任 LLM。完整模型适合资源充足场景，蒸馏模型 FACTGUARD-D 适合低成本部署。实验表明，该方法在中英文假新闻数据集上都优于 LLM-only、SLM-only、LLM-SLM 和蒸馏类基线，是一篇比较典型的“LLM 辅助但不完全依赖 LLM”的假新闻检测工作。

