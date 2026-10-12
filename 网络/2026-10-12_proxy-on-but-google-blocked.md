# 梯子开着却上不了 Google：DNS 投毒 + 订阅后端失效 + 浏览器扩展覆盖代理，三重叠加

> 日期：2026-10-12 · 环境：Arch Linux · niri + FlClash · 症状：国内网站正常，Google/ChatGPT 全部打不开

## 现象

- 代理（FlClash）显示已连接、TUN 也开了，但**Google 搜索、ChatGPT 完全打不开**
- 国内网站（百度等）访问正常
- Google 一直卡在 **"Verify it's you"** 验证页，怎么都过不去

四个看似独立的现象，实际是**三个不同层面**的问题叠在一起。

## 排查过程与根因

### 第一层：DNS 被投毒（系统解析器层面）

```bash
$ getent hosts www.google.com
104.244.42.197      # ← 这是 Twitter 的 IP！
2001::1             # ← 保留地址，根本不可达
```

系统 DNS 被污染，`www.google.com` 被解析到**完全无关的 IP**。
而走代理时是用代理端解析的，所以：

```bash
$ curl -x http://127.0.0.1:7890 https://www.google.com/    # 走代理
200                                    # ✅ 通
$ curl https://www.google.com/                              # 直连
(超时 10s)                              # ❌ 死
```

**关键判据**：`走代理通、直连不通` ≠ 代理有问题。说明**部分流量没走代理**。

### 第二层：订阅后端已失效，节点全是死地址

换的新订阅打开后，节点列表看起来正常（有名字、能选），但：

```yaml
proxies:
- name: "🇦🇱 套餐已过期"         server: 127.0.0.1  port: 443
- name: "🇦🇱 请及时续费"         server: 127.0.0.1  port: 443
- name: "🇦🇱 续费地址：kitty.fo"  server: 127.0.0.1  port: 443
```

**订阅过期了** —— 机场不再下发真实节点，改塞占位节点，服务器地址全指向本机 `127.0.0.1:443`，
而那个端口上没有任何程序监听。节点名本身就是催续费通知。

配合它的分流规则：

```yaml
- RULE-SET,google,节点选择        # Google → 节点选择
- RULE-SET,overseas-ai,节点选择   # ChatGPT → 节点选择
- RULE-SET,cn,DIRECT             # 国内 → 直连
- MATCH,节点选择
```

于是必然结果：**Google/ChatGPT 被丢给一个死地址 → 超时转圈；国内走 DIRECT → 正常**。
这正是"国内能开、国外全挂"的经典形态。

> 续费后**必须重新拉取订阅**才会下发新节点，光续费不更新是不生效的。

### 第三层：浏览器扩展覆盖了系统代理

续费、换节点后浏览器仍旧不行。查 Chrome 扩展发现装了 **Proxy SwitchyOmega (V3)**，
而它的配置**全是安装时的模板默认值，从没配对**：

| Profile | 内容 | 后果 |
|---|---|---|
| `auto switch` | 默认 `direct`，规则只有 `*.example.com` | **浏览器全部直连** |
| `proxy` | `proxy.example.com:8080` | 死地址（域名无法解析） |

`config` 里搜 `7890`（FlClash 端口）→ **0 处**。

**根因**：SwitchyOmega 是浏览器级代理控制器，**它的设置优先于系统代理**。
它一旦启用，Chrome 就不再看 GNOME 的系统代理设置，而是走它自己的 profile ——
而它配置里没有 FlClash，所以 **Chrome 一直在直连**。

同时 Chrome 的 `Preferences` 里没有任何 `proxy` 键、也没有命令行代理参数，
说明代理完全由扩展接管。

### 附带发现：Google 的 "Verify it's you" 为什么过不去

修好代理后仍卡验证页，查出出口是**机房 IP**：

```bash
$ curl -x http://127.0.0.1:7890 http://ip-api.com/line/?fields=country,isp,hosting
Germany
Hetzner Online GmbH
true          # ← 机房 IP
```

Google 对机房 IP 风控严格，会反复要求人机验证；而节点延迟 1.9~3.8 秒且剧烈抖动，
reCAPTCHA 的资源根本加载不完 → **永远停在验证页**。

改走日本节点（延迟降到 0.5 秒档）后恢复正常。

## 修复

```bash
# 1. 换掉被投毒的 DNS（改用走代理的加密 DNS）
#    在 FlClash 配置里把域名解析改为经由节点：
#    nameserver:
#      - https://dns.cloudflare.com/dns-query#节点选择
#      - https://dns.google/dns-query#节点选择
#    respect-rules: true
#    enhanced-mode: fake-ip

# 2. 订阅过期 → 续费后在 FlClash 里重新更新订阅（关键：要拉取）
#    续费后节点从 4 个占位变成 36 个真实节点

# 3. 处理 SwitchyOmega
#    方案 A：禁用扩展，让 Chrome 回落到系统代理
#    方案 B：新建一个 FixedProfile → HTTP 127.0.0.1:7890，并切到它
#    验证：chrome://net-internals/#proxy 应显示 127.0.0.1:7890

# 4. 换低延迟、非机房的节点
```

## 验证

```bash
# 节点延迟（同一节点连测多次看抖动）
for i in 1 2 3; do
  curl -s -o /dev/null -x http://127.0.0.1:7890 \
    -w "http=%{http_code} TLS=%{time_appconnect}s 总计=%{time_total}s\n" \
    https://www.google.com/
done

# 出口 IP 与类型（hosting=true 表示机房 IP，容易被风控）
curl -s -x http://127.0.0.1:7890 \
  "http://ip-api.com/line/?fields=country,isp,hosting"

# ChatGPT 要分开看：网页 403 但 API 401 说明是 Cloudflare 拦 IP，不是没连上
curl -s -o /dev/null -x http://127.0.0.1:7890 -w "chatgpt.com: %{http_code}\n" https://chatgpt.com/
curl -s -o /dev/null -x http://127.0.0.1:7890 -w "api.openai.com: %{http_code}\n" https://api.openai.com/v1/models
```

## 教训

1. **"代理开着但上不了网"要分层排查，不要笼统归因于"梯子坏了"。**
   三个独立层面：**DNS 解析** → **节点可用性** → **流量是否真的走了代理**。
   每一层的验证方法不同：
   ```bash
   getent hosts <域名>                      # DNS 层
   curl -x <代理> <url>                     # 代理层（对比直连）
   chrome://net-internals/#proxy            # 浏览器是否真的在用代理
   ```

2. **「走代理通、直连不通」= 代理没问题，是部分流量绕过了代理。**
   这条判据能把排查方向从"换节点"纠正到"查谁没走代理"。

3. **节点列表有名字 ≠ 节点可用。**
   订阅过期时机场会下发指向 `127.0.0.1` 的占位节点，
   界面上看不出异常。**要用 curl 实测，不要看列表。**

4. **浏览器扩展能覆盖系统代理，系统层面的配置对它无效。**
   装了代理类扩展时，系统代理设置可能完全不生效。
   排查浏览器网络问题**必须先确认扩展状态**。

5. **机房 IP 会被 Google/OpenAI 风控。**
   `ip-api.com` 查 `hosting=true` 即可识别。
   表现是反复人机验证、403，而不是连不上 —— 别误判成网络故障。

6. **高延迟会伪装成"打不开"。**
   1.9~3.8 秒抖动 + 资源加载超时，用户体验上就是"网页打不开"。
   测 `%{time_appconnect}`（TLS 握手耗时）比只看能否连通更有诊断价值。
