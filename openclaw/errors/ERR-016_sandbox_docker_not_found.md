### [ERR-016] Docker 沙箱未安装导致工具执行失败 (spawn docker ENOENT)

- **适用平台**: `Windows / 未安装 Docker 的环境`
- **错误现象/日志**:
  - Agent 无法执行任何工具（read/write/exec），每次调用均报错。
  - 控制面板显示红色错误信息：
    ```text
    Agent failed before reply: Sandbox mode requires Docker, but the 'docker' command 
    was not found in PATH. Install Docker (and ensure 'docker' is available) or set 
    `agents.defaults.sandbox.mode=off` to disable sandboxing. | spawn docker ENOENT | ENOENT.
    ```

---

#### 根本原因 (Root Cause)

* **沙箱模式默认启用但宿主机未安装 Docker**：
  OpenClaw 的 Agent 默认在 Docker 容器内执行工具操作以实现安全隔离。如果宿主机上未安装 Docker Desktop（Windows/Mac）或 Docker Engine（Linux），所有工具调用会因找不到 `docker` 命令而立即失败。

---

#### 解决方案 (Solution)

在 `openclaw.json` 中添加关闭沙箱的配置：

```json
"agents": {
  "defaults": {
    "sandbox": {
      "mode": "off"
    }
  }
}
```

修改后重启 Gateway 即可。

---

- **验证状态**: `已确认有效`
