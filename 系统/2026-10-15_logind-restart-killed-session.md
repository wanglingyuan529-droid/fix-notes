# 重启 systemd-logind 毁掉了整个图形会话：niri 因 DRM 权限崩溃、被踢回 SDDM

> 日期：2026-10-15 · 环境：Arch Linux · niri + Nyxuri dotfiles · SDDM
> 背景：给桌面加"永不睡眠"，改完 logind 配置后顺手重启了 logind 服务

## 现象

改完 `/etc/systemd/logind.conf.d/99-no-sleep.conf` 后执行：

```bash
sudo systemctl restart systemd-logind
```

屏幕上所有窗口**瞬间消失**，直接掉回 SDDM 登录界面。重新登录后一度黑屏无响应，
只能强制关机重启。

事后回看日志，崩溃点非常明确：

```
niri[724]: WARN niri::backend::tty: error queueing frame:
           DRM access error: Page flip commit failed on device Some("/dri/card1")
           (Permission denied (os error 13))
```

连刷十几条同样的错误，随后会话被销毁。

## 根因

**`systemctl restart systemd-logind` 会强制销毁该用户的所有会话（session）。**

logind 是所有图形/登录会话的持有者。重启它 = 把它管理的 session 全部回收，
哪怕用户还正登录着、还在用。对**正在使用的图形会话**执行这个操作，等价于
一次没有保存、没有退出的强制登出。

这里还有一层放大效应：会话被销毁时，niri 手里的 DRM master 权限被回收，
于是渲染线程持续报 `Permission denied` 直到进程退出 —— 这就是"黑屏无响应"的来源。

**最关键的一点是：这个重启从头到尾都不必要。**
`IdleAction` 属于"改完读一次就生效"的配置，不需要重启服务来应用。

## 试错

| 做法 | 结果 |
|---|---|
| 改 `/etc/systemd/logind.conf.d/*.conf` 后**重启 logind 以便立刻验证** | ❌ 会话被销毁，踢回 SDDM |
| 只想确认 `IdleAction` 是否生效，于是 `systemctl restart` 后读 `busctl` 属性 | ❌ 属性确实读到了，但代价是整个会话 |
| 直接读配置 + 读运行时属性，不重启 | ✅ 完全够用，见下 |

真正该做的是**用不改动任何东西的方式验证**。对于 logind：

```bash
# 读运行时生效值（只读，零副作用）
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 \
       org.freedesktop.login1.Manager IdleAction
# → s "ignore"
```

## 修复

配置本身是对的，只要**不重启服务**：

```ini
# /etc/systemd/logind.conf.d/99-no-sleep.conf
[Login]
IdleAction=ignore
IdleActionSec=0
```

改完就结束。`IdleAction` 在下次开机（或由系统自行重载时）生效，
**不需要、也不应该**为了"立刻看到效果"去重启 logind。

## 验证

```bash
# 1) 确认配置已落盘
cat /etc/systemd/logind.conf.d/99-no-sleep.conf

# 2) 读运行时状态（只读！）
busctl get-property org.freedesktop.login1 /org/freedesktop/login1 \
       org.freedesktop.login1.Manager IdleAction
# → s "ignore"

# 3) 确认图形会话仍完好——这才是真正该关心的
loginctl list-sessions
niri msg workspaces
```

## 教训

1. **`systemctl restart` 不是"刷新一下"，它是一个破坏性操作。**
   执行前必须能回答：**这个服务名下挂着谁的会话 / 谁的连接 / 谁的状态？**
   `systemd-logind`、`display-manager`、`systemd-user-sessions` 这类直接持有
   用户会话的服务，**在用机器上永远不要重启**。

2. **"改配置"和"重启服务"是两件事，不要顺手做第二件。**
   绝大多数配置类改动（logind.conf、sysctl、udev 之外的静态配置）
   **重启服务不是必需的**。为了"立刻验证生效"而重启服务，
   收益只是少等一次开机，风险却是丢掉整个会话。

3. **验证优先用只读手段。** 想知道配置有没有生效，先找有没有
   "不改变任何状态就能读出来"的接口（`busctl get-property`、
   `<程序> config export`、`--status` 之类）。这类接口存在时，
   重启服务永远是下策。

4. **区分"配置已读入"和"服务已重启"。** 本例改完后其实已经能通过
   只读接口确证生效了，我却多做了一步重启 —— 问题不在判断错误，
   而在于**默认用了更粗暴的动作**。

5. **被踢回登录界面后黑屏，优先怀疑残留的 session/greeter 状态**，
   而不是硬件。这次重启后能恢复，是因为重启了整个系统；
   单纯重新登录时图形栈可能仍处于半死状态。

6. **给服务分级：**

   | 类型 | 例子 | 在用机器上重启？ |
   |---|---|---|
   | 持有用户会话/显示 | `systemd-logind`、`display-manager`、`systemd-user-sessions` | ❌ 绝不 |
   | 持有硬件/输入 | `libinput` 相关、`bluetooth`、`NetworkManager` | ⚠️ 会中断输入/网络，避免 |
   | 纯后台无状态 | `cron`、`systemd-tmpfiles` | ✅ 一般安全 |

   **不确定属于哪一类时，就不要重启，改完配置等它自然生效。**
