---
name: browser-use
description: Use for browser automation via Playwright MCP tools when a task requires live website interaction such as navigation, form filling, scraping, approvals in web UIs, social posting, following and connecting, or other multi-step browser workflows, especially when human handoff, safety checks, and recovery rules matter. Do not use for general web questions, single-URL content fetches, or direct REST API calls.
metadata:
  execution_mode: sandbox
---

# Browser Use

Operate a Playwright MCP browser to navigate, interact with, and extract data from web pages on behalf of the user.

## Trigger Conditions

Activate when:
- User asks to open, navigate, or interact with a website
- User asks to fill, submit, or read a web form
- User asks to approve/reject items in a web UI (PRs, purchase orders, etc.)
- User asks to scrape, extract, or monitor web page content
- User asks to automate a multi-step web workflow
- User asks to post content on social media (Twitter, LinkedIn, etc.)
- User asks to perform social actions (follow, connect, message, etc.)
- Another skill or agent needs browser capability

Do NOT activate for factual questions answerable from knowledge, single-URL content fetches (`WebFetch`), or direct REST API calls (`curl`).

## Risk Levels

| Level | Example | Action |
|-------|---------|--------|
| LOW | Read-only browsing, data extraction | Proceed autonomously |
| MEDIUM | Form filling with user-provided data | Proceed, narrate each step |
| HIGH | Submit/approve actions, content publishing | STOP — confirm in chat before acting |
| CRITICAL | Payment, delete, legal consent, social relationships, irreversible ops | STOP — mandatory browser handoff |

## Absolute Prohibitions

Never, regardless of user request:
- Read or extract values from password, PIN, SSN, CVV, or card number fields; only `value.length > 0` is allowed for autofill verification
- Type credentials into login forms — rely on vault autofill or hand off
- Bypass CAPTCHA programmatically
- Dismiss security warnings or certificate errors
- Access sites the user hasn't explicitly asked to visit
- Continue after 3 consecutive failures of the same action
- Ignore user abort signals ("stop", "cancel", "不要了", "换个方向")

## HITL Interaction Model

### Category A — Mandatory Browser Handoff

**Before clicking any button or submitting any form, check if the current page matches any row below. If yes, STOP — do NOT interact with the page. Call `mcp__browser_session__request_human_help`, send the handoff message, then do NOT take any browser action until the user replies.**

| Scenario | Recognition signals | Emoji | Hard rule |
|----------|-------------------|-------|-----------|
| Login form | Username/password fields, "Sign in" button | 🔐 | Try vault autofill first; only hand off if autofill fails |
| 2FA / OTP | Code input, "Enter verification code" | 🔑 | Never read or guess codes |
| CAPTCHA | Image puzzle, "I'm not a robot" checkbox | 🤖 | Never attempt JS bypass |
| Biometric / WebAuthn | Face ID / Touch ID / security key prompt | 🔒 | — |
| Legal consent | TOS / Privacy Policy / Cookie consent requiring agreement | ⚖️ | Never click "Accept" / "Agree" |
| Payment / 3DS | Card input, payment password, 3DS verification | 💳 | Never type card numbers, CVVs, or payment passwords |
| Irreversible confirm | "Delete account", "Wipe data", "Unbind" confirm dialog | ⚠️ | Never click the confirm button |

**Handoff message format:**

