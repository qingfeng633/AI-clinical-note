# 项目运行与配置说明

##  当前进度

* **已实现**：读取 `./input.json` 内容，传入基于 Bionic 的本地模型进行推理，并将结果输出至 `./output.json`。
* **已实现**：**输出内容校验**——在 Schema 校验之外，对生成文本做内容层校验（数值回溯、否定/缺失信息保留、
  计划与确诊措辞、evidence 与待核事项），并写入 `validation_report.json` / `run_log.json`，见下文。
* **已实现**：**自我审查（可选）**——`python main.py --self-review` 在初稿之上追加一次模型自审调用，
  模型依据 sources 指出并修正问题；修正稿只有在「校验通过且内容分不下降」时才替换初稿，见下文。
* **已实现**：**训练数据回流**——`tools/collect_training_data.py` 把通过校验的运行结果导出为
  `reference.json` + SFT `train/val.jsonl`（含 `provenance.json` 溯源），直接接入 `train/` 的 LoRA 流程。
* **已实现**：**模型训练**——离线 LoRA/QLoRA SFT 流程：合成数据 → chat 格式数据集 → 训练 → 基座/adapter 对比评估，
  见 [`train/README.md`](train/README.md)。
* **尚待实现**：无（README 原有两项待办均已完成；训练流程受本机无 GPU 限制，只提供可复现脚本与数据自检，
  正式评测不允许替换部署模型权重）。

---

##  环境与模型配置

### 1. 基础环境
* **Bionic (LM Studio)**：`1.2.1`
* **Python**：`3.11.9`
* **本地部署模型**：`qwen3-4b-instruct-2507`
* **基础 URL**：`http://localhost:1234/v1`

### 2. 推理参数设置

| 参数项 | 配置值 |
| :--- | :--- |
| **硬件优化** | `Automatic Optimize Based on Hardware` |
| **CPU Thread Pool Size** | `8` |
| **Temperature** | `0.7` |
| **Top K** | `20` |
| **Top P** | `0.8` |
| **Min P** | `0` |
| **Seed** | `42` |

### 3. 环境变量文件

`.env`（不入库，参照 `.env.example`）：

```ini
MODEL_NAME=replace_with_model_identifier
ENDPOINT=http://127.0.0.1:1234/v1/chat/completions
```

依赖安装：`pip install -r requirements.txt`。

---

##  快速运行测试

在 PowerShell 中依次执行以下命令以准备测试数据并运行主程序。运行前请先按上一节准备 `.env`
（可执行 `Copy-Item .\.env.example .\.env` 后填入实际的 `MODEL_NAME` 与 `ENDPOINT`）；
缺少 `.env` 时 `main.py` 会以退出码 `2` 报 `CONFIGURATION_ERROR`，不会调用模型。

```powershell
# 1. 复制示例输入文件
Copy-Item -LiteralPath '.\examples\DEMO_THREE_CASES.input.json' -Destination '.\input.json'

# 2. 激活基线提示词模板
Copy-Item -LiteralPath '.\prompts\baseline.txt' -Destination '.\prompts\active.txt'

# 3. 使用虚拟环境运行主脚本
& .\.venv\Scripts\python.exe .\main.py
```

正式运行时建议改用更强的 `prompts\grounded.txt` 覆盖 `prompts\active.txt`
（更严格地要求保留否定/缺失信息、计划不写成已实施）。

`main.py` 的退出码：`0` 表示流程正常结束（个别病例失败仍返回 0，失败原因写入
`run_log.json` 的 `case_runs[i].error`）；`1` 表示运行期写盘或 IO 失败；`2` 表示配置、
输入文档或提示词等启动阶段错误，此时不会发出任何模型请求。

---

##  输出内容校验

`src/validate.py` 负责 **Schema / 结构** 校验（5 个字段齐全、evidence 结构、`source_ids` 可解析、
不许多字段）。`src/content_check.py` 补上 **内容层** 校验，对应任务评分点：

| 类别 | 权重 | 检查内容（示例） |
| :--- | :--- | :--- |
| structure | 15 | 三段齐全、每段不少于 60 字、总篇幅 800–1600 字建议区间、占位符、三段重复 |
| factual | 25 | checklist 必含事实覆盖率、资料缺失项是否在正文或待核事项中说明 |
| numeric | 20 | 数值能否回溯到资料（`NUMBER_NOT_TRACEABLE`）、单位一致、checklist 数值匹配 |
| negation | 15 | `未提供/未见/否认` 是否被写成“正常/阴性/排除”、否定范围是否被扩大、待核事项是否为空 |
| clinical_boundary | 15 | 计划写成“已实施/已完成”、初步判断写成“确诊”、引入记录截止点之后的信息 |
| evidence_review | 10 | evidence 条数、数值型 evidence 是否可在来源中回溯、`review_targets` 覆盖 |

运行 `main.py` 时自动执行，**不改变** `output.json` 的产出与退出码：

* 每例结果写入 `validation_report.json` 的 `cases[i].content_check`（分数、明细、指标）；
* 全量汇总写入 `validation_report.json` 与 `run_log.json` 的 `content_summary`；
* `run_log.json` 的 `case_runs[i]` 增加 `content_score` / `content_passed` / `content_issues_by_code`。

### checklist（病例侧校验清单）

给同一病例放一份人工清单，校验器即可知道“哪些事实必须出现、哪些缺失项不能被写成正常”：

```text
checklists/<case_id>.checklist.json   # 也可放 examples/<case_id>.checklist.json
```

字段见 `templates/checklist.template.json`，示例见 `examples/DEMO001.checklist.json`：

