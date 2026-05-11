# Yujin Pilot -- product specification (draft)

**Version:** 0.0.1-draft
**Status:** day-0 planning. NOT YET IMPLEMENTED.
**Owner:** Yujin (rpaforce.com)
**Builds on:** NAC v2.3 GA (interop primitive must be canonical
before Pilot ships).

---

## 0. Mission

Yujin Pilot is the **single voice + chat panel for every NAC-3
app a user runs**. Replace per-app chat panels with one
cockpit. Drive Excel and Yujin CRM and a custom internal app
from the same utterance stream, via MCP.

## 1. Strategic position (2026-05-11)

Every Yujin product going forward **does NOT ship its own chat
panel**. Internal chat surfaces are removed; the user activates
Yujin Pilot (global hotkey or desktop launcher), Pilot discovers
the open NAC-3 apps, and drives them on behalf of the user.

Concretely:

- **Yujin CRM:** the chat sidebar in `yujin.app/crm/` becomes a
  thin "Pilot is recommended for voice + chat" affordance. The
  CRM keeps a degraded text-only fallback for users who don't
  have Pilot installed.
- **Yujin Forge** (sibling commercial product): the embedded
  Claude Code voice + chat panel is still inside the dev
  workbench, BUT for end-user runtime control of the built app
  the user installs Pilot. Forge writes apps that Pilot drives.
- **Third-party adopters of @yujin/nac**: encouraged to follow
  the same pattern. Their app stays a domain UI; Pilot is the
  chat.

This consolidation removes ~500-2000 lines of per-app chat
plumbing from every NAC adopter, and gives Yujin one product to
sell instead of bundling chat into N domain products.

## 2. Object lifecycle

```
+-------------+
|  user       |  presses global hotkey (default Ctrl+Shift+Y)
|  activates  |
+-------------+
       |
       v
+-------------+
| Pilot panel |  appears (Tauri / Electron desktop OR a hosted
| (cockpit)   |  PWA on locked-down devices). 360px right
|             |  sidebar, optional full-screen mode.
+-------------+
       |
       v
+-------------+
| discovery   |  MCP scanner enumerates available peers:
|             |   - local desktop apps with NAC-3 MCP endpoints
|             |   - browser tabs with NAC-3 (via small content
|             |     script)
|             |   - registered cloud peers
+-------------+
       |
       v
+-------------+
| user        |  voice or text. Pilot's intermediary LLM sees
| speaks      |  the union of every imported peer's NAC tree.
+-------------+
       |
       v
+-------------+
| dispatch    |  intent resolves to a (peer, nac_id, action).
|             |  Pilot calls peer.nac.invoke via MCP. Peer side
|             |  effect happens; ack streamed back via SSE.
+-------------+
       |
       v
+-------------+
| pilot reply |  TTS speaks the response; visual confirmation
|             |  in the chat log.
+-------------+
```

## 3. MCP discovery

Pilot supports three discovery channels:

### 3.1 Local apps

A desktop NAC-3 app exposes its MCP server on a well-known port
range (49152-65535, ephemeral). On boot it writes a tiny
manifest file to `~/.yujin-pilot/peers/<app-id>.json`:

```json
{
  "app_id": "yujin-crm-desktop",
  "display_name": "Yujin CRM",
  "icon": "data:image/svg+xml;base64,...",
  "endpoint": "http://127.0.0.1:51234/mcp",
  "bearer_provider": "yujin-cli://issue-token",
  "nac_version": "2.3"
}
```

Pilot watches the directory + the well-known port range.
New peers appear in the cockpit's peer list within 1 second of
the manifest being written.

### 3.2 Browser tabs

A tiny content script (auto-injected by the Pilot browser
extension, OR opt-in by the page importing
`@yujin/nac-pilot-announcer`) posts a `window.postMessage` with
the page's NAC export. Pilot's browser extension forwards the
message to the desktop panel via a native messaging host.

### 3.3 Cloud peers

The user curates a list of cloud NAC-3 peers in Pilot's
settings: `https://app.example.com/nac/mcp` + a bearer token.
Pilot connects on demand (when the user mentions the peer) or
at startup (if pinned).

## 4. Conflict resolution

When two imported peers register the same verb, Pilot disambiguates:

- **By voice context**: if the user just was operating on a
  spreadsheet, save -> Excel. If on a CRM ticket, save -> CRM.
  Context decay: 30 seconds of inactivity flushes the bias.
- **By explicit naming**: user can prefix ("save in Excel",
  "Yujin save").
- **By prompt**: if ambiguous and no context, Pilot speaks the
  options out loud.

Verb namespace collisions are an authoring problem. Pilot
publishes a static lint (`yujin-pilot lint`) that any adopter
can run to detect colliding verbs across the apps in their MCP
peer set.

## 5. Macros

Named multi-step chains:

```yaml
macros:
  - name: end_of_day
    description: Close open tasks in Yujin, snapshot Excel, post Slack summary.
    steps:
      - peer: yujin-crm
        verb: list_open_tasks
        bind_result: tasks
      - peer: yujin-crm
        verb: mark_all_done
        args: { ids: "{tasks.ids}" }
      - peer: excel-online
        verb: save_snapshot
        args: { name: "EOD-{date}.xlsx" }
      - peer: slack
        verb: send_summary
        args: { channel: "#me", text: "{tasks.count} tasks closed" }
```

