### [ERR-007] Clash Fake-IP 劫持触发 SsrfBlockedError 导致 Gemini CLI OAuth 登录失败

- **适用平台**: `通用 (常见于使用 Clash/Surge 等代理软件 Fake-IP 模式的环境，包括 Docker 部署和 macOS/Windows 本地安装)`
- **错误现象/日志**:
  - 执行 Google Gemini OAuth 登录（例如在终端执行 `openclaw models auth login --provider google-gemini-cli`），在浏览器完成授权并把重定向回调 URL 粘贴回控制台后，登录被强制中断。
  - 报错信息如下：
    ```text
    09:58:18 [security] blocked URL fetch (url-fetch) targetOrigin=https://cloudcode-pa.googleapis.com reason=Blocked: resolves to private/internal/special-use IP address
    Gemini CLI OAuth failed
    Trouble with OAuth? Ensure your Google account has Gemini CLI access.
    SsrfBlockedError: Blocked: resolves to private/internal/special-use IP address
    ```

---

#### 根本原因 (Root Cause)
当本地开启了 Clash、Surge 等科学上网软件且使用的是 **Fake-IP（虚拟 IP）模式**时，任何经过代理的 DNS 解析请求都会被劫持并返回一个 `198.18.0.0/15` 段的虚拟 IP（如 `198.18.1.75`）。
由于该网段是 RFC 2544 标准规定的基准测试/特殊用途保留网段，OpenClaw 内置的安全请求拦截器（SSRF 守卫）在拦截出站请求并检查 IP 时，会判定该请求目的地为"局域网/内网/特殊用途 IP"，从而为了防御 SSRF（服务端请求伪造）攻击而强制拒绝请求。导致 Gemini CLI 与谷歌验证服务器的通信在本地被强行斩断。

---

#### 解决方案 A：Docker 部署环境

在 Docker 配置文件中使用 **静态域名-IP 绑定**，将谷歌 OAuth 的相关接口直接指向真实的公网 IP，避开 Clash DNS 的 Fake-IP 劫持，进而通过 SSRF 安全校验。

##### 步骤 1：解析谷歌 OAuth 接口的真实公网 IP
为了防止本地解析结果被 Clash 劫持，可以使用 DNS over HTTPS（DoH）向谷歌公共 DNS 发送安全查询，获取真实 IP：
```bash
curl -s "https://dns.google/resolve?name=cloudcode-pa.googleapis.com"
curl -s "https://dns.google/resolve?name=daily-cloudcode-pa.sandbox.googleapis.com"
curl -s "https://dns.google/resolve?name=autopush-cloudcode-pa.sandbox.googleapis.com"
```
记下返回 JSON 的 `"data"` 字段对应的 IP 地址（例如：`142.251.215.170`）。

##### 步骤 2：在 Docker-Compose 中配置静态解析
打开工作区目录下的 `docker-compose.yml`，在 `openclaw-gateway` 与 `openclaw-cli` 服务的 `extra_hosts` 配置项中添加这三个域名的真实 IP 映射：
```yaml
    extra_hosts:
      - "oauth2.googleapis.com:142.250.101.95"
      - "www.googleapis.com:142.251.215.170"
      - "generativelanguage.googleapis.com:142.251.215.202"
      # 新增谷歌验证域名解析映射：
      - "cloudcode-pa.googleapis.com:142.251.215.170"
      - "daily-cloudcode-pa.sandbox.googleapis.com:142.250.101.81"
      - "autopush-cloudcode-pa.sandbox.googleapis.com:142.250.141.81"
```

##### 步骤 3：重建容器并重新登录
1. 运行以下命令，强制重建容器以让 `extra_hosts` 静态解析生效：
   ```bash
   docker compose up -d --force-recreate
   ```
2. 重新在容器中安装依赖的 `gemini-cli` 工具（若使用容器运行）：
   ```bash
   docker exec -u root openclaw-in-docker-openclaw-gateway-1 npm install -g @google/gemini-cli
   ```
3. 重新在容器控制台运行登录指令，复制粘贴重定向网址即可一次性通过验证。

---

