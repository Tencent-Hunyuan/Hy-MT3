# TransRubric：基于 Rubric 的翻译自动评估

[English](README.md) | 中文

TransRubric 是一种基于 Rubric 的 LLM-as-a-Judge 翻译评估方法，用于实现细粒度且可解释的翻译质量评估。给定翻译任务、候选译文和任务特定的 Rubric，TransRubric Judge 会识别具体问题及其错误类别和严重程度，并输出 0–100 的 Pointwise 翻译质量分数。

本仓库发布用于应用 TransRubric 的配套资源：基于 FLORES-200、WMT26 GenMT 和 HardMTBench 构建的四个测试集样本级 Rubric、翻译 Rubric 构建 Prompt，以及 Pointwise 翻译 Judge Prompt。原文、参考译文、任务指令和测试集特有元数据来自对应的上游测试集；样本级 `rubric` 字段是本次新增的评估资源。

## 发布的 Rubric 资源

| 文件 | 上游测试集 | 覆盖范围 | 翻译方向/语言对 | 样本数 |
| --- | --- | --- | --- | ---: |
| `testsets/flores200_33langs_rubric.jsonl` | FLORES-200 | 33 种语言 | 1,056 个方向 | 106,656 |
| `testsets/flores200_low_resource_rubric.jsonl` | FLORES-200 | 64 种与中文或英语配对的额外低资源语言 | 256 个方向 | 25,856 |
| `testsets/wmt26_genmt_rubric.jsonl` | WMT26 GenMT | 20 个语言对 | 20 个语言对 | 4,303 |
| `testsets/hardmtbench_rubric.jsonl` | HardMTBench | 12 个知识密集型领域，中文与英语 | 2 个方向 | 20,000 |

WMT26 GenMT 子集移除了英语到利古里亚语、拉迪恩语和北萨米语三个语言对。

## 数据格式

四个 JSONL 文件使用相同的顶层字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 唯一样本标识。WMT26 沿用 `doc_id`；FLORES 按发布文件中的记录顺序生成。 |
| `benchmark` | string | 上游测试集：`flores200`、`wmt26` 或 `hardmtbench`。 |
| `subset` | string | 子集：`33langs`、`low_resource`、`genmt` 或 `full`。 |
| `source_language` | string | 上游数据中的源语言名称或代码。 |
| `target_language` | string | 上游数据中的目标语言名称或代码。 |
| `source_text` | string | 待翻译原文。 |
| `task_prompt` | string | 完整的翻译任务 Prompt。 |
| `reference` | string 或 null | 上游参考译文；不存在时为 `null`。 |
| `reference_type` | string 或 null | 上游提供的参考译文类型，例如 `human`、`synthetic` 或 `none`。 |
| `rubric` | list[string] | TransRubric 新增的样本级评估标准。 |
| `meta` | object | 上游测试集特有的元数据。 |

语言标识保留各上游测试集的原始表示方式：FLORES 使用中文语言名称，WMT26 使用语言及文字代码，HardMTBench 使用 `Chinese` 和 `English`。

对于 HardMTBench，`meta` 保留上游的 `pair_id`、翻译方向、领域与子领域、领域关键词、双语术语以及难度评分。发布文件不包含 Rubric 构建候选和模型生成过程字段，例如 `response_list` 和 `model_res`。

示例：

