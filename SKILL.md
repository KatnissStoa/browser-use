---
name: browser-use
description: Use for browser automation via Playwright MCP tools when a task requires live website interaction such as navigation, form filling, scraping, approvals in web UIs, social posting, or other multi-step browser workflows, especially when human handoff, safety checks, and recovery rules matter. Do not use for general web questions, single-URL content fetches, or direct REST API calls.
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

## Non-Trigger Conditions

Do NOT activate when:
- User asks a factual question answerable from knowledge (no browser needed)
- User asks to fetch a single URL for content reading (use `WebFetch`)
- User asks to call a REST API directly (use `curl` / `Bash`)
- User asks about browser concepts without needing an actual browser

## Confirmation Gates

Confirm with user before:
- Submitting a form that triggers payment or financial transaction
- Clicking "delete", "remove", or any destructive/irreversible action
- Submitting anything on a production system not explicitly named by user
- Downloading or uploading files via the browser
- Accepting legal terms, privacy policies, or cookie consents
- Publishing content on any platform (social media, forums, comments)

## Stop Conditions

Stop immediately when:
- 2FA/OAuth/biometric prompt detected → hand off to user
- CAPTCHA or reCAPTCHA detected → hand off to user
- Payment flow or 3DS verification detected → hand off to user
- Legal/compliance consent screen detected → hand off to user
- Social relationship action requested (follow/unfollow/block) → confirm with user in chat before executing (use Social Relationship Confirmation template)
- Same action failed 3 consecutive times → escalate, do not retry
- Page shows account suspension, IP block, or rate limit error
- Sensitive data (PII, credentials) unexpectedly visible → do not read, notify user
- User says "stop", "cancel", "换个方向", or similar abort signal → halt immediately

## Risk Levels

See `references/safety-and-recovery.md` for full definitions.

| Level | Example | Action |
|-------|---------|--------|
| LOW | Read-only browsing, data extraction | Proceed autonomously |
| MEDIUM | Form filling with user-provided data | Proceed, narrate each step |
| HIGH | Submit/approve actions, content publishing | Confirm before each action |
| CRITICAL | Payment, delete, legal consent, irreversible ops | Mandatory browser takeover |

## HITL Interaction Model

All human-in-the-loop interactions follow this mechanism:

**AI calls `mcp__browser_session__request_human_help`** → AI sends a **chat message** containing:
1. What happened / why user intervention is needed
2. The **browser URL** for the user to click and open
3. Exactly what the user should do in the browser
4. A **completion indicator** — what the user should see when done
5. Instruction: "After you finish, reply here so I can resume"

**The user clicks the URL → opens the browser → performs the action → replies in chat.**
**AI then takes a snapshot to verify the result before proceeding.**

There is NO popup. The user receives a text message with a clickable browser link.

Default browser session idle timeout: if the browser session has been idle for 10 minutes or more before work resumes, treat the session and page state as at risk of expiry. When resuming, verify the page state immediately before assuming the session is still valid.

See `references/hitl-protocol.md` for all 15 interaction scenarios and their message templates.

## User-Facing Language

- Do not mention internal skill mechanics, such as "according to the browser-use skill", "HIGH risk", "CRITICAL", or similar policy labels
- Do not explain actions in terms of internal governance categories unless the user explicitly asks about the policy itself
- When confirmation is needed, ask naturally and directly, for example: "I'm ready to post this on Twitter. Please confirm the final text before I publish it."
- Keep user-facing explanations grounded in the concrete action, such as publishing, approving, deleting, logging in, or completing verification
- Do NOT output internal monologue or self-narration (e.g. "Now let me check…", "The user wants me to…", "Let me first…"). Every message must be written directly to the user with clear, actionable information.

## Workflow

### 1. Pre-flight

- If the user already named the target website and the task to perform, open the site in browser use first and navigate to the page most relevant to the user's request before asking for confirmation
- If the user mentions a product, company, person, service, listing, or other target object without giving a URL, first use the agent's `web_search` tool to identify the most likely official or intended website/page
- Prefer the most authoritative destination available: official site, product page, company domain, verified profile, or the exact page implied by the task
- Do not ask the user to provide a URL when the target can be reasonably located via `web_search`
- After resolving a likely URL, immediately open it with browser use and inspect the page to determine which specific product, profile, listing, record, or other information the user most likely meant
- If the page presents multiple plausible matches or selectable targets, narrow them down as far as possible in the browser first, then ask the user to choose only if a real selection is still required
- If search results are ambiguous, there are multiple plausible destinations, or a wrong choice could cause a risky action on the wrong site, pause and confirm the destination with the user
- State the resolved target URL, the page you found, and the current goal after navigation context is clear
- Check if already on the target page (avoid redundant navigation)
- If a session may already be authenticated, verify before requesting login

