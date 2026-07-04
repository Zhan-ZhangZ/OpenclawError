### [ERR-015] DeepSeek API Key 鉴权失败 (401 Authentication Fails)

- **适用平台**: `通用 (使用 DeepSeek 模型的部署)`
- **错误现象/日志**:
  - 飞书/Telegram 等渠道的机器人无法回复，Agent 执行立即失败。
  - Gateway 日志中出现反复的 401 鉴权错误：
    ```text
    [feishu] embedded run agent end: isError=true model=deepseek-v4-flash provider=deepseek
    error=LLM request failed. rawError=401 Authentication Fails, Your api key: ****1a92 is invalid
    [model-fallback/decision] decision=candidate_failed reason=auth next=none
    ```

---

#### 根本原因 (Root Cause)

* **API Key 被删除、过期、欠费或拼写错误**：
  DeepSeek 开放平台的 API Key 可能因账户余额耗尽、手动重新生成（旧 Key 自动失效）或输入错误等原因变为无效状态。

---

#### 诊断方法 (Diagnosis)

1. **使用 openclaw CLI 测试**：
   ```bash
   openclaw infer model run --model deepseek/deepseek-v4-flash --prompt "回复一个字好" --local --json
   ```

2. **使用 curl 直接测试**：
   ```bash
   curl -i -H "Authorization: Bearer <API_KEY>" https://api.deepseek.com/v1/models
   ```

3. **检查其他中转渠道是否可用**（如配置了 custom_yunzhi 等）：
   ```bash
   openclaw infer model run --model custom_yunzhi/deepseek-v4-flash --prompt "回复一个字好" --local --json
   ```

---

#### 解决方案 (Solution)

1. **更换 API Key**：登录 [DeepSeek 开放平台](https://platform.deepseek.com/api_keys) 生成新 Key，替换 `openclaw.json` 中 `models.providers.deepseek.apiKey`。

2. **临时切换中转渠道**：若配置了可用的中转 provider，可临时修改默认模型：
   ```json
   "agents": {
     "defaults": {
       "model": {
         "primary": "custom_yunzhi/deepseek-v4-flash"
       }
     }
   }
   ```

3. 修改后重启 Gateway。

---

- **验证状态**: `已确认有效`
- **实际案例**: 2026-07-01 服务器 106.54.35.73，官方 Key ❌，两个中转渠道 ✅
