# Recovery Playbook

When something doesn't work as expected, do not retry the same action and hope. Diagnose, then choose the right recipe below.

## General rule

Before any recovery, take a fresh `screenshot()` and *describe what you actually see*. Most recovery failures come from acting on a stale mental model.

---

## Failure: element not found

Symptoms: `click()` or `type()` returns "no such element", or the next screenshot shows the page unchanged.

Diagnose:

1. Re-screenshot. Is the element actually on screen?
2. If not visible: scroll toward where it should be, screenshot, retry.
3. If visible: the locator is probably wrong. Switch to a different identification strategy (text → role → label → nearest landmark).

Stop after **two** locator changes. Tell the user what you tried and what you saw.

---

## Failure: navigation didn't load

Symptoms: blank page, infinite spinner, or 5xx error after `navigate()`.

Diagnose:

1. Wait 3 seconds, screenshot once more.
2. If still blank: try the URL one more time. Many SPAs fail their first hydration after a cold connector start.
3. If still blank: report to the user with the URL and the status (blank vs. error page). Don't keep retrying.

---

## Failure: modal or popup blocking

Symptoms: a screenshot shows the goal area covered by a dialog (cookie banner, signup nag, "We use cookies", "Subscribe to our newsletter").

Recipe:

1. Identify the dismissive option by text: "Accept", "Accept all", "Reject all", "Continue without accepting", "Close", "×", "No thanks".
2. Prefer the user-friendly outcome: accept *necessary* cookies, reject tracking when there's a clear option, dismiss newsletter popups.
3. Screenshot and confirm the dialog is gone before continuing toward the original goal.

If the popup keeps reappearing every navigation, mention it once and proceed.

---

## Failure: captcha or bot check

Symptoms: hCaptcha, reCAPTCHA, Cloudflare "checking your browser", Turnstile, "Press and hold to confirm you're human".

**Stop.** Do not attempt to solve.

1. Screenshot.
2. Tell the user: *"This page is asking for a CAPTCHA. I can't solve it — can you complete it, then tell me to continue?"*
3. Wait for the user's go-ahead.
4. Re-screenshot before resuming.

Never click "I'm not a robot" yourself even if the checkbox seems to "just work" sometimes — many sites silently flag automated solves.

---

## Failure: auth wall / sign-in required

Symptoms: redirected to a login screen, or the page shows a "Sign in to continue" panel.

Recipe:

1. Confirm the domain in the URL bar matches the expected service (Microsoft 365 → `login.microsoftonline.com`, Google → `accounts.google.com`, etc.). If the domain looks off, **stop** — possible phishing.
2. Tell the user which account/credential is needed.
3. Pause and let the user complete sign-in.
4. After resuming, screenshot before continuing — auth flows often land you on a different URL than where you started.

Never type a password, OTP, or recovery code on the user's behalf, even if they've shared one in the conversation.

---

## Failure: rate limit / blocked

Symptoms: HTTP 429, "Too many requests", "Access denied", or a site that abruptly returns 403 after a few clicks.

Recipe:

1. Stop interacting with that site for this session.
2. Tell the user: *"This site is rate-limiting the browser session. We should pause and come back later, or switch approach."*
3. If the task is urgent, suggest an alternative (API, mobile site, RSS) when one exists.

---

## Failure: page state desynced

Symptoms: the screenshot disagrees with what you thought you did three steps ago — wrong tab, wrong page, lost form data.

Recipe:

1. Stop. Don't try to "click back to where you were".
2. State the current URL and what's visible.
3. Decide whether to:
   - **Reset**: navigate to a known starting URL and begin again.
   - **Resume**: if you can identify a way forward from here, name it explicitly and continue from a screenshot.
4. Don't redo destructive actions (form submits, deletes) blindly.

---

## Failure: irreversible action ahead

Symptoms: next step would be Submit / Pay / Send / Delete / Confirm.

Always:

1. Screenshot the summary.
2. State exactly what's about to happen ("This will charge $42 to the card ending 1234 and place the order").
3. Ask the user to confirm. Don't proceed on implicit consent.

---

## Hard stops

Drop everything and report when:

- The URL domain doesn't match what was expected.
- A page is asking for credentials, payment info, or 2FA codes.
- You've gone three loop iterations without progress on the user's actual goal.
- The connector returns an error twice in a row.
- The user says any form of "stop", "wait", "hold on".