```json
{
  "id": "flores200_33langs_000001",
  "benchmark": "flores200",
  "subset": "33langs",
  "source_language": "中文",
  "target_language": "英语",
  "source_text": "他们通常会提供特别的食品、饮料和娱乐活动，让客人保持愉悦，留在赌场。",
  "task_prompt": "将以下文本翻译为英语，注意只需要输出翻译后的结果，不要额外解释：\n他们通常会提供特别的食品、饮料和娱乐活动，让客人保持愉悦，留在赌场。",
  "reference": "They usually have special food, drink and entertainment offers, to keep guests in a good mood, and keep them at the premise.",
  "reference_type": null,
  "rubric": [
    "指令遵循：输出必须仅为英文译文，不包含解释或其他多余内容。",
    "语义完整：译文须覆盖原文的核心信息，不得漏译或增译关键信息。",
    "逻辑关系：须准确表达提供食品、饮料和娱乐活动是为了让客人保持愉悦并继续留在赌场，不得弱化或颠倒该目的关系。",
    "行为主体：须明确留在赌场的是客人，不得因代词或句法结构不当而误解为提供服务的一方留在赌场。"
  ],
  "meta": {
    "l1": "wikivoyage",
    "l2": "英语",
    "l3": "中文",
    "l4": "travel",
    "class": "中文-英语-wikivoyage"
  }
}
```

## Rubric 构建 Prompt

[`transrubric_prompts.py`](transrubric_prompts.py) 提供了三阶段 Rubric 构建流程所使用的完整 Prompt 模板：

1. `RUBRIC_DRAFT_PROMPT`：根据翻译任务生成初始 Rubric。
2. `RUBRIC_DELTA_PROMPT`：基于无标签候选译文识别缺失的判别性标准。
3. `RUBRIC_MERGE_PROMPT`：保守地将有效新增标准合并到已有 Rubric 中。

对应的消息构造函数为：

```python
build_rubric_draft_messages(...)
build_rubric_delta_messages(...)
build_rubric_merge_messages(...)
```

Rubric 检查项使用简体中文编写，目标是提供具体、可验证的评估标准，而不是规定唯一的参考译法。

## 翻译 Judge Prompt

`TRANSLATION_JUDGE_PROMPT` 用于结合翻译任务和样本级 Rubric，对单个候选译文进行 Pointwise 评估。Judge 以结构化格式输出具体问题、整体评价、置信度以及 0–100 的翻译质量分数。

下面的辅助函数仅负责构造模型消息。使用者可以将返回的消息传递给自行选择的多语言模型或推理框架。

```python
import json

from transrubric_prompts import build_translation_evaluation_messages

with open(
    "testsets/flores200_33langs_rubric.jsonl",
    encoding="utf-8",
) as f:
    sample = json.loads(next(f))

messages = build_translation_evaluation_messages(
    task_prompt=sample["task_prompt"],
    candidate_response="候选译文",
    rubric=sample["rubric"],
    response_id=sample["id"],
)
```

Rubric 是用于辅助核验的检查清单，并非参考答案，也不意味着候选译文一定存在对应错误。对于语义等价且自然的译法，应当予以接受。

## 项目结构

```text
TransRubric/
├── README.md
├── README_zh.md
├── transrubric_prompts.py
└── testsets/
    ├── flores200_33langs_rubric.jsonl
    ├── flores200_low_resource_rubric.jsonl
    ├── wmt26_genmt_rubric.jsonl
    └── hardmtbench_rubric.jsonl
```

## 数据来源

本资源中的测试数据来自开源的 FLORES-200、WMT26 GenMT 和 HardMTBench 测试集。发布文件保留其原文、参考译文、任务指令和测试集特有元数据，并新增样本级 Rubric 和统一的发布字段格式。

- [FLORES-200](https://github.com/facebookresearch/flores)
- [WMT26 General Machine Translation Shared Task](https://github.com/wmt-conference/wmt26-general-mt)
- [HardMTBench](https://github.com/jasonNLP/HardMTBench)

## 参考文献

使用 TransRubric 测试集时，请同时引用对应的上游测试集：

- NLLB Team et al. (2022). *No Language Left Behind: Scaling Human-Centered Machine Translation*.
- Kocmi et al. (2026). *Findings of the WMT26 General Machine Translation Shared Task: Contrastive Dynamic Human Evaluation at Scale*.
- Li, Z., Zheng, M., Song, M., and Fei, T. (2026). [*HardMTBench: Stress-Testing Chinese-English Translation on Knowledge-Intensive Domains*](https://arxiv.org/abs/2605.28315). arXiv:2605.28315.
