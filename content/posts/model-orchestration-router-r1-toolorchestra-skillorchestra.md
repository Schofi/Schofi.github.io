---
title: "模型调度怎么学：从 Router-R1、ToolOrchestra 到 SkillOrchestra"
date: 2026-11-06T19:10:00+08:00
draft: false
author: "xiangxin"
description: "用完整执行案例理解三个模型与工具调度项目：PPO 和 GRPO 更新什么，奖励从哪里来，工具描述如何参与泛化，技能手册如何替代参数训练，以及怎样迁移到 Simulink 开发。"
summary: "从“让模型找其他模型帮忙”出发，区分执行循环与训练循环、题目与在线轨迹、专家能力与调度知识。保留奖励算分、数据与环境、技能画像案例、源码差异、Simulink 迁移方案，以及学习中修正过的常见误解。"
tags: ["Model Routing", "Agentic RL", "PPO", "GRPO", "Tool Use", "Skills", "Simulink"]
ShowToc: true
TocOpen: false
isCJKLanguage: true
cover:
  image: "/images/posts/model-orchestration-learning-map.png"
  hidden: true
---

给一个系统这样的任务：

> 查出两项参数，计算它们的差值，说明结论和依据。优先控制费用。

系统可以一直使用最强模型，也可以先搜索，再交给程序计算，最后请一个较便宜的模型组织答案。两个流程都可能答对，但费用、耗时和可靠性未必一样。

调度研究关心的就是这类选择：当前应该做什么，交给谁，传入什么信息，什么时候继续，什么时候停止。

今天依次学习了 Router-R1、ToolOrchestra 和 SkillOrchestra。最初看上去，它们都在做“让模型选择模型或工具”。随着问题展开，真正需要分清的是：**执行经验究竟改变了什么？是调度模型参数，还是一份外部的技能手册？**

本文把学习过程整理成一条完整路线。除明确标注来源的数据规模与配置外，文中的算分数字、产品参数和 Simulink 案例均为教学示例；不报告新的训练结果，也不代表三个项目已经验证过 Simulink 开发。

源码核对基于以下本地快照。文中同时标注论文设计、源码行为和迁移建议，三者不能混用。

| 项目 | 本次核对的提交 |
|---|---|
| Router-R1 | `801a240e37701577907c32de27a548af4e6c4430` |
| ToolOrchestra | `c1c0a4002be128b18accb267aac5ffdc26a28103` |
| SkillOrchestra | `be0603bc29f4656a1c301c2a5d78dc6f1d7a5bd4` |

## 1. 先把执行、学习和专家能力分开

一次任务的执行循环可以写成：

```text
任务、工具说明、当前历史
  → 调度器生成下一步动作
  → 程序执行工具或请求专家
  → 返回观察
  → 调度器继续决策
  → 结束任务
```

工具调用是模型生成的一段请求。真正的网络访问、程序运行、数据库修改或仿真，由执行器完成。模型说“已经完成”也不等于任务通过了验证。

训练则是外面的另一层循环：

```text
取一批任务
  → 当前策略实际执行，产生轨迹
  → 评分器计算结果与开销
  → 计算学习信号
  → 更新模型参数，或更新外部手册
  → 继续采集和验证
```

接收一次工具结果，只是在增加上下文。执行优化器更新，才会改变模型权重。

| 对象 | 负责什么 | 是否因为一次调用自动变强 |
|---|---|---|
| 调度器 | 决定做什么、交给谁 | 不会，取决于是否执行学习更新 |
| 专家模型 | 解决被分配的子任务 | 通常不会，项目主要学习调度 |
| 执行器 | 真实运行工具 | 不会，它是程序 |
| 验证器 | 判断结果是否满足要求 | 不会，它提供反馈 |
| 外部手册 | 保存能力统计与执行经验 | 可以通过独立的学习管线更新 |

三个项目最重要的区别，可以先记成下面这张表：

| 项目 | 主要执行对象 | 学习结果主要保存在什么地方 |
|---|---|---|
| Router-R1 | 候选大模型，多轮问答 | PPO 更新后的调度模型参数 |
| ToolOrchestra | 模型、搜索、代码与业务工具 | GRPO 更新后的调度模型参数 |
| SkillOrchestra | 模型或带工具的专家 Agent | 技能定义、专家画像和执行经验手册 |

[![三种调度方法对照图。Router-R1 调度候选大模型，经多轮轨迹评分用 PPO 和 GAE 更新调度模型。ToolOrchestra 调度模型与工具，经同题轨迹比较用 GRPO 更新调度模型。SkillOrchestra 按技能需求选择专家，从探索轨迹归纳技能、统计专家表现，并通过验证选定外部手册。](/images/posts/model-orchestration-learning-map.png)](/images/posts/model-orchestration-learning-map.png)

