### [ERR-021] Web UI 打开失败：ERR_EMPTY_RESPONSE (协议不匹配)

- **适用平台**: `云服务器控制台` / `Web 浏览器`
- **错误现象/日志**:
  在云平台（如阿里云/腾讯云等）的“应用管理”或“控制台”点击一键登录 OpenClaw Web UI 时，浏览器弹出一个白屏错误页面，提示：
  ```text
  该网页无法正常运作
  [IP地址] 未发送任何数据。
  ERR_EMPTY_RESPONSE
  ```
  云平台提供的访问链接类似于：`http://[IP]:17616/xxxx#token=xxxx`

---

#### 根本原因 (Root Cause)
此问题是由于**云平台生成的访问链接协议**与 **OpenClaw 服务端的实际监听协议**发生了冲突：
1. **OpenClaw 默认开启了 TLS（HTTPS 加密）**：在服务端的 `openclaw.json` 配置中，`"tls": { "enabled": true }`。这使得 17616 端口处于强加密模式，只能接收并解析 `https://` 的请求。
2. **云平台生成的链接是明文 HTTP**：云平台不知道服务端开启了加密，机械地生成了 `http://` 开头的快捷登录链接。
3. 当浏览器带着 `http://` 的明文请求去敲打 `https` 的强加密端口时，OpenClaw 服务端无法解析该请求，会直接粗暴地断开 TCP 连接。这导致浏览器突然失去连接，从而抛出 `ERR_EMPTY_RESPONSE` 错误。

---

#### 解决方案 (Solution)

有两种解决办法，推荐使用方法一（治本）。

**方法一：修改服务端配置关闭 TLS（推荐，彻底解决云平台链接兼容问题）**
1. SSH 登录到服务器后台。
2. 打开 OpenClaw 的配置文件（通常位于 `/home/admin/.openclaw/openclaw.json` 或 `/etc/openclaw/openclaw.json`）。
3. 找到 `gateway` -> `tls` 配置块，将 `enabled` 的值改为 `false`。
   *可以使用 `jq` 命令行工具一键修改（假设配置文件在 admin 目录下）：*
   ```bash
   jq '.gateway.tls.enabled = false' /home/admin/.openclaw/openclaw.json > /tmp/oc.json && cp /tmp/oc.json /home/admin/.openclaw/openclaw.json
   ```
4. 重启 OpenClaw 服务：
   ```bash
   systemctl restart openclaw
   # 或 pm2 restart openclaw
   ```
5. 此时再去点击云平台上的 `http://` 链接，即可瞬间秒开。

**方法二：手动修改浏览器地址栏（治标）**
如果不方便修改服务器配置，可以直接在浏览器顶部弹出的报错页面的地址栏中，将链接最前面的 **`http://`** 手动修改为 **`https://`** 并回车。
*(注意：此时浏览器可能会报红提示“您的连接不是私密连接”，只需点击下方的“高级” -> “继续前往（不安全）”即可强行进入面板。)*

---

- **验证状态**: `已确认有效`
