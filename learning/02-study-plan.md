# 四阶段学习路线

建议每天：10 分钟回顾，40–60 分钟读代码或实践，10–20 分钟记录。每次只追踪一个问题，不认识的后端实现先记下入口。

## 预备知识：按需补齐

- Python：dataclass、类型注解、动态导入、async/await。练习解释为什么 `generate` 是异步函数，而顶层 `generate_rollout` 是同步入口。
- PyTorch：tensor shape、autograd、mask、optimizer。用一个小 tensor 手算 masked mean，再核对计算结果。
- 概率与 RL：token 概率、log probability、reward、advantage、策略梯度。先能解释各自输入输出，不必先推导所有算法。
- 分布式：理解 Ray `.remote()` 与 `ray.get()`，以及 DP/TP 的基本区别。
- Linux/Bash：理解数组、环境变量、`source` 和启动脚本。当前 Windows 工作区可以用于代码学习；训练实践按仓库 Linux/GPU 环境进行。

## 第一周：建立整体模型

| 次序 | 学习内容 | 必须留下的产出 |
| --- | --- | --- |
| 1 | README 架构、项目地图 | 用 200 字解释框架的用途 |
| 2 | `train.py` 初始化与主循环 | 一张包含生成、训练、同步的时序草图 |
| 3 | `scripts/run-qwen3-4B.sh` 与模型配置脚本 | 参数分组表，标记路径、采样、算法、并行 |
| 4 | `Sample` 与 `RolloutFnTrainOutput` | 一条假想样本的字段表 |
| 5 | 快速开始、调试指南；复述主流程 | 环境清单与三个待解决问题 |

验收：脱离源码，能说出一次训练从什么开始、样本在哪里产生、参数在哪里更新、何时同步给推理端。

## 第二周：跟踪一条样本

依次阅读 `RolloutDataSource.get_samples` → `generate_rollout` → `generate_rollout_async` → `generate_and_rm_group` / `generate_and_rm` → `generate` / `async_rm` → `RolloutManager._convert_samples_to_train_data` → `_split_train_data_by_dp`。

每天追踪一个边界：prompt 分组、生成返回、奖励附加、mask 构造、训练分片。对每个边界记录“输入类型、输出类型、新增字段、不变量”。阅读 `test_sample.py`、`test_rm_f1.py` 和 plugin contracts，具备依赖时执行选定 CPU 测试。

验收：能解释 `tokens` 与 `response_length` 的关系、为何 loss mask 按 response 对齐，以及为什么生成结果还需要转换和分片。

## 第三周：理解策略更新

依次阅读 `RayTrainGroup.async_train` → `MegatronTrainRayActor.train` / `train_actor` → `compute_advantages_and_returns` → `policy_loss_function` → `model.py` 中训练与优化器调用。

先只追一套配置：Qwen3-4B 脚本中的 GRPO 路线。把“原始 reward → 奖励后处理 → advantage → policy loss → 梯度更新”写成字段变化表。阅读 `ppo_utils.py` 和对应 policy loss 测试核对公式与形状。完成后再比较 critic 分支；不同算法不能直接套用同一套 advantage 解释。

验收：能区分 rollout log probability、训练侧 log probability、reference log probability；能解释 mask 和 KL 配置在哪些步骤影响训练。

## 第四周：完成一次扩展与调试

1. 参考自定义指南，在独立实验目录写一个规则奖励函数，用假样本检查输入输出。
2. 有 GPU 环境时，先验证 rollout-only，再保存 rollout dump；固定输入做 train-only，最后完成短训练闭环。
3. 比较 `train.py` 与 `train_async.py`，标记下一轮生成提前启动的位置、权重同步前等待生成的位置。
4. 选择一个进阶方向，完成一份“入口、数据契约、关键参数、验证方法”的笔记。

验收：能指出扩展函数如何被加载、哪些字段必须保留，以及如何把生成问题与训练问题分开定位。没有 GPU 时，交付扩展函数与 CPU 输入输出验证，GPU 项保持待办。

## 后续方向

| 兴趣 | 建议入口 | 学习重点 |
| --- | --- | --- |
| Agent RL | [agent 指南](../docs/zh/get_started/agent.md)、[multi_agent](../examples/multi_agent/README.md) | 多轮轨迹、mask、拆分样本 |
| 长尾异步生成 | [fully_async](../examples/fully_async/README.md) | 并发、缓冲、权重版本 |
| 部署与同步 | [vLLM 配置](../docs/zh/advanced/vllm-config.md)、[delta sync](../docs/zh/advanced/delta-weight-sync.md) | 推理拓扑、权重格式与传输 |
| 算法与蒸馏 | [OPD 文档](../docs/zh/advanced/on-policy-distillation.md) | teacher 概率与损失 |

如果已有 RL 与分布式经验，可将前两周压缩成 2–3 天；如果 Python/PyTorch 基础薄弱，先完成预备练习，再进入训练后端。
