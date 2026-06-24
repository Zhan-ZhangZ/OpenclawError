### [ERR-004] 域名配置错误（如 api.deepseek.com.cn）导致 ECONNREFUSED 报错

- **适用平台**: `通用`
- **错误现象/日志**:
  - 发送消息后模型无回复，网页端出现 Cooldown（冷却）或账号不可用提示。
  - 控制台日志中在极短时间（通常为 3-18ms）内抛出连接被拒绝异常：
    ```text
    [provider-transport-fetch] [model-fetch] error provider=deepseek api=openai-completions model=deepseek-v4-flash elapsedMs=18 name=TypeError code=undefined causeName=Error causeCode=ECONNREFUSED message=fetch failed
    ```
  - 无论如何清空系统代理环境变量、重启服务，依然瞬间报错。

---

#### 根本原因 (Root Cause)
在 OpenClaw 的子配置文件（如 `.openclaw/config/providers.json`）中，模型供应商的 `baseUrl` 域名书写错误（例如将 DeepSeek 官方的 `https://api.deepseek.com` 错写为了不存在的 `https://api.deepseek.com.cn`）。
当客户机开启 Clash 等代理软件时，该错误域名的 DNS 查询被代理软件的 Fake-IP 机制接管，并返回了一个虚拟段的 Fake-IP（如 `198.18.x.x`）。当 Node.js 的 fetch 尝试连接该虚拟 IP 的 443 端口时，由于本地没有该服务运行，在本地协议栈便被瞬间拒绝（`ECONNREFUSED`，耗时仅几毫秒）。由于该域名本身就不存在，即使清空系统代理也无法连通。

---

#### 解决方案 (Solution)

##### 步骤 1：定位并打开 providers.json 配置文件
在客户机上找到以下路径的配置文件：
- **Windows**: `C:\Users\<用户名>\.openclaw\config\providers.json` 或 `工作区目录\.openclaw\config\providers.json`
- **macOS / Linux**: `~/.openclaw/config/providers.json` 或 `工作区目录/.openclaw/config/providers.json`

##### 步骤 2：纠正 baseUrl 域名
找到报错的模型提供商（如 `deepseek`），将其 `baseUrl` 字段修改为正确的官方 API 地址：
* **修正前**：`"baseUrl": "https://api.deepseek.com.cn"`
* **修正后**：`"baseUrl": "https://api.deepseek.com"`

##### 步骤 3：保存并重启 OpenClaw
保存配置文件，并重启 OpenClaw 服务以清除由于此前多次失败导致的 Cooldown 冷却状态。

---

- **验证状态**: `已确认有效`
