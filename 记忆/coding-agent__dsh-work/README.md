<!-- 由 memory-vault 自动生成，请勿手改 -->

# 记忆快照 · coding-agent::dsh-work

> 导出时间：2026-10-18 19:08 · 事实 32 条 · 知识页 3 张 · 来源：本地 Hindsight

这份快照由 `memory-vault` 自动生成：Hindsight 里存的是给 agent 检索的结构化事实，本文件是它的人可读投影，用于版本化与跨机器搬运。

## 事实

### observation（16 条）

- **2026-10-13** 2026-10-13 的那次升级覆盖掉了补丁，导致本次补丁失效。
- **2026-10-18** ark-usage-patch 补丁包位于 ~/.dsh/plugins/ark-usage-patch/，包含 apply.mjs（幂等打补丁）、snippets/{constants,ark-block}.js 以及 volc_usage.py 兜底脚本；补丁目标是 ~/.dsh/profiles/web/node_modules/dsh-balance/lib/{index.js,client.js}。dsh-balance 版本 1.3.0，来源 github:GeekRicardo/dsh-balance，相关文档在 ~/dsh-work/docs/ark-coding-plan-dsh-config.md。
- **2026-10-18** 补丁失效机制：dsh-balance 是 GitHub 依赖，每次 dsh plugin add/update 或 pnpm install 都会用上游 tarball 覆盖 node_modules，补丁会静默消失——这正是火山用量插件反复失效的原因。
- **2026-10-18** 判断补丁是否存在用 node ~/.dsh/plugins/ark-usage-patch/apply.mjs --check；恢复用 node apply.mjs，恢复前会先把 lib/*.js 备份为 lib/*.js.pre-ark-patch。
- **2026-10-18** 补丁不会被热加载：dsh-hmr 明确跳过 /node_modules/ 路径的模块（dsh-hmr/lib/index.js 第 201 行），市场的 /dsh-market/toggle 启停只重跑 apply、不清 ESM 缓存，因此改完必须重启 dsh web 进程；可用市场自带的 /dsh-market/restart，它用分离子进程拉起同一个 boot invocation。
- **2026-10-18** ~/.dsh/.credentials.yaml 的 refs 下需要账号级的 ARK_ACCESS_KEY_ID / ARK_SECRET_ACCESS_KEY，而不是推理用的 ARK_API_KEY。
- **2026-10-18** 当前凭据文件里存的值可用，不要改写：文件中的 SK 与手工复制的那串仅末两位不同，以 TQ== 结尾的才有效，手工复制的 TT0= 会返回 SignatureDoesNotMatch，误改成无效 SK 会导致签名失败。
- **2026-10-18** 火山签名 V4 走 open.volcengineapi.com，service=ark、region=cn-beijing、version=2024-01-01，scope 以 request 结尾、algorithm 为 HMAC-SHA256；canonical headers 顺序不敏感，补丁的固定序 host;x-date;x-content-sha256;content-type 与官方 SDK 的字母序都能通过。
- **2026-10-18** 判错技巧：AK 无效时返回 InvalidAccessKey（100009）并回显 AK；AK 有效但 SK 不匹配时返回 SignatureDoesNotMatch（100010），据此可区分问题出在 AK 还是 SK。
- **2026-10-18** 用量查询先调 GetAFPUsage（Agent Plan），没有 QuotaUsage 时回落到 GetCodingPlanUsage；QuotaUsage[].Level 的 session/weekly/monthly 分别对应状态栏的 5 小时/7 天/30 天。
- **2026-10-18** 市场路由需要浏览器 cookie 鉴权：用启动时打印的 launch token 兑换，命令为 curl -i -H 'Host: 127.0.0.1:3080' 'http://127.0.0.1:3080/?token=<launchToken>'，返回的 set-cookie 即凭据；cookie 名为 dsh-auth-<hash>，用 ~/.dsh/.credentials.yaml 中 client-connection/browser-session 的 secret 签名，30 天有效。
- **2026-10-18** 用户于 2026-10-18 发起新计划「记忆金库」：把记忆自动沉淀并同步到 fix-notes 运维仓库，让记忆可带走、可复核、可版本化，而不是只躺在本地 pg0 里。
- **2026-10-18** 用户的本地 Hindsight 已跑通：使用 ~/.pg0、systemd 服务 hindsight-memory、bge-m3 嵌入与 DeepSeek 抽取，作为记忆金库计划的基础设施。
- **2026-10-18** 记忆金库计划新增 DSH skill「memory-vault」：对值得长期保留的排障结论/决策双写——一份进 Hindsight 库供检索与反思，一份按 fix-notes 仓库规范写成 Markdown（分类/YYYY-MM-DD_主题.md，结构为现象→根因→修复→验证→教训，并自动重建两级索引），推送前做密钥扫描，再 commit+push 到 wanglingyuan529-droid/fix-notes。
- **2026-10-18** memory-vault 将提供反向「回灌」子命令：换电脑时把笔记重新灌进本地库以补齐向量，恢复记忆检索能力。
- **2026-10-18** 用户于 2026-10-18 把知识页刷新频率从默认每小时（H * * * *，每小时 5 页是最主要的固定 token 开销）调整为每天一次。

### world（16 条）

- **2026-10-13** 本次补丁失效是 2026-10-13 那次升级覆盖掉的 | When: 2026-10-13 —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** ark-usage-patch 补丁包位于 ~/.dsh/plugins/ark-usage-patch/，含 apply.mjs（幂等打补丁）、snippets/{constants,ark-block}.js 与 volc_usage.py 兜底脚本；被补丁目标是 ~/.dsh/profiles/web/node_modules/dsh-balance/lib/{index.js,client.js}；dsh-balance 版本 1.3.0，来源 github:GeekRicardo/dsh-balance；相关文档在 ~/dsh-work/docs/ark-coding-plan-dsh-config.md —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** 补丁失效机制：dsh-balance 是 GitHub 依赖，每次 dsh plugin add/update 或 pnpm install 都会用上游 tarball 覆盖 node_modules，补丁会静默消失 | 解释了火山用量插件反复失效的原因 —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** 判断补丁是否存在用 node ~/.dsh/plugins/ark-usage-patch/apply.mjs --check；恢复用 node apply.mjs，恢复前会先备份 lib/*.js.pre-ark-patch —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** 补丁不会被热加载：dsh-hmr 明确跳过 /node_modules/ 路径的模块（dsh-hmr/lib/index.js 第 201 行），市场 /dsh-market/toggle 启停只重跑 apply、不清 ESM 缓存，所以改完必须重启 dsh web 进程；可用市场自带的 /dsh-market/restart，它会用分离子进程拉起同一 boot invocation —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** ~/.dsh/.credentials.yaml 的 refs 下需要账号级 ARK_ACCESS_KEY_ID / ARK_SECRET_ACCESS_KEY，而不是推理 key ARK_API_KEY —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** 当前凭据文件里存的值可用，不要改写：文件中的 SK 与手工复制的那串仅末两位不同，以 TQ== 结尾的才有效，手工复制的 TT0= 会返回 SignatureDoesNotMatch | 避免误改成无效 SK 导致签名失败 —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** 火山签名 V4 走 open.volcengineapi.com，service=ark、region=cn-beijing、version=2024-01-01，scope 以 request 结尾、algorithm 为 HMAC-SHA256；canonical headers 顺序不敏感，补丁的固定序 host;x-date;x-content-sha256;content-type 与官方 SDK 的字母序都能通过 —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** 判错技巧：AK 无效返回 InvalidAccessKey(100009) 并回显 AK；AK 有效但 SK 不匹配返回 SignatureDoesNotMatch(100010) | 用于区分是 AK 还是 SK 的问题 —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** 用量查询先调 GetAFPUsage（Agent Plan），无 QuotaUsage 时回落到 GetCodingPlanUsage；QuotaUsage[].Level 的 session/weekly/monthly 对应状态栏的 5 小时/7 天/30 天 —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** 市场路由需要浏览器 cookie 鉴权：用启动时打印的 launch token 兑换，命令 curl -i -H 'Host: 127.0.0.1:3080' 'http://127.0.0.1:3080/?token=<launchToken>' 返回的 set-cookie 即凭据；cookie 名为 dsh-auth-<hash>，用 ~/.dsh/.credentials.yaml 中 client-connection/browser-session 的 secret 签名，30 天有效 —— DSH 会话中排查火山用量插件失效的全过程
- **2026-10-18** 用户发起新计划「记忆金库」：把记忆自动沉淀并同步到 fix-notes 运维仓库，目标是让记忆可带走、可复核、可版本化，而不是只躺在本地 pg0 里。 | When: 2026-10-18 | Involving: user | 避免记忆只存在本地数据库，实现可携带与版本化 —— initiative marker for [[page:kp-41797a47c0fa49cda20791ca116d10cc]]
- **2026-10-18** 用户已跑通本地 Hindsight：使用 ~/.pg0、systemd 服务 hindsight-memory、bge-m3 嵌入与 DeepSeek 抽取。 | Involving: user | 作为记忆金库计划的基础设施 —— initiative marker for [[page:kp-41797a47c0fa49cda20791ca116d10cc]]
- **2026-10-18** 计划新增 DSH skill「memory-vault」，对值得长期保留的排障结论/决策双写：一份进 Hindsight 库供检索与反思，一份按 fix-notes 仓库规范写成 Markdown（分类/YYYY-MM-DD_主题.md，结构为现象→根因→修复→验证→教训，并自动重建两级索引），推送前做密钥扫描，再 commit+push 到 wanglingyuan529-droid/fix-notes。 | Involving: user | 实现排障结论的长期保留与版本化 —— initiative marker for [[page:kp-41797a47c0fa49cda20791ca116d10cc]]
- **2026-10-18** memory-vault 将提供反向「回灌」子命令，换电脑时把笔记重新灌进本地库以补齐向量。 | Involving: user | 支持换机后恢复记忆检索能力 —— initiative marker for [[page:kp-41797a47c0fa49cda20791ca116d10cc]]
- **2026-10-18** 用户把知识页刷新频率从默认每小时（H * * * *，5 页/小时是最主要的固定 token 开销）调整为每天一次。 | When: 2026-10-18 | Involving: user | 每小时刷新 5 页是最主要的固定 token 开销 —— initiative marker for [[page:kp-41797a47c0fa49cda20791ca116d10cc]]

## 知识页

见同目录 `知识页/` 下的 Markdown（由 Hindsight 合成）。
