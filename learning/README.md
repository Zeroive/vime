# Vime 项目学习导航

这是一套面向当前仓库的中文学习资料。目标是：能解释训练闭环、跟踪一条样本、理解关键配置，并通过扩展接口完成一个小改动。

资料依据：当前工作区代码及仓库内中文文档；整理日期：2026-10-11。代码更新后，以实际函数与参数为准。文档中的练习是待执行任务，不代表已经完成运行验证。

## 从哪里开始

1. 阅读 [项目地图](01-project-map.md)，建立模块与数据流的整体印象。
2. 按 [学习路线](02-study-plan.md) 安排时间，每阶段完成一个可检查的产出。
3. 阅读代码时使用 [源码阅读指南](03-code-reading.md)，沿调用链追踪，不必逐文件通读。
4. 用 [实践与调试](04-practice.md) 检验理解；没有 GPU 也能完成前半段。
5. 把问题、证据和实验写入 [学习记录](05-study-notes.md)。
6. 准备数据时查阅 [训练数据格式](06-data-format.md)，区分 RL、SFT 与多模态样本。

默认按“具备基础 Python 能力，但不熟悉大模型 RL 工程”安排，每天 60–90 分钟，约四周。时间安排只是参考；按阶段验收决定是否继续。若不熟悉 PyTorch、async/await 或概率基础，先补学习路线中的预备知识。

## 第一小时

- 15 分钟：阅读 [仓库介绍](../README_zh.md) 的架构总览和代码阅读路径。
- 20 分钟：阅读 [train.py](../train.py)，标记初始化、生成、训练、同步、评估与保存。
- 15 分钟：找到 [Sample](../vime/utils/types.py) 的定义，认识 `tokens`、`response_length`、`reward`、`loss_mask`。
- 10 分钟：在学习记录中用自己的话写出“一条 prompt 如何影响模型参数”。

## 原始文档入口

| 想解决的问题 | 入口 |
| --- | --- |
| 配置环境、准备权重和数据 | [快速开始](../docs/zh/get_started/quick_start.md) |
| 了解参数和运行模式 | [使用说明](../docs/zh/get_started/usage.md) |
| 加入自己的生成或奖励逻辑 | [自定义指南](../docs/zh/get_started/customization.md) |
| 分离推理与训练排查问题 | [调试指南](../docs/zh/developer_guide/debug.md) |
| 理解测试的覆盖范围 | [CI 说明](../docs/zh/developer_guide/ci.md) |
| 学习 agent 工作流 | [Agentic RL 路线](../docs/zh/get_started/agent.md) |

学习资料与原有 `docs/` 分开维护；实验产物建议放在 `learning/experiments/`，日志和模型权重使用仓库外的目录。
