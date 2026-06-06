---
name: weixin-mp-api-ip-whitelist
version: 0.1.0
description: Add or update WeChat Official Account / Weixin MP backend API IP whitelist entries for a specific WeChat Official Account AppID through OpenCLI Browser. Use when the user asks to configure API IP whitelist, add server IPs, update IP whitelist, handle QR-code login/admin verification for a Weixin developers console mp product URL, or when another WeChat/Weixin API workflow such as posting articles, uploading images, or publishing to WeChat fails or stalls because the current server/export IP is not in the API IP whitelist.
---

# Weixin MP API IP Whitelist

## Overview

Use OpenCLI Browser to open the Weixin MP developer console for the target Official Account AppID, handle QR-code login or administrator verification in the visible browser window, and append API IP whitelist entries without overwriting existing entries. If another WeChat API workflow is blocked by an IP whitelist warning, discover the outbound public IP of the machine running this skill, then add that IP with this same workflow.

Target console URL template:

```text
https://developers.weixin.qq.com/console/product/mp/<WECHAT_APP_ID>?tab1=basicInfo&tab2=dev
```

## Prerequisites

Before relying on this skill during WeChat publishing or upload workflows, make sure the target Official Account AppID is already configured and easy to read.

- On the first use in a conversation, tell the user: this skill will read `WECHAT_APP_ID` from `~/.baoyu-skills/.env` when it is being used to unblock `baoyu-post-to-wechat`; if they are not using `baoyu-post-to-wechat`, they need to manually configure or provide `WECHAT_APP_ID`.
- Prefer configuring `WECHAT_APP_ID` as an environment variable in the shell/session that runs Codex or the WeChat workflow:

```bash
export WECHAT_APP_ID=wx_your_official_account_appid
```

- When this skill is triggered by `baoyu-post-to-wechat`, read `WECHAT_APP_ID` from `~/.baoyu-skills/.env` if it is not already available in the current environment or task context.
- If the WeChat-related publishing/upload skill uses a project `.env` or config file, put the same `WECHAT_APP_ID` value there and make sure that skill exposes or preserves it in the task context.
- Do not hard-code a personal AppID inside this skill.
- If multiple Official Accounts are used, keep the active `WECHAT_APP_ID` explicit in the task context so the whitelist update is applied to the correct account.
- If `WECHAT_APP_ID` is missing when an IP whitelist failure happens, pause and ask the user to configure or provide it before opening the Weixin developers console.

## Quick Start

1. On the first use in a conversation, notify the user that `WECHAT_APP_ID` is read from `~/.baoyu-skills/.env` only when this skill is being used with `baoyu-post-to-wechat`; otherwise the user must configure or provide it manually.
2. Read `WECHAT_APP_ID` from the existing WeChat-related skill configuration, environment, workflow settings, user request, developer console URL, or nearby workflow context. AppIDs usually look like `wx...`.
3. If this skill was triggered by `baoyu-post-to-wechat` and `WECHAT_APP_ID` is still missing, read it from `~/.baoyu-skills/.env`.
4. If `WECHAT_APP_ID` is still missing or ambiguous, ask the user to configure or provide the target Official Account AppID before opening the console.
5. Collect the IPs or CIDRs from the user request. If the user did not provide an IP and the trigger is a WeChat API whitelist failure, get the current outbound IP first; see "Outbound IP Discovery".
6. Normalize obvious separators mentally before pasting: accept one IP/CIDR per line in the final whitelist value.
7. Open the console after replacing `<WECHAT_APP_ID>`:

```bash
opencli browser --session weixin-mp-ip --window foreground open 'https://developers.weixin.qq.com/console/product/mp/<WECHAT_APP_ID>?tab1=basicInfo&tab2=dev' && opencli browser --session weixin-mp-ip state
```

8. If the page shows login or verification QR code, leave the browser window visible and wait for the user/admin to scan. Do not ask the user for credentials.
9. Use `opencli browser --session weixin-mp-ip state` after every navigation or modal change. Use element indices from state for clicks/types.
10. Confirm the page AppID matches `WECHAT_APP_ID` before editing any whitelist.
11. Locate the API IP whitelist editor in the development/basic-info area, read existing entries, append missing IPs, then submit.

