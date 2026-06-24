### [ERR-011] Hermes 配置自定义中转站报错 (unknown provider)

- **适用平台**: `Hermes v0.16.0+`
- **错误现象/日志**:
  - 启动 Hermes 后台服务或通过 WebUI 发送消息时，请求失败，并在日志中提示被回退（Fallback）到了其他默认提供商（如 openrouter）。
  - 底层控制台或 `~/.hermes/logs/agent.log` 日志中出现如下警告：
    ```text
    WARNING agent.auxiliary_client: resolve_provider_client: unknown provider 'maimai'
    ```
    或者使用通用的 openai 名字也会出现：
    ```text
    WARNING agent.auxiliary_client: resolve_provider_client: unknown provider 'openai'
    ```

---

#### 根本原因 (Root Cause)

1. **配置区域用错（误入白名单）**：用户往往凭直觉，将自创的中转站名称（如 `maimai`）直接添加到了 `config.yaml` 的 `providers:` 字典区域下。但 `providers:` 实则是一个受底层代码严格管控的**官方内置白名单区域**，只认识原生支持的大厂名称（如 `deepseek`、`openrouter`）。一旦读到不认识的名字，就会直接拦截并抛出 `unknown provider`。
2. **通用 `openai` 外壳被废弃**：在一些较旧的开源 AI Agent 项目中，常规做法是把中转站名字改成 `openai` 借壳生效。但在较新版的 Hermes（v0.16.0 及以上版本）中，官方从白名单里**删除了通用无后缀的 `openai` 名称**。因此，旧版经验在此失效。

---

#### 解决方案 (Solution)

##### 最佳实践：使用官方引导菜单配置

为了避免手动修改 YAML 带来的缩进问题和结构错误，强烈推荐使用 Hermes 内置的向导。

1. 在终端中运行指令：
   ```bash
   hermes model
   ```
2. 在弹出的提供商列表（Select provider）中，使用键盘方向键向下找到并选中：
   `> (o) custom (direct API)`
3. 接着在子菜单中选择手动输入配置：
   `> (o) Custom endpoint (enter URL manually)`
4. 根据终端向导的提示，依次输入完整的、兼容 OpenAI 协议的 API 地址（例如 `https://maimai.it.com/v1`）和 API 密钥。遇到兼容模式选择时，默认回车选择 `Auto-detect` 即可。
5. **最重要的一步：重启服务**。配置写完后，必须终止旧的后台进程（如 `hermes dashboard --stop` 或 `Ctrl+C`）并重新启动服务，新配置才会载入内存。

##### 进阶：手动修改 `config.yaml` 文件的标准规范

如果出于自动化部署或其他原因必须手动修改文件，需按照以下**上下两步联动**的格式修改：

1. **在文件末尾新建自定义列表**：必须在文件的空白处（不要放入 `providers:` 字典内部）新建一个 `custom_providers:` 列表。
   ```yaml
   custom_providers:
   - name: maimai                           # 自定义端点名称
     base_url: https://maimai.it.com/v1     # 需包含完整的兼容路径
     key_env: MAIMAI_API_KEY                # 推荐使用环境变量引用密钥
   ```

2. **在顶部设置全局调用时，加上特定前缀**：
   要想让顶层的主配置识别到底部的自定义列表，必须在提供商名字前加上 **`custom:`** 前缀作为底层查找的“暗号”。
   ```yaml
   model:
     base_url: ''                       
     default: gpt-5.5                   
     provider: custom:maimai            # ⚠️ 核心秘诀：千万别漏了 `custom:` 
   ```

保存后，重启进程即可生效。

---

- **验证状态**: `已确认有效`