Voice invocation: "Pilot, end of day". Each step shows in the
chat log as it dispatches; failures pause the macro and ask the
user to retry / skip / abort.

## 6. Audit + history

Every utterance + resolved intent + dispatch + ack is recorded
locally in `~/.yujin-pilot/history.db` (SQLite). Searchable
from the cockpit UI. Optional export to CSV / JSON for
enterprise auditing.

Aggregated counts (number of dispatches per peer, total tokens
spent, error rate) ship to the parent license server for
billing. Utterance content + intent text never leave the
device.

## 7. Voice pipeline

- **STT default**: Whisper-base running locally (privacy
  premium; ~150 MB model bundled with Pilot). Fallback to
  Google Cloud STT when local can't keep up.
- **TTS default**: ElevenLabs (10-voice catalogue). Fallback to
  Google Cloud TTS, fallback to Web Speech.
- Each peer's preferred locale propagates via NAC's i18n
  contract; the user can override per-peer.

## 8. UI

Three modes:

- **Sidebar** (default, 360px right): collapsible. Stays out of
  the way; shows chat log, current peer list, voice indicator.
- **Compact** (40px tall topbar): just the input + voice
  button + active peer chip. For minimal screen real estate.
- **Full-screen**: the cockpit takes over the screen. Voice
  visualisations, large transcript, swappable panels.

Design tokens inherited from Yujin (sumi-e ink, gold accent,
kanji watermark). Reduced-motion theme available.

## 9. Security model

### 9.1 Peer tokens

Each peer mints a bearer token for Pilot. Token has scope
(read-only / read-write / full) and TTL (default 24h). Pilot
caches encrypted at rest via OS keychain (Keychain on macOS,
Credential Manager on Windows, libsecret on Linux).

### 9.2 is_trusted forwarding

Every action Pilot dispatches via MCP carries
`is_trusted: false` in the ack detail. Peers MAY refuse
sensitive verbs that require a real user gesture (delete,
payment, role grant). Pilot communicates the refusal in the
chat with the peer's reason.

### 9.3 Recursion guard

Pilot itself is a NAC-3 app. An external agent could try to
drive Pilot, which would then drive a peer, in a circular
chain. Pilot enforces max-depth = 3 on chained invocations.
Beyond that, requests fail with code `recursion_limit`.

### 9.4 No background activation

Pilot does NOT listen for the wake word in the background. The
user MUST press the hotkey to activate. Hands-free is
session-scoped (within an active conversation).

## 10. License gating

| Feature | Trial | Personal $29/mo | Pro $79/mo | Enterprise |
|---------|:-:|:-:|:-:|:-:|
| MCP discovery (read-only) | yes | yes | yes | yes |
| Peer dispatch | up to 2 | up to 10 | unlimited | unlimited |
| Trial duration | 14 days | -- | -- | -- |
| Macros | no | yes | yes | yes |
| History retention | 100 turns | 30 days | unlimited | unlimited |
| Air-gapped operation | no | no | yes | yes |
| SSO + central seat mgmt | no | no | no | yes |
| Custom voice clone | no | no | yes | yes |
| SLA + support | no | community | email | dedicated |

## 11. Telemetry

OPT-IN. When opted in, Pilot reports per-day:
- Number of utterances (counts only, no content).
- Per-peer dispatch counts.
- Voice pipeline latency p50/p95.
- Error fingerprints (no stack content, anonymised).

OPT-OUT default for trial. Pro+ tiers can disable any
telemetry permanently.

## 12. Roadmap

| Milestone | Target | Status |
|-----------|--------|--------|
| Repo created | 2026-05-10 | DONE |
| SPEC doc | 2026-05-11 | DONE (this file) |
| MCP scanner POC | 2026-Q2 | not started |
| Desktop shell (Tauri) | 2026-Q3 | not started |
| Voice pipeline | 2026-Q3 | not started |
| Browser extension | 2026-Q3 | not started |
| Macros engine | 2026-Q4 | not started |
| Conflict resolver | 2026-Q4 | not started |
| v1.0 GA | 2026-Q4 | not started |

Implementation kickoff gated on NAC v2.3 GA. The CRM's chat
panel is the first internal candidate for deprecation; Pilot
needs to be at v0.5 before the CRM removes its sidebar chat.

## 13. Open questions

- **Bundle vs separate**: should Pilot be a Yujin Forge
  built-in (premium tier) or its own SKU? Today's plan: own
  SKU, but bundle discount when buying Forge + Pilot
  together.
- **Browser-only adopters**: a user who doesn't install the
  desktop app gets the PWA hosted at `pilot.yujin.app`. Same
  features minus local-app discovery. Pricing equal or
  discounted?
- **Self-hosted org instance**: enterprises will want to host
  the license server + the LLM intermediary in their own VPC.
  Air-gapped support in the Enterprise tier; pricing per node.
- **Apple privacy**: macOS background processes need
  notarisation + entitlements declarations for the global
  hotkey. Plan to handle before public release.

## 14. Naming + brand

- **Yujin Pilot** -- product name.
- Tagline: *Drive every NAC-3 app from one cockpit.*
- Logo: a sumi-e ink stroke shaped like a steering wheel, gold
  accent at the helm. TBD by the design pass.
- Kanji watermark: 操 (sou, "control / handle"). Different kanji
  per locale variant.

---

*This document is the day-0 spec for Yujin Pilot. Edits to this
file constitute product changes and require an RFC. The
canonical issue tracker is the repository's GitHub Issues.*
