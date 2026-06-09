# Repository Guidelines

## Project Structure & Module Organization

This repository packages a Codex/Claude skill for updating Weixin MP API IP whitelists through OpenCLI Browser.

- `SKILL.md` is the primary skill definition. It contains front matter metadata plus the operational workflow agents should follow.
- `README.md` is the user-facing overview, installation context, usage examples, and safety notes.
- `agents/openai.yaml` configures the OpenAI/Codex agent interface, display name, default prompt, and implicit invocation policy.
- There is no application source tree, generated asset directory, or formal test directory at present.

Keep workflow changes in `SKILL.md` first, then update `README.md` and `agents/openai.yaml` when descriptions, prompts, version numbers, or invocation behavior change.

## Build, Test, and Development Commands

This repo does not require a build step. Useful local checks are:

```bash
cat SKILL.md
cat agents/openai.yaml
git diff --check
```

Use `git diff --check` before committing to catch whitespace errors. To manually validate command examples, use a real OpenCLI Browser installation and a test Weixin MP AppID:

```bash
opencli browser --session weixin-mp-ip --window foreground open \
  "https://developers.weixin.qq.com/console/product/mp/${WECHAT_APP_ID}?tab1=basicInfo&tab2=dev"
```

## Coding Style & Naming Conventions

Write Markdown in concise, instructional language. Use fenced code blocks with `bash`, `text`, or another accurate language tag. Keep command examples copyable, with placeholders such as `<WECHAT_APP_ID>` or environment variables such as `${WECHAT_APP_ID}`.

YAML files use two-space indentation and quoted user-facing strings where helpful. Skill names and sessions should remain stable: `weixin-mp-api-ip-whitelist` and `weixin-mp-ip`.

## Testing Guidelines

There is no automated test framework. Validate changes by reviewing rendered Markdown, checking YAML syntax, and walking through the documented OpenCLI workflow when behavior changes. For safety-sensitive edits, confirm the guide still requires reading existing whitelist entries before appending new IPs.

## Commit & Pull Request Guidelines

Recent commits use short, direct messages, often in Chinese, such as `增加版本号` or `更新 WECHAT_APP_ID 配置说明`. Follow that style: one concise summary of the user-visible change.

Pull requests should describe the workflow or documentation change, note any manual OpenCLI validation performed, and call out changes to safety behavior, AppID handling, or whitelist merge rules. Link related issues when available.

## Security & Configuration Tips

Never hard-code personal AppIDs, credentials, QR codes, or real whitelist contents. Keep `WECHAT_APP_ID` in the environment or external configuration. The workflow must preserve existing whitelist entries and must not ask users for Weixin passwords.
