### [ERR-006] 镜像升级导致配置 Schema 不兼容（Unrecognized keys）

- **适用平台**: `通用 (Docker / 本地运行)`
- **错误现象/日志**:
  - OpenClaw 服务（如 Gateway 容器）更新镜像版本（例如升级到 `2026.6.2-beta.1` 及以上）后启动失败，状态显示为 `Exited (1)`。
  - 容器或程序日志中输出以下报错信息：
    ```text
    Gateway failed to start: Invalid config at /home/node/.openclaw/openclaw.json.
    agents.defaults: Invalid input
    ```
  - 运行 `openclaw doctor` 或查看深层诊断日志时，显示具体的 Schema 校验错误：
    ```text
    Error: Invalid config at /home/node/.openclaw/openclaw.json:
    - agents.defaults: Unrecognized keys: "embeddedPi", "agentRuntime"
    ```

---

#### 核心配置项解析 (Config Parameter Meaning)
在旧版本的 OpenClaw 中，`agents.defaults` 下存在以下两个关键配置项：
1. **`embeddedPi`** (或 `embeddedAgent`):
   - **作用**: 用于定义本地/嵌入式 Agent 在执行任务时的策略合约（Execution Contract）。
   - **经典参数**: `"executionContract": "strict-agentic"`（严格代理模式）。在此模式下，如果 Agent 在当前轮次中有可用工具（Tools），则不允许只生成“计划（Plan）”而不执行具体工具。这能够有效防止 Agent 陷入“只口头答应却不干活”的死循环。
2. **`agentRuntime`**:
   - **作用**: 指定 Agent 的底层执行运行时平台（如 `google-gemini-cli` 或 `codex` 等），用于决定如何调度底层的推理引擎和技能集。

---

#### 版本更迭变化 (Version Changes)
在 `2026.6.2` 版本升级中，OpenClaw 对底层 Agent 运行时的绑定进行了重构和解耦，将这些属于运行时特性的控制参数从全局的 `agents.defaults` 块中**废弃（Deprecated）或迁移到了更底层的插件与绑定中**。
由于新版本采用了更加严格的 Zod Schema 格式校验，如果用户的 `openclaw.json` 中依然残留了旧版的 `"embeddedPi"` 和 `"agentRuntime"` 字段，就会触发 `Unrecognized keys`（未识别的键名）校验失败，导致 Gateway 服务直接拒绝启动。

---

#### 解决方案 (Solution)

##### 步骤 1：备份当前配置文件
修改前，请务必备份现有的配置文件：
- **Windows**: 备份 `C:\Users\<用户名>\.openclaw\openclaw.json`
- **macOS / Linux**: 备份 `~/.openclaw/openclaw.json`

##### 步骤 2：手动删除废弃字段
用文本编辑器打开 `openclaw.json`，在 `agents.defaults` 块中找到并删除以下字段及其包裹的代码：
1. 删除 `"embeddedPi"` 字段块：
   ```json
   // 需删除此块
   "embeddedPi": {
     "executionContract": "strict-agentic"
   },
   ```
2. 删除 `"agentRuntime"` 字段块：
   ```json
   // 需删除此块
   "agentRuntime": {
     "id": "google-gemini-cli"
   }
   ```
3. **注意 JSON 格式**：删除字段后，请检查前一个字段（如 `memorySearch`）末尾的逗号 `,`，如果它变成了 defaults 块的最后一个字段，需要**删掉该逗号**，以保证 JSON 格式正确。

##### 步骤 3：使用 Doctor 命令修复并重启
1. 在 Docker 环境中，可运行 CLI 容器进行自动修复与校验：
   ```bash
   docker compose run --rm openclaw-cli doctor --fix
   ```
2. 修复后，重新启动 Gateway 容器或服务：
   ```bash
   docker compose up -d
   ```

---

- **验证状态**: `已确认有效`
