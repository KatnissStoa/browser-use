# HITL (Human-in-the-Loop) Protocol

All human handoffs follow one mechanism: **AI sends a chat message with the browser URL**. The user clicks the URL to open the browser, performs the required action, then replies in chat. No popup — it's a text message with a clickable link.

Default timeout rule: if the browser session remains idle for 10 minutes or more before work resumes, AI should assume the session or page state may no longer be reliable and must verify the current state before resuming.

---

## Interaction Categories

### Category A: Mandatory Browser Takeover (8 scenarios)

AI **must** call `mcp__browser_session__request_human_help` and send the browser URL. AI cannot proceed until user completes the action.

### Category B: Conditional Takeover (2 scenarios)

AI tries to handle it first. If unable → escalates to browser takeover (Category A).

### Category C: Chat-Level Interaction (5 scenarios)

AI asks a question or presents options in the chat. No browser takeover needed — user replies in text.

---

## Category A — Mandatory Browser Takeover

### A1. Account Login (username/password)

**Trigger:** Login form detected on page.

**AI tries first:**
1. Click the username or password field once and wait about 1 second for vault autofill
2. Verify that autofill happened by checking:
   - username field is non-empty
   - password field `value.length` is non-zero
3. Never read or print the password value itself
4. If autofill worked, submit the form and continue
5. If autofill did not work, escalate with browser takeover

**AI sends:**
> 🔐 I need you to log in to **[site name]**.
>
> 👉 Open the browser: [browser URL]
>
> Steps:
> 1. Enter your username and password
> 2. Complete any additional prompts
> 3. Wait until you see **[completion indicator, e.g. "the dashboard with your name"]**
>
> Once done, reply here so I can continue.

**Post-handoff:** Snapshot → verify logged-in indicators → proceed or re-request.

---

### A2. Two-Factor Authentication (OTP / Authenticator)

**Trigger:** 2FA/MFA prompt detected (OTP input, authenticator app, SMS code).

**AI sends:**
> 🔑 A two-factor verification is required.
>
> 👉 Open the browser: [browser URL]
>
> Please enter your verification code (check your authenticator app or SMS).
> After the page advances past the 2FA screen, reply here.

---

### A3. CAPTCHA / reCAPTCHA

**Trigger:** CAPTCHA challenge detected on page.

**AI sends:**
> 🤖 There's a CAPTCHA I can't solve.
>
> 👉 Open the browser: [browser URL]
>
> Please complete the CAPTCHA verification. Once the page moves forward, reply here.

---

### A4. Biometric Authentication (Face ID / Touch ID / WebAuthn)

**Trigger:** Page requests biometric or hardware key verification.

**AI sends:**
> 🔒 This page requires biometric verification (Face ID / Touch ID / security key) that I can't perform.
>
> 👉 Open the browser: [browser URL]
>
> Please complete the biometric prompt on your device. Once authenticated, reply here.

---

### A5. Legal / Compliance Consent (TOS, Privacy Policy, Cookie Consent)

**Trigger:** Page shows terms of service, privacy policy consent, or cookie banner requiring explicit agreement.

**AI sends:**
> ⚖️ This page requires you to accept legal terms / privacy policy / cookie consent. I cannot agree to legal terms on your behalf.
>
> 👉 Open the browser: [browser URL]
>
> Please review and accept (or decline) the terms. Once the page moves past the consent screen, reply here.

**Rule:** AI must NEVER click "Accept" / "Agree" on legal/compliance screens.

---

### A6. Payment Confirmation (card entry, payment password, 3DS)

**Trigger:** Payment form, credit card input, payment password prompt, or 3DS verification page.

**AI sends:**
> 💳 A payment step requires your input. I cannot enter payment credentials.
>
> 👉 Open the browser: [browser URL]
>
> What's needed: [describe — e.g. "enter credit card details", "confirm payment of $XX", "complete 3DS verification"]
>
> After the payment is processed (or cancelled), reply here.

**Rule:** AI must NEVER type card numbers, CVVs, or payment passwords.

---

### A7. Irreversible Operation Confirmation (delete account, wipe data, unbind)

**Trigger:** Page shows destructive confirmation dialog — account deletion, data wipe, unbinding, revoking access.

**AI sends:**
> ⚠️ **Irreversible action detected.** This page is asking to confirm: **[exact action, e.g. "Delete account permanently"]**.
>
> 👉 Open the browser: [browser URL]
>
> I cannot perform irreversible operations on your behalf. Please review carefully and confirm or cancel yourself. Reply here when done.