> [emoji] [What is blocking and why I can't proceed]
>
> 👉 Open the browser: [browser URL]
>
> Please [exact action]. Once you see [completion indicator], reply here so I can continue.

After user replies: take `browser_snapshot`, verify expected state (login → dashboard visible; payment → receipt shown; consent → screen gone; etc.), then resume. If multi-step auth, warn user up front that multiple handoffs may be needed.

If session was idle 10+ minutes before resuming: re-check login indicators before assuming the session is still valid.

### Category B — Conditional Escalation

Try once autonomously. If still blocked:
1. Take a `browser_screenshot` to capture the current state
2. Send the screenshot + the following message in chat:

> 🚫 I'm stuck at **[current URL]**
>
> 👉 Open the browser: [browser URL]
>
> **What happened:** [brief description]
> **What I tried:** [actions taken]
> **Progress so far:** [what was completed]
>
> Please [specific ask]. Reply here when ready.

- Page error / unresponsive: try `location.reload()` once → still stuck → escalate as above
- Multi-option ambiguity: describe options in chat → user still can't decide → escalate as above

### Category C — Chat Confirmation (no browser handoff)

STOP all browser actions. Ask in chat and **do NOT proceed until the user replies**. Then act on their response.

- **AI-drafted content** (comment, reply, review, post, message, or any text the AI composed rather than the user providing word-for-word) → draft the text in chat FIRST, before typing anything into the browser → "Here's what I plan to submit: [full text]. Post as-is, edit, or cancel?" → only open the browser text input and type after the user confirms
- **State-changing action** (submit, approve, reject, merge, close, reopen) → "I'm about to [action] on [target]. Proceed?"
- **Social relationship action** (follow / unfollow / block / connect on any platform) → "I'm about to [action] [@target] on [platform]. Proceed or cancel?"
- **Missing required field** → ask for the specific value

On reply: confirm → execute; edit → apply change, re-confirm; cancel → abort.

## User-Facing Language

- Do not mention internal labels ("HIGH risk", "Category A", "CRITICAL", "browser-use skill")
- Ask confirmations naturally: "I'm ready to post this. Please confirm the text before I publish."
- Ground explanations in the concrete action, not policy categories

## Workflow

### 1. Pre-flight

- If the user named the target site, open it immediately and navigate to the most relevant page
- If no URL given, use `web_search` to find the most authoritative destination; open it before asking follow-up questions
- If multiple plausible matches exist, narrow down in the browser first; ask only if a real choice remains
- If a wrong choice could trigger a risky action on the wrong site, confirm destination first
- Check if already authenticated before requesting login

### 2. Navigation & Interaction

**Before every browser action:**
1. Does the page match any Category A scenario? → STOP, call `mcp__browser_session__request_human_help`, wait.
2. Am I about to type AI-drafted text into the browser, or perform a state-changing / irreversible action? → STOP, show the draft or confirm in chat (Category C), wait for reply, then decide.
3. Only if neither applies → proceed.

- State what you're about to do before each action
- Use `browser_snapshot` to read page state before acting
- Use `browser_click` / `browser_type` for individual interactions; switch to `browser_evaluate` for 3+ repetitive actions
- After key actions, snapshot to confirm the expected result

### 3. Authentication Handoff

Login page detected:
1. Click username field, wait ~1s for vault autofill
2. Verify: username non-empty + password `value.length > 0` via `browser_evaluate`
3. Autofill succeeded → submit; failed → Category A handoff (🔐 template)
4. After user signals done → snapshot to verify; still on login page → ask user to retry

2FA / OAuth / biometric detected → Category A handoff immediately (🔑 / 🔒 template).

### 4. Session Awareness

- Check for logged-in indicators before multi-step flows
- Session expired mid-task → pause, report progress ("Completed steps 1–N"), Category A re-login handoff
- Session idle 10+ minutes → re-verify login state before any action
- Track progress: "✅ Step N/M — [specifics]"; never retry silently

### 5. Failure Recovery

```
Action failed
  ├─ Page unresponsive → reload once → still stuck → Category B handoff
  ├─ Element not found → re-snapshot → try 2 alternative selectors → not found → Category B handoff
  ├─ Click/type no effect → snapshot to verify → try browser_evaluate → 3 attempts max → Category B handoff
  ├─ Navigation timeout → snapshot first (timeout ≠ load failure) → check navigator.onLine
  │     online + usable → continue; online + unusable → retry once; second failure → Category B handoff
  ├─ Session expired → pause, report progress → Category A re-login handoff
  └─ Unexpected error page → read error via snapshot, report to user → Category B handoff if needed
```

Retry limits: page reload ×1, click/type ×3, same-URL navigation ×2, form submission ×1 then confirm before retry.

If repeating the same action 3+ times toward a single goal, switch to `browser_evaluate` instead.

### 6. Sensitive Field Protection

- Never read or extract from password, PIN, SSN, CVV, card number fields
- Writing user-provided values INTO sensitive fields is allowed when the task requires it
- Unexpected sensitive data visible on page → stop, notify user, do not read it

### 7. Form Filling

- Fill all fields using user-provided values
- Missing required value → Category C (ask in chat)
- After filling, hand off for CAPTCHA or other blocked steps

### 8. Social Media & Content Publishing

- Before publishing any content (tweet, post, comment, DM): Category C — show full preview, wait for confirmation
- Social relationship actions (follow/unfollow/block/connect): Category C — wait for explicit chat approval before clicking

### 9. Completion & Batch Tasks

Single task: summarize pages visited, actions taken, results, failures.

Batch task (operating on a list):
1. Track completed items by ID or index
2. Failure at item K → report "Completed K−1 of N; failed on K: [reason]" → ask "Skip and continue, or stop?"
3. Final summary: completed / skipped / failed
