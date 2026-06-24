### [ERR-010] 飞书插件默认账号配置层级不匹配导致机器人不回复

- **适用平台**: `通用 (飞书/Feishu 渠道配置)`
- **错误现象/日志**:
  - 飞书机器人收到用户消息后显示已分发至 Agent（`dispatching to agent`），但机器人始终不进行回复。
  - 网关日志（如 `/tmp/openclaw/openclaw-xxxx.log`）中可以观察到如下警告：
    ```text
    Retry failed for delivery <id>: LarkClient[default]: appId and appSecret are required
    ```
  - 在网关诊断日志或插件调试信息中，发现解析出来的 `default` 账号状态为：
    ```text
    default account: enabled:false / configured:false
    ```
  - 投递队列中积压了大量 `accountId: "default"` 的待发送回复，由于账号被判定为未启用而无法投递成功。

- **根本原因 (Root Cause)**:
  飞书插件（`channels.feishu`）在解析默认账号配置时存在层级读取的硬编码设计：
  - 虽然 JSON 配置文件中将默认账号写在了 `channels.feishu.accounts.default` 路径下，但插件在读取 `default` 账号时，会直接从 `channels.feishu` 的**顶层属性**中读取。
  - 这导致放在 `accounts.default` 下的配置被忽略，使系统判定默认账号为 `disabled`。

- **解决方案 (Solution)**:
  1. **备份配置文件**：备份 `openclaw.json` 以防万一。
  2. **调整 JSON 配置结构**：
     将 `accounts.default` 内的所有配置字段（如 `appId`、`appSecret`、`encryptKey`、`verificationToken` 等）**上移/复制到 `channels.feishu` 的根层级**下，同时保留 `accounts` 节点用于存放其他自定义多账号（如 `xiaoba`）。
     *修改前配置示例：*
     ```json
     "feishu": {
       "accounts": {
         "default": { "appId": "cli_xxx", "appSecret": "xxx" },
         "xiaoba": { "appId": "cli_yyy", "appSecret": "yyy" }
       }
     }
     ```
     *修改后配置示例：*
     ```json
     "feishu": {
       "appId": "cli_xxx",       // default 账号字段移至顶层
       "appSecret": "xxx",       // default 账号字段移至顶层
       "accounts": {
         "xiaoba": { "appId": "cli_yyy", "appSecret": "yyy" }
       }
     }
     ```
  3. **重启网关服务**：重启 OpenClaw 守护进程或网关容器，验证队列中的消息是否开始正常发送。

- **验证状态**: `已确认有效`
