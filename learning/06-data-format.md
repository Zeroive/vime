# 训练数据格式速查

依据当前工作区的 [使用说明：数据格式](../docs/zh/get_started/usage.md)、[数据加载实现](../vime/utils/data.py) 和 [SFT 示例](../docs/zh/examples/qwen3-4b-base-openhermes.md)。

## 文件与字段

默认文件读取器支持 `.jsonl` 和 `.parquet`，Parquet 需要 `pyarrow`。JSONL 每行是一条完整 JSON 对象，不是整个文件一个 JSON 数组。下面展开的 JSON 示例写入 JSONL 时需压成一行。

| 参数 | 作用 | 默认值 |
| --- | --- | --- |
| `--prompt-data` | 数据文件路径 | 按启动配置指定 |
| `--input-key` | 输入字段名 | `input` |
| `--label-key` | 标签字段名 | `None`，不读取标签 |
| `--metadata-key` | 附加信息字段名 | `metadata` |
| `--multimodal-keys` | 媒体类型到字段名的映射 | `None` |

字段名可以更换，启动参数必须匹配实际数据。标签是否必需取决于任务和奖励函数；读取器并不强制所有训练都有 label。

## RL：输入问题，在线生成回答

```json
{
  "prompt": [{"role": "user", "content": "计算 1+1，回答最终结果。"}],
  "label": "2",
  "metadata": {"source_name": "math"}
}
```

数据相关参数：

```text
--prompt-data /path/to/train.jsonl
--input-key prompt
--label-key label
--apply-chat-template
```

`prompt` 也可以是字符串，开启 chat template 时会包装成 user 消息。RL 的回答通常由模型在线生成；label 供奖励函数评判，不是直接作为 SFT 目标回答。label 的内容应符合具体奖励函数的要求。

## SFT：输入完整示范对话

```json
{
  "messages": [
    {"role": "user", "content": "计算 1+1。"},
    {"role": "assistant", "content": "2"}
  ]
}
```

输入字段设置为 `--input-key messages`，并切换到 `--rollout-function-path vime.rollout.sft_rollout.generate_rollout` 与 `--loss-type sft_loss`。完整配置参考 [SFT 启动脚本](../scripts/run-qwen3-4B-base-sft.sh)。

该 SFT 路径需要保留 messages 列表，由 loss mask generator 处理模板和监督范围；示例没有启用 `--apply-chat-template`。不要直接套用 RL 的预先模板渲染配置。消息可附带 `step_loss_mask: 0` 屏蔽该轮监督；实际 token mask 由模板和 mask generator 决定。

## VLM：问题、媒体与标签

以下是图像占位符路径的形状示例，图片路径需替换为训练环境可访问的真实文件：

```json
{
  "problem": "<image>图中有几个三角形？",
  "images": ["/data/images/example.png"],
  "answer": "3"
}
```

使用 `--input-key problem`、`--label-key answer`、`--apply-chat-template` 和 `--multimodal-keys '{"image":"images"}'`。最后一个参数的引用方式是 Bash 示例；其他 shell 需保持传入内容为合法 JSON。

此路径会按出现顺序用 images 替换 `<image>` 占位符，并校验图片与占位符数量一致。模型的 processor 负责处理媒体数据。参考 [VLM 示例](../examples/geo3k_vlm/README.md) 和 [实际启动配置](../examples/geo3k_vlm/run_geo3k_qwen35.sh)。

## 原始数据与内部训练数据

原始文件一般提供输入、标签及附加信息。`tokens`、`response_length`、`reward`、`rollout_log_probs` 和 `loss_mask` 等字段是在数据处理、生成或 SFT mask 构造过程中产生的，随后转换为训练 batch。准备普通训练集时，不需要先手工填满 `Sample` 的所有字段。

自定义数据源可以通过扩展接口改变加载方式，上述文件限制指默认文件读取器。本文参数片段只描述数据配置，不是完整训练启动命令。
