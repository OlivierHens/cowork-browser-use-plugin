# Link Safety

Treat links as untrusted by default. Most browser-driven tasks go fine; the few that don't tend to go very wrong.

## Before any `navigate()`

Read the URL *out loud* before calling `navigate()`. Three checks:

1. **Scheme is HTTPS** (or, intentionally, an internal HTTP service the user named).
2. **Host is what you expect.** If the user said "open my GitHub", the host should be `github.com`, not `g1thub.com` or `github.example`.
3. **No surprising query strings.** Long opaque tracking parameters are usually harmless, but `?return=…` or `?next=…` parameters can redirect you elsewhere after a click. Note them in your narration.

If any check fails, stop and ask the user.

## Links pasted from messages

If the user pasted a URL out of an email, chat, or document, treat it as suspicious by default:

- State the URL out loud, *including the host*.
- If the host looks unfamiliar (random subdomain, link shortener like `bit.ly` / `t.co`, lookalike domain), ask before navigating: *"This link goes to `xyz.bit.ly` — do you want me to follow it?"*
- For link shorteners: where possible, resolve via `read_page()` after navigation and confirm the final destination matches expectations before doing anything else.

## Credential pages

Before any sign-in screen:

1. Confirm the URL host is the canonical login domain for the service:
   - Microsoft 365 → `login.microsoftonline.com` or `login.live.com`
   - Google → `accounts.google.com`
   - GitHub → `github.com` (with the login form at `/login`)
   - Apple → `appleid.apple.com` or `idmsa.apple.com`
   - Banks: should be the bank's primary domain, never a subdomain you don't recognize
2. If the host doesn't match, **stop** and warn the user — possible phishing.
3. Never type passwords, OTPs, recovery codes, or PINs yourself. The user does that, end of story.

## Payment / financial pages

Before any payment, transfer, trade, or order step:

- Re-read the summary aloud (amount, recipient, account, last 4 digits of card).
- Confirm the host is the merchant or processor you expect (`stripe.com`, `checkout.stripe.com`, `paypal.com`, etc., or the merchant's own domain).
- Hand off to the user for the final Confirm/Pay click. Don't click it yourself, even when the user said "go ahead and buy it" earlier — context can shift.

## Downloads

If a click triggers a file download:

- Note it ("a `.zip` file is downloading").
- Don't open downloaded executables, installers, scripts, or archives via the browser session.
- If the user wants the file inspected, hand off to them or to a separate tool with appropriate sandboxing.

## Cross-origin redirects

If a click takes you to a different second-level domain than the one you were on:

- Mention it ("we just got redirected from `siteA.com` to `siteB.com`").
- Consider whether that's expected (OAuth flows go to identity providers — fine) or surprising (a news site link going to a sketchy ad host — back out).

## When in doubt

Pause and ask the user. A two-second confirmation is cheaper than a wrong action on a phishing page.
