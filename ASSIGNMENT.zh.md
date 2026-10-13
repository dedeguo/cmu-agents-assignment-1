# 作业 1：构建 Agent Harness

Agent harness（智能体框架）提供了一层接口，使语言模型（本质上只会生成「看起来可能的字符串」）能够观察某个环境并在其中采取行动。构建 agent harness 最流行的框架之一是 [ReAct](https://arxiv.org/abs/2210.03629)，它在提示词中交错放置与环境观察、推理/思维链、以及 agent 动作相对应的文本。在本作业中，你将在 ReAct 框架下构建一个 harness。

在第 1 部分中，你将实现基础 [`Agent`](./src/assignment/agent/base.py) 的 ReAct 循环；该 Agent 会被实例化为一个在终端中行动、用于解决软件问题的 [`CodeAgent`](./src/assignment/agent/code_agent.py)。你的 ReAct 循环将是抽象的，并可应用到多个领域，从而让你看清 agent harness 的通用结构。你将实现必要的机制，让编码 agent 能够在终端环境中观察与行动，并应用可复用的 agent skill 来完成任务。随后，你要让这个编码 agent 修复一个国际象棋应用中的问题。

在第 2 部分中，你将研究编码 agent 在解决更复杂软件问题时如何使用上下文。你将实现一种简单的上下文压缩（context compaction）方法，来管理 agent 所占用的语言模型上下文窗口大小。

在第 3 部分中，你将使用第 1 部分构建的基础 `Agent`，将其实例化为 `ChessAgent`，在你用 `CodeAgent` 修复过的应用上对弈国际象棋。你将为该 agent 实现工具规格说明，使语言模型能够通过 API 与应用内运行的棋局对弈，对手是一个基于规则的 bot（你可以旁观！）。你还将探索程序化工具调用（programmatic tool calling），它把编程的力量带入 agent 的动作，使 agent 能够通过执行 Python 代码来实施更复杂的策略。

三个部分之间的关系如下：

```text
                SWE-Bench issue
                      |
                      v
buggy chess app -> CodeAgent -> fix.patch -> repaired chess server
                                              ^
                                              |
                                    ChessAgent tools
```

## 环境搭建

安装 [uv](https://docs.astral.sh/uv/)，然后运行：

```bash
make setup
```

这会配置好 Python 环境（使用 `uv`），并下载安装相关依赖。如果固定版本的 `chess_app` 源码（供你的 agent 处理的某个任务所需）缺失，或不在正确的 commit 上，此步骤会失败。

本作业会使用 [Modal](https://modal.com/)——一个用于运行代码的云平台。该平台允许你在云端机器上安全地运行 agent 生成的代码。你的 agent 所处的环境是一个远程 Modal 沙箱，用于执行 agent 动作的代码在其中运行。首先，创建一个 Modal 账号（如果你还没有），并按照将发送给你的说明获取 Modal 的计算额度。然后运行：

```bash
uv run modal setup
```

以登录并在你的环境中完成 Modal 配置。

完成 Modal 配置后，运行：

```bash
cp .env.example .env
```

以创建包含环境机密信息的文件——主要是 LLM API 详情。如果你是这门课程的学生，你会收到如何获取某个 LLM 提供方额度的说明。按说明生成 API key，并确保正确配置该服务的 base URL。

```dotenv
OPENAI_BASE_URL=<OpenAI 兼容的 base URL>
OPENAI_API_KEY=...
OPENAI_MODEL=deepseek/deepseek-v4-flash-0731
OPENAI_MAX_RETRIES=5
```

默认情况下，如上所示，你会使用 DeepSeek-V4-Flash 模型。在有指示的地方，你应改用其他模型。如果额度允许，欢迎探索同一提供方的其他模型，但我们期望使用本模型完成本作业。

我们把任何会消耗 API 额度（Modal 或 LLM 提供方）的活动称为 _billable_（计费）。你会在整个作业中运行多次计费评测。我们建议监控相关仪表盘，确保 API 额度使用得当。

在一次计费运行之前，先在不启动沙箱、也不生成 token 的情况下校验子模块、Modal 认证以及模型端点：

```bash
make doctor
```

_（可选）_ 如果你想测试所有 Modal 组件是否按预期运行，运行：

```bash
make test-modal
make test-chess-modal
```

作为一般性提示，你可以用

```bash
modal container list
```

来检查是否有计费的 Modal 沙箱正在运行。如果环境被异常关闭，沙箱可能仍在运行并耗尽额度。你可以用 `modal container stop <container ID>` 停止未正确终止的沙箱。

## 公开测试

`make test` 快速、离线且不计费。初始代码会故意在与学生 TODO 相关的测试上失败。请把这些里程碑当作指引；如果新增了澄清性测试，具体的 pytest 数量可能会有所变化。

通过公开测试并不代表完全正确。私有测试还覆盖：失败时的清理、重复/畸形的 skill、棋类的并行调用、传输错误、产物一致性、补丁重放，以及真实的 Modal 集成。

## 规则

- 不要修改 `tests/`、`tasks/`、`chess_app/`。
- 不要改动已提供的日志/清理逻辑，也不要在子类中重复实现共享的 ReAct 循环。
- 不要为通过测试而硬编码 agent 任务的预期解法。你的 agent 应为给定任务自主生成走法或补丁。
- **绝不暴露、记录或提交 API key 等凭据。** 请注意 `.env` 文件的内容。
- 在整个作业过程中，你会使用多个 agent，以及这些 agent 的多个版本。在任何时刻，只使用该 agent 预期具备的工具集。

## 第 1 部分：构建编码 agent 并修复 chess 应用

共享的 `Agent` 必须实现一个常规的 ReAct 循环：构建请求、获得 assistant 动作、执行其工具调用、添加关联的观察，并重复直到完成。

### 1. 构建提示词

给语言模型的提示词是一串消息，用于在交互的每一步查询语言模型。消息是一个字典对象，至少包含 `role` 和 `content` 两个键，还可以包含额外信息。消息的 `content` 可以是字符串，也可以是包含推理 token 的结构化对象，或者是（在本作业未涉及的场景中）图片。消息的 `role` 可以取若干值之一。

`system` 角色用于提供常驻指令——关于领域的说明、环境信息、需遵循的规则、跨任务可能有用的通用策略等。`user` 角色提供任务信息[^1]，以及必要时的额外解题指引。`assistant` 角色消息表示 LLM 的生成结果，`tool` 角色消息表示工具调用的结果，它们作为对环境的观察。你的任务是组织这些消息的序列。按惯例，恰好有一个 `system` 消息位于最前，随后在任何 assistant 消息之前有一个 `user` 消息。由于 LLM 每次只生成一条响应，因此 `assistant` 消息之后应当跟 `tool` 消息（揭示工具调用的结果）或一个 `user` 消息。关于这些消息的结构，可参阅 [OpenAI API reference](https://developers.openai.com/api/reference/python/resources/chat/subresources/completions/methods/create) 以及 vLLM 文档中的这个[示例](https://docs.vllm.ai/en/v0.7.2/getting_started/examples/openai_chat_completion_client_with_tools.html)。

你需要实现 `Agent.build_prompt` 方法。给定作为 agent 类属性可访问的系统提示词和任务提示词，以及你可能自行添加的任何额外记账信息，该方法应准备好交给语言模型的输入。如你所见，我们提供了 `Agent.query_language_model` 方法，它直接使用该方法的输出去查询语言模型 API。你的实现需要为这次查询提供正确的输入。`Agent.query_language_model` 方法会向语言模型 API 提供工具和推理参数，并返回一个包含全部相关信息的数据结构。它同时维护步数计数——即查询 API 的次数。API 负责把所给的提示词和工具渲染成单一的 token 序列，这部分不需要你处理。它还提供了额外的机制来记录你的请求以供评分，这部分你不应修改。

agent 的 LLM 客户端会在遇到临时性提供方故障时，最多重试 `OPENAI_MAX_RETRIES` 次。

> **TODO(1.1.a)**
> 添加必要的机制，在 agent 采取行动并观察结果时维护其状态。构造一个构成语言模型提示词的消息序列。这应包含常驻指令、任务说明、此前的交互（包括观察、推理以及先前轮次的动作）。注意该方法应与领域无关，并以一种适用于任何继承它的领域特定 agent 的方式构建提示词。

接下来，你需要为 `CodeAgent` 构建系统提示词和任务提示词。system 消息 _必须_ 逐字包含以下代码块，并使用环境提供的值：

```text
<system_information>
{
  "machine": <machine>,
  "release": <release>,
  "system": <system>,
  "version": <version>
}
</system_information>
```

> **TODO(1.1.b)**
> 为 `CodeAgent` 构建系统提示词和 `task_prompt`。它们应能被 `Agent.build_prompt` 方法使用。

### 2. 运行 ReAct 循环

实现 `Agent.run` 的方法体。

> **TODO(1.2)**
> 运行 ReAct 循环。编排这样一串过程：提示语言模型产生推理与动作、从模型输出中提取工具调用、执行工具调用以获得下一步的 agent 观察。确保通过设置 `Agent.finished` 来判定 agent 已完成任务。若 agent 超过 `step_limit`，抛出 `StepLimitError`。

### 3. 执行编码工具

实现 `CodeAgent.execute_tool_calls`。编码 agent 支持两个工具：`execute` 和 `send_message`。你可以在[这里](./src/assignment/agent/tools.py)找到它们的定义。你需要实现一个机制来执行这些工具，并准备它们的输出以展示给语言模型。这些输出同样是形如 `{"role": str, "tool_call_id": str, "content": str}` 的消息。每条消息是执行某个工具所得的观察，其角色为特殊角色 `tool`。[^2] 你应使用 `Environment.execute` 方法，以确保代码在正确的 Modal 沙箱中执行。可选参数你可以保持当前默认值，但应允许 agent 在其工具调用中覆盖它们。

> **TODO(1.3)**
> 让 `execute` 和 `send_message` 工具对 agent 可用。解析每次调用，执行被识别的工具，并每次调用返回一条消息（一次 agent 响应中可能有多个工具调用！）。畸形的 JSON 和未知工具必须变成可恢复的观察并转达给 agent，而不是抛出异常。

### 4. 加载 skills

你可能觉得万事俱备、可以针对软件问题运行 agent 了，但还差一点。`CodeAgent` 需要的最后一块知识，是如何提交解决方案的说明。我们使用的具体协议是：agent 解决问题后，把解决方案保存到名为 `patch.txt` 的文件中。评测时会从环境中提取该文件。

为了教会 agent 这个协议，我们将使用一个 skill。Skill 是以专门、可复用的工作流来扩展 agent 能力的方式。[Agent Skills protocol](https://agentskills.io/home) 定义了 skill 应遵循的标准，从而可被多个 agent 框架支持。`tasks/code-skills/submit-task` 定义了一个用于按该协议提交解决方案的基础版 skill。该协议还有更多你可以探索的高级特性，但本作业中你只使用一个最小化的 skill。

你需要实现的核心特性是 _progressive disclosure_（渐进式披露）。为了让 agent 能使用许多可能很复杂的 skill，skill 中的信息以逐级递进的细节程度呈现给 agent，agent 按需访问所需信息。本作业中你将实现一种简单的渐进式披露：在 _系统提示词_ 中给出所有可用 skill 的描述。同时你还要给 agent 提供 `invoke_skill` 工具，agent 可用 skill 名称调用它以查看完整 skill。这样，当 agent 查看完整的 `submit-task` skill 后，它就知道如何提交修复以供评分。我们会检查系统提示词是否提及 skill 名称和描述。当 agent 没有任何可用 skill 时，提示词中不应有任何部分提及 `patch.txt` 或给出提交指令。

skill 文件在[本地](./tasks/code-skills/)可用；当 agent 以使用 skills 的方式运行时，`Agent.skills_path` 指向该位置（否则它被设为 `None`）。注意本作业中 agent 只会用一个 skill（即 `./tasks/code-skills/` 下的一个子目录），但原则上 agent 可能拥有并使用许多 skill，那时渐进式披露会更重要也更有用。

> **TODO(1.4)**
> 校验 ``skills_path``，为每个子目录发现一个 ``SKILL.md``，解析其 YAML frontmatter（文件头部两个 `---` 标签之间的内容），并返回一个以 frontmatter ``name`` 为键的映射。每个值必须包含一段简洁的 ``metadata`` 字符串（用于模型的 skill 目录），以及 skill 文件的完整 ``content``（供 ``invoke_skill`` 使用）。对重复名称，以及畸形或缺失的 frontmatter，抛出清晰的 ``ValueError``。若 agent 有可用 skill，请在提示词中让这些 skill 的描述/元数据对 agent 可见。

例如，`SKILL.md` 的 `content` 为：

```
---
name: hello-world
description: Write "hello, world" to the terminal
---

echo "hello, world"
```

其 `metadata` 为：

```
name: hello-world
description: Write "hello, world" to the terminal
```

### 5. 运行并检查修复

现在，让 agent 来修复一个软件问题。

```bash
make run-code-agent
make check-part1
```

默认模型为 `deepseek/deepseek-v4-flash-0731`。agent 会收到 `tasks/chess-terminal-move/problem_statement.md`，在 `/testbed` 中工作，并且必须复现、修复并验证该失败。一次运行会产生：

- `artifacts/fix.patch`
- `artifacts/part1-trajectory.json`

`make check-part1` 会把生成的补丁应用到一个全新的 testbed 上，并运行公开回归测试以及 chess-app 测试套件。不要直接编辑 `chess_app/` 中的目标文件。

完成第 1 部分后，你的方案应通过与提示词构建、截断、步数上限、畸形/未知工具、补丁提交相关的测试。

## 第 2 部分：实现上下文压缩

冗长的 ReAct 会话记录会推高成本，并最终挤占掉有用的上下文。随着 agent 承担越来越复杂、周期越来越长的任务，它们最终也会触及语言模型上下文窗口的极限。为了让 agent 能高效应对长时间运行的任务，我们希望只保留与先前动作和观察相关的信息。做法是把上下文压缩成一份工作记忆。现在，你将在共享的 `Agent` 中实现一份由模型生成的工作记忆；不要使用特定提供方的压缩端点。

> **TODO(2.1)**
> 实现 `Agent.compact_context`。提示模型压缩上下文。压缩所用的系统提示词应要求输出简洁的事实性工作记忆，并保留目标、约束、涉及的文件、命令、编辑、具体结果、失败的尝试、测试、阻塞项和下一步动作。只对较早的前缀做总结；原样保留原始的 system/task 消息，以及至少最近一个完整的 assistant 动作及其关联的所有工具观察。生成的摘要应改变 `build_prompt` 的输出，并缩短提示词长度。

**不要**触碰 agent 类的 `api_prompt` 和 `api_responses` 属性。它们用于记账和评测。

> **TODO(2.2)**
> 在共享循环中，每次发起新的动作请求之前调用 `maybe_compact_context()`。它已经能估算活跃 token 数、处理阈值，并记录压缩事件以供日志。

以 6,000 token 的阈值运行内置的 `django__django-15368` 任务：

```bash
COMPACT_THRESHOLD=6000 \
SWEBENCH_PATCH=artifacts/django__django-15368.patch \
SWEBENCH_TRAJECTORY=artifacts/django__django-15368-trajectory.json \
make run-swebench-agent INSTANCE=django__django-15368
make check-swebench INSTANCE=django__django-15368
```

设置 `COMPACT_THRESHOLD=0` 可省略压缩标志，运行一个全上下文基线。若要同时保留两次运行，请使用不同的输出名称，例如：

```bash
COMPACT_THRESHOLD=0 \
SWEBENCH_PATCH=artifacts/django__django-15368-baseline.patch \
SWEBENCH_TRAJECTORY=artifacts/django__django-15368-baseline-trajectory.json \
make run-swebench-agent INSTANCE=django__django-15368
```

提交的压缩运行必须至少触发一次压缩、实质性地减少活跃上下文，并产生一个能通过 `check-swebench` 的补丁。生成过程是随机的，因此它不必比每个基线样本使用更少的 ReAct 步数。

一旦你同时产出了压缩与不压缩两条轨迹，请比较各轨迹的 token 用量。将你的观察写入 `artifacts/token-usage-analysis.md`，并对观察到的趋势给出简要解释。压缩与不压缩两种条件之间，在上下文使用上的权衡是什么？

## 第 3 部分：构建 `ChessAgent`

你的 agent 在第 1 部分已修复 chess 应用的问题，现在可以构建一个下国际象棋的 agent 了。`ChessAgent` 复用第 1、2 部分中完全相同的循环。它执白方，服务器上的确定性 bot 执黑方。服务器会在每个合法的白方走子之后自动回复。

### 1. 实现 `play_move`

你首先要定义一个新工具，使 agent 能够在应用中运行的对局里落子。请遵循 [OpenAI function calling 指南](https://developers.openai.com/api/docs/guides/function-calling)来定义工具。

> **TODO(3.1.a)**
> 定义一个名为 ``play_move`` 的 OpenAI function-tool schema。它必须恰好接受一个名为 ``move`` 的必填字符串参数，说明走子使用 [UCI 记谱法](https://en.wikipedia.org/wiki/Universal_Chess_Interface)（例如 `e2e4` 和 `e7e8q`），并拒绝多余参数。

然后，你要实现运行该工具的机制。[^3]

> **TODO(3.1.b)**
> 在 `chess_tools.py` 中实现 `_play_move`。解析参数并向 `/api/move` POST `{"move": <uci move>}`，返回其序列化后的 JSON 对象。捕获工具抛出的任何错误，并返回夹在 `<chess_error></chess_error>` 之间的错误消息，供 agent 处理。需覆盖：参数 JSON 畸形、参数不是对象、缺失或非字符串的 fen、非字符串的 move、服务器拒绝的局面或走子，以及传输失败。

你可以查看 `chess_app` 了解 API 如何工作。

用 `format_state` 方法格式化成功返回的状态，更新 `last_state`，并根据 `game_over` 设置 `finished`。使用原始的 `tool_call_id` 关联该观察。畸形 JSON、未知工具、非法走子和网络错误都要变成 `<chess_error>...</chess_error>` 观察。由于落子后实时局面会变化，一组并行调用中最多只执行一个走子，其余以可恢复的方式拒绝。

初始观察和落子后的成功观察已经展示了棋盘、黑方回复和下一步合法走子。不需要单独的读棋盘工具。

运行：

```bash
make run-chess-agent
```

它会应用你的 `fix.patch`、启动修复后的服务器，并保存：

- `artifacts/part3-trajectory.json`
- `artifacts/game-result.json`

打印出的 HTTPS URL 提供棋盘与 API。agent 落子期间棋盘会轮询实时状态；其按钮可在不重置对局的情况下刷新状态。`CHESS_TIMEOUT=1800` 控制沙箱生命周期。

### 2. 运行观察 A/B 实验

为观察工具接口对 agent 行为的影响，你将对两个模型比较「仅棋盘」观察与「棋盘加合法走子」观察。运行器无需修改源码即可配置，并使用不同的文件名：

```bash
make run-obs-deepseek-no-legal
make run-obs-deepseek-legal
make run-obs-gpt-oss-no-legal
make run-obs-gpt-oss-legal
```

对四次运行中的每一次，记录 `play_move` 调用总数、因非法而被拒绝的调用数、无效走子比例，以及是否达到 `game_over: true`。在 `artifacts/observation-experiment.md` 中写一份简短的比较。评分依据是实验与证据，而非某个特定结果，也不要求赢得对局。

### 3. 添加 `simulate_move`

为了让 agent 能规划更复杂的策略，你将赋予它模拟走子的能力。模拟允许 agent 模拟走子的效果，但不会真正改变实际棋盘的状态。你将定义并注册 `SIMULATE_MOVE_TOOL`、实现 `_simulate_move`，并把它加入现有的分发器。

> **TODO(3.3.a)**
> 像 `play_move` 工具一样定义 `simulate_move` 工具。

`simulate_move(fen, move=None)` 会调用 `POST /api/simulate`。

单独传入一个完整的六字段 [FEN](https://en.wikipedia.org/wiki/Forsyth%E2%80%93Edwards_Notation)，会返回该局面及其合法走子。传入一个 FEN 加一个 UCI 走子，则返回恰好走子一步之后的局面（双方皆可）。返回的 JSON 应可供 Python 使用其中的 `fen`、`squares`、`turn`、`legal_moves` 及终局结果字段。

> **TODO(3.3.b)**
> 在 `chess_tools.py` 中实现 `_simulate_move`。解析参数，调用所提供的 `/api/simulate` 端点，传入 FEN 和可选的 move，并返回其序列化后的 JSON。捕获工具抛出的任何错误，并返回夹在 `<chess_error></chess_error>` 之间的错误消息，供 agent 处理。需覆盖：参数 JSON 畸形、参数不是对象、缺失或非字符串的 fen、非字符串的 move、服务器拒绝的局面或走子，以及传输失败。

你应该使用 `ChessAgent.chess_client` 发送请求。

### 4. 添加 `run_python`

具备了模拟走子的能力后，agent 现在可以进行更复杂形式的规划。能够执行代码将使 agent 更可靠地执行它想出的计划，你将通过程序化工具调用来启用这一能力。定义并注册 `RUN_PYTHON_TOOL`、实现 `_run_python`，并把它加入分发器。模型提供 `run_python(code)`，代码片段可把 `simulate_move` 和 `play_move` 当作普通的同步 Python 函数调用。

> **TODO(3.4)**
> 解析参数，并在沙箱中运行代码，其中已注册的工具可以按名称使用。`/opt/assignment/sandbox_python.py` 是 `env` 沙箱上的一个脚本，它可以访问本文件中的同一份工具定义。用它来运行模型作为 run_python 工具参数产生的代码。该脚本接受两个位置参数——`port` 和一段 base64 编码的代码字符串（以避免引号问题）。实现这个工具调用。
> 该脚本打印一个包含 `stdout`、`stderr` 和 `error` 的 JSON 对象——请原样返回该字符串。非零 returncode 意味着沙箱本身失败，而非模型代码有问题。将 `exception_info` 或 `stderr` 作为 <chess_error> 报告。
> 若存在类型不匹配或解析失败之类的问题，返回 <chess_error>{message}</chess_error>。

模型编写的代码绝不能在本地 agent 进程中运行。`_run_python` 通过 `env.execute` 将 base64 编码的代码发送给所提供的沙箱运行器：

```text
python /opt/assignment/sandbox_python.py <port> <base64-code>
```

返回运行器的 JSON 字符串，其中含 `stdout`、`stderr` 和 `error`。沙箱命令非零退出时变为 `<chess_error>`。代码片段抛出的 Python 异常属于运行器成功调用，应放入其 `error` 字段中。每段代码执行后，重新读取实时棋盘，更新 `last_state` 和 `finished`，并把格式化后的状态追加到观察中；否则模型可能会重放代码片段已经执行过的走子。

用以下命令启用这些工具：

```bash
uv run assignment-play-chess --programmatic-tools
```

完成这一部分后，你的 agent 应能生成一段选择走子的代码，并在对局中走出该步。你可能仍会发现你的 chess agent 并不利用手头的工具来走好棋，而是经常直接用 `play_move` 行动。为了给它更多结构与策略，我们再次回到之前探讨过的 skill 这一思路。

### 5. 加载并使用 chess skill

虽然执行 Python 代码的能力给了 agent 执行复杂计划的选项，但 agent 未必有此倾向。为了给 agent 一个可执行的具体策略，我们在 `tasks/chess-skills/select-move` 中提供了另一个 skill。为让模型使用它，你将把 `CodeAgent` 的部分 skill 使用能力赋予 `ChessAgent`。由于工具执行机制略有不同，你必须为 `ChessAgent` 重新实现对 `invoke_skill` 工具的处理（位于 `_invoke_skill(skills, arguments)`），仅在 skills 被加载时才注册 `INVOKE_SKILL_TOOL`，并返回具名的完整内容。

> **TODO(3.5)**
> 解析参数并返回具名 skill 的内容。若存在类型不匹配或解析失败等问题，返回 <chess_error>{message}</chess_error>。

```bash
uv run assignment-play-chess \
  --programmatic-tools \
  --skills-path tasks/chess-skills \
  --trajectory artifacts/part3-python-skill-trajectory.json
```

该轨迹必须展示 `invoke_skill`，随后是调用 `simulate_move` 进行搜索、并只调用一次 `play_move` 落子的 `run_python` 代码。先读 skill、然后每轮只做一次直接 `play_move` 调用，并不能体现这些工具的协同。对局不必结束，也不要求获胜。

## 评分

本作业总分 **100 分**。每一行独立评分；随机的模型运行失败不会抹掉无关实现部分的得分。

| 部分 | 评分项 | 分值 | 证据 |
|---|---|---|---|
| 1 | 提示词构建与有效的动作/观察历史 | 6 | 私有单元测试 |
| 1 | ReAct 生命周期、纯文本恢复、步数上限、清理、轨迹 | 6 | 私有单元测试 |
| 1 | 编码工具分发、可恢复错误、补丁提交 | 6 | 私有单元测试 |
| 1 | Chess 补丁可应用并通过私有/回归测试 | 8 | 在全新 testbed 中重放补丁 |
| 1 | Skill 发现与 `invoke_skill` 行为 | 4 | 私有测试 |
| 2 | 压缩触发与模型生成的摘要 | 6 | 私有测试与轨迹 |
| 2 | 原始指令与最近的完整工具步骤保持有效 | 6 | 私有测试与轨迹 |
| 2 | 可审计的压缩实质性地减少活跃上下文 | 4 | 压缩事件与用量 |
| 2 | 全上下文 vs. 压缩的 token 用量分析报告 | 4 | 报告 |
| 2 | SWE-bench 补丁通过 FAIL_TO_PASS 与 PASS_TO_PASS | 8 | 补丁重放 |
| 3 | `play_move` schema 与注册 | 4 | 私有测试 |
| 3 | `play_move` 状态更新与可恢复错误 | 6 | 私有测试 |
| 3 | 基础 chess 轨迹到达终局状态 | 4 | 轨迹与结果 |
| 3 | 四次运行的观察 A/B 实验完整 | 4 | 四条轨迹、四个结果 |
| 3 | 观察 A/B 实验报告 | 4 | 报告 |
| 3 | `simulate_move` 无状态行为与错误处理 | 6 | 私有单元/集成测试 |
| 3 | `run_python` 沙箱执行、状态刷新、错误处理 | 6 | 使用沙箱替身的私有测试 |
| 3 | Skill 轨迹结合 skill、程序化搜索与真实落子 | 6 | 轨迹重放 |
| — | 完整、可解析、符合规则的提交 | 2 | 归档校验 |
| | **总计** | **100** | |

评分者会重放补丁与提交的轨迹，不会发起新的 LLM 调用。缺失或前后不一致的证据只会影响对应行丢分。讲师测试与参考补丁不包含在本仓库中。

## 提交

提交一个 ZIP 压缩包，包含你在 `src/assignment/agent/` 下改动的文件，以及以下产物：

```text
artifacts/fix.patch
artifacts/part1-trajectory.json
artifacts/django__django-15368.patch
artifacts/django__django-15368-trajectory.json
artifacts/token-usage-analysis.md
artifacts/part3-trajectory.json
artifacts/game-result.json
artifacts/part3-no-legal-moves-deepseek.json
artifacts/part3-no-legal-moves-deepseek-result.json
artifacts/part3-legal-moves-deepseek.json
artifacts/part3-legal-moves-deepseek-result.json
artifacts/part3-no-legal-moves-gpt-oss.json
artifacts/part3-no-legal-moves-gpt-oss-result.json
artifacts/part3-legal-moves-gpt-oss.json
artifacts/part3-legal-moves-gpt-oss-result.json
artifacts/observation-experiment.md
artifacts/part3-python-skill-trajectory.json
```

仅在你修改了 `src/assignment/prompts.py` 时才包含它。**不要**提交凭据、`.env`、任务文件、测试、子模块内容或讲师文件。你的 ZIP 应包含 `src/` 和 `artifacts` 两个子文件夹，以及一个 `AI_USAGE.md` 文件。`AI_USAGE.md` 应详细说明你在本作业中对 AI 技术的使用情况。列出你使用的所有工具，并清楚描述你如何使用每个工具。如果你没有使用任何 AI 辅助，请在该文件中声明。我们不会为你提交的 `AI_USAGE.md` 评分，但会通过测验（细节另行公布）考察你对所提交代码的理解。

[^1]: 在像这里这样的离线评测环境中，你通常通过把任务组织成来自用户的请求来指定任务。更多示例请参阅这些[文档](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/chatgpt?tabs=python-key%2Cdotnet-secure%2Cjavascript-secure&pivots=programming-language-python)。

[^2]: 提供方返回重复的 `call_0` ID 是有效的：请把每个工具观察与同一 assistant 动作中的调用相匹配，不要假设 ID 在整条轨迹中全局唯一。

[^3]: `_play_move` 等工具被隔离在 `chess_tools.py` 中，以便这些工具能在远程沙箱中执行。请在该代码的结构内正确使用 Modal 沙箱来执行国际象棋走子。