#### 解决方案 B：macOS 本地安装环境

如果 OpenClaw 是直接在 macOS 上通过 `npm install -g openclaw` 安装运行（非 Docker），则需要修改系统 `/etc/hosts` 文件来绕过 Fake-IP。

##### 步骤 1：解析谷歌 OAuth 接口的真实公网 IP
同方案 A，使用 DoH 查询真实 IP（该命令本身走 HTTPS 不受 Clash DNS 劫持影响）：
```bash
curl -s "https://dns.google/resolve?name=cloudcode-pa.googleapis.com" | python3 -c "import sys,json; [print(a['data']) for a in json.load(sys.stdin).get('Answer',[])]"
curl -s "https://dns.google/resolve?name=daily-cloudcode-pa.sandbox.googleapis.com" | python3 -c "import sys,json; [print(a['data']) for a in json.load(sys.stdin).get('Answer',[])]"
curl -s "https://dns.google/resolve?name=autopush-cloudcode-pa.sandbox.googleapis.com" | python3 -c "import sys,json; [print(a['data']) for a in json.load(sys.stdin).get('Answer',[])]"
curl -s "https://dns.google/resolve?name=oauth2.googleapis.com" | python3 -c "import sys,json; [print(a['data']) for a in json.load(sys.stdin).get('Answer',[])]"
curl -s "https://dns.google/resolve?name=www.googleapis.com" | python3 -c "import sys,json; [print(a['data']) for a in json.load(sys.stdin).get('Answer',[])]"
```
记下每个域名返回的 **A 记录**（IPv4 地址，如 `142.251.215.170`），忽略 CNAME 记录。

##### 步骤 2：编辑 macOS 系统 hosts 文件
用 `sudo` 权限编辑 `/etc/hosts`：
```bash
sudo nano /etc/hosts
```
在文件末尾追加以下内容（IP 地址替换为步骤 1 查到的实际值）：
```text
# === OpenClaw Gemini OAuth - 绕过 Clash Fake-IP ===
142.251.215.170  cloudcode-pa.googleapis.com
142.250.101.81   daily-cloudcode-pa.sandbox.googleapis.com
142.250.141.81   autopush-cloudcode-pa.sandbox.googleapis.com
142.250.101.95   oauth2.googleapis.com
142.251.215.170  www.googleapis.com
142.251.215.202  generativelanguage.googleapis.com
# === END ===
```

##### 步骤 3：刷新 macOS DNS 缓存
保存 hosts 文件后，必须刷新系统 DNS 缓存才能生效：
```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

##### 步骤 4：验证解析是否绕过了 Fake-IP
```bash
# 应该返回你在 hosts 中填写的真实公网 IP，而不是 198.18.x.x
ping -c 1 cloudcode-pa.googleapis.com
```
如果看到的是 `198.18.x.x`，说明 hosts 没有生效，检查：
- hosts 文件格式是否正确（IP 和域名之间用空格或 Tab 分隔）
- DNS 缓存是否已刷新
- Clash 是否在系统级别接管了 DNS（需要在 Clash 中将这些域名加入**直连规则**）

##### 步骤 5：重新登录
```bash
openclaw models auth login --provider google-gemini-cli
```
完成浏览器授权后粘贴回调 URL，应可一次通过。

##### ⚠️ 注意事项
- **IP 地址会变**：Google 的服务 IP 不是固定的，如果过一段时间后再次遇到连接问题，需要重新执行步骤 1 查询最新 IP 并更新 hosts。
- **Clash 直连规则（更优方案）**：如果不想手动维护 hosts，可以在 Clash 配置中将 `*.googleapis.com` 域名加入**直连规则**或使用 **Real-IP 模式**匹配这些域名，从根本上避免 Fake-IP 劫持：
  ```yaml
  # 在 Clash 配置的 dns.fake-ip-filter 中添加：
  dns:
    fake-ip-filter:
      - "*.googleapis.com"
  ```
  这样这些域名的 DNS 解析会返回真实 IP，无需手动改 hosts。

---

- **验证状态**: `已确认有效（Docker 方案）/ 待验证（macOS 本地方案，逻辑等价）`
