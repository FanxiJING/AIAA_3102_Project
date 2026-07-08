# Final Project - Topic C：SWE-bench 代码库接管

## Adopt a Codebase, Hard Mode

*修复五个成熟 Python libraries 中的真实 bugs，用 SWE-bench 验证 patches，并解释每个修复背后的代码库 reasoning。*

### I. 背景

现代软件工程越来越多地使用 AI agents，但 agent 并不能替代你对代码的理解。一个 patch 可能通过 local smoke test，却遗漏真实 edge case；agent 也可能给出听起来合理、但在检查相关 subsystem 后发现错误的解释。真实代码库工作需要 localization、testing、patch review 和 judgment。

在本项目中，你将处理五个来自常用 Python libraries 的历史 GitHub issues。每个 repository 都冻结在 upstream fix 之前的 commit。released metadata 会给出 issue text 和 base commit，但不会提供 gold patch 或 hidden tests。你的任务是为每个 issue 生成 patch，通过课程 SWE-bench 路径验证，并解释 patch 为什么正确，或为什么失败。

### II. 项目概览

你将处理五个 pinned SWE-bench Verified instances：

| # | Instance id | Library | Bug summary |
|---|---|---|---|
| 1 | `django__django-16485` | django | `floatformat('0.00', 0)` raises `ValueError: valid range for prec`. |
| 2 | `django__django-14580` | django | A generated migration references `models.Model` without importing `models`. |
| 3 | `sphinx-doc__sphinx-8595` | sphinx | Empty `__all__ = []` is ignored; autodoc documents all members. |
| 4 | `pylint-dev__pylint-7080` | pylint | `pylint --recursive=y` ignores `ignore-paths`. |
| 5 | `pytest-dev__pytest-7571` | pytest | `caplog.set_level()` is not restored after a test. |

有些修复代码行数很少，但 reasoning 并不简单。强提交应解释 bug 位于哪里，相关 subsystem 如何工作，哪些 local evidence 支持 patch，以及 AI agent 错了什么或漏了什么。

### III. 正确性与评估

每个 task 的 released metadata 包含：

- repository；
- base commit；
- issue text (`problem_statement`)；
- `sb-cli` 使用的 task id。

Correctness 通过 SWE-bench harness 评估：

- **FAIL_TO_PASS:** 捕捉该 bug 的 tests 应在 patch 后通过。
- **PASS_TO_PASS:** 原本通过的 tests 应继续通过。

通过 hidden tests 有帮助，但这本身不够。无法解释的 passing patch 几乎没有 analysis credit。失败的 patch 仍可能有价值，前提是 reproduction、localization 和 failure diagnosis 足够仔细且诚实。

### IV. SWE-bench 包结构与提交格式

starter package 提供 task metadata、problem statements、public self-check guidance 和 patch naming expectations。请使用其中的 `README.md` 查看 allowed materials、clone targets、patch names 和 prediction-format expectations。clone repositories、运行 checks 和 packaging predictions 都由你自己完成。Local Docker evaluation 是 optional。

### V. 信息来源规则与预算

Allowed sources：

- assigned base commit 下的 source code；
- 该 checkout 中可用的 project documentation；
- 理解 public APIs 所需的 package documentation；
- provided issue text。

Forbidden sources：

- SWE-bench gold patch 或 hidden test patch；
- 原始 upstream PR 或 fixing commit；
- 使用 instance id 加 `"fix"`、`"patch"` 或 exact error 进行 web search；
- 复制其他团队的 transcript 或 patch。

不需要 GPU。请遵守课程对 agent turns 或 API spending 的 cap。报告 tokens、turns、runtime 和 failures。

### VI. 分析范围

你的 analysis 应证明你理解代码库和 bug mechanism，而不只是让 agent 生成 patch。对每个 task，请把 reproduction evidence、localization、patch reasoning、local checks 和 `sb-cli` status 连接成一个 coherent explanation。

跨所有 tasks，至少比较一次 zero-orchestration one-shot agent run 与 guided run。说明 prompt、context、tests 或 review process 有何变化，以及 guided run 为什么改善或没有改善 outcome。Exact artifact formats 在 starter README 中说明。

### VII. 报告要求

最终的 `report.pdf` 应总结你的 methodology、analysis、key findings 和 conclusions。它应该是一份自洽的项目报告，而不是 patch logs 的集合。

一份优秀报告应清楚说明你如何接管陌生代码库，如何定位每个 bug，为什么每个 patch 正确或不正确，以及哪些证据支持你的判断。对于本 topic，报告也应反思 agent orchestration：agent 在哪里有帮助，在哪里失败，以及 one-shot 与 guided runs 之间发生了什么变化。

报告中也应设置单独的 **Difficulties and Solutions** section，说明至少三个具体、可验证的项目困难，以及你如何解决它们。请包含 AI usage declaration，说明你如何使用 AI agents，以及如何验证 AI 输出。

### VIII. 提交内容

```
repo/
|-- patches/
|   |-- <instance_id>.patch
|   `-- predictions.jsonl
|-- analysis/
|   |-- orientation.md
|   |-- root-cause.md
|   `-- analysis.md
|-- results/sb-cli/
|-- logs/
|   |-- <instance_id>-guided.md
|   |-- <instance_id>-one-shot.md
|   `-- README.md
|-- report.pdf
`-- README.md
```

starter README 会说明 expected patch files、reproduction evidence、root-cause notes 和 SWE-bench status artifacts。

### IX. 学术诚信与 AI 使用

允许并鼓励使用 AI agents。不允许查找 gold patches、hidden tests 或 upstream fixes。你的 patch 自然可能触及真实 fix 的相同行；重合本身不是问题，缺少 derivation trail 才是问题。你需要能在 oral defense 中解释自己的 reasoning。
