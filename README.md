# Browser Use for Copilot Cowork

A Microsoft 365 **Copilot Cowork (Frontier)** plugin that lets Cowork drive a real web browser the way a person would: navigate to a URL, look at the page with a screenshot, then click and type to accomplish a goal — handling cookie banners, verifying every action, and recovering when things go sideways.

The plugin contains two pieces:

- A **`browser-use` skill** — the prompt-level workflow that teaches Cowork *how* to use a browser like a human (look before clicking, verify after every action, slow down on auth, surface unfamiliar URLs).
- A **Browser Use connector** — a remote MCP server (e.g., [Browser Use Cloud](https://browser-use.com) or your own self-hosted Browser Use instance) that exposes the actual `navigate` / `screenshot` / `click` / `type` / `scroll` / `wait` tools.

---

## How it works (one paragraph)

Copilot Cowork plugins ship as M365 App Packages (`.zip`) and can only call **remote** MCP servers over HTTPS — there's no local-process / stdio support and no way to bundle a Chromium binary in a Cowork plugin. So instead of running the browser on your desktop, this plugin connects Cowork to a hosted browser-automation MCP server, and adds a skill that tells Cowork to use it human-style. If you want to drive the browser sitting on your own machine, that's a different category of tool (e.g., the Claude in Chrome extension) — not a Cowork plugin.

---

## Prerequisites

Before installing, make sure you have:

1. A **Microsoft 365 tenant with Copilot Cowork enabled**.
2. Enrollment in the **[Frontier preview program](https://adoption.microsoft.com/en-us/copilot/frontier-program/)**. Your admin account also needs to be enrolled (Copilot → Settings → Frontier) — otherwise Cowork won't appear in Agent management in the admin center.
3. **Custom App Upload** enabled for your tenant (most M365 E3/E5/Business Premium tenants have this on by default).
4. A reachable **Browser Use MCP endpoint**:
   - [Browser Use Cloud](https://browser-use.com) account (easiest), **or**
   - A self-hosted Browser Use server with HTTPS + TLS 1.2+ at a public URL.
5. **PowerShell 5.1+** (only if you're building from source on Windows).

---

## Install — Option A: Build from source

```powershell
git clone https://github.com/OlivierHens/cowork-browser-use-plugin
cd cowork-browser-use-plugin
```

Edit `manifest.json` and replace the two `REPLACE_ME` values (see [Configure the Browser Use connector](#configure-the-browser-use-connector) below), then zip:

```powershell
Compress-Archive -Path manifest.json, color.png, outline.png, skills `
  -DestinationPath cowork-browser-use-plugin.zip -Force
```

You now have `cowork-browser-use-plugin.zip` ready to sideload.

> **Note**: replace `color.png` and `outline.png` with real branded icons before submitting to the Microsoft 365 App Store. The shipped versions are solid-color placeholders so the package validates.

## Install — Option B: Download a prebuilt release

1. Grab `cowork-browser-use-plugin.zip` from the [**Releases**](https://github.com/OlivierHens/cowork-browser-use-plugin/releases) page (once one is published).
2. Unzip it, edit `manifest.json` to set your connector URL + reference ID, then re-zip with the same command above.

---

## Configure the Browser Use connector

The connector talks to a remote MCP server you provide. The plugin doesn't ship one — you bring your own endpoint and credential.

### 1. Get an MCP endpoint URL

- **Browser Use Cloud**: sign up at [browser-use.com](https://browser-use.com), enable the MCP server in your project settings, and copy the URL. It looks something like `https://api.browser-use.com/mcp` (consult the current docs for the exact path).
- **Self-hosted**: run Browser Use behind a public HTTPS endpoint (Caddy / Cloudflare Tunnel / etc.) with TLS 1.2 or higher. The endpoint must speak Streamable HTTP MCP (JSON-RPC 2.0) and respond to `tools/list` and `tools/call`.

### 2. Register an auth client

Follow [Configure authentication for MCP and API plugins in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/plugin-authentication) to register an **API key** or **OAuth** client with the Microsoft Enterprise Token Store.

When registering, set **Usage by organization** to **Any Microsoft 365 Organization** so the connector works for all your users.

You'll get back a **referenceId** (looks like `A1bC2dE3fH4iJ5kL6mN7oP8qR9sT0u`). The secret itself never enters the package.

### 3. Patch `manifest.json`

| Field | Path | Value |
| --- | --- | --- |
| Endpoint | `agentConnectors[0].toolSource.remoteMcpServer.mcpServerUrl` | Your MCP endpoint URL |
| Reference ID | `agentConnectors[0].toolSource.remoteMcpServer.authorization.referenceId` | The referenceId from step 2 |

If your server takes an OAuth flow instead of an API key, also change `authorization.type` from `"ApiKeyPluginVault"` to `"OAuthPluginVault"`.

Re-zip with `Compress-Archive` after editing.

---

## Sideload into Cowork

1. Sign in to the [Microsoft 365 admin center](https://admin.microsoft.com) as a **Global**, **Teams**, or **Copilot** admin.
2. **Settings → Integrated apps → Upload custom apps**.
3. Choose **App package file** and pick `cowork-browser-use-plugin.zip`.
4. Review the requested permissions, assign the app to users/groups, and confirm.
5. Open Cowork → **Sources & Skills**. You should see:
   - **`browser-use`** under **Skills**
   - **Browser Use** under **Connectors**
6. The first time you trigger the skill, complete the connector sign-in (API key prompt or OAuth consent).

---

## Try it

Drop one of these into Cowork:

- *"Open https://news.ycombinator.com and tell me the top three story titles."*
- *"Go to wikipedia.org and search for 'octopus cognition'. Summarize the lead section."*
- *"Open the BBC homepage, take a screenshot, and describe the top headline."*

You should see Cowork narrate what it's doing — *"I see the homepage with a search box at the top. I'm clicking the search box. Confirmed: it has focus and the cursor is in it…"* — instead of acting blind.

For a task that touches authentication or unfamiliar links, the skill will surface the destination URL and pause before submitting credentials. That's the `link-safety` rules in `skills/browser-use/references/link-safety.md` doing their job.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Cowork doesn't show in Agent management | Admin not enrolled in Frontier | Enroll: Copilot → Settings → Frontier in admin center |
| Sideload upload rejected | Manifest validation error | Re-check the [validation checklist](#validation-checklist) below |
| Skill never triggers | Trigger phrases don't match what users actually say | Edit the `description:` block in `skills/browser-use/SKILL.md` and re-package |
| Connector returns 401 | `referenceId` doesn't match registered client | Re-register the auth client and patch `manifest.json` |
| Connector hangs / 504 | Browser Use server is slow or down | Cowork allows up to 30 s per tool call — check your hosted endpoint |
| Tool calls return data but the skill misuses them | Tool names in SKILL.md don't match what your MCP server actually exposes | Hit your `tools/list` endpoint, then rename the tools in `SKILL.md` to match |

---

## Validation checklist

Before zipping, double-check:

- `manifest.json` parses as JSON and `manifestVersion` is `"1.28"`.
- `id` is a valid GUID. (The shipped one is fine for sideload; generate a new one if you're forking.)
- `agentSkills[0].folder` is `./skills/browser-use` and that folder contains `SKILL.md`.
- `SKILL.md` has YAML frontmatter between `---` delimiters with `name: browser-use` (matches the folder, kebab-case).
- `agentConnectors[0]`:
  - has a unique `id`,
  - has exactly one of `plugin` or `remoteMcpServer`,
  - has an HTTPS `mcpServerUrl`,
  - has `authorization.referenceId` if `type` is not `None`.
- `color.png` is 192×192 and `outline.png` is 32×32.
- No companion file in `references/` exceeds 5 MB, and the total stays under 10 MB.

---

## Updating

1. Bump `"version"` in `manifest.json` (semver: PATCH for fixes, MINOR for new behavior, MAJOR for breaking changes).
2. Re-zip with the same command.
3. Re-upload via **Integrated apps → Upload custom apps**. As long as the `id` (GUID) is unchanged, the admin center treats this as an update rather than a new app.

---

## What's not included (out of scope)

- **Local browser control.** Cowork's plugin model is HTTPS-only — no way to ship a local browser binary. Use a hosted Browser Use server.
- **Slash commands, sub-agents, hooks.** Those are Claude-plugin features that Cowork's manifest doesn't support yet.
- **Microsoft 365 App Store submission.** This README stops at sideload-ready `.zip`. To publish, you need real branded icons, a real privacy URL, a real terms URL, and Partner Center onboarding.

---

## Repo layout

```
cowork-browser-use-plugin/
├── manifest.json                # M365 Unified App Manifest v1.28
├── color.png                    # 192x192 placeholder icon
├── outline.png                  # 32x32 placeholder icon
├── README.md
├── .gitignore
└── skills/
    └── browser-use/
        ├── SKILL.md             # The human-like browsing workflow
        └── references/
            ├── interaction-patterns.md
            ├── recovery-playbook.md
            └── link-safety.md
```

---

## References

- [Build plugins for Cowork (Frontier)](https://learn.microsoft.com/en-us/microsoft-365/copilot/cowork/cowork-plugin-development) — Microsoft Learn (authoritative)
- [Available plugins for Cowork (Frontier)](https://learn.microsoft.com/en-us/microsoft-365/copilot/cowork/cowork-available-plugins)
- [Plugin manifest schema 2.4 for Microsoft 365 Copilot](https://learn.microsoft.com/microsoft-365/copilot/extensibility/plugin-manifest-2.4)
- [Configure authentication for MCP and API plugins](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/plugin-authentication)
- [Browser Use](https://browser-use.com) — the open-source browser-automation library this plugin is designed to drive

---

## License

MIT — see `LICENSE` (or this section, until one's added). Contributions welcome via PR.