* `required_facts`：必须被复述的关键事实（文本覆盖率阈值 0.35/0.65）；
* `numeric_facts`：必须出现的数值 + 单位；
* `negation_scope`：否定范围说明（不得扩大为“排除”）；
* `missing_items`：资料缺失项（正文或待核事项提到即算保留）；
* `forbidden_claims`：禁止出现的断言（如“血压正常”）；
* `review_targets`：待核事项应覆盖的重点；
* `clinical_review_status`：现有示例均为 `not_expert_reviewed`，仅作教学与回归用。

没有 checklist 时只运行通用规则（数值回溯、否定保留、边界措辞、结构等），仍然可用。

### 离线审计与改进验证

```powershell
# 对已有 output.json 重新做内容校验（不调用模型）
python tools/check_output.py --input input.json --output output.json --report content_report.json

# baseline vs improved：同一批病例两次输出对比，输出收益/局限
python tools/check_output.py --input input.json --output output_grounded.json `
  --baseline-output output_baseline.json --report reports/comparison.json
```

`--fail-on-error` 可在存在 error 级内容问题时返回退出码 1，便于接入 CI。

---

##  自我审查（可选，每例追加一次模型调用）

`main.py --self-review` 在内容校验之后追加一轮「模型自审」：把病例 sources、初稿和规则校验发现一起发给
同一个模型（提示词 `prompts/self_review.txt`），要求它返回结构化审查结果：

```json
{
  "verdict": "pass | revise",
  "issues": [{"code": "...", "severity": "error|warning", "field": "...", "message": "..."}],
  "revised_record": null,
  "notes": ["..."]
}
```

修正稿（`revised_record`）只有在同时满足以下条件时才会**替换** `output.json` 中的初稿，否则保留原文：

1. 仍然通过 `validate_medical_record`（五字段、evidence 结构、`source_id` 真实存在）；
2. 内容分不低于初稿（`check_record`）；
3. 没有引入新的 error 级问题代码（避免「修好一个、弄坏另一个」）。

留痕位置：

* `run_log.json` 的 `case_runs[i].self_review`：`called` / `verdict` / `issues` / `accepted` / `reason` / `score_before` / `score_after`；
* `validation_report.json` 的 `cases[i].self_review`：同一份摘要；
* `artifacts/raw/<run_id>/<i>.review.txt`：自审原始响应（便于复核模型是否胡说）。

计数口径：默认每例 1 次调用；开启后成功的病例会变成 2 次，`run_log.json` 的 `total_model_calls` 按「实际 HTTP 调用」累加
（`attempt + review_attempt`），并新增 `self_review.reviewed_cases/accepted_cases` 汇总。自审失败（超时、截断、JSON 不可解析）
**不会**让病例失败，只记录 `reason` 并保留初稿。

```powershell
# 在原有命令上追加 --self-review 即可
python main.py --self-review
```

---

##  模型训练

`train/README.md` 给出完整流程（离线研究轨道；正式评测不允许替换部署模型权重）：

```powershell
# 0. 可选：把一次正式运行中通过校验的结果回流成 SFT 数据
python tools/collect_training_data.py --input input.json --output output.json `
  --report validation_report.json --out-dir data/collected

# 1. 合成数据 → chat 格式 SFT 数据
python tools/build_sft_dataset.py --input data/synth/input.json `
  --reference data/synth/reference.json --out-dir data/sft

# 2. 数据/配置自检（无需 GPU 依赖）
python train/train_lora.py --dry-check --train-file data/sft/train.jsonl `
  --val-file data/sft/val.jsonl --output-dir checkpoints/qwen3-4b-lora

# 3. GPU 环境训练（需 requirements-train.txt）
python train/train_lora.py --train-file data/sft/train.jsonl `
  --val-file data/sft/val.jsonl --output-dir checkpoints/qwen3-4b-lora

# 4. 基座 vs adapter 在同一验证集上评估
python train/evaluate.py --val-file data/sft/val.jsonl --output reports/base_eval.json
python train/evaluate.py --val-file data/sft/val.jsonl `
  --adapter checkpoints/qwen3-4b-lora --output reports/lora_eval.json
```

---

##  目录结构

```text
main.py                     运行入口（生成 + 校验 + 留痕）
src/model_client.py         模型调用（OpenAI 兼容端点，无鉴权头，超时/错误分类）
src/validate.py             输入 / Schema 校验
src/content_check.py        内容层校验（新增）
src/self_review.py          自我审查：模型自审 + 修正稿准入判定（新增）
src/checklist.py            checklist 加载与文本对齐 / 数值 / 缺失项抽取工具
prompts/                    baseline / grounded / self_review / active 提示词
templates/                  medical_record.schema.json、checklist.template.json
examples/                   三例演示输入及其 checklist
tools/check_output.py       已有 output.json 的离线内容审计与前后对比
tools/collect_training_data.py  通过校验的运行结果 → SFT 数据（含 provenance）
tools/build_sft_dataset.py  训练数据构造
tools/probe_model.py        单请求端点探测（不写 output.json）
train/                      LoRA/QLoRA 训练与评估脚本（离线轨道）
tests/                      单元与集成测试（unittest）
```

---

##  测试

```powershell
& .\.venv\Scripts\python.exe -m unittest discover -s tests -v
```

覆盖：输入/Schema 校验、模型错误分类、主流程退出码与留痕、内容校验各规则、checklist 加载、
自我审查准入逻辑与端到端回环（本地 stub 端点）、训练数据构造/回流与前后对比工具。
`tests/test_model_client.py` 中的真实端点用例默认跳过（设置 `RUN_LIVE_TESTS=1` 时启用）。
