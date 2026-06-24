### [ERR-008] OpenAI OAuth 登录令牌交换超时（SSRF 守卫忽略环境代理）

- **适用平台**: `通用 (国内网络环境下使用代理的系统，特别是 WSL)`
- **错误现象/日志**:
  在执行 OpenAI Codex OAuth 认证登录时，粘贴回调 URL 后控制台等待 30 秒超时并报错：
  ```text
  [fetch-timeout] fetch timeout after 30000ms
  url=https://auth.openai.com/oauth/token

  Error: OpenAI Codex token exchange timed out after 30000ms
  ```
- **根本原因 (Root Cause)**:
  OpenClaw 的 OAuth 令牌交换流程内部使用了安全出站过滤器 `fetchWithSsrFGuard`。该过滤器以严格直连的方式发送请求，从而忽略了 Node 原生环境代理设置（如 `HTTP_PROXY`、`HTTPS_PROXY`、`NODE_USE_ENV_PROXY` 等）。在国内直连 OpenAI 会被长城防火墙（GFW）阻断，导致连接超时。
- **解决方案 (Solution)**:
  1. **手动修改打包运行时文件**：
     打开或编辑 OpenClaw 全局或本地包中的以下文件（具体哈希后缀可能不同）：
     `~/.npm-global/lib/node_modules/openclaw/dist/openai-chatgpt-oauth-flow.runtime-Bxb_5P_d.js`
  2. **将令牌交换的请求参数修改为信任并使用环境代理**：
     ```js
     fetchWithSsrFGuard({
       mode: "trusted_env_proxy",
       proxy: "env",
       dangerouslyAllowEnvProxyWithoutPinnedDns: true,
       url: TOKEN_URL,
       // ... 其他原有参数
     });
     ```
  3. **重新执行登录**：重新运行 `openclaw models auth login --provider openai`，请求将在 3 秒内成功送达并完成授权。
- **验证状态**: `已确认有效`
