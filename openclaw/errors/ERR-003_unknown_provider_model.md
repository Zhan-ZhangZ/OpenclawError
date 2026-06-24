### [ERR-003] 模型供应商配置缺失导致主模型无法识别 (Unknown model)

- **适用平台**: `通用`
- **错误现象/日志**:
  - 启动 OpenClaw 或运行相关 Agent 时，控制台或日志报错并无法正常工作。
  - 错误日志中包含类似以下字样：
    ```text
    FailoverError: Unknown model: deepseek/deepseek-chat. Found agents.defaults.models["deepseek/deepseek-chat"], but no matching models.providers["deepseek"].models[] entry. Add { "id": "deepseek-chat" } to models.providers["deepseek"].models[] to register this provider model.
    ```

---

#### 根本原因 (Root Cause)
在 OpenClaw 的配置文件中，主模型设置指向了某个特定模型（例如 `deepseek/deepseek-chat`），且在别名列表 `agents.defaults.models` 中完成了映射，但最底层的模型供应商节点 `"models.providers"` 下完全没有配置该供应商（例如 `"deepseek"`）的信息，导致 OpenClaw 无法找到该模型的接口端点（`baseUrl`）以及定义，抛出 `Unknown model` 异常。

---

#### 解决方案 (Solution)

##### 步骤 1：定位配置文件
打开客户机对应的 OpenClaw 配置文件：
- **macOS / Linux**: `~/.openclaw/config.json`（即 `/Users/用户名/.openclaw/config.json`）
- **Windows**: `C:\Users\<用户名>\.openclaw\config.json`

##### 步骤 2：在 `models.providers` 中补齐供应商定义
找到 `"models.providers"` 段落，在该节点下添加对应的模型供应商及其模型定义。
* **提示**：如果系统环境中已经存在该供应商的 API 密钥变量，在配置文件中可以省略 `apiKey` 字段，或不显式配置，系统会自动读取环境变量。

以补充 `deepseek` 供应商及 `deepseek-chat` 模型为例，添加以下配置块：
```json
"models": {
  "mode": "merge",
  "providers": {
    // ... 原有的其他 providers (如 openai 等) ...
    "deepseek": {
      "baseUrl": "https://api.deepseek.com",
      "api": "openai-completions",
      "injectNumCtxForOpenAICompat": true,
      "authHeader": true,
      "models": [
        {
          "id": "deepseek-chat",
          "name": "DeepSeek Chat",
          "api": "openai-completions",
          "reasoning": true,
          "input": [
            "text"
          ],
          "contextWindow": 128000,
          "maxTokens": 8192
        }
      ]
    }
  }
}
```

##### 步骤 3：保存并重启
保存 `config.json`，然后新开终端重启 OpenClaw 服务即可。

---

- **验证状态**: `已确认有效`
