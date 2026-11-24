# OpenCUA 论文学习笔记

论文是 Xinyuan Wang 等人的 *OpenCUA: Open Foundations for Computer-Use Agents*（2025，arXiv:2508.09123，NeurIPS 2025 Spotlight）。本笔记依据第一作者公开的 42 页预印本，章节、图表号和页码均对应这份 PDF。后续项目更新单独标明。 [仓库内 PDF](opencua-paper.pdf) · [作者发布的 PDF](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf) · [arXiv](https://arxiv.org/abs/2508.09123) · [作者项目页](https://opencua.xlang.ai/)

## 1. 论文想解决什么问题

给模型一句任务指令，让它看屏幕、操作鼠标键盘，再根据操作后的屏幕决定下一步。比如「在 Features 幻灯片里插入一张 5 行 2 列的表格」，模型要依次找到插入入口、设定行列、确认插入，并判断任务是否完成。这个循环同时考验**界面定位、操作规划、纠错和停止判断**。

作者问的是，怎样把日常电脑操作录成大量可用于训练的示范，并让模型随着示范增加而学到更强的操作能力？§3 给了一个反例。直接用约 27K 条轨迹的状态—动作对微调，基线在 OSWorld 上只有 **4.4%**。作者因此同时改进了动作表示、推理标注、历史上下文和训练数据配比。图 2 展示了录制工具、数据集、训练流水线、模型和评测怎样接在一起。 [原文 §1–3](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=2)

## 2. 数据拆解：人类录屏怎样变成训练样本

| 环节 | OpenCUA 的做法 | 学习时关注什么 |
| --- | --- | --- |
| 录制 | AgentNetTool 同步记录屏幕视频、键鼠事件和无障碍树（AXTree）。标注者填写任务目标，检查后提交。 | 视频回答「屏幕上有什么」，事件回答「人做了什么」，AXTree 提供控件结构和边界框等辅助信息。 |
| 动作归约 | 把连续按键合并成文本输入，把修饰键组合归为快捷键，把同向滚动量累加，合并双击和拖拽等手势。 | 动作粒度决定学习目标。模型可以学习一次有意义的点击，而无需模仿鼠标移过的每个像素。 |
| 状态—动作对齐 | 对点击，回溯到鼠标开始移动前，再找到最近一帧有视觉变化的画面。最后补上终止帧和终止动作。 | 贴着点击时间截帧，光标可能已经悬停在答案上，甚至触发了悬停提示。模型可能靠这些线索猜坐标。 |
| 反思标注 | 为动作补充观察、推理和自然语言动作描述，识别错误或冗余步骤。 | 人类轨迹也会走弯路。要保留错后状态来解释纠正动作，并避免把错误动作本身当作监督目标。 |

第 2 节公布的 **AgentNet 有 22,625 个人工标注任务**，平均 **18.6 步**，覆盖 Windows、macOS、Ubuntu，以及 140 多种应用和约 190 个网站。§3.1 的脚注另列实际训练规模，OpenCUA-7B 和 32B 使用了 **41,428 条**轨迹。理解实验时，要区分公开数据集的规模与具体模型使用的训练规模。工具采集 AXTree，部分 grounding 数据来自 AXTree 框标注，论文的在线 agent 则使用截图作为视觉观测。 [原文 §2–3](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=3) · [公开数据处理代码](https://github.com/xlang-ai/OpenCUA/tree/main/data/data-process)

## 3. 方法拆解：模型每一步学什么

把任务写成 `指令 → (截图₀, 动作₀) → (截图₁, 动作₁) → …`。每一轮以指令、当前截图和历史为条件，预测下一动作。运行时执行器把模型生成的动作落实成点击、输入或快捷键，再返回真实界面。预测下一状态的文字也可以帮助下一轮对照实际变化。 [原文 §2.1、§3.2](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=3)

### 反思式长 CoT（§3.1，图 5、图 6）

| 格式 | 生成内容 | 图 6 的表格任务示意 |
| --- | --- | --- |
| L1 | Action，再输出可执行动作 | 选中「行数」输入框中的 `1`。 |
| L2 | Thought + Action，再输出可执行动作 | 列数已正确，行数还是 `1`，目标是 `5`。先选中旧值，下一步替换。 |
| L3 | Observation + Thought + Action，再输出可执行动作 | 先描述对话框及其可见字段，再给出上述判断和动作。 |

训练标注由三个组件合成。**Reflector** 对比操作前后截图、动作代码和生成描述，判断上一步有没有生效、是否多余，并写出解释。**Generator** 结合目标、历史、反思、截图和示范动作生成各层文本。对于坐标动作，标注阶段会提供目标点标记和放大图块。**Summarizer** 把模糊目标改写得更具体，并按目标一致性、效率、难度给轨迹打分。论文用 Claude 3.7 Sonnet 合成这三类标注。生成标签时能看到示范动作及事后反馈，实际执行时则由模型根据可见历史自己决定下一步。 [原文 §3.1 与图 6](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=5) · [合成代码](https://github.com/xlang-ai/OpenCUA/tree/main/data/cot-generate)

### 历史、输出和数据配比是三个独立选择（§3.2–3.3）

- **历史输入**用 L1 的简短动作描述和总计 3 张截图。文字保留操作脉络，截图让模型看到界面变化。第 5 节的消融中，从 1 张增至 3 张收益明显，从 3 张增至 5 张只再带来小幅收益，且多消耗约 3K 上下文 token。
- **当前输出**在训练时混合 L1/L2/L3，在线推理时默认 L2。刚做了什么可以用短历史记录，下一步为什么这么做则值得用一段简洁的反思来判断。
- **监督微调（SFT）**混合三类材料。Grounding 样本教模型定位元素，任务轨迹教它选择动作和推进目标，通用图文样本覆盖指令理解、OCR、视觉问答等。论文试了仅第二阶段、两阶段和联合训练。32B 先强化定位和通用理解，再训练操作规划。7B 联合训练的配比是 20% 规划、20% grounding、60% 通用数据。两者得分差包含模型容量、训练配方等多种影响。

[原文 §3.2–3.3](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=6) · [附录 G 的 L1/L2/L3 样本](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=33)

## 4. 实验拆解：每个分数在测什么

| 评测方式 | 看什么 | 初版结果示例 |
| --- | --- | --- |
| GUI grounding | 给定目标描述时，模型能否在截图上找到目标位置。 | 32B 的 ScreenSpot-Pro 为 55.3%，基础 Qwen2.5-VL-32B 为 39.4%。 |
| AgentNetBench 离线评测 | 给定已录好的截图和历史，下一步动作是否符合一个标注的有效选择。覆盖 Windows、macOS 上的 100 个留出任务。 | 32B 的总体单步分数为 79.1%，其中坐标类 81.9%，终止类 55.7%。 |
| OSWorld-Verified 在线评测 | 在真实交互环境里逐步执行，最终状态是否满足任务判定标准。 | 32B 在 15、50、100 步预算下分别为 29.7%、34.1%、34.8%，同表 OpenAI CUA 在 100 步下为 31.4%。 |

**79.1% 和 34.8% 可以同时成立。** 离线评测让不同模型面对相同轨迹的相同步骤，便于快速诊断下一步选得如何。在线执行时，一个错误就可能把模型带进示范中没见过的页面，接下来还要自己恢复，并在正确时机结束。离线评测为单步标注多个有效动作，能容纳部分合理差异，但覆盖不了所有替代路线。 [原文 §4、表 3–4、图 7、附录 B](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=8)

初版摘要的 **34.8%** 属于 OpenCUA-32B。项目于 2025 年 10 月发布 OpenCUA-72B，官方仓库报告其 100 步预算下的平均成功率为 **45.0%**。引用时带上模型、评测版本、步数预算和运行口径。初版表 3 中 Claude 4 Sonnet 的 43.9% 位于 **50 步**列，100 步为 41.5%。 [初版表 3](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=9) · [后续模型与成绩](https://github.com/xlang-ai/OpenCUA#performance)

## 5. 怎样读消融、扩展性和失败案例

| 结果 | 能从中学到什么 |
| --- | --- |
| 短 CoT **11.5%** → 反思式长 CoT **15.3%**（表 5） | 在该消融设置下，整套反思式标注带来 **3.8 个百分点**。作者推测一部分收益来自自我纠错。要验证这个机制，可进一步统计错误发生后的重新定位和成功恢复率。 |
| 只用 L2 训练 **13.1%**，混合格式 **18.5%**。推理格式 L1 / L2 / L3 分别是 **16.9% / 18.5% / 17.6%**（表 5） | 训练时可以提供不同层次的监督。推理时，L2 在这组 15 步设置里表现最好。作者观察到 L3 有时会把无关界面元素写进描述。 |
| 从 7K Ubuntu 到 10K Ubuntu + 17K Win/macOS，OSWorld 成功率 **9.8% → 18.5%**（§4.2，图 1） | 跨系统数据混合有收益。这个对比同时增加了 Ubuntu 和 Win/macOS 数据。若想看跨系统迁移，也要读图 8 的分来源曲线。图中同平台数据通常更匹配相应基准。 |
| 32B 在 50 步预算的平均单次成功率约 **34.2%**，**Pass@3 约 45.6%**（附录表 10） | Pass@3 按任务统计三次尝试中是否至少成功一次，依赖评测器知道每次是否成功。它反映多尝试的潜力。实际部署还要解决重试、状态重置和结果选择。 |

增加**单次最大步数**和增加**独立尝试次数**是两种不同的测试时计算投入。前者给同一条轨迹更多机会，模型也可能把额外步数花在循环上。后者探索不同解法，甚至温度为 0 时也可能受网页延迟、系统时间等环境扰动影响。论文 §5 的 Pass@16 实验使用 **OpenCUA-Qwen2-7B**，与主表里的 **OpenCUA-7B** 有不同的基础模型和配方。 [原文 §4.2、§5、附录 D](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=11)

附录 E 把失败分成应用知识不足、像素级定位错误、动作重复、终止误判、长任务上下文丢失、错误感知与恢复不足。想象模型需要把 `H2O` 中的 `2` 改成下标，知道该选字属于规划，只选中 `2` 属于精细定位，检查是否改错字符属于反馈与纠错。附录 F 给出一次恢复成功的完整轨迹。把它和附录 E 对读，可以看清个案展示与整体失败分布分别提供了什么证据。 [原文附录 E–F](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=28)

复现时还要考虑两项约束。真实人类示范需要人工成本，知情同意和隐私筛查会影响样本代表性。作者公开了录制、处理、评测和模型等材料，但官方仓库仍把完整训练代码列为待办，且大模型使用的轨迹规模大于公开 AgentNet。可以用公开样本验证方法环节，原训练结果的完整复现还依赖更多资产。 [原文附录 A、C.3.2](https://github.com/XinyuanWangCS/OpenCUA-Website/blob/8b6d576bcc9772cfee7518caaea643ba3f4de35b/static/pdf/OpenCUA_arxiv.pdf#page=21) · [官方仓库](https://github.com/xlang-ai/OpenCUA#todo)

## 6. 一小时学习路线

1. **10 分钟读图 2 和 §2。** 画出「录制 → 归约 → 截图对齐 → 合成推理 → SFT → 在线执行」，标注每处用到的视频、键鼠事件、AXTree 或文本。
2. **20 分钟读图 5、图 6 和附录 G。** 用同一个步骤分别写 L1、L2、L3。假设上一步点错，写下当前画面如何告诉你点错了，以及接下来哪一步能把任务拉回正轨。
3. **20 分钟读表 3–5、图 8–12。** 自己算一遍 32B 从 15 到 50 步的 **+4.4 个百分点**，再解释为什么 50 步 Pass@3 的 45.6% 与 50 步平均单次成功率不同。看每组消融时写下模型、改动项、评测指标和预算。
4. **10 分钟看一条公开示范。** 在 AgentNetBench 示例中任选一个任务，根据相邻两张截图写出预期状态和实际状态。为其中一步写出第二种合理操作，再判断离线标注能否给它分数、在线环境会怎样验证最终结果。 [示例轨迹和截图](https://github.com/xlang-ai/OpenCUA/tree/main/evaluation/agentnetbench/sample_data)

读完可以用三个问题自测。

- 为什么对齐训练样本时要回溯到鼠标开始移动之前？
- 为什么语言层面能选对动作，执行后仍可能失败？分别用 grounding、历史和终止判断举例。
- 如果要测跨操作系统数据本身的贡献，如何对齐基础模型、总样本量、训练 token 预算与评测设置？
