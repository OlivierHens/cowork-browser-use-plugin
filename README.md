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

## Pick your install path

Cowork supports three ways to add a custom plugin. Most non-admin users want **Option 1**.

| Option | Who it's for | What you get | Admin needed |
| --- | --- | --- | --- |
| 1. **In-app upload** (Cowork → Manage plugins → Browse plugins → My plugins → upload) | You, on your own account | Full plugin (skill + connector) | No |
| 2. **OneDrive folder-drop** (`Documents/Cowork/skills/`) | You, but only the skill | Skill only — no MCP connector | No |
| 3. **Admin-center deploy** | Your IT admin, rolling out to the whole tenant | Full plugin, auto-installed for users | Yes |

---

## Option 1: In-app upload (recommended)

This matches the UI you see when you click **+ → Manage plugins → Browse plugins → My plugins → "upload a plugin"** in Cowork. No admin role required.

### 1. Get the package

Either build from source:

```powershell
git clone https://github.com/OlivierHens/cowork-browser-use-plugin
cd cowork-browser-use-plugin
```

…or download `cowork-browser-use-plugin.zip` from the repo's [**Releases**](https://github.com/OlivierHens/cowork-browser-use-plugin/releases) page once one is published.

### 2. Configure the Browser Use connector

See [Configure the Browser Use connector](#configure-the-browser-use-connector) below. You need to set two values in `manifest.json` before packaging:

- `mcpServerUrl` — the HTTPS URL of your Browser Use MCP server
- `authorization.referenceId` — the credential reference ID from Microsoft's token store

### 3. Zip it

From the repo root:

```powershell
Compress-Archive -Path manifest.json, color.png, outline.png, skills `
  -DestinationPath cowork-browser-use-plugin.zip -Force
```

The `manifest.json` must be at the root of the zip — don't include the outer folder.

### 4. Upload in Cowork

1. Open **Cowork** (m365.cloud.microsoft → Cowork).
2. Above the chat input, click **+ → Manage plugins**.
3. In the **Manage plugins** dropdown, click **Browse plugins**.
4. Switch to the **My plugins** tab.
5. Click **upload a plugin** (next to "Add a marketplace or upload a plugin").
6. Pick `cowork-browser-use-plugin.zip`. Cowork validates the manifest and skill.
7. Toggle the plugin **on**. The first time you trigger it, complete the connector sign-in (API key prompt or OAuth consent).

You should now see `browser-use` under **Sources & Skills** and **Browser Use** under **Connectors**.

---

## Option 2: OneDrive folder-drop (skill only, no connector)

The fastest way to get *just the skill* into Cowork — useful if you want to read or tweak the workflow without wiring up a Browser Use server yet. Cowork auto-discovers `SKILL.md` files in `Documents/Cowork/skills/` and loads them at the start of every conversation, up to 50 per user.

### Caveat

This path installs only the skill. **It does not install the Browser Use MCP connector.** Without the connector, the skill has no `navigate` / `screenshot` / `click` tools to call — Cowork will know *how* a person would browse but won't actually be able to. Use this path when:

- You already have a different way to get browser tools into Cowork (e.g. a separate connector already deployed by your admin), or
- You just want to read and customize the skill content offline.

For the full plugin (skill + connector), use Option 1.

### Steps

1. In File Explorer, open your OneDrive root and navigate to `Documents/Cowork/skills/`. Create the `skills` folder if it doesn't exist.
2. Copy the entire `skills/browser-use/` folder from this repo into it. The result should look like:
   ```
   OneDrive/
     Documents/
       Cowork/
         skills/
           browser-use/
             SKILL.md
             references/
               interaction-patterns.md
               recovery-playbook.md
               link-safety.md
   ```
3. Open a fresh Cowork conversation. The skill loads automatically; you'll see it appear in the side panel when relevant.

To remove it, just delete the `browser-use` folder from OneDrive.

---

## Option 3: Admin-center deploy (org-wide)

For an admin rolling the plugin out to a whole tenant — skip this section if you're installing for yourself.

1. Build/zip the package as in Option 1, steps 1–3.
2. Sign in to the [Microsoft 365 admin center](https://admin.microsoft.com) as a **Global**, **Teams**, or **Copilot** admin.
3. **Settings → Integrated apps → Upload custom apps** (or **Copilot → Agents → All agents → Upload**).
4. Choose **App package file** and pick `cowork-browser-use-plugin.zip`.
5. Review the requested permissions, assign to users/groups, and deploy.
6. Targeted users see the plugin labeled **Managed by your organization** in their **Added Plugins** tab; they can enable/disable per conversation but can't remove it.

---

## Configure the Browser Use connector

The connector talks to a remote MCP server you provide. The plugin doesn't ship one — you bring your own endpoint and credential.

### 1. Get an MCP endpoint URL

- **Browser Use Cloud**: sign up at [browser-use.com](https://browser-use.com), enable the MCP server in your project settings, and copy the URL. It looks something like `https://api.browser-use.com/mcp` (consult the current docs for the exact path).
- **Self-hosted**: run Browser Use behind a public HTTPS endpoint (Caddy / Cloudflare Tunnel / etc.) with TLS 1.2 or higher. The endpoint must speak Streamable HTTP MCP (JSON-RPC 2.0) and respond to `tools/list` and `tools/call`.

### 2. Register an auth client

Follow [Configure authentication for MCP and API plugins in Microsoft 365 Copilot](https://learn.microsoft.com/en-us/microsoft-365/copilot/extensibility/plugin-authentication) to register an **API key** or **OAuth** client. This step happens in the Teams Developer Center / Agents Toolkit and returns a `referenceId` you paste into the manifest — the secret itself never enters the package.

You'll get back a **referenceId** that looks like `A1bC2dE3fH4iJ5kL6mN7oP8qR9sT0u`.

> **No tenant admin?** You can still register a personal OAuth/ApiKey client tied to your own account — Microsoft's token store works for individual users too. If you hit a permissions wall, ask your admin to register the client once on your behalf and share the `referenceId`.

### 3. Patch `manifest.json`

| Field | Path | Value |
| --- | --- | --- |
| Endpoint | `agentConnectors[0].toolSource.remoteMcpServer.mcpServerUrl` | Your MCP endpoint URL |
| Reference ID | `agentConnectors[0].toolSource.remoteMcpServer.authorization.referenceId` | The referenceId from step 2 |

If your server uses OAuth instead of an API key, change `authorization.type` from `"ApiKeyPluginVault"` to `"OAuthPluginVault"`.

Re-zip after editing (Option 1 step 3).

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
| Cowork itself doesn't appear in m365.cloud.microsoft | You (or your admin) aren't enrolled in Frontier | Enroll via the Frontier preview program; admins also need Copilot → Settings → Frontier |
| `Browse plugins` button missing in Cowork | Plugin system is disabled at the tenant level | Ask your admin to re-enable it |
| `upload a plugin` rejected | Manifest validation error | Re-check the [validation checklist](#validation-checklist) below |
| OneDrive skill not loading (Option 2) | Folder path or `SKILL.md` filename wrong, or you're over the 50-skill cap | Path must be exactly `Documents/Cowork/skills/<name>/SKILL.md` (case-sensitive), `name` in frontmatter must match the folder |
| Skill never triggers | Trigger phrases don't match what users actually say | Edit the `description:` block in `skills/browser-use/SKILL.md` and re-upload |
| Connector returns 401 | `referenceId` doesn't match a registered client, or your sign-in expired | Re-register the auth client and patch `manifest.json`; or revoke + re-consent from **Sources & Skills** |
| Connector hangs / 504 | Browser Use server is slow or down | Cowork allows up to 30 s per tool call — check your hosted endpoint |
| Tool calls return data but the skill misuses them | Tool names in `SKILL.md` don't match what your MCP server actually exposes | Hit your `tools/list` endpoint, then rename the tools in `SKILL.md` to match |

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
3. Re-upload through whichever path you originally used:
   - **In-app upload (Option 1)**: Cowork → Manage plugins → Browse plugins → My plugins → remove the old version, then upload the new `.zip`. The `id` (GUID) staying the same lets Cowork recognize it as the same plugin.
   - **OneDrive folder-drop (Option 2)**: just overwrite the files in `Documents/Cowork/skills/browser-use/`. The next conversation picks up the new version.
   - **Admin-center deploy (Option 3)**: re-upload via Integrated apps → Upload custom apps; same `id` is treated as an update.

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