**Rule:** AI never clicks "Delete"/"Deactivate"/"Unbind" on irreversible confirmation dialogs.

---

### A8. Social Relationship Actions (follow/unfollow/block/connect)

**Trigger:** User asks AI to follow, unfollow, block, unblock, connect, disconnect, send friend request, or remove a contact on any social platform.

**AI sends (chat-level confirmation, no browser takeover):**
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

**Rule:** AI MUST send this confirmation message and wait for explicit user approval before clicking any follow/unfollow/block/connect button. This applies to ALL social platforms (Twitter/X, LinkedIn, Instagram, Facebook, etc.). No browser takeover needed — confirmation happens in chat.

---

## Category B — Conditional Takeover

### B1. Page Error / Unresponsive

**Trigger:** Page is blank, shows error, or doesn't respond after AI tried `location.reload()` once.

**AI sends:**
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

### B2. Multi-Option Ambiguity

**Trigger:** Page shows choices AI can't resolve (plan selection, region picker, shipping method) and user hasn't specified preference.

**AI first tries (chat-level):**
> The page is showing multiple options:
> 1. Option A — [details]
> 2. Option B — [details]
> 3. Option C — [details]
>
> Which one should I select?

**If user can't decide from description → escalate to browser takeover:**
> 👉 Open the browser to see the options yourself: [browser URL]
>
> Please select the option you want, then reply here.

---

## Category C — Chat-Level Interaction

### C1. Approve / Reject

**Trigger:** AI is about to perform a state-changing action (submit, approve, reject, send).

**AI sends (in chat, no browser takeover):**
> I'm about to **[action]** on **[target]**.
> Details: [specifics — PR title, order amount, etc.]
>
> Proceed? Reply **yes** or **no**.

For batch operations: confirm first item, then ask "Apply the same to the remaining N items?"

---

### C2. Edit / Revise

**Trigger:** AI has drafted content (form values, email text, comment) and wants user to review before submitting.

**AI sends:**
> Here's what I plan to submit:
>
> **Field 1:** value
> **Field 2:** value
> **Field 3:** value
>
> Want me to submit as-is, or would you like to change anything? Tell me what to modify.

**User replies with edits → AI applies changes → confirms final version → submits.**

---

### C3. Supplementary Input

**Trigger:** AI encounters a required field with no user-provided value, or session expired and re-login is needed.

**AI sends:**
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

**Trigger:** User says "stop", "cancel", "不要了", "换个方向", "skip this", or similar abort signal.

**AI response:**
> ⏸ Stopped. Here's where I was:
> - Current task: [description]
> - Progress: [what was completed]
> - Pending: [what was not done]
>
> What would you like to do instead?

**Behavior:** AI immediately halts the current browser flow. Does NOT click any more buttons or navigate. Waits for new direction.

---

### C5. Content Publishing Confirmation

**Trigger:** AI is about to publish content on any platform — tweet, post, LinkedIn update, comment, reply, direct message, forum post.

**AI sends:**
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

**Rule:** AI must NEVER publish content without showing the full preview and getting explicit confirmation. This applies to all platforms, all content types (posts, replies, DMs, comments).

---

## Post-Handoff Verification (for all Category A & B scenarios)

After user signals completion:

1. Take `browser_snapshot` immediately
2. Check for expected state:
   - Login → user avatar/name visible, no login form
   - Payment → confirmation/receipt page
   - Legal consent → consent screen gone
   - Error recovery → target page loaded
   - Social action → action completed (button state changed)
3. If expected state NOT reached → inform user with specifics, request retry
4. If multi-step auth (OAuth chain) → may need another handoff round

If the browser session was idle for 10+ minutes:
5. Re-check whether the session is still authenticated and whether the page is still on the intended step
6. If the site returned to a login page, expired state, or start page → treat it as session expiry and resume via the re-login flow
7. If the page is still valid but drifted to a nearby state, explain what changed before continuing

## Session Expiry Mid-Task

1. Pause immediately — do not interact with login form
2. Report progress: "I completed steps 1–N. Session expired at step N+1."
3. Send browser URL for re-login (A1 template)
4. After re-login verified → navigate back → resume

## Idle Session Expiry Risk

If browser session idle time exceeds 10 minutes:

1. Do not assume the previous page state is still usable
2. Take a fresh snapshot before any new action
3. If login indicators are gone or a login page appears, treat as session expiry
4. If the original target page is no longer open, navigate back only after confirming the session is still valid
5. Tell the user briefly whether the session survived, expired, or the page context changed
