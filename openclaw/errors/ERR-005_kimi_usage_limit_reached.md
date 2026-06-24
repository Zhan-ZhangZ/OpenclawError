### [ERR-005] Kimi (Moonshot AI) API 账户额度耗尽或频率限制

- **适用平台**: `通用`
- **错误现象/日志**:
  - 在 OpenClaw 中发送消息后，模型无法正常回答，系统返回如下错误信息：
    ```text
    You've reached your usage limit for this period. Your quota will be refreshed in the next period. Upgrade to get more: https://www.kimi.com/code/console?from=limit-upgrade
    ```
  - 该提示可能以通知气泡或对话框形式直接反馈在聊天界面上。

---

#### 根本原因 (Root Cause)
当前运行环境正通过 **Moonshot (Kimi)** API 获取模型响应，该报错是由 Kimi 官方接口返回的限制指令。这通常是由于该 API Key 对应账户发生了以下问题：
1. **免费体验额度用尽**：新账户附带的赠送额度被全部消耗。
2. **账户欠费或余额不足**：付费账户中的余额已被扣减至零或负值。
3. **并发调用频率（Rate Limit）超限**：短时间内发送的请求数量（RPM）或 Token 流量（TPM）超出了 Kimi 分配给当前账户等级的上限限制。

---

#### 解决方案 (Solution)

##### 方法 1：登录官方控制台充值升级（推荐）
1. 在浏览器中打开 Moonshot 开发者平台：[https://www.kimi.com/code/console?from=limit-upgrade](https://www.kimi.com/code/console?from=limit-upgrade)。
2. 登录对应的账户，在“费用中心”或“账户总览”中查看余额。
3. 若已欠费或余额耗尽，进行适当金额的充值，API 将在几分钟内自动恢复可用。

##### 方法 2：更换其他可用的 Kimi API Key
1. 取得其他有余额的有效 Kimi API Key。
2. 打开 OpenClaw 配置文件 `openclaw.json`：
   - **Windows**: `C:\Users\<用户名>\.openclaw\openclaw.json`
   - **macOS / Linux**: `~/.openclaw/openclaw.json`
3. 寻找到 `moonshot` 配置节点，更新其绑定的 `apiKey`：
   ```json
   "moonshot": {
     "enabled": true,
     "config": {
       "webSearch": {
         "apiKey": "替换为新的_KIMI_API_KEY",
         "baseUrl": "https://api.moonshot.cn/v1"
       }
     }
   }
   ```
4. 保存配置文件并重启 OpenClaw 服务。

##### 方法 3：临时切换至其他可用模型通道
在 OpenClaw 聊天页面的顶部“Default Model”下拉菜单中，选择切换为其他有余额或正常连接的模型通道（如 DeepSeek），即可绕过 Kimi API 额度的限制继续使用。

---

- **验证状态**: `已确认有效`
