# 运行平台 Modal 说明

本文整理本作业在运行平台 Modal 上的依赖情况：Modal 在代码中如何被集成、一次运行的完整链路、哪些环节必须真实调用 Modal 云端、哪些可以纯本地离线完成。

## 1. 凭据与依赖

| 项目 | 位置 | 说明 |
|---|---|---|
| Python 包 `modal` | `pyproject.toml`，锁定 `modal==1.5.3`（`uv.lock`） | 硬依赖，非可选 extra |
| 远程运行时 `swe-rex[modal]` | 同上 | 提供 `RemoteRuntime` 与 `ModalDeployment` |
| Modal 凭据 | `~/.modal.toml`（由 `uv run modal setup` 写入） | **不放在 `.env` 里** |
| LLM 凭据 | `.env`（由 `.env.example` 复制而来） | `OPENAI_BASE_URL` / `OPENAI_API_KEY` / `OPENAI_MODEL` / `OPENAI_MAX_RETRIES` |

读取 Modal 凭据用的是 `modal.config.Config()`，见 [doctor.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/doctor.py#L48-L52)。

`.env.example` 只有 9 行、仅含 OpenAI 兼容端点配置，不含任何 Modal 配置项。

## 2. Modal 集成方式

关键点：**不是** `@app.function` 装饰器风格，而是 **`modal.Sandbox.create` + SWE-ReX 远程运行时**。

仓库中没有 `modal.App(...)` 定义，也没有 `@app.function`；Modal App 由 SWE-ReX 内部创建（测试中通过 `modal.App.lookup("swe-rex", create_if_missing=True)` 拿到）。

涉及的关键文件：

| 文件 | 作用 |
|---|---|
| [env.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/env.py) | 核心：`AssignmentModalDeployment` + `Environment` |
| [utils/image.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/utils/image.py#L87-L118) | `build_testbed_image`，用 `modal.Image.from_dockerfile` 构建 testbed 镜像 |
| [chess_sandbox.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/chess_sandbox.py) | `ChessSandbox(Environment)`，暴露 chess server 端口 |
| [eval/harness.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/eval/harness.py) | 评测复用同一个 `Environment` |
| [doctor.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/doctor.py) | 只读校验凭据，不启动沙箱 |

## 3. 一次运行的完整链路

以 `Environment(...)` 的实例化为例：

**第 1 步：构建镜像（Modal 侧）**

[image.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/utils/image.py#L102-L116)：

```python
image = modal.Image.from_dockerfile(
    str(task.dockerfile),
    context_dir=str(task.source),
    force_build=force_build,
    ignore=is_ignored,
)
if task.pins:
    image = image.pip_install(*task.pins)
```

- `context_dir=str(task.source)` —— 构建上下文是本地仓库目录（你当前打开的 [image.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/utils/image.py#L104) 就在这里）
- `strict=True` 时先校验 checkout 是否位于任务的 base commit，不匹配直接抛 `SourceMismatch`（`verify_source`）
- `pip_install(*task.pins)` 放在 Dockerfile 之后追加：Modal 会在老版本 image builder 上把依赖安装拼到 Dockerfile 命令之后，不放在最后会让 Dockerfile 里的版本被覆盖

**第 2 步：创建沙箱**

[env.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/env.py#L49-L60)：

```python
self._sandbox = await modal.Sandbox.create.aio(
    "/usr/bin/env", "bash", "-c", self._start_swerex_cmd(token),
    image=self._image,
    timeout=int(self._deployment_timeout),
    encrypted_ports=encrypted,
    unencrypted_ports=unencrypted,
    app=self._app, **extra_kwargs,
)
```

沙箱内启动的是 swe-rex 服务，端口由 SWE-ReX 分配（`self._port`），走 `encrypted_ports`。

**第 3 步：取隧道地址，接上运行时**

[env.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/env.py#L61-L75)：

```python
tunnels = await self._sandbox.tunnels.aio()
tunnel = tunnels[self._port]
self._runtime = RemoteRuntime(host=tunnel.url, timeout=..., auth_token=token, ...)
```

这里有一处对上游的修补：`_tls_port_configuration`（[env.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/env.py#L13-L24)）强制把 SWE-ReX 控制端口放进 `encrypted_ports`。原因是 SWE-ReX 1.4.0 把控制端口以未加密 TCP 打开，却用 `Tunnel.url`（Modal 的 HTTPS 端点）初始化 HTTP 客户端，不改会导致 URL 与传输层协议不匹配。

**第 4 步：构造函数里同步阻塞启动**

[env.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/env.py#L150-L163)：

```python
async def _start():
    await self.deployment.start()
    await self.deployment.is_alive()

asyncio.run(_start())
# ...
self.system, self.release, self.version, self.machine = self.execute(
    "uname -s; uname -r; uname -v; uname -m"
)["output"].splitlines()
```

这意味着 **实例化 `Environment` 就等于启动一个 Modal 沙箱**，无法绕过。启动后立刻在容器内跑 `uname`，为 Part 1 的 `<system_information>` 块提供 `machine` / `release` / `system` / `version` 值 —— 这正是作业 1.1.b 要求逐字包含的那块。

**第 5 步：执行命令**

[env.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/env.py#L252-L264)：

```python
result = asyncio.run(self.deployment.runtime.execute(Command(**arguments)))
```

所有命令（agent 的 `execute` 工具、`make check-*` 的 pytest 等）都走这条 HTTP 隧道打到云端容器。

## 4. Chess 部分

[chess_sandbox.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/chess_sandbox.py#L98)：

```python
modal_sandbox_kwargs={"encrypted_ports": [port]}
```

端口常量 `CHESS_PORT = 8000` 定义在 [chess_tools.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/agent/chess_tools.py#L16)。沙箱内启动 chess server，本地通过 `tunnel_url(port)` 拿到 HTTPS 地址，用 httpx 访问 API。

`run_python` 工具（作业 3.4）也是走这条路：模型编写的代码 base64 编码后通过 `env.execute` 送到 `/opt/assignment/sandbox_python.py`，**绝不在本地 agent 进程中执行**。

## 5. 超时与清理

- `deployment_timeout`（默认 600s）：Modal 回收沙箱的最长存活时间
- `startup_timeout`（默认 600s）：等 SWE-ReX 运行时就绪
- `runtime_timeout`（默认 600s）：单条命令的超时
- `CHESS_TIMEOUT=1800` 控制棋局沙箱生命周期
- [env.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/env.py#L288-L308) 的 `stop()` 显式调用 `sandbox.terminate()`，因为上游 `ModalDeployment.stop()` 的 poll 判断反了，会漏掉沙箱从而烧额度
- `is_alive()` 同样不信 handle 的存在性，直接问沙箱本身

所以作业文档里才提示：环境异常关闭时沙箱可能仍在运行，可用 `modal container list` / `modal container stop <ID>` 手动清理。

## 6. 哪些必须真 Modal

| 环节 | 原因 |
|---|---|
| 任何 `Environment` / `ChessSandbox` 实例化 | 构造函数即启动沙箱 |
| `build_testbed_image` | 镜像构建本身在 Modal 侧 |
| `make run-code-agent` / `run-swebench-agent` / `run-chess-agent` / `run-obs-*` | 都要 agent 在容器内真实执行代码 |
| `make check-part1` / `check-swebench` | 打补丁到全新 testbed 后要真跑 pytest 判题 |
| `make test-modal` / `test-chess-modal` | 真实沙箱集成测试 |

对应 [Makefile](file:///Users/chende/Documents/github/assignment-1/Makefile#L43-L53) 自己的注释：`` `test` is fast, free ``，而 `test-modal` 是 `` Slow and billable: launches real Modal sandboxes ``。

## 7. 哪些可以纯本地离线

| 环节 | 说明 |
|---|---|
| `make setup` | `uv sync` + `git submodule update --init` + `verify-sources`，纯 git 操作 |
| `make test` | 默认 `-m 'not modal'`（见 `pyproject.toml`），不收集计费测试 |
| `make verify-sources` | 校验子模块是否在正确 commit 且干净 |
| `make doctor --offline` | 只读校验，不启动沙箱、不探测推理端点 |

离线测试的替身（不需要 Modal 账号、不烧 LLM 额度）：

- `FakeEnvironment` —— 鸭子类型替身，记录命令但不执行
- `FakeChessEnvironment` + `httpx.MockTransport` —— 假 chess 服务器
- 脚本化的 `client.chat.completions.create` —— 完全不发网络请求的假模型

也就是说：**作业的实现部分（Agent 循环、prompt 构造、上下文压缩、step limit、畸形工具调用、chess 工具 schema）可以完全离线写完并用 `make test` 验证；但产出 artifacts 并通过 `check-*` 这一步绕不开 Modal。**

## 8. 使用成本与前置检查

- `make doctor` 在任何计费运行前先校验子模块、Modal 认证、模型端点，且**不会启动沙箱或生成 token**（[doctor.py](file:///Users/chende/Documents/github/assignment-1/src/assignment/doctor.py#L104-L106)）
- `make test-modal` / `make test-chess-modal` 可选，用于确认所有 Modal 组件按预期运行
- 作业把任何消耗 API 额度（Modal 或 LLM 提供方）的活动称为 _billable_

## 9. 一次性准备清单

```bash
make setup                          # 依赖 + 子模块
uv run modal setup                  # Modal 账号与凭据（写入 ~/.modal.toml）
cp .env.example .env                # LLM 配置
make doctor                         # 计费前的只读校验
make test                           # 离线测试，不计费
```