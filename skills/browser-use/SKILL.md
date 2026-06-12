---
name: browser-use
description: |
  Drives a real web browser like a human — navigates to a URL, looks at the
  page via a screenshot, and clicks and types to accomplish a goal. Use when
  the user asks to "open a website", "go to <URL>", "browse to …",
  "log into …", "fill out this form", "buy …", "book …", "order …",
  "scrape this page", "click around …", "check the price of …", or any task
  that requires interacting with a live web UI. Also use whenever a request
  cannot be answered from search results alone and needs real-time page
  state, an authenticated session, or a multi-step interaction.
license: MIT
metadata:
  author: Olivier Hens
  version: "0.1"
---

# Browser Use — Human-like Web Browsing

## What this skill does

Uses the **Browser Use** connector to drive a real browser one small step at a time, the way a careful person would: look at the page, decide on a single next action, do it, then look again to confirm what changed. The goal is to be slow, observable, and recoverable — not fast and brittle.

## When to activate

Activate as soon as the user wants you to interact with a webpage rather than just search the web. Telltale signals:

- A URL or a specific website is named.
- The user wants the *current* state of a page (prices, availability, dashboard, inbox).
- The task involves filling a form, logging in, clicking through, or scraping.
- A previous search-only attempt didn't produce a usable answer.

If you're not sure whether to use it, ask the user once: *"Want me to actually open that in a browser?"*

## Connector tools you'll use

The Browser Use connector exposes (names may vary slightly depending on the server version — fall back to whatever `tools/list` returns):

- `navigate(url)` — load a URL in the active tab.
- `screenshot()` — capture what's currently on screen. **Always your first move on a new page.**
- `read_page()` — get a structured outline of the page (text + interactive elements, ids/labels).
- `click(target)` — click an element identified by visible text, role, or label.
- `type(target, text)` — type into a field identified the same way.
- `scroll(direction, amount)` — scroll the viewport.
- `wait(seconds)` or `wait_for(condition)` — pause; use sparingly.

Reference these tools **by name** in your narration so the user can follow along.

## The human-like loop

For every page or every step, run this loop:

1. **Look.** Call `screenshot()` (and `read_page()` if the page is text-heavy). Describe out loud what you see: page title, the part that matters, any modals or banners.
2. **Decide one action.** Pick the single next thing a person would do — not three at once. Identify the target by *visible* attribute: button text, link text, field label, aria-role. Not by CSS selector unless nothing else works.
3. **Handle blockers first.** Cookie banners, "are you 18?" prompts, GDPR consent, login walls, "verify you're human" — deal with these *before* pursuing the goal. If a captcha or hard bot check appears, stop and report (see `references/recovery-playbook.md`).
4. **Act.** Call the tool (`click`, `type`, `scroll`).
5. **Wait briefly.** A short `wait(1)` after a click or navigation lets the page settle. Not every action needs one, but auth pages and SPA route changes usually do.
6. **Look again.** Call `screenshot()` and confirm the expected change happened. State the confirmation: *"Confirmed: the search results page is loaded, I can see 10 results."* If the expected change *didn't* happen, do not retry the same action — diagnose first (`references/recovery-playbook.md`).
7. **Repeat** from step 2 until the goal is met or you need to stop.

## Output format — narrate as you go

Speak in present-tense, first-person, one short line per loop iteration. Three slots:

```
I see  → <one line: what's on screen right now>
I'm doing → <the single tool call you're making>
Confirmed → <one line: what changed after the action>
```

Example for *"Open Hacker News and tell me the top story"*:

```
I see  → Hacker News homepage, story list visible. Top item is "Show HN: …".
I'm doing → reading the page to grab the title and points cleanly.
Confirmed → Top story: "Show HN: A new approach to …" (231 points).
```

Drop the narration only for trivial fall-throughs (e.g., a sub-second redirect).

## Rules of the road

- **Never act blind.** Always `screenshot()` before the first `click()` on a page.
- **Identify by what a person would see.** Visible text > aria-label > role > nth-of-type, in that order.
- **One action per loop.** No multi-clicks or batched calls — Cowork can't recover if the second one fails.
- **Scroll into view before clicking** an element below the fold.
- **Trust your eyes, not your assumptions.** If the screenshot disagrees with what you expected, the screenshot is right.
- **Be careful with submit buttons.** Re-read the form before pressing "Submit", "Pay", "Confirm", "Delete". State what you're about to do and pause.
- **Don't follow links from messages.** If the user pasted a URL from email/chat, name the destination domain out loud and ask before navigating to it. See `references/link-safety.md`.
- **Never log in for the user.** Stop at the login form. Tell the user what credential is needed and let them complete the auth.
- **Stay on the user's goal.** If you find yourself three pages deep in unrelated territory, back out and re-anchor.

## When to stop

Stop immediately and report when:

- A captcha, bot check, or 2FA prompt appears (see `references/recovery-playbook.md`).
- A "Confirm payment", "Confirm delete", or other irreversible action is the next step. The user must approve.
- The page domain isn't what was expected (potential redirect / phishing).
- You've retried the same step twice without progress.
- The user said "stop" or revoked the connector.

## Additional resources

- **`references/interaction-patterns.md`** — How to identify elements reliably; pacing rules; scroll/lazy-load handling.
- **`references/recovery-playbook.md`** — Failure modes (element-not-found, captcha, modal, auth wall, rate limit) with concrete back-out recipes.
- **`references/link-safety.md`** — Rules for unfamiliar URLs, credential pages, and links from untrusted messages.
