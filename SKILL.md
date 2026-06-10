---
name: weixin-mp-api-ip-whitelist
version: 0.2.2
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
opencli browser open 'https://developers.weixin.qq.com/console/product/mp/<WECHAT_APP_ID>?tab1=basicInfo&tab2=dev'
opencli browser state
```

8. If the page shows login or verification QR code, leave the browser window visible and wait up to 60 seconds for the user/admin to scan. Do not ask the user for credentials.
9. After login, prefer the "JS Fast Path" to find the IP whitelist editor, merge entries, and submit. Use `state`/indexed clicks only as a fallback when the script reports that it cannot locate the editor or submit control.
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

- Use the OpenCLI Browser command format supported by the installed version. OpenCLI Browser 1.7.x uses a single automation window and does not accept `--session` or `--window`.
- Use `opencli browser open <url>` to open the console.
- Use `opencli browser state` as the primary inspection command.
- Use `opencli browser click <index>`, `type <index> <text>`, `keys <key>`, and `get value <index>` for interaction.
- If a different OpenCLI installation supports session flags, check `opencli browser --help` first before adding them; do not pass unsupported flags.
- Use `eval` for the JS fast path below. It may edit the whitelist only after the visible page is confirmed to belong to the intended `WECHAT_APP_ID`, and it must return the old value, merged value, and submit status.
- Run `state` after every page-changing click.
- Prefer exact visible labels such as `开发`, `开发设置`, `基本配置`, `IP白名单`, `服务器IP`, `修改`, `配置`, `确定`, `保存`, or `提交`.
- If a QR-code verification modal appears after saving, wait up to 60 seconds for the scan, then verify the success state.

## JS Fast Path

Use this branch after the console page has loaded and login/admin verification is complete. Replace `wx_your_appid` and `1.2.3.4` before running. Keep one IP or CIDR per string in `targetEntries`.

```bash
opencli browser eval '(() => {
  const expectedAppId = "wx_your_appid";
  const targetEntries = ["1.2.3.4"];
  const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));
  const text = (node) => (node && (node.innerText || node.textContent || node.value) || "").trim();
  const visible = (node) => {
    if (!node) return false;
    const rect = node.getBoundingClientRect();
    const style = getComputedStyle(node);
    return rect.width > 0 && rect.height > 0 && style.visibility !== "hidden" && style.display !== "none";
  };
  const normList = (value) => String(value || "")
    .split(/[\n,;，；\s]+/)
    .map((v) => v.trim())
    .filter(Boolean);
  const uniq = (items) => Array.from(new Set(items));
  const click = (node) => {
    node.scrollIntoView({ block: "center", inline: "center" });
    node.dispatchEvent(new MouseEvent("click", { bubbles: true, cancelable: true, view: window }));
  };
  const setValue = (el, value) => {
    const proto = el instanceof HTMLTextAreaElement ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
    const setter = Object.getOwnPropertyDescriptor(proto, "value").set;
    setter.call(el, value);
    el.dispatchEvent(new Event("input", { bubbles: true }));
    el.dispatchEvent(new Event("change", { bubbles: true }));
  };
  return (async () => {
    if (expectedAppId && !document.body.innerText.includes(expectedAppId)) {
      return { ok: false, reason: "appid_mismatch_or_not_visible", expectedAppId };
    }
    const labels = ["IP白名单", "IP 白名单", "服务器IP", "服务器 IP", "接口权限"];
    const buttons = [...document.querySelectorAll("button, a, [role=button], .weui-desktop-btn")]
      .filter(visible);
    const editButton = buttons.find((btn) => {
      const btnText = text(btn);
      if (!/(修改|配置|设置|编辑|查看|详情)/.test(btnText)) return false;
      const scope = btn.closest("tr, li, section, .weui-desktop-form__item, .weui-desktop-panel, .weui-desktop-card") || btn.parentElement;
      return labels.some((label) => text(scope).includes(label));
    });
    if (editButton) {
      click(editButton);
      await sleep(800);
    }
    const fields = [...document.querySelectorAll("textarea, input[type=text], input:not([type])")]
      .filter(visible);
    const field = fields.find((el) => {
      const current = el.value || "";
      const scope = el.closest(".weui-desktop-dialog, .weui-desktop-form__item, form, section, tr, li") || el.parentElement;
      return labels.some((label) => text(scope).includes(label)) || /^\s*(\d{1,3}\.){3}\d{1,3}/.test(current);
    }) || fields.find((el) => (el.placeholder || "").includes("IP"));
    if (!field) return { ok: false, reason: "whitelist_field_not_found" };
    const oldValue = field.value || "";
    const merged = uniq([...normList(oldValue), ...normList(targetEntries.join("\n"))]).join("\n");
    setValue(field, merged);
    await sleep(200);
    if ((field.value || "").trim() !== merged.trim()) {
      return { ok: false, reason: "value_write_failed", oldValue, attemptedValue: merged, actualValue: field.value };
    }
    const submitButton = [...document.querySelectorAll("button, a, [role=button], .weui-desktop-btn")]
      .filter(visible)
      .find((btn) => /^(确定|保存|提交|确认)$/.test(text(btn)) || /(确定|保存|提交|确认)/.test(text(btn)));
    if (!submitButton) return { ok: false, reason: "submit_button_not_found", oldValue, mergedValue: merged };
    click(submitButton);
    await sleep(1000);
    return { ok: true, oldValue, mergedValue: merged, added: targetEntries, pageTextHint: document.body.innerText.slice(0, 300) };
  })();
})()'
```

If the command returns `ok: false`, do not keep retrying blindly. Run `opencli browser state`, inspect the current labels, and continue with the indexed fallback procedure.

## Safe Update Procedure

Always append, never replace blindly:

1. Open the whitelist edit dialog or text area.
2. Confirm the visible page AppID is the intended `WECHAT_APP_ID`.
3. Read the current textarea/input value with `opencli browser get value <index>`.
4. Merge current entries with the target list, keeping one IP/CIDR per line.
5. Preserve existing entries exactly unless a harmless whitespace cleanup is needed.
6. Select all in the editor, type the merged list, and verify with `get value`.
7. Click the save/submit control only after confirming the value contains all previous entries plus the requested additions.

## Login And Verification

If login is required:

```bash
opencli browser wait time 60
opencli browser state
```

Repeat until the page leaves the login screen. Tell the user the visible browser window is ready for scanning only if they have not already scanned.

If saving requires administrator confirmation or QR scan, keep the modal open and wait. After scanning, run:

```bash
opencli browser wait time 60
opencli browser state
```

## Remote QR Authorization

Use this when the user is not physically near the computer where the browser window appears.

1. Keep the QR-code modal open.
2. Capture the visible browser window or QR-code area with OpenCLI:

```bash
opencli browser screenshot /tmp/weixin-mp-auth-qr.png
```

3. Share the screenshot/image with the user in the conversation so they can scan it from the Codex mobile client with WeChat.
4. If the QR code expires, refresh/reopen the authorization modal and capture a new screenshot.
5. After the user scans and confirms, wait up to 60 seconds and re-run `state` to verify the page shows `设置成功` or the requested whitelist entry.

## Completion Check

Before reporting success:

- Confirm the page shows a success toast/state, or reopen/read the whitelist field and verify all requested IPs are present.
- Confirm the final page still belongs to the target `WECHAT_APP_ID`.
- Report the exact IPs added and mention any IPs that were already present.
- If unable to find the whitelist editor because the UI changed, stop after opening the relevant console page and report the current visible labels from `opencli browser state`.
