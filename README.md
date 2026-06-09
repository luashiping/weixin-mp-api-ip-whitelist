# Weixin MP API IP Whitelist

> **GitHub 描述：** Update Weixin MP (微信公众号) backend API IP whitelist entries for a specific Official Account AppID via OpenCLI Browser — handles QR login, admin verification, and outbound IP discovery without overwriting existing entries.

[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex Skill](https://img.shields.io/badge/Codex-Skill-4B0082)](https://developers.openai.com/codex)
[![Version](https://img.shields.io/badge/version-0.2.0-blue)](#)
[![OpenCLI](https://img.shields.io/badge/powered%20by-OpenCLI%20Browser-orange)](#)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

一个用于 Claude Code / Codex 的 Skill，通过 [OpenCLI Browser](https://github.com/opencli/opencli) 驱动真实浏览器，在微信公众号（Weixin MP）开发者控制台为指定 AppID 维护「接口权限 > IP 白名单」。在追加新 IP 时**保留**已有条目，配套支持二维码登录、远程扫码核验，以及出口公网 IP 自动发现。

---

## 何时使用

- 部署/迁移服务后，调用微信公众号 API（发文、上传素材、`access_token`）时被白名单拦截
- 现有 IP 白名单条目较多，需要在不丢失历史 IP 的前提下追加新条目
- 微信公众号发布 / 上传 skill 因 IP 不在白名单内失败，需要快速打通
- 操作员不在电脑旁，需要把二维码截图发到移动端扫码

不适用于：服务号 / 订阅号身份认证、第三方平台代开发、移动 App AppID 的白名单。

---

## 特性

- **AppID 显式隔离**：操作前确认控制台 AppID 与任务上下文中的 `WECHAT_APP_ID` 一致，避免把 IP 加到错误公众号上
- **追加而非覆盖**：读取已有白名单内容，与目标 IP 合并后再回写，绝不盲替换
- **出口 IP 自动发现**：通过 `api.ipify.org` / `ifconfig.me` / `4.ipw.cn` 探测当前机器出口公网 IPv4
- **可视化登录**：保留前台浏览器窗口，等待用户 / 管理员扫码，不会索要账号密码
- **远程扫码授权**：把二维码截图回传到对话，让用户从手机微信扫码
- **JS 快速路径**：登录后优先用 `opencli browser eval` 自动定位白名单编辑器、合并 IP 并保存，失败时再回退到元素索引流程

---

## 前置条件

- 已安装并能正常启动 [OpenCLI Browser](https://github.com/opencli/opencli)
- 一个有效的微信公众号 AppID（形如 `wx...`）
- 当前机器能访问 `developers.weixin.qq.com`

将 AppID 放进当前会话的环境变量，避免硬编码到 skill 内：

```bash
export WECHAT_APP_ID=wx_your_official_account_appid
```

首次使用时，本 skill 会告知用户：如果是为了配合 `baoyu-post-to-wechat` 解除 IP 白名单阻塞，会读取 `~/.baoyu-skills/.env` 中的 `WECHAT_APP_ID`；如果不是使用 `baoyu-post-to-wechat`，需要用户手动配置或提供 `WECHAT_APP_ID`。

多公众号场景下，请明确在任务上下文中指出当前 `WECHAT_APP_ID`，避免误操作。

---

## 快速开始

```bash
# 1. 启动 OpenCLI Browser，打开目标公众号控制台
opencli browser --session weixin-mp-ip --window foreground open \
  "https://developers.weixin.qq.com/console/product/mp/${WECHAT_APP_ID}?tab1=basicInfo&tab2=dev"

# 2. 让浏览器处理登录 / 扫码
opencli browser --session weixin-mp-ip wait time 5 && \
  opencli browser --session weixin-mp-ip state

# 3. 登录完成后优先使用 SKILL.md 的「JS Fast Path」
#    自动定位编辑入口 → 读取当前值 → 合并新 IP → 写回 → 保存
opencli browser --session weixin-mp-ip eval '<paste JS Fast Path here>'

# 4. 如果 JS 返回 ok:false，再回退到 state/index 手动定位
opencli browser --session weixin-mp-ip state

# 5. 看到「设置成功」或重新读回白名单确认新增 IP 已落库
opencli browser --session weixin-mp-ip state
```

详细流程、标签定位、安全合并规则见 [`SKILL.md`](./SKILL.md)。

---

## 工作流概览

```
┌──────────────────┐
│  触发：API 报错   │  ← 发文/上传/素材 40165 / invalid ip
│  或用户显式请求   │
└────────┬─────────┘
         ▼
┌──────────────────┐
│  解析 WECHAT_APP_ID │ ← env / .env / 任务上下文 / 用户输入
└────────┬─────────┘
         ▼
┌──────────────────┐
│  收集目标 IP     │  ← 用户提供 / 出口 IP 探测
└────────┬─────────┘
         ▼
┌──────────────────┐
│ OpenCLI 打开控制台 │  ← 前台窗口，保留扫码能力
└────────┬─────────┘
         ▼
┌──────────────────┐
│  登录 / 管理员核验 │  ← wait + state，必要时截图远程扫码
└────────┬─────────┘
         ▼
┌──────────────────┐
│ JS 读 → 合并 → 写 │  ← 永远追加，不覆盖
└────────┬─────────┘
         ▼
┌──────────────────┐
│  校验 & 重试原 API │
└──────────────────┘
```

---

## 与其他 Skill 协作

通常作为「上游修复」被调用：

| 上游 Skill | 失败症状 | 修复动作 |
| --- | --- | --- |
| `baoyu-post-to-wechat` | 发布文章时 `invalid ip` / `not in whitelist` | 探测出口 IP → 追加白名单 → 重试发文 |
| 自定义素材上传脚本 | `access_token` 获取失败且返回 IP 相关错误码 | 同上 |
| 多账号批量发布 | 当前 IP 切换导致部分账号失败 | 逐个 AppID 调用本 skill |

调用方应在 `WECHAT_APP_ID` 缺失时先确认触发来源：如果来自 `baoyu-post-to-wechat`，读取 `~/.baoyu-skills/.env`；其他场景应暂停并询问用户手动配置或提供 AppID，而不是猜测。

---

## 出口 IP 探测

```bash
curl -fsS https://api.ipify.org \
  || curl -fsS https://ifconfig.me/ip \
  || curl -fsS https://4.ipw.cn
```

如果沙箱环境网络受限，需要提权重试，**不要猜 IP**。请使用「运行本 skill 的那台机器」的出口 IP，不要把别的执行环境的 IP 加进来。

---

## 安全注意事项

- 本 skill **不会**收集、存储或回传账号密码，全程只让用户在浏览器里手动扫码
- 操作前必须 `get value` 校对白名单当前内容，确认 AppID 后再写入
- 远程二维码截图请走受信任的会话通道，避免被缓存到公开图床
- 多个管理员协作时，建议在提交后保留操作记录（AppID、追加的 IP、时间）

---

## 开发与测试

```bash
# 进入仓库
cd weixin-mp-api-ip-whitelist

# Skill 元数据
cat SKILL.md
cat agents/openai.yaml
```

提交新版本前请确认：

- `SKILL.md` 的描述行与 `agents/openai.yaml` 的 `default_prompt` 仍能覆盖新场景
- 新增标签（如控制台改版后的新按钮名）已写入「OpenCLI Rules」或「JS Fast Path」段落
- 合并规则仍然「追加」而非「覆盖」

---

## 许可证

MIT
