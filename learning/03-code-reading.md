# 源码阅读指南：沿边界追踪

## 第一条线：控制流

从 [train.py](../train.py) 的 `train` 开始，依次标记：

1. `create_placement_groups`：资源划分。
2. `create_rollout_manager`：创建 rollout manager 与推理引擎。
3. `create_training_models`：创建训练模型组。
4. 初始 `actor_model.update_weights()`：训练权重加载后同步到推理端。
5. `rollout_manager.generate.remote(...)`：获取一轮训练数据。
6. `actor_model.async_train(...)`：调度训练；使用 critic 时还会走 critic 分支。
7. 保存、显存管理、再次同步权重、周期评估及资源清理。

注意：方法名称里的 `async` 不保证整个训练循环与 rollout 重叠。`train.py` 使用 `ray.get` 等待；重叠关系要看实际等待位置。对比 [train_async.py](../train_async.py) 后画出时间线。

## 第二条线：数据流

| 阅读入口 | 要回答的问题 |
| --- | --- |
| [data_source.py](../vime/rollout/data_source.py)：`get_samples` | 数据如何读入？一个 prompt 如何复制成一组样本？index 如何赋值？ |
| [base_types.py](../vime/rollout/base_types.py)：输出类型 | 训练输出为什么是嵌套样本列表？评估输出有何不同？ |
| [vllm_rollout.py](../vime/rollout/vllm_rollout.py)：`generate_rollout` | 训练与评估在哪里分支？同步入口如何调用异步代码？ |
| 同文件：`generate_rollout_async` | 如何调度生成？如何处理过滤、未完成样本和批量目标？ |
| 同文件：`generate_and_rm`、`generate` | 自定义生成如何接入？token 和 log probability 如何回写？ |
| [rm_hub/__init__.py](../vime/rollout/rm_hub/__init__.py)：`async_rm` | 自定义奖励与内置 `rm_type` 的优先级是什么？ |
| [ray/rollout.py](../vime/ray/rollout.py)：`generate` | dump 保存发生在转换前还是后？rollout-only 在哪里返回？ |
| 同文件：`_convert_samples_to_train_data` | rewards、raw_reward、loss_masks、rollout_ids 如何构造？ |
| 同文件：`_split_train_data_by_dp` | 数据如何按训练并行配置分发？返回值如何跨 Ray 边界传递？ |

跟踪一个简化例子：prompt 有 3 个 token，response 有 2 个 token，总 `tokens` 长度为 5，`response_length=2`，`loss_mask=[1, 1]`。这是形状示意；真实 token 值、模板与概率需要实际 tokenizer 和生成结果。推理输出、序列总长度与参与损失的 token 数不要混用。

## 第三条线：训练与权重

入口为 [actor_group.py](../vime/ray/actor_group.py) 的 `async_train`，然后阅读 [actor.py](../vime/backends/megatron_utils/actor.py) 的 `train`、`_get_rollout_data` 和 `train_actor`。

重点跟踪：数据迭代器与 microbatch 调度；reference/teacher/actor 的选择；概率计算或复用的条件；advantage 计算；postprocess hook；模型训练与保存。第一次可以跳过 routing replay、低精度和模型结构细节。

在 [loss.py](../vime/backends/megatron_utils/loss.py) 中只先找四个入口：`get_log_probs_and_entropy`、`compute_advantages_and_returns`、`policy_loss_function`、`loss_function`。再按它们的调用深入 [ppo_utils.py](../vime/utils/ppo_utils.py)。每读一个公式，记录对应 tensor 的长度与 mask 范围。

最后沿 `update_weights` 阅读 [update_weight/__init__.py](../vime/backends/megatron_utils/update_weight/__init__.py) 和实际选中的实现。配置决定传输方式，不应假设所有训练都使用相同同步路径。

## 避免常见混淆

- `--custom-generate-function-path` 替换生成步骤；`--rollout-function-path` 替换更大的 rollout 编排范围。
- 样本数量关系先按普通单轮生成理解；fan-out、动态采样与特殊工作流需要核对其专门契约。
- `raw_reward`、后处理的 `rewards` 和 `advantages` 不是同一个字段。
- `Sample.status` 和 `loss_mask` 都要查；不能只看有没有 response 字符串。
- `rollout_id` 的分组语义会影响 loss 聚合，拆分样本必须保留相应关系。

## 搜索方法

在仓库根目录使用以下只读命令；PowerShell 与 Bash 都可直接运行：

```text
rg -n "def train|async_train|update_weights" train.py train_async.py vime/ray vime/backends/megatron_utils
rg -n "class Sample|class RolloutFnTrainOutput" vime
rg -n "custom_generate_function_path|custom_rm_path" vime
rg -n "compute_advantages_and_returns|policy_loss_function" vime tests
```

笔记优先写文件路径、函数名与结论。行号随代码变化，仅作为临时定位参考。
