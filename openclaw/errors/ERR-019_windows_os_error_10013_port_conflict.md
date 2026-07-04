### [ERR-019] Windows 系统下 Codex 登录失败 (os error 10013) 端口被保留冲突

- **适用平台**: `Windows`
- **错误现象/日志**:
  - 打开 Codex 时，启动页上方弹出红色报错提示：`登录失败: failed to start login server: 以一种访问权限不允许的方式做了一个访问套接字的尝试。(os error 10013)`。
  - 无法正常进入 Codex 主界面，卡在“开始使用 Codex / 使用 ChatGPT 登录”界面。

---

#### 根本原因 (Root Cause)
`os error 10013` 是 Windows 系统的套接字权限错误（WSAEACCES），通常表示请求的端口被其他进程占用，或者**更常见的情况是：该端口被 Windows NAT (WinNAT) 服务或 Hyper-V 虚拟机平台作为动态端口范围保留并锁定了**。
当 Codex 尝试启动本地的 Login Server 监听特定端口时，如果该端口恰好落入了 WinNAT 随机保留的端口排除范围（Port Exclusion Range）内，系统就会拒绝访问，抛出 10013 错误。

---

#### 解决方案 (Solution)

需要以管理员身份重置 TCP 的动态端口排除范围，并重启 WinNAT 服务以释放被异常占用的端口。

##### 步骤 1：以管理员身份运行命令行
在 Windows 开始菜单搜索 `PowerShell` 或 `cmd`，右键点击并选择“**以管理员身份运行**”。

##### 步骤 2：停止 WinNAT 服务
在终端中输入以下命令并回车，暂时停止 Windows NAT 网络服务：
```powershell
net stop winnat
```

##### 步骤 3：重置 TCP 动态端口范围
输入以下命令并回车，将 IPv4 和 IPv6 的 TCP 动态端口起始值调整为大于 1455 的数值（如 1555），以此释放被系统意外保留占用的低位端口：
```powershell
netsh int ipv4 set dynamicport tcp start=1555 num=64000
netsh int ipv6 set dynamicport tcp start=1555 num=64000
```

##### 步骤 4：恢复启动 WinNAT 服务
输入以下命令并回车，重新启动 WinNAT 服务：
```powershell
net start winnat
```

##### 步骤 5：验证结果
完成以上命令后，关闭命令行窗口。重新打开 Codex 应用程序，登录服务即可正常启动，不再报 `os error 10013`。

---

- **验证状态**: `已确认有效`
