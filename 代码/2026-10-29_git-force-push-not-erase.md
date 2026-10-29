# GitHub 彻底清除历史：force push 只让对象不可达，删仓库重建才真删

> 日期：2026-10-29 · 环境：Arch Linux · git + gh 2.102.0 · FlClash 代理（127.0.0.1:7890）· 状态：✅ 已解决

> 一句话结论：`git push -f` 不是数据销毁——它只让旧 commit 变成不可达的孤儿对象，GitHub 服务端仍长期保留；知道 SHA 就能通过 API 看到当时的完整内容，要真正清除只能删仓库重建。

## 现象

公开仓库里误提交了不该公开的内容（学习笔记的原稿、他人资料），需要从历史里彻底移除。

第一步是很标准的做法：`git checkout --orphan` 重写成单个无父提交，再强制覆盖远程。

```bash
git checkout --orphan clean-root
git add -A && git commit -m "初始化..."
git branch -D main && git branch -m main
git push -f origin main
#  + c040875...0c84b50 main -> main (forced update)
```

推完**看起来已经完全干净**：

```bash
$ gh api repos/OWNER/REPO/commits --jq 'length'
1
$ gh api repos/OWNER/REPO/commits --jq '.[].sha[0:7]'
0c84b50
```

commit 列表只剩一个，网页历史页也只剩一条，目录里那些文件早就删了。

但拿旧 SHA 直接查，对象还活着：

```bash
$ gh api repos/OWNER/REPO/commits/1ef4b94 --jq '.sha'
1ef4b9420c8569166b93f0bee9d0b171eb7370c6
$ gh api repos/OWNER/REPO/commits/1dc99b4 --jq '.sha'
1dc99b448569243833e1ece494cc4e678988174d
```

不只 commit 元数据在，那两次提交里删掉的文件内容也能完整取出来。

## 根因

**GitHub 不会因为对象不可达就立即销毁。**

git 的对象存储是内容寻址、只增不改的结构。`push -f` 做的是**移动 remote ref 的指针**：旧 commit 从「有 ref 指向」变成「没有任何 ref 指向」，也就是 unreachable / 孤儿对象。对象本身一个字节都不会被删。

服务端什么时候 GC 这些孤儿对象，GitHub 没有公开承诺；实践中长期保留，并且**支持按 SHA 精确访问**。所以：

- 「commit 列表干净」只证明**没有 ref 指向它**
- 不证明**对象不存在**

这两件事在 git 里是完全不同的概念，而 `git log` 只能看到前者——所以任何自查都查不出来。

顺带一提，这也是为什么泄露密钥后「删掉那个文件再提交」毫无意义：密钥一直留在历史对象里。

## 修复

删仓库 → 同名重建 → 重新推送。仓库 URL 不变，但所有旧对象的存放位置随之销毁。

```bash
# 1) 拿到 delete_repo scope。默认 gh 登录只有 gist / read:org / repo，没有这个
gh auth refresh --hostname github.com --scopes delete_repo
#    给出 one-time code + https://github.com/login/device，浏览器输入后完成

# 2) 删除（不可逆）
gh repo delete OWNER/REPO --yes

# 3) 同名重建，空仓库，公开
gh repo create OWNER/REPO --public --description "..."

# 4) 推本地那个干净的单提交
git -c http.proxy=http://127.0.0.1:7890 -c https.proxy=http://127.0.0.1:7890 \
    push -u origin main
```

第 1 步有个坑：**`gh auth refresh` 需要 TTY**，在非交互环境（脚本、后台任务）直接跑会**挂起且没有任何输出**，看起来像卡死。要这样跑：

```bash
gh auth refresh --hostname github.com --scopes delete_repo </dev/null \
    > /tmp/ghauth.log 2>&1 &
# 等几秒后读文件拿码
cat /tmp/ghauth.log
#  ! One-time code (XXXX-XXXX) copied to clipboard
#  Open this URL to continue in your web browser: https://github.com/login/device
```

本地那侧同样要清，否则 reflog 里还能把旧提交捞回来：

```bash
git reflog expire --expire=now --all
git gc --prune=now
```

## 验证

**唯一可信的验证方式：拿旧 SHA 打 API，必须返回 422。** 返回 `200` 或拿到 commit 对象，就是没清干净。

```bash
$ gh api repos/OWNER/REPO/commits/1ef4b94
{"message":"No commit found for SHA: 1ef4b94","status":"422"}

$ git cat-file -e 1ef4b94        # 本地应报错
```

重建后的完整核对：

```bash
gh api repos/OWNER/REPO/commits --jq 'length'          # → 1
git ls-remote origin main                              # → 与本地 HEAD 一致
gh api repos/OWNER/REPO --jq '.size'                   # → 0（新仓库）
```

⚠️ 写验证脚本时注意退出码陷阱：

```bash
gh api .../commits/$sha 2>&1 | grep -o 'Not Found' | head -1 || echo "仍可访问"
```

这行永远走不到 `||` 分支——管道的退出码是**最后一个命令** `head` 的，而 `head` 恒成功。要取真实状态就单独判：

```bash
out=$(gh api .../commits/$sha 2>&1); echo "$out" | head -2
```

## 教训

1. **`push -f` ≠ 删数据。** 它只是移动 ref，对象一个都没少。任何「清历史」的承诺，都不能只看 commit 列表。
2. **判断标准要选对。** `git log` / commit 列表看不出真相，必须拿旧 SHA 打 API，**422 才叫清干净**。同理，泄露密钥后删文件重提交是假修复。
3. **顺序是先确认影响面，再动手删仓库。** 删仓库重建会重置 star / watch / issue / PR / releases，私有仓库还会丢 webhook 和 secrets。动手前确认仓库里没有这些。
4. **`delete_repo` 是危险权限，用完考虑收回**：`gh auth refresh -s repo,gist,read:org`。比默认权限多一个「token 泄漏即可删仓库」的口子。
5. **`gh auth refresh` 非交互会静默挂起**，必须 `</dev/null` + 重定向文件 + 后台运行再读设备码。
6. **别把 git 命令接进管道再取 `$?`**，退出码会变成管道末端命令的；推送是否成功只看日志里有没有 `旧SHA..新SHA  main -> main`。