这不是一条“后者完全替代前者”的升级路线。它们展示了同一类问题的不同建模和学习方式。

## 2. Router-R1：把模型调用变成可学习的多轮动作

### 2.1 它执行的不是一次固定分派

假设问题是“某本书的作者出生在哪座城镇”。一次示意执行可以是：

```text
原问题
  → 选择模型 A，询问作者是谁
  → 得到作者姓名
  → 选择模型 B，询问该作者出生地
  → 整合已有信息
  → 输出最终答案
```

第二次调用的问题，是在看到第一次回复之后决定的。调度器也可以直接回答，或把完整问题一次交给专家，并不要求每题必须使用多个模型。

本地调用协议用标签表达：

```xml
<search>候选模型名称:具体子问题</search>
<information>专家回复</information>
<answer>最终短答案</answer>
```

这里的 `search` 是模型请求标签，不代表搜索引擎。原始提示词还要求生成 `think` 分析块；上面只展示动作和回包协议。

Actor 没有额外的“六分类路由头”。模型名称、子问题和最终答案都由语言模型生成，再由程序解析。当前上下文中的任务、模型描述和已有回复，共同影响下一次生成。

对应源码：[提示词](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/data_process/prompt_pool.py)、[模型请求服务](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/router_r1/llm_agent/route_service.py)。

### 2.2 数据分为“题目”和“在线轨迹”

训练脚本准备的是问题及标准答案。默认从 `RUC-NLPIR/FlashRAG_datasets` 读取 NQ 和 HotpotQA，各采样 7,000 条，共 14,000 条。

一条样本的核心内容可以简化为：

```json
{
  "prompt": [{"role": "user", "content": "调用规则、模型描述和问题"}],
  "reward_model": {
    "style": "rule",
    "ground_truth": {"target": ["标准答案"]}
  }
}
```

`reward_model` 在这里是字段名，`style: rule` 表示规则评分。不能仅凭这个名字就认定训练了一个奖励神经网络。标准答案交给评分器，不放入 Actor 的问题上下文。

数据也不要求预先标注“第一步必须找 A，第二步必须问 B”。这些调用轨迹由当前策略在训练中采样产生，专家 API 会被真实请求。失败轨迹同样提供学习信息。

数据入口：[qa_train_merge.py](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/data_process/qa_train_merge.py)。

### 2.3 环境是模型 API、协议和上下文管理

环境接收动作，映射模型名称，发送专家请求，再把回复插入上下文。专家主要收到本次子问题及固定指令，不会自动继承调度器的全部历史；必要背景要写进请求。

```text
Actor 生成调用
  → 解析名称与子问题
  → 请求专家 API
  → 包装成 information 观察
  → 更新历史与活动样本标记
  → 再次生成
```

`max_turns=4` 控制主交互循环。主循环后，对尚未结束的轨迹还有一次关闭真实专家调用的额外生成机会，因此不能简单把它说成“最多生成四次”。

多轮执行入口：[generation.py](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/router_r1/llm_agent/generation.py)。

### 2.4 奖励先保证合法与正确，再考虑成本

以本地默认 EM 分支为例，令 $C\in[0,1]$ 为成本收益，越便宜通常越高；$\alpha$ 为 `cost_coe`。主要分支可概括为：

$$
R(\tau)=
\begin{cases}
-1, & \text{格式不合法},\\
0, & \text{格式合法但答错},\\
(1-\alpha)+\alpha C, & \text{格式合法且答对}.
\end{cases}
$$

格式检查包括标签、最终答案数量、模型名称与子问题等。EM 会做文本归一化后进行完全匹配，代码另支持词重叠 F1。

这个条件顺序很重要：随便答错的便宜轨迹，不会因为低成本就得到正奖励。下面的分数仅用于演示：

| 轨迹 | 结果 | 成本收益 | 当 $\alpha=0.3$ 时的奖励 |
|---|---|---:|---:|
| A | 格式正确、答对 | 0.9 | 0.97 |
| B | 格式正确、答对 | 0.2 | 0.76 |
| C | 格式正确、答错 | 1.0 | 0 |
| D | 格式错误 | — | -1 |

当前 `train.sh` 的 `cost_coe=0.0`。因此默认任务奖励主要是合法答对得 1、合法答错得 0、格式错误得 -1，不能把它描述成默认就启用了成本优化。

成本也有实现边界：路由服务使用专家输出 Token 数乘配置单价；没有覆盖输入费用、调度器自身开销和延迟，单价按百万 Token 配置时，相乘处没有除以百万。它随后经过平方根变换与滑动窗口分位数归一化，是相对成本信号，不是完整账单。

源码：[格式与成本处理](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/verl/trainer/main_ppo.py)、[答案与组合奖励](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/verl/utils/reward_score/qa_em.py)、[默认训练配置](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/train.sh)。

### 2.5 PPO 更新的是调度模型生成动作的概率

