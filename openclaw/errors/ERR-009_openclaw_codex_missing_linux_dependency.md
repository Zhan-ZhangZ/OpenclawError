### [ERR-009] WSL 环境下 Codex 启动提示缺少 @openai/codex-linux-x64 原生依赖

- **适用平台**: `WSL (Linux x64)`
- **错误现象/日志**:
  OAuth 登录完成后，Dashboard 中启动嵌入式 Agent 失败，控制台或系统弹窗报错：
  ```text
  Error: Missing optional dependency @openai/codex-linux-x64.
  Reinstall Codex: npm install -g @openai/codex@latest
  ```
  报错源码文件路径位于：
  `~/.openclaw/npm/projects/openclaw-codex-XXXXXX/node_modules/@openclaw/codex/node_modules/@openai/codex/bin/codex.js`
- **根本原因 (Root Cause)**:
  在 WSL（Linux x64）环境下，`@openai/codex` 运行时需要对应的平台特定二进制依赖包 `@openai/codex-linux-x64`（其对应的是 `npm:@openai/codex@0.135.0-linux-x64`）。OpenClaw 的独立沙箱项目在自动安装插件依赖时，未能完整下载或拉取该可选依赖（Optional Dependency），导致运行时报错。
- **解决方案 (Solution)**:
  1. **进入对应的隔离项目路径**，或直接通过 `--prefix` 在隔离项目路径中补装缺失的可选依赖包。运行以下命令：
     ```bash
     npm install \
       --prefix ~/.openclaw/npm/projects/openclaw-codex-8902d781d4 \
       --no-save \
       --include=optional \
       @openai/codex-linux-x64@npm:@openai/codex@0.135.0-linux-x64
     ```
     *(注意：将命令中的 `openclaw-codex-8902d781d4` 替换为您实际报错路径中的文件夹名称)*
  2. **验证是否修复成功**：
     运行 `codex app-server --help`，正常输出帮助信息不报错即代表成功。
- **验证状态**: `已确认有效`