### 2. Navigation & Interaction

- Before each action, briefly state what you are about to do
- Once on the resolved site, identify the concrete information or target object the user referred to before asking follow-up questions
- Prefer reaching the user-relevant page or record first, then decide whether a confirmation gate is actually needed for the next action
- Use `browser_snapshot` to read page state before acting (prefer over screenshot)
- Use `browser_click` / `browser_type` for individual interactions
- For repetitive actions (3+ similar), switch to `browser_evaluate` with JS
- After key actions, take a snapshot to confirm the expected result
- If the same action fails 2-3 times without progress, change approach instead of repeating it

### 3. Authentication Handoff

When a login page is detected:
1. Try vault autofill first: click the username or password field and wait about 1 second
2. Verify autofill without reading secrets: use `browser_evaluate` to confirm the username is non-empty and the password field has `value.length > 0`
3. If autofill succeeded, click the login / submit button
4. If autofill did not happen, call `mcp__browser_session__request_human_help`
5. After user signals done, snapshot to VERIFY login succeeded
6. Still on login page, inform user and request re-login

When 2FA/OAuth/biometric is detected:
1. Call `mcp__browser_session__request_human_help` immediately
2. Send message with browser URL + clear instructions (see HITL protocol)
3. After user signals done, snapshot to VERIFY the flow completed
4. Multi-step OAuth may require multiple handoffs; warn the user up front

### 4. Session Awareness

- Before multi-step flows, check for logged-in indicators (avatar, dashboard, username)
- Session expired mid-task → pause, report progress, request re-login
- If the browser session sits idle for 10+ minutes, assume the session may have expired; re-check login state or target page state before resuming
- Track progress: "✅ Step N/M done — [specifics]"
- Never retry silently — always surface blockers

### 5. Failure Recovery

See `references/safety-and-recovery.md`. Summary:
1. Page unresponsive → `browser_evaluate(() => location.reload())` once
2. Navigation timeout → inspect the actual page state before retrying
3. Still stuck → hand off to user with browser URL + description
4. Element not found → re-snapshot, try alternatives, then escalate
5. Max 3 retries per action
6. Batch tasks: report partial progress on failure

### 6. Sensitive Field Protection

- NEVER read/extract values from: password, PIN, SSN, CVV, card number fields
- NEVER use JS `.value` on sensitive inputs
- Writing user-provided values INTO these fields is allowed
- For login forms, never type credentials yourself; either rely on vault autofill or hand off to the user
- Unexpected sensitive data on page → stop, notify user

### 7. Form Filling

- Fill all fields you can using user-provided values, including identity and application fields
- Writing user-provided values into SSN and similar sensitive fields is allowed when the task requires it
- If a required field is missing, ask for that specific value in chat
- After filling all possible fields, hand off to the user for CAPTCHA or other blocked steps

### 8. Social Media & Content Publishing

- Before publishing ANY content (tweet, post, comment, message): AI MUST stop and send the C5 Content Publishing Confirmation message. Do NOT click the publish/post/send button until the user explicitly confirms.
- User-provided content in the original prompt does NOT count as confirmation. Confirmation must be obtained immediately before the publish action.
- User can edit content in chat before AI executes the publish action.
- For social relationship actions (follow/unfollow/block/connect): AI MUST stop and send the Social Relationship Confirmation message before executing. Do NOT click the follow/unfollow/block/connect button until the user explicitly confirms.

### 9. Completion

- Summarize: pages visited, actions taken, results achieved
- Report failures or skipped items

## HITL Message Templates

These are the exact message templates AI must use for each scenario. Templates in `references/hitl-protocol.md` provide additional context but are NOT automatically loaded — use the templates below.

---

### A1. Login Required

> 🔐 I need you to log in to **[site name]**.
>
> 👉 Open the browser: [browser URL]
>
> Steps:
> 1. Enter your username and password
> 2. Complete any additional prompts
> 3. Wait until you see **[completion indicator]**
>
> Once done, reply here so I can continue.

---

### A2. Two-Factor Authentication

> 🔑 A two-factor verification is required.
>
> 👉 Open the browser: [browser URL]
>
> Please enter your verification code (check your authenticator app or SMS).
> After the page advances past the 2FA screen, reply here.

---

### A3. CAPTCHA

