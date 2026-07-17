### [ERR-020] OpenClaw 崩溃：gateway tls failed to load cert (EACCES)

- **适用平台**: `Linux` (CentOS/Ubuntu/Debian 等服务端)
- **错误现象/日志**:
  程序无法启动，且在日志文件（如 `/home/admin/.openclaw/logs/stability/*startup_failed.json`）中存在如下 JSON 报错信息：
  ```json
  "error": {
    "name": "Error",
    "message": "gateway tls: failed to load cert (Error: EACCES: permission denied, open '/etc/openclaw/tls/server.key')"
  }
  ```

---

#### 根本原因 (Root Cause)
这是一个典型的 **Linux 文件权限越界问题 (Permission Denied)**。
OpenClaw 核心网关进程通常由普通业务用户（例如 `admin` 或 `ubuntu`）运行，而 TLS 加密所需的 SSL 证书（尤其是私钥 `server.key`）往往是 root 账户通过脚本自动生成或申请的，导致该密钥文件的所属权（Ownership）停留在 `root:root`，并且权限极为严格（如 `600`）。
当 `admin` 用户的 OpenClaw 尝试挂载这个只允许 root 摸的 `server.key` 证书时，会被 Linux 操作系统无情拒绝，从而报出 `EACCES: permission denied` 并引发启动失败闪退。

---

#### 解决方案 (Solution)

需要通过 SSH 登录到服务器，将证书文件夹的归属权移交给启动 OpenClaw 的那个业务账号（假设账号是 `admin`）。

**执行以下修复命令：**
1. **移交归属权 (Chown)**：
   让 `admin` 用户成为证书文件夹的新主人：
   ```bash
   sudo chown -R admin:admin /etc/openclaw/tls
   ```
   *(注：如果运行 OpenClaw 的是其他账号如 ubuntu，请将 admin 替换为 ubuntu)*

2. **修正私钥文件权限 (Chmod)**：
   确保私钥只能被主人读取（防泄漏），同时保证程序能顺畅加载它：
   ```bash
   sudo chmod 600 /etc/openclaw/tls/server.key
   sudo chmod 644 /etc/openclaw/tls/server.crt
   ```

3. **重启服务**：
   执行完权限转移后，重新拉起 OpenClaw：
   ```bash
   pm2 restart openclaw
   # 或者
   systemctl restart openclaw
   ```

---

- **验证状态**: `已确认有效`
