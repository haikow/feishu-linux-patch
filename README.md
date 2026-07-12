# feishu-linux-patch · 飞书 Linux 桌面端补丁

给飞书 Linux 桌面版（Electron，`/opt/bytedance/feishu`）打两个补丁，**一个脚本搞定**：

- 🕵️ **防撤回**：对方撤回的消息，你仍能看到原文 —— 显示为「xxx撤回了一条消息**: 原文内容**」。
- 👀 **防对方已读**：你看过对方（私聊/群聊）的消息后，**对方界面仍显示「未读」**。

配套 **systemd 自动重打**：飞书自动更新覆盖补丁后自动重新打上，装一次基本一劳永逸。

> 💡 **防撤回缓存法思路参考 Windows「吾乐吧」补丁 [flydoos/FeiShuRevokeMsgPatcher](https://github.com/flydoos/FeiShuRevokeMsgPatcher)**。
>
> 📌 **当前实测生效版本：飞书 Linux `7.58.14`**（记录于 2026-07）。其它版本可能因 minified JS 结构变化需重新锚定正则，脚本失配时会中止且不改原文件。

> ⚠️ 仅供学习研究与个人使用，风险自负。改的是飞书本地 asar 文件，不碰账号、不联网上传任何数据。

---

## 原理

飞书桌面端是 Electron 应用，聊天逻辑在打包的 `messenger-next.asar`（JS）里。本补丁解包 → 正则注入 → 重打包。

### 防撤回（内容缓存法）
桌面撤回会**抹掉本地原文**、只留「xxx撤回了一条消息」系统提示 —— 所以**不能事后渲染，要提前缓存**：
1. **撤回前**：在会话预览更新（`upsertPreviews`）时，把每条消息的 `{id, 内容}` 存进 `localStorage`；
2. **撤回时**：系统提示渲染处按消息 id 取回缓存，拼到提示后面。

### 防对方已读
飞书看过消息后会调用 `…info("updateMessagesMeRead", { …messageIds })` 把已读 id 上报服务器。
补丁在该调用前注入 `t.messageIds=[],` 清空上报列表 → 服务器收不到你的已读回执，对方一直显示未读。
（你本地 UI 照常显示已读，本地读状态独立。）

> 详细逆向过程见 [`docs/防撤回-复盘-封档误判到缓存法破局.md`](docs/防撤回-复盘-封档误判到缓存法破局.md)。

---

## 环境要求
- Linux + 飞书桌面版装在 `/opt/bytedance/feishu`（其它路径可作参数传入）
- Python 3
- `sudo`（asar 是 root 拥有）

## 快速开始

```bash
git clone <本仓库>
cd feishu-linux-patch

# 打补丁（关掉飞书后执行）
sudo python3 patch_feishu.py                    # 默认 /opt/bytedance/feishu
# sudo python3 patch_feishu.py /自定义/飞书路径

# 重启飞书生效
pkill -9 -x feishu ; /opt/bytedance/feishu/feishu &
```

首次运行会把原 asar 备份为 `messenger-next.asar.bak`。输出示例：
```
[已读未读]      注入点: 2
[防撤回·缓存写] upsertPreviews 注入: 1
[防撤回·缓存读] 撤回提示拼接: 2
✅ 已写入 …/messenger-next.asar
```

## 一劳永逸：自动重打（推荐）

飞书**自动更新会覆盖 asar、抹掉补丁**。装上 systemd 自动重打，更新后自动补上：

```bash
sudo bash autopatch/install.sh      # 部署 + 启用
journalctl -u feishu-autopatch.service -f   # 看日志
sudo bash autopatch/uninstall.sh    # 卸载
```

- `.path` 监听 asar 变化（更新瞬间触发）+ `.timer` 开机/每 30 分钟兜底；
- 幂等：补丁在位就跳过，缺失才重打，不循环、不空转；
- 详见 [`autopatch/README.md`](autopatch/README.md)。

> 注：自动重打改的是磁盘文件，**正在运行的飞书需重启一次**才加载新补丁（飞书更新一般会自己重启）。

## 飞书更新后（没装自动重打的话）
重跑一次即可：
```bash
sudo python3 patch_feishu.py && (pkill -9 -x feishu ; /opt/bytedance/feishu/feishu &)
```
脚本会自动判断：当前 asar 是干净版（更新后）就把它设为新基线；已打补丁版才用现有 `.bak`。**不会拿旧基线覆盖新版**。

## 回滚
```bash
sudo cp /opt/bytedance/feishu/webcontent/messenger-next.asar.bak \
        /opt/bytedance/feishu/webcontent/messenger-next.asar
pkill -9 -x feishu ; /opt/bytedance/feishu/feishu &
```

## 验证补丁是否在位
```bash
ASAR=/opt/bytedance/feishu/webcontent/messenger-next.asar
grep -c recalledMessageCacheList "$ASAR"   # 防撤回 >0
grep -cF 't.messageIds=[],' "$ASAR"        # 防已读 =2
```

---

## 局限
**防撤回**：缓存来自会话预览 = 各聊天**最后一条**，撤回最近消息最稳；很老的历史消息可能没缓存；只对**打补丁后新到**的消息有效；图片/表情显示 `[图片]`/`[表情]`。

**防已读**（只堵主入口）：引用回复、给消息贴表情、接收对方文件仍会向对方泄露「已读」。

**版本兼容**：拦的是 minified JS 的结构特征，飞书大改版可能失配 —— 脚本会**中止且不改原文件**，日志会提示。届时按复盘文档思路重新锚定正则。

## 已知踩坑（写在这省得你再踩）
- 飞书新版实际加载的是 **`messenger-next.asar`**，不是遗留的 `messenger.asar`（改错文件=形态全对但零效果）。用 `sudo lsof | grep asar` 可确认。
- 注入禁忌：别插在 `let{...}=this.props` 后（截断声明表 → 飞书「页面异常」崩）。

---

## 目录结构
```
patch_feishu.py            补丁脚本（防撤回 + 防已读）
patch_feishu_unread.py     旧版（仅防已读，保留备查）
asar.py                    asar 解包/打包库
autopatch/                 systemd 自动重打（install/uninstall + 单元文件）
docs/                      逆向复盘文档
```

## 致谢与来源
- 防撤回缓存法思路参考 Windows「吾乐吧」补丁 [flydoos/FeiShuRevokeMsgPatcher](https://github.com/flydoos/FeiShuRevokeMsgPatcher)。
- 防已读思路参考 [starccy/feishu-unreadme](https://github.com/starccy/feishu-unreadme)。
- `asar.py` 源自 BeautifulDiscord（MIT）。

## 免责声明
本项目仅用于个人学习研究。使用者需自行承担风险并遵守飞书用户协议与当地法律。作者不对任何后果负责。
