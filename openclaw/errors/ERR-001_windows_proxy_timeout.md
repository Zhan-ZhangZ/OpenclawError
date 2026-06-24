### [ERR-001] Windows 代理环境变量异常导致模型连接超时

- **适用平台**: `Windows`
- **错误现象/日志**:
  - OpenClaw 运行时模型回答无响应、提示超时，且无其他明显报错。
  - 即使重新执行 `openclaw onboard --install-daemon` 引导重新设置模型配置，也无法解决该问题。
  - 在 PowerShell 中使用 `Invoke-WebRequest` / `curl` 访问服务商 API 网址（如 `https://api.deepseek.com`）可以正常返回 200，但 OpenClaw 内部依然报错超时。

---

#### 根本原因 (Root Cause)
系统或当前用户环境变量中设置了异常或残留的代理配置（如 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 等）。
OpenClaw 及其底层的网络请求库（如 Node.js fetch/axios）在发起 HTTP/HTTPS 请求时，会自动读取这些环境变量并试图通过它们路由流量。如果代理服务器已关闭、配置不正确或不支持该服务，就会导致模型请求在代理节点处挂起，最终导致超时，而外部独立的 PowerShell 测试由于不走该代理可能表现为正常。

---

#### 解决方案 (Solution)

在 Windows PowerShell 中执行以下步骤，彻底清空当前会话、用户级、系统级代理环境变量以及 WinHTTP 代理和注册表代理设置。

##### 步骤 1：清空当前会话的临时代理环境变量
```powershell
Remove-Item Env:HTTP_PROXY -ErrorAction SilentlyContinue
Remove-Item Env:HTTPS_PROXY -ErrorAction SilentlyContinue
Remove-Item Env:ALL_PROXY -ErrorAction SilentlyContinue
Remove-Item Env:NO_PROXY -ErrorAction SilentlyContinue

Remove-Item Env:http_proxy -ErrorAction SilentlyContinue
Remove-Item Env:https_proxy -ErrorAction SilentlyContinue
Remove-Item Env:all_proxy -ErrorAction SilentlyContinue
Remove-Item Env:no_proxy -ErrorAction SilentlyContinue
```

##### 步骤 2：清除持久的“用户(User)”级别环境变量
```powershell
[Environment]::SetEnvironmentVariable("HTTP_PROXY", $null, "User")
[Environment]::SetEnvironmentVariable("HTTPS_PROXY", $null, "User")
[Environment]::SetEnvironmentVariable("ALL_PROXY", $null, "User")
[Environment]::SetEnvironmentVariable("NO_PROXY", $null, "User")
```

##### 步骤 3：清除持久的“系统/计算机(Machine)”级别环境变量
```powershell
[Environment]::SetEnvironmentVariable("HTTP_PROXY", $null, "Machine")
[Environment]::SetEnvironmentVariable("HTTPS_PROXY", $null, "Machine")
[Environment]::SetEnvironmentVariable("ALL_PROXY", $null, "Machine")
[Environment]::SetEnvironmentVariable("NO_PROXY", $null, "Machine")
```

##### 步骤 4：重置 WinHTTP 代理
```powershell
# 查看当前 WinHTTP 代理设置
netsh winhttp show proxy

# 重置 WinHTTP 代理为直连模式
netsh winhttp reset proxy
```

##### 步骤 5：关闭注册表中的 Internet 选项代理设置
```powershell
# 查看当前注册表代理状态
Get-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" ProxyEnable,ProxyServer

# 禁用注册表代理设置（将其值设为 0）
Set-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" ProxyEnable 0
```

##### 步骤 6：验证清除结果与重启
1. 在当前终端执行以下命令，确认无代理相关输出：
   ```powershell
   Get-ChildItem Env: | findstr /i proxy
   [Environment]::GetEnvironmentVariable("HTTPS_PROXY", "User")
   [Environment]::GetEnvironmentVariable("HTTPS_PROXY", "Machine")
   ```
2. **非常重要**：关闭当前的 PowerShell 窗口，**新开一个终端窗口**以使环境变量的修改完全生效。
3. 重新启动 OpenClaw，即可正常调用模型。

---

- **验证状态**: `已确认有效`
