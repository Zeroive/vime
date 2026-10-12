# 实践与调试

## A. 无需安装依赖：静态追踪

读 [启动脚本](../scripts/run-qwen3-4B.sh)，为 MODEL、CKPT、ROLLOUT、GRPO、PERF、VLLM 等参数组各写一句作用。追踪至少 5 个参数到 [框架参数](../vime/utils/arguments.py)、[vLLM 参数](../vime/backends/vllm_utils/arguments.py) 或 Megatron 参数处理代码。

普通单轮生成中，假设 `rollout-batch-size=4`、`n-samples-per-prompt=2`，则目标生成 8 个样本。若 `global-batch-size=4`，用这些样本执行 2 次更新时，`num-steps-per-rollout=2`。对照快速开始中的约束，再查参数校验；不要把这两个 optimizer step 理解成两次推理端权重同步。

产出：参数表、样本字段表、一轮训练时序图。

## B. CPU：利用已有测试检验理解

前提：当前 Python 环境能够导入被选测试所需依赖。项目声明 Python >=3.10，CPU CI 使用 Python 3.11；CPU 不等于没有依赖。依赖范围可参考 [.buildkite/pipeline.yml](../.buildkite/pipeline.yml)，不必为了入门安装完整 GPU 栈。

在仓库根目录、准备好的 Python 环境中执行：

```text
python -m pytest tests/test_rm_f1.py -q
python -m pytest tests/test_sample.py -q
```

先预测断言，再运行：

- F1 测试：规范化、完全匹配、部分重叠、零重叠如何影响奖励？
- Sample 测试：序列化是否保留状态与字段？finish reason 如何转为样本状态？

如果发生 import error，记录缺少的模块，依据选定测试和 CI 环境补齐；不要直接从运行全套 `tests/` 开始，里面包含 GPU 与模型相关测试。

## C. CPU：最小奖励扩展

在 `learning/experiments/` 中建立独立实验，参考 [自定义指南](../docs/zh/get_started/customization.md) 与 [async_rm](../vime/rollout/rm_hub/__init__.py) 的调用签名。

任务：写一个规则奖励函数，比较 `sample.response` 与 `sample.label` 的去首尾空白结果，相同返回 1.0，不同返回 0.0；显式处理空答案和缺失 label。该函数只作为学习接口的练习，不是数学评估器。

验收输入：匹配答案、不匹配答案、空回答、缺失标签。随后阅读 [plugin contracts](../tests/plugin_contracts/)，核对函数路径加载、异步行为与返回类型。将预期值、实际值和命令记在学习记录中。

## D. GPU：分三步验证闭环

以下是实验安排，需要按 [快速开始](../docs/zh/get_started/quick_start.md) 准备 Linux/GPU 环境、容器依赖、模型、数据和 checkpoint。当前 Windows 工作区不代表训练环境已经就绪。硬件、镜像、路径及并行规模应以你的实际环境和仓库说明为准。

原有启动脚本可能包含清理进程、启动 Ray 等操作。先完整阅读，再复制成实验脚本修改模型路径和资源参数。学习阶段只需短任务，不必先跑完整示例规模。

1. **rollout-only**：在实验启动配置中加入 `--debug-rollout-only`，并设置 `--save-debug-rollout-data /path/to/learning-dumps/rollout_{rollout_id}.pt`。检查生成文本、reward、status、response 长度、mask 与概率字段。
2. **train-only**：用 `--load-debug-rollout-data /path/to/learning-dumps/rollout_{rollout_id}.pt` 固定输入。该参数会启用 train-only；检查 advantage、loss 与训练指标，必要时设置 `--save-debug-train-data /path/to/learning-dumps/train_{rollout_id}.pt`。
3. **短闭环**：去掉分离调试配置，完成至少两轮生成与训练，核对训练日志、权重同步记录、评估和 checkpoint。两轮可帮助观察第二轮是否经过同步；短任务 reward 不一定上升。

上述路径是占位示例，必须替换。参数应加到实际 `train.py` 提交命令中，并保留必需的模型、数据和并行配置；这些参数本身不是完整启动命令。

## 排查顺序

| 现象 | 优先检查 |
| --- | --- |
| 推理引擎未启动或请求失败 | 引擎日志、模型路径、router 地址与资源配置 |
| 有文本但 reward 异常 | label、答案格式、rm_type、自定义函数及 metadata |
| 有样本但训练信号异常 | response 长度、mask、样本分组、原始与后处理奖励 |
| 概率或 loss 异常 | 固定 rollout 输入，对照概率计算、reference 与算法配置 |
| 权重同步异常 | 实际同步实现、模型格式、同步日志与版本 |
| GPU 显存不足 | 序列长度、并发、batch、并行和 offload；每次只改一个因素 |

具体参数和 dump 格式参考 [调试指南](../docs/zh/developer_guide/debug.md)。CPU 测试通过只能验证其覆盖的数据契约或算法逻辑，GPU 端到端闭环需要单独执行。
