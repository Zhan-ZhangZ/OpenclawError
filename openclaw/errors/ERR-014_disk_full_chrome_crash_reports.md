### [ERR-014] 服务器磁盘 100% 撑满导致飞书不回复 (Chrome Crash Reports)

- **适用平台**: `Linux 服务器 (启用 browser/Playwright 插件)`
- **错误现象/日志**:
  - 飞书机器人突然无法回复消息，Gateway 服务运行但无输出。
  - `df -h` 显示根分区使用率达 100%，剩余空间为 0。
  - 大量磁盘被 Chrome 崩溃转储文件占用：
    ```text
    /root/.config/google-chrome-for-testing/Crash Reports/pending  →  15GB+
    ```

---

#### 根本原因 (Root Cause)

* **Playwright/Browser 插件的 Chrome 崩溃缓存无限堆积**：
  OpenClaw 启用了 browser 插件后，后台运行的 Chrome for Testing 实例在异常退出时会生成 crash dump 文件。这些文件持续写入 `~/.config/google-chrome-for-testing/Crash Reports/pending/` 目录且永远不会自动清理，最终导致磁盘占满。磁盘满后 SQLite 数据库无法写入，飞书消息队列无法落盘，机器人完全卡死。

---

#### 解决方案 (Solution)

1. **清理 Chrome 崩溃缓存**：
   ```bash
   sudo rm -rf "/root/.config/google-chrome-for-testing/Crash Reports/pending"
   sudo mkdir -p "/root/.config/google-chrome-for-testing/Crash Reports/pending"
   ```

2. **验证磁盘恢复**：
   ```bash
   df -h
   ```

3. **重启 Gateway 服务**：
   ```bash
   sudo XDG_RUNTIME_DIR=/run/user/0 systemctl --user restart openclaw-gateway
   ```

4. **预防措施（可选）**：添加 cron 定期清理：
   ```bash
   echo '0 3 * * 0 rm -rf "/root/.config/google-chrome-for-testing/Crash Reports/pending"/*' | sudo crontab -
   ```

---

- **验证状态**: `已确认有效`
- **实际案例**: 2026-07-01 服务器 106.54.35.73，清理后磁盘从 100% 恢复至 66%