一次训练迭代包括轨迹采样、任务评分、参考概率与价值计算、GAE、Critic 更新和 Actor 更新。

GAE 使用：

$$
\delta_t=r_t+\gamma V(s_{t+1})-V(s_t),\qquad
A_t=\delta_t+\gamma\lambda A_{t+1}.
$$

这里的 $t$ 是训练序列中的 Token 位置，不是专家调用轮次。Actor 的 PPO 概率比为：

$$
\rho_t=\frac{\pi_\theta(y_t\mid s_t)}{\pi_{\mathrm{old}}(y_t\mid s_t)}.
$$

核心策略损失仍然是标准裁剪形式：

$$
L_{\mathrm{policy}}=-\mathbb E_t\left[
\min\left(\rho_t A_t,
\operatorname{clip}(\rho_t,1-\epsilon,1+\epsilon)A_t\right)
\right].
$$

优势为正时，训练倾向于增加相关动作概率；优势为负时倾向于降低，裁剪控制更新幅度。Old Policy 用于新旧概率比，Reference 用于 KL 约束，两者不是同一个角色。

默认配置使用 PPO + GAE，裁剪系数 0.2，KL 系数与熵系数均为 0.001。它没有提出新的 PPO 概率比、Clipping 或优势估计公式。

对应实现：[训练主循环](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/verl/trainer/ppo/ray_trainer.py)、[核心算法](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/verl/trainer/ppo/core_algos.py)、[基础配置](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/verl/trainer/config/ppo_trainer.yaml)。

### 2.6 专家回复可被读取，但不作为 Actor 自己的动作

```text
Actor 调用请求      专家回复        Actor 下一步生成
参与策略损失       不参与          参与策略损失
```

`state_masking=true` 时，专家回复通过 `info_mask` 从 Actor 策略损失中排除，默认 KL 奖励计算也使用该遮罩。它仍然是后续生成能够读取的上下文。

这个结论不能扩大成“专家 Token 在所有计算中都删除了”：当前 GAE 与 Critic 路径仍使用普通有效 Token 遮罩。

实现位置：[信息块遮罩](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/router_r1/llm_agent/generation.py)、[Actor 损失](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/verl/workers/actor/dp_actor.py)。

### 2.7 数据范围决定结论范围

论文在 NQ 和 HotpotQA 上训练，在七个问答集上评测。NQ、HotpotQA 属于训练中使用过的数据集，评测样本另取；其他五个检验域外表现。

| 评测集 | 主要任务 |
|---|---|
| NQ、TriviaQA、PopQA | 一般知识与事实问答 |
| HotpotQA、2WikiMultiHopQA、MuSiQue、Bamboogle | 多跳问答 |

