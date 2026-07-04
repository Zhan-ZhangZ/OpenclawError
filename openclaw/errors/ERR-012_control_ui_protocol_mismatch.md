### [ERR-012] OpenClaw Control UI 协议版本不匹配报错 (protocol mismatch)

- **适用平台**: `OpenClaw 所有平台`
- **错误现象/日志**:
  - 在 OpenClaw 网页端仪表盘 (OpenClaw Dashboard) 界面中，连接状态栏显示红色报错：`protocol mismatch`。
  - 在 OpenClaw 网关后台终端或日志文件中，出现类似如下的 WebSocket 连接错误信息：
    ```text
    00:13:49 [ws] protocol mismatch conn=7ecb3fee-1b16-4585-950e-ebb5673a44f6 peer=127.0.0.1:51375->127.0.0.1:18789 remote=127.0.0.1 client=openclaw-control-ui webchat v2026.5.5 min=3 max=3 expected=4 probeMin=4
    00:13:49 [ws] closed before connect conn=7ecb3fee-1b16-4585-950e-ebb5673a44f6 peer=127.0.0.1:51375->127.0.0.1:18789 remote=127.0.0.1 fwd=n/a origin=http://127.0.0.1:18789 ua=... code=1002 reason=unauthorized: protocol mismatch
    ```

---

#### 根本原因 (Root Cause)

* **客户端与服务端协议版本不一致**：
  客户在浏览器中打开的 **网页端仪表盘 (Control UI)** 版本过旧（例如日志中显示的协议版本仅支持 `min=3 max=3` 的 `webchat v2026.5.5`），而本地正在运行的 **OpenClaw 网关 (Gateway)** 服务已经升级，要求对接更高的通信协议版本（例如日志中要求的 `expected=4` 或 `probeMin=4`）。
* **常见触发场景**：
  客户直接访问了浏览器中历史缓存的旧版网页端书签，或者直接双击打开了以往下载的旧版离线 Web 页面文件，而没有通过当前网关重新生成对应的仪表盘地址。

---

#### 解决方案 (Solution)

##### 步骤 1：通过命令行重新拉起匹配的仪表盘
在运行 OpenClaw 网关的电脑（Windows / macOS / Linux）终端中，执行以下指令重新生成与当前网关版本完全配套的控制台链接：
```bash
npx openclaw dashboard
```
*(如果本地已经全局安装了 openclaw，也可以直接执行 `openclaw dashboard`)*

运行后，程序会自动获取当前网关最新的 Web 控制台 URL（已匹配对应协议和最新 Token），并在浏览器中自动拉起新版控制台，此时连接将恢复正常。

##### 步骤 2：若仍不匹配，整体更新 openclaw
如果本地网关和 CLI 版本出现混乱，建议将其统一更新至最新版本：
1. 更新命令：
   ```bash
   npm install -g openclaw@latest
   ```
2. 重新启动服务：
   ```bash
   openclaw serve --daemon
   ```
3. 重新获取仪表盘页面：
   ```bash
   openclaw dashboard
   ```

---

- **验证状态**: `已确认有效`
