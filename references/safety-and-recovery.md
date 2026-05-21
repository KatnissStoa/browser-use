# Safety and Recovery

## Severity Levels

### LOW — Read-Only Operations
- Navigating to a URL
- Taking snapshots / reading page content
- Extracting text, prices, statuses
- Checking login state

**Action:** Proceed autonomously. No user confirmation needed.

### MEDIUM — Data Entry with User-Provided Values
- Filling form fields with data the user explicitly provided
- Typing into search bars or filter fields
- Selecting options from dropdowns (when user specified)

**Action:** Proceed, but narrate each step ("Filling in your name as John Doe"). User can interrupt.

### HIGH — State-Changing Actions & Content Publishing
- Clicking "Submit", "Approve", "Reject", "Send"
- Creating new records (tickets, orders, accounts)
- Modifying existing records (status changes, assignments)
- Publishing content on social media (posts, tweets, comments, messages)

**Action:** Chat-level confirmation (HITL C1 or C5). Show exactly what will be submitted/published. For batch operations, confirm first item, then ask about the rest.

### CRITICAL — Mandatory Browser Takeover
- Payment flows (card entry, payment password, 3DS)
- Irreversible operations (delete account, wipe data, unbind)
- Legal / compliance consent (TOS, privacy policy, cookie consent)
- Biometric / device-level authentication
- Social relationship actions (follow, unfollow, block, connect, friend request)

**Action:** ALWAYS call `request_human_help` and send browser URL to user. AI never performs these actions itself.

## Escalation Thresholds

- LOW → MEDIUM: action writes data (even non-destructive)
- MEDIUM → HIGH: action changes system state visible to others, or publishes user content
- HIGH → CRITICAL: action involves money, deletion, legal consent, social relationships, or is irreversible

## Prohibitions

Regardless of user request, NEVER:
- Read/extract values from password, PIN, SSN, CVV, or card number fields
- Store or log credentials seen on screen
- Bypass CAPTCHA programmatically
- Type credentials into login forms yourself
- Read password contents directly; only password `value.length` may be checked for vault autofill verification
- Click "Accept" / "Agree" on legal/compliance consent screens
- Click "Delete" / "Deactivate" / "Unbind" on irreversible confirmation dialogs
- Enter payment card numbers, CVVs, or payment passwords
- Click follow/unfollow/block/connect on social platforms — always hand off
- Publish any content (tweet, post, comment, DM) without showing full preview and getting explicit confirmation
- Continue after 3 consecutive failures of the same action
- Dismiss security warnings or certificate errors
- Access sites the user hasn't explicitly asked to visit
- Ignore user abort signals ("stop", "cancel", "不要了")

## Failure Recovery Decision Tree

```
Action failed
  ├─ Page unresponsive / blank
  │   ├─ Try reload: browser_evaluate(() => location.reload())
  │   ├─ Success → continue
  │   └─ Still stuck → send browser URL to user (HITL B1)
  │
  ├─ Element not found
  │   ├─ Re-snapshot the page
  │   ├─ Check if page changed (redirect, modal, overlay)
  │   ├─ Try alternative selector (role, text, CSS)
  │   ├─ Found → continue
  │   └─ Not found after 2 alternatives → escalate (HITL B1)
  │
  ├─ Click/type had no effect
  │   ├─ Snapshot to check if state actually changed
  │   ├─ Try browser_evaluate to set value / trigger click via JS
  │   ├─ Effect confirmed → continue
  │   └─ No effect after 3 attempts → escalate (HITL B1)
  │
  ├─ Navigation timeout
  │   ├─ Treat as "page may still be rendering", not an immediate hard failure
  │   ├─ Inspect the page via snapshot or browser_evaluate before retrying
  │   ├─ Check network: browser_evaluate(() => navigator.onLine)
  │   ├─ Online and page looks usable → continue or use a content-based wait
  │   ├─ Online but page still unusable → retry once
  │   ├─ Offline → inform user in chat
  │   └─ Second failure on same URL → change strategy or send browser URL to user (HITL B1)
  │
  ├─ Session expired (login page appeared mid-task)
  │   ├─ Pause immediately
  │   ├─ Report progress so far
  │   └─ Send browser URL for re-login (HITL A1)
  │
  ├─ Browser session idle for 10+ minutes
  │   ├─ Take a fresh snapshot before acting
  │   ├─ Check whether login indicators still exist
  │   ├─ Session still valid → continue carefully from current page state
  │   └─ Session invalid or page reset → treat as session expiry and re-login
  │
  └─ Unexpected page / error page
      ├─ Read error message via snapshot
      ├─ Report to user with the error text
      └─ Send browser URL if user intervention needed (HITL B1)
```

## Retry Limits

| Action type | Max retries | Then |
|-------------|-------------|------|
| Page reload | 1 | Send browser URL to user |
| Element click/type | 3 | Switch to JS or send URL to user |
| Navigation on same URL | 2 | Change strategy or send browser URL to user |
| Form submission | 1 | Confirm with user before retry |
| Browser session idle | 10 minutes | Re-snapshot and re-validate session/page state |

## Efficiency Rule

If you've performed the same UI action (click, type, scroll) on the same or similar element 3+ times consecutively toward a single goal, pause and consider a programmatic approach (`browser_evaluate` to set values directly). If the remaining repetitions exceed 3, switch to JS.

If the same action fails 2-3 times without progress, do not keep repeating it. Change approach first:
- For SPAs, navigate to a shallower page and click through
- If the page is already loaded, interact with visible content instead of re-navigating
- If data is available from a related page or backend API, use that path instead

## Navigation Timeout Guidance

A timeout such as "waiting until domcontentloaded/load exceeded" often means the browser event never fired, not that the network request failed.

After a navigation timeout:
1. Inspect the page snapshot to see what actually rendered
2. Use `browser_evaluate` to read a specific signal when faster than retrying
3. Use a content-based wait only when you know what element or text should appear
4. Avoid another long navigation retry unless the page truly did not load

## Batch Task Recovery

For tasks operating on a list (e.g. approve N PRs, fill N forms):

1. Track completed items by ID or index
2. On failure at item K: "Completed K-1 of N. Failed on item K: [reason]"
3. Ask user: "Skip this item and continue, or stop here?"
4. If user says continue, proceed from K+1
5. At the end, produce a summary: completed / skipped / failed

## Escalation Message Format

When sending browser URL for user intervention:

```text
I'm stuck at [current URL].

Open the browser: [browser URL]

What happened: [brief description]
What I tried: [actions taken]
Progress so far: [what was completed]

Please [specific ask]. Reply here when ready.
```
