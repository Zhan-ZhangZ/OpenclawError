### [ERR-013] 飞书/Lark 插件导出解析方法缺失错误 (resolveFeishuAccount)

- **适用平台**: `通用 (飞书/Feishu 渠道)`
- **错误现象/日志**:
  - 启动 OpenClaw 服务时，飞书通道的机器人账号（模式为 websocket 或 webhook）启动失败并陷入自动重启循环（auto-restart loop）。
  - 控制台或运行日志中出现如下报错信息：
    ```text
    [feishu] starting feishu[xun-yu] (mode: websocket)
    [feishu] [xun-yu] channel exited: The requested module './accounts-CRcvqpsl.js' does not provide an export named 'resolveFeishuAccount'
    [feishu] [xun-yu] auto-restart attempt 4/10 in 41s
    ```

---

#### 根本原因 (Root Cause)

* **插件版本或模块包不兼容**：
  OpenClaw 核心程序在加载飞书通道账号时，需要从安装的飞书插件模块包（如旧版或损坏的 `@openclaw/feishu`）内部加载 `resolveFeishuAccount` 方法。
  由于版本升级引起的 API 不兼容、本地编译缓存残留、或拉取了不匹配版本的依赖包，导致插件的入口分流模块文件（如 `accounts-CRcvqpsl.js`）内未能成功导出该方法，最终引发运行时模块加载异常。

---

#### 解决方案 (Solution)

直接在网关所在的服务器或电脑终端中执行以下命令，使用最新的官方飞书套件包重新覆盖安装并替代原飞书插件：

1. **执行官方覆盖安装指令**：
   ```bash
   npx -y @larksuite/openclaw-lark install
   ```
   *（该指令会使用 `-y` 参数静默同意并下载最新版本的 `@larksuite/openclaw-lark`，覆盖并修复本地不兼容的飞书依赖组件。）*

2. **重启 OpenClaw 网关服务**：
   安装完成后，杀死现有的 openclaw 进程，重新启动网关加载最新依赖：
   ```bash
   openclaw serve --daemon
   ```

---

- **验证状态**: `已确认有效`