> 🤖 There's a CAPTCHA I can't solve.
>
> 👉 Open the browser: [browser URL]
>
> Please complete the CAPTCHA verification. Once the page moves forward, reply here.

---

### A4. Biometric Authentication

> 🔒 This page requires biometric verification (Face ID / Touch ID / security key) that I can't perform.
>
> 👉 Open the browser: [browser URL]
>
> Please complete the biometric prompt on your device. Once authenticated, reply here.

---

### A5. Legal / Compliance Consent

> ⚖️ This page requires you to accept legal terms / privacy policy / cookie consent. I cannot agree to legal terms on your behalf.
>
> 👉 Open the browser: [browser URL]
>
> Please review and accept (or decline) the terms. Once the page moves past the consent screen, reply here.

---

### A6. Payment Confirmation

> 💳 A payment step requires your input. I cannot enter payment credentials.
>
> 👉 Open the browser: [browser URL]
>
> What's needed: [describe — e.g. "enter credit card details", "confirm payment of $XX", "complete 3DS verification"]
>
> After the payment is processed (or cancelled), reply here.

---

### A7. Irreversible Operation

> ⚠️ **Irreversible action detected.** This page is asking to confirm: **[exact action, e.g. "Delete account permanently"]**.
>
> 👉 Open the browser: [browser URL]
>
> I cannot perform irreversible operations on your behalf. Please review carefully and confirm or cancel yourself. Reply here when done.

---

### Social Relationship Confirmation (follow / unfollow / block / connect)

> 👤 I'm about to perform a social relationship action on your behalf:
>
> **Platform:** [Twitter / LinkedIn / etc.]
> **Action:** [follow / unfollow / block / connect]
> **Target:** [@username or full name]
>
> This will send a notification to the other party and may affect your visibility. Want me to proceed, or cancel?

**User options:**
- "yes" / "proceed" / "go" → AI executes the action
- "cancel" → AI aborts, does not click the button

---

### B1. Page Error / Stuck

> 🚫 I'm stuck at **[current URL]**.
>
> 👉 Open the browser: [browser URL]
>
> What happened: [description]
> What I tried: [actions]
> Progress so far: [e.g. "completed 8/10 orders"]
>
> Please try refreshing or navigating to **[target URL]**. Reply here when ready.

---

### B2. Multi-Option Ambiguity (chat-level first)

> The page is showing multiple options:
> 1. Option A — [details]
> 2. Option B — [details]
> 3. Option C — [details]
>
> Which one should I select?

If user can't decide from description, escalate to browser takeover:

> 👉 Open the browser to see the options yourself: [browser URL]
>
> Please select the option you want, then reply here.

---

### C1. Approve / Reject

> I'm about to **[action]** on **[target]**.
> Details: [specifics — PR title, order amount, etc.]
>
> Proceed? Reply **yes** or **no**.

For batch operations: confirm first item, then ask "Apply the same to the remaining N items?"

---

### C2. Edit / Revise

> Here's what I plan to submit:
>
> **Field 1:** value
> **Field 2:** value
> **Field 3:** value
>
> Want me to submit as-is, or would you like to change anything?

---

### C3. Supplementary Input

> I need additional information to continue:
> - **[field name]**: [what's needed]
>
> Please provide it here.

For session expiry:

> ⏱ Session expired at **[site]**. I completed **[progress summary]**.
>
> 👉 Please re-login: [browser URL]
>
> After you see **[indicator]**, reply here and I'll resume from where I left off.

---

### C4. Abort / Redirect

> ⏸ Stopped. Here's where I was:
> - Current task: [description]
> - Progress: [what was completed]
> - Pending: [what was not done]
>
> What would you like to do instead?

---

### C5. Content Publishing Confirmation

> 📝 I'm about to publish the following content:
>
> **Platform:** [Twitter / LinkedIn / etc.]
> **Account:** [which account]
> **Action:** [new post / reply to @xxx / direct message to xxx]
> **Content:**
> > [full text of the post/message, exactly as it will appear]
>
> [If applicable: **Attachments:** image/link preview description]
>
> Want me to publish as-is, edit something, or cancel?

**User options:**
- "publish" / "go" / "yes" → AI executes the publish action
- "change X to Y" → AI updates content, shows new preview, asks again
- "cancel" → AI aborts, does not publish

**Rule:** AI MUST NEVER publish content without sending this message and receiving explicit confirmation. User-provided content in the original prompt does NOT count as confirmation.

---

## References

- **HITL interaction protocol (full 15 scenarios with context)**: See `references/hitl-protocol.md`
- **Safety, risk, and recovery**: See `references/safety-and-recovery.md`
