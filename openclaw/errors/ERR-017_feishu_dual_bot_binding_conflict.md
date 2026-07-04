### [ERR-017] 飞书双机器人消息全部路由到同一 Agent (bindings 匹配冲突)

- **适用平台**: `通用 (飞书/Feishu 多机器人部署)`
- **错误现象/日志**:
  - 配置了两个飞书机器人（如 main + artist_bot），但所有消息均被路由到 main agent 处理。
  - artist_bot 机器人虽然能收到消息，但实际执行的 agent 身份和系统提示词与 main 一致。

---

#### 根本原因 (Root Cause)

* **bindings 匹配规则过于宽泛 + 顺序优先级问题**：

  **错误写法**（客户原配置）：
  ```json
  "bindings": [
    { "agentId": "main",   "match": { "channel": "feishu" } },
    { "agentId": "artist", "match": { "channel": "feishu:artist_bot" } }
  ]
  ```
  
  问题在于：
  1. `"channel": "feishu"` 是前缀匹配，会捕获所有飞书消息（包括 artist_bot 的）
  2. bindings 按数组顺序匹配，首条命中即停止，第二条规则永远不会生效
  3. `"channel": "feishu:artist_bot"` 拼接写法也不是正确的 accountId 匹配语法

---

#### 解决方案 (Solution)

**正确写法**（使用 `accountId` 精确匹配）：

```json
"bindings": [
  {
    "agentId": "artist",
    "match": {
      "channel": "feishu",
      "accountId": "artist_bot"
    }
  },
  {
    "agentId": "main",
    "match": {
      "channel": "feishu",
      "accountId": "main"
    }
  }
]
```

同时 `channels.feishu` 部分必须改为**命名 accounts 写法**，每个机器人独立声明：

```json
"channels": {
  "feishu": {
    "enabled": true,
    "domain": "feishu",
    "connectionMode": "websocket",
    "accounts": {
      "main": {
        "appId": "cli_xxx",
        "appSecret": "xxx",
        "groupPolicy": "allowlist",
        "allowFrom": ["*"],
        "groupAllowFrom": ["*"]
      },
      "artist_bot": {
        "appId": "cli_yyy",
        "appSecret": "yyy",
        "groupPolicy": "allowlist",
        "allowFrom": ["*"],
        "groupAllowFrom": ["*"]
      }
    }
  }
}
```

**关键要点**：
1. 使用 `accountId` 字段而非 channel 名称拼接
2. 更具体的规则（artist_bot）放在数组前面，宽泛的（main fallback）放后面
3. 不要把 appId/appSecret 放在 channels.feishu 顶层，全部移入 accounts 子对象

---

- **验证状态**: `已确认有效`
- **实际案例**: 2026-07-03 客户 49.233.244.105 飞书双机器人路由修复
