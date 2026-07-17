### [ERR-022] Windows 下新版本 ACL 权限校验失败导致启动拦截

- **适用平台**: `Windows`
- **错误现象/日志**:
  - OpenClaw 从旧版本（如 `2026.6.6`）升级到 `2026.7.1` 等新版本后，执行 `openclaw gateway` 启动网关服务失败。
  - 报错信息提示权限未验证，且明确指出 ACL 不可用：`SecretProviderResolutionError: secrets.providers.openclaw-file.path ACL verification unavailable on Windows for C:\Users\...\.openclaw\secrets.json | permission-unverified`。

---

#### 根本原因 (Root Cause)
新版本 OpenClaw 针对存放敏感密钥的 `secrets.json` 引入了强制的本地文件权限校验（ACL Verification）机制，以防止越权读取。
但由于 Windows 系统的访问控制列表（ACL）机制与 Unix-like 系统差异较大，当前版本的代码实现无法在 Windows 平台上完成该校验（即抛出 `unavailable on Windows`）。基于“安全第一”的兜底拦截策略，程序拒绝加载未经验证的凭据，从而中断启动。这属于新版本在 Windows 平台的兼容性 Bug（破坏性变更）。

---

#### 解决方案 (Solution)

在官方提供 Windows 端的 ACL 修复补丁之前，最彻底的临时解决方案是：**放弃分离管理机制，将密钥直接写回 `openclaw.json` 主配置文件中，并移除对独立密钥文件的配置引用。**

##### 步骤 1：移除主配置中的 secrets 块
打开主配置文件（例如 `C:\Users\Administrator\.openclaw\openclaw.json`），在文件中找到并彻底删除顶部的整个 `"secrets"` 配置块：

```json
  "secrets": {
    "providers": {
      "openclaw-file": {
        ...
      }
    }
  },
```
删除后，程序启动时将不再尝试加载独立的密钥文件。

##### 步骤 2：将密钥直接写入主配置
将原 `secrets.json` 中的实际密钥，覆盖主配置中对应的引用位置。
例如：
- 把 `"apiKey": {"source": "file", "provider": "openclaw-file", "id": "/vllm_api_key"}` 
- 替换为 `"apiKey": "真实的_api_key_字符串"`
（对其他如各 channel 的 `appSecret`、gateway 的 `token` 执行同样处理）

##### 步骤 3：清理残留文件
将该目录下的 `secrets.json` 文件删除或重命名（如 `secrets.json.bak`），确保文件不留存。

##### 步骤 4：验证重启
保存主配置文件后，新开 PowerShell 并重新运行 `openclaw gateway`。

---

- **验证状态**: `已确认有效`
