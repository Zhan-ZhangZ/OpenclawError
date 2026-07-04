### [ERR-018] 智谱 GLM 模型 404 端点不匹配 (coding vs standard endpoint)

- **适用平台**: `通用 (使用智谱 AI/ZhipuAI GLM 模型)`
- **错误现象/日志**:
  - Agent 调用 GLM-5.2 或 GLM 编程模型时返回 404 Not Found：
    ```text
    provider=zai model=glm-5.2 rawError=404 "Not Found"
    ```
  - 或返回 429 余额不足（但账户实际有余额）。

---

#### 根本原因 (Root Cause)

* **标准端点与 Coding 端点的模型可用列表不同**：
  智谱 AI 有两套 API 端点：
  - 标准端点：`https://open.bigmodel.cn/api/paas/v4` — 仅支持通用模型（glm-4 等）
  - Coding 端点：`https://open.bigmodel.cn/api/coding/paas/v4` — 支持编程专用模型（glm-5.2 等）

  使用标准端点调用 coding 专属模型会返回 404 或 429。

---

#### 解决方案 (Solution)

将 provider 的 `baseUrl` 改为 coding 端点：

```json
"models": {
  "providers": {
    "zhipu": {
      "baseUrl": "https://open.bigmodel.cn/api/coding/paas/v4",
      "apiKey": "<your-api-key>",
      "api": "openai-completions",
      "models": [
        {
          "id": "glm-5.2",
          "name": "GLM 5.2"
        }
      ]
    }
  }
}
```

---

#### 验证方法 (Verify)

```bash
curl -s -X POST https://open.bigmodel.cn/api/coding/paas/v4/chat/completions \
  -H "Authorization: Bearer <API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"model":"glm-5.2","messages":[{"role":"user","content":"hello"}],"max_tokens":10}'
```

正常返回 200 即可。

---

- **验证状态**: `已确认有效`
- **实际案例**: 2026-07-02 客户智谱 GLM-5.2 调用 404，切换至 coding 端点后恢复
