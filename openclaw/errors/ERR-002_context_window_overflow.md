### [ERR-002] 模型上下文窗口(contextWindow)配置过小导致无回复/上下文溢出

- **适用平台**: `通用`
- **错误现象/日志**:
  - 向 OpenClaw 发送消息后，模型没有任何回复。
  - 控制台或日志文件中持续循环输出以下错误信息：
    ```text
    [agent/embedded] [context-overflow-precheck] stale token state had no real conversation messages for [model-id]; resetting the context snapshot and retrying prompt
    [agent/embedded] promptBudgetBeforeReserve=xxxx overflowTokens=xxxx toolResultReducibleChars=0 reserveTokens=16384 effectiveReserveTokens=8000...
    [agent/embedded] Auto-compaction failed (Context overflow: prompt too large for the model (precheck).). Preserving existing session mapping...
    ```

---

#### 根本原因 (Root Cause)
在 OpenClaw 的配置文件中，当前使用的主模型（`primary`）的 `"contextWindow"`（上下文窗口）被设置得过小（例如 `16000`）。
在 OpenClaw 的 Agent 运行机制中，系统会自动为系统提示词（System Prompt）、已启用的技能与工具定义（Tools/Skills Definition）预留一定的 Token，并且会分配一个默认的回复预留空间（`reserveTokens`，通常为 16384 左右）。如果预留空间加上 Prompt 的估算大小超出了主模型配置的 `contextWindow`，预检机制（Precheck）就会直接判定上下文溢出并拒绝发起请求，导致模型无回复。

---

#### 解决方案 (Solution)

##### 方法 1：修改配置文件，调大模型的上下文窗口（推荐）
1. 定位并打开 OpenClaw 的配置文件：
   - **Windows**: `C:\Users\<您的用户名>\.openclaw\config.json`
   - **macOS / Linux**: `~/.openclaw/config.json`
2. 找到当前使用模型的配置节点（在 `models.providers` 下的对应模型列表中），将其 `"contextWindow"` 数值调大。
   - 例如，对于 DeepSeek 兼容模型，官方通常支持 64,000 或 128,000 的上下文窗口：
     ```json
     {
       "id": "deepseek-v4-pro",
       "name": "deepseek-v4-pro (Custom Provider)",
       "contextWindow": 128000,  // <-- 将此处的 16000 修改为 128000 或 64000
       "maxTokens": 4096
     }
     ```
3. 保存文件并重启 OpenClaw 服务。

##### 方法 2：切换为主上下文窗口更充足的内置模型
如果不想手动编辑 JSON 的参数，可以通过重新运行配置引导，或者在配置文件的 `agents.defaults.model.primary` 中，将主模型修改为其他默认自带大上下文配置的模型（例如已配置为 `128000` 的 `deepseek/deepseek-chat`）。

---

- **验证状态**: `已确认有效`