论文每集抽取 500 条，Bamboogle 约 120 条。仓库的 `qa_test_merge.py` 默认每个来源抽 100 条，`qa_test_gen.py` 对指定数据集抽最多 500 条，不能混淆脚本用途和论文实验规模。[论文实验设置](https://arxiv.org/html/2506.09033v1#S4)、[验证集脚本](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/data_process/qa_test_merge.py)、[独立评测脚本](https://github.com/ulab-uiuc/Router-R1/blob/801a240e37701577907c32de27a548af4e6c4430/data_process/qa_test_gen.py)。

Router-R1 的方法价值在多轮路由与信息整合的建模，以及相应的奖励与交互设计。它也没有精确标注每次调用的因果贡献：某次多余调用，可能跟随成功轨迹一起得到强化。

## 3. ToolOrchestra：把模型与程序一起纳入资源分配

### 3.1 先纠正一个容易夸大的差别

“Router-R1 只训练单步，ToolOrchestra 才训练整个执行过程”是不准确的。两者都可以用完整轨迹结果影响多个调度决策。

ToolOrchestra 扩大了执行对象与任务环境，并显式纳入延迟和工具偏好。它是一个研究方法与实现；Orchestrator-8B 是用该方法得到的模型，不能仅凭基准效果就认定项目已广泛用于生产。

它也不是发明了新的策略梯度优化器。论文采用 GRPO，和 Router-R1 默认 PPO + GAE 的主要区别包括优势来源，而不是因此自动获得更多工具能力。[ToolOrchestra 方法](https://arxiv.org/html/2511.21689#S3)。

### 3.2 一次问答可以怎样执行

本地部分问答配置提供三类高层工具：

| 工具 | 在这份配置中的用途 |
|---|---|
| `search` | 选择模型辅助获取缺失信息 |
| `enhance_reasoning` | 选择模型生成辅助代码并执行 |
| `answer` | 选择模型根据已有信息作答 |

例如工具调用可能只含模型别名：

```json
{"name": "enhance_reasoning", "arguments": {"model": "reasoner-3"}}
```

执行器会拼接原问题、已有资料和代码结果。HLE 的代码辅助分支会请求模型写出 Python，再运行它。因此，调度器不一定亲自写每段代码；最终作答也可以委托专家。

```text
任务与用户偏好
  → 选择搜索模型
  → 获得资料
  → 选择代码模型，执行计算
  → 获得结果
  → 选择回答模型
  → 结束
```

实际路径依反馈而变，并非固定流水线。上述三个封装也不等于项目支持的所有工具类型。

对应源码：[工具定义](https://github.com/NVlabs/ToolOrchestra/blob/c1c0a4002be128b18accb267aac5ffdc26a28103/training/tools.json)、[HLE 执行器](https://github.com/NVlabs/ToolOrchestra/blob/c1c0a4002be128b18accb267aac5ffdc26a28103/evaluation/eval_hle.py)、[模型服务适配](https://github.com/NVlabs/ToolOrchestra/blob/c1c0a4002be128b18accb267aac5ffdc26a28103/LLM_CALL.py)。

### 3.3 奖励来自验证器与执行统计

任务成功判断有两类典型来源：问答与参考答案比较，或检查业务环境的实际状态。本地问答分支先检查简单匹配，不直接匹配时可请求 GPT-5 裁判。业务任务不能只看结束语，还要检查操作效果。

按照论文的奖励设计，轨迹特征可写为：

$$
M(\tau)=[n_1,\ldots,n_k,\mathrm{success}(\tau),-\mathrm{cost}(\tau),-\mathrm{latency}(\tau)].
$$

各维归一化后，与偏好权重 $P$ 组合：

$$
R(\tau)=
\begin{cases}
\widehat M(\tau)\cdot P,&\text{任务成功},\\
0,&\text{任务失败}.
\end{cases}
$$

没有差异的维度不贡献分数。自然语言偏好进入模型输入，对应权重用于评分。描述提供选择依据，奖励反馈执行是否值得强化，没有单独的“读懂描述加一分”。

下面按论文规则构造一组算分例子，暂不考虑特定工具偏好：

| 轨迹 | 结果 | 成本，示意单位 | 延迟 | 正确性分 | 低成本分 | 低延迟分 |
|---|---|---:|---:|---:|---:|---:|
| A | 成功 | 0.01 | 10 秒 | 1 | 1 | 1 |
| B | 成功 | 0.05 | 50 秒 | 1 | 0 | 0 |
| C | 失败 | 0.01 | 10 秒 | 0 | 1 | 1 |

取正确性、低成本、低延迟权重为 0.5、0.3、0.2，则 A 得 1，B 得 0.5，C 因失败门控得 0。权重和为 1 是本例的选择，不是论文的一般必要条件。

这里的“低成本分”为何越便宜越高？它归一化的是负成本。例如成本从 0.01 到 0.05，负成本从 -0.05 到 -0.01，0.01 对应的 -0.01 数值更大。

来源：[论文奖励设计](https://arxiv.org/html/2511.21689#S3.SS2)、[本地评分与轨迹统计](https://github.com/NVlabs/ToolOrchestra/blob/c1c0a4002be128b18accb267aac5ffdc26a28103/training/lead_agent/llm_agent/generation_quick3.py)。

### 3.4 “训练整个执行策略”具体是什么意思

同一任务采样多条轨迹，用组内比较得到优势：

$$
A_i=\frac{R_i-\operatorname{mean}(R)}{\operatorname{std}(R)+\varepsilon}.
$$

上例得分为 1、0.5、0，均值为 0.5。因此 A 的优势为正，B 约为零，C 为负。失败的原始奖励可以是 0，但减去组内基线后，优势仍可以为负。

对一条被保留的轨迹，各轮记录可以组织为：

| 本轮输入 | 当时生成的动作 | 训练依据 |
|---|---|---|
| 原任务与工具说明 | 选择搜索模型 | 这条轨迹的相对优势 |
| 任务与搜索结果 | 选择计算能力 | 同一轨迹的相对优势 |
| 任务、资料与计算结果 | 选择回答模型 | 同一轨迹的相对优势 |

本地代码确实按任务与轨迹编号分组，将归一化信号分配到各轮生成，再经格式和有效输出等筛选送入训练。它不是精确算出了“第二次调用贡献 70%”。

重要的是每轮输入不同：模型学习的是“在什么状态下选什么”，不是把一条成功流程永久写死。核心策略更新仍采用裁剪目标。

源码入口：[多轮生成与分组信号](https://github.com/NVlabs/ToolOrchestra/blob/c1c0a4002be128b18accb267aac5ffdc26a28103/training/lead_agent/llm_agent/generation_quick3.py)、[训练接入](https://github.com/NVlabs/ToolOrchestra/blob/c1c0a4002be128b18accb267aac5ffdc26a28103/training/recipe/algo/grpo_ray_trainer_quick3.py)、[RewardManager](https://github.com/NVlabs/ToolOrchestra/blob/c1c0a4002be128b18accb267aac5ffdc26a28103/training/recipe/algo/main_grpo_quick3.py)。

### 3.5 源码不能直接当成论文公式的逐字实现

当前快照中，前一段效率奖励计算后，奖励字典又被重置并进入偏好重算。因此，只读前面的“正确性减成本与延迟”分支，会误解最终路径。

静态阅读还发现：成本和延迟的范围统计使用负值，后续一些加权表达式使用正值，方向与论文公式存在差异。若仅看这类表达式，成本范围是负值、分子却使用正成本，会产生昂贵轨迹分数反而增加的问题。相关维度还经过多处累加，需要端到端核对。

这些是**指定源码快照的静态检查结论**，本次没有修改代码、复跑训练，也没有验证其对最终指标的影响。上面的教学算分使用论文预期规则，不能当作本地程序的实测输出。

复现时应先为最终奖励路径做数值核验，再开始昂贵的训练。[相关代码段](https://github.com/NVlabs/ToolOrchestra/blob/c1c0a4002be128b18accb267aac5ffdc26a28103/training/lead_agent/llm_agent/generation_quick3.py#L1390)。

### 3.6 可验证环境也是训练数据的一部分

ToolScale 合成流程不只生成用户问题，还构造数据库、工具 API、参考操作和必要过程信息。它让任务能够通过执行结果验收。官方评测主要包括 HLE、FRAMES 和 τ²-Bench；这些结果不能直接证明通用软件开发或 Simulink 开发能力。[ToolOrchestra 官方说明](https://research.nvidia.com/labs/lpr/ToolOrchestra/)

从方法贡献看，值得学习的是偏好相关的多目标训练、可验证环境合成，以及变化工具和价格配置。把 PPO 换成 GRPO 本身不是项目的原创算法贡献。

## 4. 工具描述驱动的调度，两个项目都能做

### 4.1 它与普通 tool-use 后训练相通

一个工具可以描述为“计算两项数据的差值”，一个模型专家可以描述为“擅长数学推理”。对调度器而言，都是基于任务、历史和说明生成调用动作。

因此，“读取描述后使用训练时未见的候选”不是 Router-R1 或 ToolOrchestra 独有的能力。更早的 ToolLLM 已研究工具微调与未见 API 泛化。[ToolLLM 原论文](https://arxiv.org/abs/2307.16789)

Router-R1 本身已有模型描述；ToolOrchestra 还变化训练中的工具子集和价格配置。因此，不能把“想研究按描述选择”当作必须放弃 Router-R1 的理由。

### 4.2 接入新工具不一定要重新训练

新工具接入需要名称、描述、参数、执行器、调用分发和反馈格式。完成这些后，可以先测试已有模型。

| 观察到的问题 | 优先处理 |
|---|---|
| 参数格式经常错误 | 接口、描述、示例 |
| 不能正确解释返回结果 | 反馈结构与领域知识 |
| 典型流程不熟练 | 可考虑收集可靠轨迹做 SFT |
| 能完成但调用策略差、开销高 | 有可靠评分后再考虑 RL |
| 结果是否正确都无法确定 | 先完善验证器 |

更换同类型专家与加入全新软件环境不是同等难度。论文中新增候选模型的泛化证据，也不能自动推广成“任意新工具零训练即可熟练使用”。能力描述是先验，不是对真实可靠性的证明。

### 4.3 怎样证明模型真的在读描述

可以额外设计以下对照实验：

| 实验 | 检查什么 |
|---|---|
| 保留描述，将名称匿名化 | 是否依赖知名模型身份 |
| 保留名称，移除能力描述 | 描述的增量价值 |
| 改变名称与执行器绑定，同步更新真实描述 | 是否能跟随当前说明选择 |
| 提供未见候选及真实描述 | 新候选泛化 |
| 在相同预算下比较 SFT、RL 和提示策略 | 训练方法的增量价值 |

例如第一批 `tool_A` 修改模型、`tool_B` 执行仿真；第二批互换身份，并同步更改描述和执行绑定。如果模型仍能正确安排操作，才更支持描述依赖。

不能只改文字却让描述与真实功能相矛盾，也不能仅靠合法 JSON 判断任务成功。“未参与本次路由训练”还不等于“底座预训练完全没见过这个名称”。

## 5. SkillOrchestra：让执行经验变成可以查询和修改的手册

### 5.1 专家没有变强，为什么总体可以变好

先看一组假设数据：20 个任务，其中延时逻辑与采样时间诊断各 10 个。

| 专家 | 延时逻辑 | 采样时间诊断 | 总体通过率 |
|---|---:|---:|---:|
| A | 9/10 | 3/10 | 60% |
| B | 3/10 | 9/10 | 60% |

只看总榜，两个专家相同。如果当前技能识别完全正确，延时交给 A、诊断交给 B，则可以完成 18/20，即 90%。这是理想化教学计算，不是项目实验指标。

专家能力没有改变，改变的是任务分配。SkillOrchestra 要系统化发现和利用这种细分差异。

### 5.2 手册包含什么

| 内容 | 回答的问题 |
|---|---|
| 模式级执行经验 | 当前应该搜索、编码还是执行其他操作？ |
| 技能定义与适用线索 | 当前需要哪项具体能力？ |
| 专家画像 | 谁擅长、成本多少、有什么常见限制？ |

这里的技能是一种能力分类，可以由描述和适用线索表示。它不一定是可执行程序，也不等同于一个 `SKILL.md` 文件。

专家仍由系统接入。论文中的 Agent 可以是“模型 + 可用工具”的组合。技能手册不会自动训练出原本不存在的专家能力。

### 5.3 离线流程：探索、归纳、统计、调整和验证

```text
让不同专家执行任务，收集成败与成本
  → LLM 比较成功和失败轨迹，归纳技能
  → 程序更新技能相关的能力与成本统计
  → 提议拆分、合并技能
  → 在验证任务上比较手册版本
  → 保存选定手册
```

因此，“通过 LLM 总结手册”只说对了一部分。LLM 负责归纳与反思，程序负责结构化存储、统计和评测，环境提供执行证据。自然语言总结不能代替真实结果。

官方入口分成 `explore → learn → select → test`，对应代码包括发现器、画像构建器、细化器、手册和候选评测模块。[官方管线](https://github.com/jiayuww/SkillOrchestra/blob/be0603bc29f4656a1c301c2a5d78dc6f1d7a5bd4/README.md)、[学习管线](https://github.com/jiayuww/SkillOrchestra/blob/be0603bc29f4656a1c301c2a5d78dc6f1d7a5bd4/skillorchestra/learning/pipeline.py)。

### 5.4 优化对象从参数变成手册

论文用技能相关能力与成本辅助选择，并通过成功、失败更新能力估计；技能粒度会继续细化或合并，手册还需要针对调度器容量做验证选择。[论文方法](https://arxiv.org/html/2602.19672#S4)

可以把手册选择目标写成：

$$
H^*=\operatorname*{argmax}_H\mathbb E_{q\in D_{\mathrm{val}}}
\left[R(\tau_H(q))-\lambda C(\tau_H(q))\right].
$$

这表达的是“哪份手册带来更好的完整任务表现与成本”，实际通过候选生成、评测和选择实现，不代表穷举所有手册或找到全局最优。

分得太粗，专家差异被掩盖；分得太细，小调度模型可能识别错技能。没有验证结果支持的细化，未必有益。

### 5.5 论文与当前源码的选择逻辑也要区分

论文给出技能能力加权减成本的选择表达，并讨论 Beta 后验均值。当前本地快照的 `get_competence()` 返回经验成功率；`weighted_competence()` 调用的是这个结果。

`SkillHandbook.select_agent()` 还采用分层比较：先比技能能力；平局再比类别能力、整体成功率；仍平局才按成本调整概率并抽样。因此，不能把这个函数解释成“所有候选都直接执行能力减成本”。

代码中某些注释或记录字段使用简化评分表述，也不能代替追踪实际选择路径。不同 routing strategy 还可能调用不同路径。

对应源码：[能力数据结构](https://github.com/jiayuww/SkillOrchestra/blob/be0603bc29f4656a1c301c2a5d78dc6f1d7a5bd4/skillorchestra/core/types.py)、[实际手册选择函数](https://github.com/jiayuww/SkillOrchestra/blob/be0603bc29f4656a1c301c2a5d78dc6f1d7a5bd4/skillorchestra/core/handbook.py)、[调度结果包装](https://github.com/jiayuww/SkillOrchestra/blob/be0603bc29f4656a1c301c2a5d78dc6f1d7a5bd4/skillorchestra/routing/orchestrator.py)。

### 5.6 在线流程：每一步重新判断当前需要什么

```text
收到新需求
  → 识别当前操作与技能
  → 查询专家画像
  → 选择专家执行
  → 读取真实反馈
  → 当前状态改变，重新选择
```

例如先由 A 修改延时逻辑，仿真却出现采样时间错误；下一步就可以转给 B 诊断，而不是整道任务始终绑定 A。以上仍是 Simulink 迁移示意。

构建手册与执行任务是两段过程，不是每遇到新需求都重新测一遍全部专家。部署期新增记录可以进入之后的更新流程，但单次结果不能直接认定为稳定能力。

### 5.7 不训练模型参数，也不是没有学习成本

| 对象 | 核心方法中是否更新参数 |
|---|---|
| 调度模型 | 不通过 PPO、GRPO 或 SFT 更新 |
| 专家模型 | 不更新 |
| 分析轨迹的 LLM | 调用已有模型，不更新 |
| 技能定义、能力统计和执行经验 | 会更新，但这是外部知识更新 |

它仍然要支付执行、LLM 归纳与验证的费用。只有同类任务反复出现、专家存在互补能力时，这些经验才更可能值得积累。专家都不会的任务，手册不能凭空解决。

从创新角度看，它的重点是显式技能中间层、经验驱动的画像与结构调整，以及调度器相关的粒度选择，不是新的策略梯度损失。

## 6. 迁移到 Simulink：先把开发闭环建起来

### 6.1 一个可以检查的最小任务

先限定以下需求，而不是直接要求开发整个控制系统：

```text
输入：布尔信号 enable
输出：布尔信号 active
行为：enable 连续为真达到 T 秒，active 置真；
      enable 变假，active 立即复位，计时清零。
初始状态：输出为假，计时为零。
示例参数：T=3 秒，离散采样时间 0.01 秒。
结构要求：只用约定的原生模块，不允许 MATLAB Function。
```

在实验前还要统一计时起点、采样点上的触发语义和时间容差，避免模型与测试对“达到 3 秒”的解释不同。

### 6.2 工具、状态与重置

可以设计下面这组接口。名称是建议，不是三个仓库已有的 Simulink 工具：

```text
inspect_model：读取模块、端口、参数和连接
edit_model：实施受控修改
compile_model：更新模型并检查错误
simulate_model：执行指定输入序列
verify_requirements：运行独立验收
```

每个 episode 从独立目录和确定的初始模型开始。状态同时包含模型文件、配置与历史结果，不只是聊天文本。外部 MATLAB 会话、工具版本和必要依赖也要明确。

Simulink 提供程序化建模与仿真接口，可以在其上封装执行器；可用 API 应按目标 MATLAB 版本核对。[模型编辑接口](https://www.mathworks.com/help/simulink/programmatic-modeling.html)、[程序化仿真](https://www.mathworks.com/help/simulink/ug/using-the-sim-command.html)

### 6.3 验证器要同时检查行为与交付结构

| 检查 | 例子 |
|---|---|
| 可执行性 | 模型可加载、更新、仿真 |
| 结构合规 | 接口、模块白名单、禁止模块 |
| 正常行为 | 持续输入达到阈值时触发 |
| 复位行为 | 输入中断后清零，重新计时 |
| 边界条件 | 阈值附近、中途抖动、不同参数 |
| 交付完整性 | 模型与必要依赖完整 |

模型运行成功不代表需求满足，行为测试通过也不代表结构约束合规。预期输出应来自独立规则或参考实现。双路径实现能减少同一实现错误自证正确，但无法消除共同的需求解释错误。

训练 Agent 不应能任意修改验收规则。可区分开发期反馈用例和保留的最终测试，以检验是否只适应少量公开输入。[行为评估与输出比较接口](https://www.mathworks.com/help/sltest/test-sequences-and-assessments.html)

### 6.4 三个项目可以分别迁移什么

| 当前目标 | 可以借用的部分 |
|---|---|
| 只想完成一两个需求 | 先使用现有模型与 Simulink 工具闭环 |
| 学习按描述选择不同专家 | Router-R1 的模型池与多轮策略即可作为起点 |
| 学习修改、仿真、修复过程中的成本与偏好调度 | ToolOrchestra 的环境和奖励组织 |
| 重复处理任务，积累细分专家优势 | SkillOrchestra 的手册与画像 |

若要做 RL，奖励可以先设置可执行性与结构硬门槛，再按行为结果评分；完整通过后再考虑效率。这是迁移设计建议，不是原项目已有配置。

若要做技能手册，可以先积累延时逻辑、滞回控制、采样时间诊断等少数任务族。划分数据时要隔离相近模板和参数变体；只把同一个需求换个数字放进测试集，不能充分说明泛化。

Simulink 执行通常比短问答复杂，应先验证小规模闭环，再决定是否训练调度器。先接工具不等于必须训练，已有模型可以作为基线。

## 7. 怎样判断创新与能力，而不被项目名称带偏

| 项目 | 主要方法贡献 | 不能由此推出 |
|---|---|---|
| Router-R1 | 多轮模型路由与整合建模、规则与成本奖励 | 提出了新的 PPO 优化器 |
| ToolOrchestra | 异构工具协调、偏好多目标训练、可验证环境合成 | 任意新软件都能直接熟练操作 |
| SkillOrchestra | 技能中间层、经验画像、手册粒度与验证选择 | 自动训练出新的专业能力 |

“功能范围更广”不等于“所有能力都更强”。公平比较还需要相同任务、专家池、预算和验收方式。把昂贵专家开放给系统，系统效果提高，并不能单独证明路由策略更好。

“工具调用和多轮 Agent 是常见范式”也不等于某个具体仓库已经广泛用于生产。公开论文、代码和基准结果是研究证据；生产采用需要另外的部署证据。本文不根据 Star 数或作者机构推断采用规模。

### 7.1 三种反馈各自不等于什么

```text
工具返回结果 ≠ 自动产生参数更新
轨迹得分较高 ≠ 每一次调用都有必要
LLM 归纳出技能 ≠ 该技能解释已被因果证明
```

改善调度还可能来自提示词、描述、工具封装和验证器。实验要将这些因素与参数训练、统计更新分别记录。

## 8. 把今天的追问整理成一张核对表

| 问题 | 更准确的理解 |
|---|---|
| Router-R1 也是 PPO 吗？ | 默认是 PPO + GAE，主要改变任务建模与奖励。 |
| 它只训练一步路由吗？ | 不是，完整多轮轨迹会影响多个生成决策。 |
| 训练前必须收集正确调用路径吗？ | 默认问答数据提供问题与答案，调用轨迹在线采样。 |
| 奖励都是 LLM 打分吗？ | 不是。可以来自答案规则、模型裁判、环境状态与执行统计。 |
| ToolOrchestra 为什么能训练整个过程？ | 完整执行后评分，再用轨迹反馈更新各轮调度行为。 |
| GRPO 是否天然比 PPO 更会用工具？ | 不会。工具接口、环境、数据和奖励决定可学习的行为。 |
| ToolOrchestra 完整包含 Router-R1 吗？ | 高层功能有大量重叠，但不是代码、接口和效果的严格超集。 |
| 按描述使用新候选是独有能力吗？ | 不是，普通工具学习、Router-R1、ToolOrchestra 都可研究。 |
| 新增工具一定要再训练吗？ | 不一定。先完成接入并测任务表现，再分析失败原因。 |
| 会写合法调用等于会完成任务吗？ | 不等于，还需要执行、解释结果和完成验证。 |
| SkillOrchestra 需要自己提供专家吗？ | 需要接入候选模型与工具；技能定义可从经验中归纳。 |
| 它只是提高效率吗？ | 合理分配也可能提高成功率，但不会改变专家本身能力。 |
| SkillOrchestra 是 LLM 写总结吗？ | LLM 归纳、程序统计、结构调整与实际验证共同组成方法。 |
| 不训练模型就没有成本吗？ | 仍有探索、调用、执行和验证成本。 |
| 系统越复杂越应该用技能手册吗？ | 更关键的是任务重复性、专家互补性和验证可靠性。 |
| 可以直接用于 Simulink 吗？ | 需构造执行器、状态管理、任务集和验证器，不能把 QA 结果直接外推。 |

## 9. 下一次阅读与实验从哪里开始

阅读时不必先钻进庞大的训练框架。先沿着一条轨迹，回答四个问题：当前输入是什么，动作是谁生成的，实际由谁执行，最终什么数据进入更新。

| 项目 | 建议阅读顺序 |
|---|---|
| Router-R1 | `infer_vllm.py` → `prompt_pool.py` → `route_service.py` → `generation.py` → 奖励函数 → PPO 主循环 |
| ToolOrchestra | `tools.json` → 实际工具执行器 → 多轮记录 → 最终奖励重算 → 各轮训练记录 → 策略更新 |
| SkillOrchestra | `scripts/pipeline.py` → `discoverer.py` → `profiler.py` → `refiner.py` → `handbook.py` → 验证选择 |

如果要做一个 Simulink 小实验，可以固定专家池、初始模型、任务集和独立验收，先比较三种分派方式：始终使用同一专家、仅靠描述选择、使用经验手册选择。记录成功率、费用、仿真次数与修复轮数。确认分派确实有可测价值以后，再决定是否值得投入 RL。

本文完成的是论文与源码学习整理，没有执行这些实验。后续结果必须单独报告模型版本、任务划分、预算、验证方式和真实运行记录。

## 原始资料与相关笔记

- [Router-R1 论文](https://arxiv.org/abs/2506.09033)与[官方仓库](https://github.com/ulab-uiuc/Router-R1)。
- [ToolOrchestra 论文](https://arxiv.org/abs/2511.21689)、[官方仓库](https://github.com/NVlabs/ToolOrchestra)与[项目主页](https://research.nvidia.com/labs/lpr/ToolOrchestra/)。
- [SkillOrchestra 论文](https://arxiv.org/abs/2602.19672)与[官方仓库](https://github.com/jiayuww/SkillOrchestra)。
- [ToolLLM 论文](https://arxiv.org/abs/2307.16789)：用于核对工具学习与未见 API 泛化的既有研究。
- [从工具调用到 Skill 自进化：用旅行 Agent 理解奖励、课程学习与联合 RL](/posts/tool-use-rl-skills-learning-notes/)：此前的工具执行、奖励和 Skill 学习笔记。