## Outbound IP Discovery

Use this branch when another skill or workflow that calls WeChat APIs stalls or errors on an IP whitelist prompt, especially article publishing, draft creation, image upload, media upload, or API token access.

1. Read the error text if available and confirm it is an IP whitelist problem. Typical clues include `IP`, `白名单`, `not in whitelist`, `invalid ip`, `access_token`, or a WeChat API response that says the request IP is not allowed.
2. If the error already includes the rejected IP, use that IP directly.
3. Otherwise, query the current machine's outbound public IPv4:

```bash
curl -fsS https://api.ipify.org || curl -fsS https://ifconfig.me/ip || curl -fsS https://4.ipw.cn
```

4. If the command fails because sandboxed network access is blocked, rerun it with escalation. Do not guess the IP.
5. Treat the discovered IP as the target whitelist entry and continue with "Safe Update Procedure".
6. After adding the IP and seeing `设置成功`, retry the original WeChat API operation that was blocked.

Use the outbound IP of the machine where this skill is running. Do not add unrelated IPs or guess alternate execution environments.

## OpenCLI Rules

- Use a named foreground session: `opencli browser --session weixin-mp-ip --window foreground ...` for `open`, and `opencli browser --session weixin-mp-ip ...` for later commands.
- Use `opencli browser --session weixin-mp-ip state` as the primary inspection command.
- Use `opencli browser --session weixin-mp-ip click <index>`, `type <index>`, `keys`, and `get value <index>` for interaction.
- Use `eval` only for read-only extraction when `state` is insufficient.
- Run `state` after every page-changing click.
- Prefer exact visible labels such as `开发`, `开发设置`, `基本配置`, `IP白名单`, `服务器IP`, `修改`, `配置`, `确定`, `保存`, or `提交`.
- If a QR-code verification modal appears after saving, wait for the scan, then verify the success state.

## Safe Update Procedure

Always append, never replace blindly:

1. Open the whitelist edit dialog or text area.
2. Confirm the visible page AppID is the intended `WECHAT_APP_ID`.
3. Read the current textarea/input value with `opencli browser --session weixin-mp-ip get value <index>`.
4. Merge current entries with the target list, keeping one IP/CIDR per line.
5. Preserve existing entries exactly unless a harmless whitespace cleanup is needed.
6. Select all in the editor, type the merged list, and verify with `get value`.
7. Click the save/submit control only after confirming the value contains all previous entries plus the requested additions.

## Login And Verification

If login is required:

```bash
opencli browser --session weixin-mp-ip wait time 5 && opencli browser --session weixin-mp-ip state
```

Repeat until the page leaves the login screen. Tell the user the visible browser window is ready for scanning only if they have not already scanned.

If saving requires administrator confirmation or QR scan, keep the modal open and wait. After scanning, run:

```bash
opencli browser --session weixin-mp-ip wait time 2 && opencli browser --session weixin-mp-ip state
```

## Remote QR Authorization

Use this when the user is not physically near the computer where the browser window appears.

1. Keep the QR-code modal open.
2. Capture the visible browser window or QR-code area with OpenCLI:

```bash
opencli browser --session weixin-mp-ip screenshot /tmp/weixin-mp-auth-qr.png
```

3. Share the screenshot/image with the user in the conversation so they can scan it from the Codex mobile client with WeChat.
4. If the QR code expires, refresh/reopen the authorization modal and capture a new screenshot.
5. After the user scans and confirms, wait and re-run `state` to verify the page shows `设置成功` or the requested whitelist entry.

## Completion Check

Before reporting success:

- Confirm the page shows a success toast/state, or reopen/read the whitelist field and verify all requested IPs are present.
- Confirm the final page still belongs to the target `WECHAT_APP_ID`.
- Report the exact IPs added and mention any IPs that were already present.
- If unable to find the whitelist editor because the UI changed, stop after opening the relevant console page and report the current visible labels from `opencli browser --session weixin-mp-ip state`.
