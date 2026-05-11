# Yujin Pilot

**One voice and chat panel that pilots every NAC-3 compliant app
on your machine, across origins, via MCP.**

Pull every NAC-3 app into a single cockpit. Sit in front of the
Pilot panel; talk to it. The panel detects the open apps, imports
their NAC trees through their MCP servers, and dispatches your
intent to the right peer. No more per-app chat widget; no more
context switching to ask each app the same thing.

> Status: **day 0**, 2026-05-10. Planning + branding anchor.
> Implementation kickoff after NAC v2.3 GA.

## What it is

Yujin Pilot is the **commercial NAC controller** -- a sibling
product to Yujin Forge in the Yujin commercial line. Where Forge
is for *building* NAC apps, Pilot is for *driving* them.

- A native or web-based panel with voice + chat, NAC-3 conformant
  itself.
- An MCP discovery layer that scans the user's environment (local
  apps, browser tabs, registered cloud peers) and lists every
  NAC-3 endpoint it finds.
- A unified intent router: the LLM intermediary sees the union of
  every imported peer's NAC tree. "Set cell A1 to 100" finds
  Excel; "open invoice #INV-005" finds Yujin CRM; the same
  utterance routes the action correctly.
- A handoff metaphor: when the user says "Yujin, jump to Excel,"
  Pilot doesn't open Excel from scratch -- it focuses the
  already-open Excel window and proxies its NAC controls back
  into the Pilot conversation.
- License-gated by Yujin parent (rpaforce.com).

## Why a separate product

NAC apps each ship their own chat panel today. That's friction:

- Different shortcuts to invoke chat per app.
- No cross-app intent ("paste the deal total from Yujin into
  the spreadsheet I have open").
- Per-app voice setup (locale, TTS voice, microphone gain).

Yujin Pilot consolidates: one chat, ten apps. The apps don't
need to ship a chat at all (saves their build budget). The user
gets a single voice + visual interface; the apps stay focused on
their domain UI.

## What it ships (v1.0 target)

- **Desktop app (Tauri or Electron)** for macOS / Windows / Linux.
  Stays in the taskbar; activates with a global hotkey.
- **Web build** for ChromeOS, kiosk, mobile -- consumed as a
  hosted PWA.
- **MCP scanner**: detects (a) local NAC-3 apps via well-known
  ports, (b) browser tabs running NAC-3 (via a tiny content
  script that announces `nac.health`), (c) cloud peers from a
  user-curated registry.
- **Voice pipeline**: STT via Google Cloud / Whisper / local
  Coqui (configurable). TTS via ElevenLabs / Google Cloud /
  Web Speech.
- **Conflict resolver**: when two imported apps both register
  a verb `save`, Pilot disambiguates with the user ("save in
  Excel or save in Yujin?").
- **Macros**: chain a sequence of cross-app intents into a named
  shortcut ("end-of-day": close Yujin tasks, snapshot Excel,
  send Slack summary).
- **History**: every utterance + intent + dispatch + ack is
  recorded for audit. Searchable.

## License gating

### Trial

- 14 days. Up to 2 peers connected simultaneously.
- Hotkey activation works. Macros disabled. History capped at
  100 turns.

### Paid seat

- Issued by the parent Yujin license server.
- N peers (plan-dependent: Personal $29/mo / 10 peers, Pro
  $79/mo / unlimited peers + macros).
- Org plans allow N seats with central billing.

### What stays free

- The MCP discovery layer (without dispatch) -- so the user can
  audit what apps Pilot would see, without committing.
- The web build's read-only inspector mode.

## Architecture

```
       +----------------+
       |  Yujin Pilot   |     desktop panel + global hotkey
       |  (you)         |     voice in, voice out
       +-------+--------+
               |
               | MCP discover + import
               v
  +------------+------------+
  |  MCP scanner            |  scans local + browser + cloud peers
  +------+--------+---------+
         |        |         |
  +------v---+ +--v--+ +----v-----+
  | NAC app A| | B   | | C        |  any NAC-3 conformant app
  +----------+ +-----+ +----------+
```

When Pilot imports a peer, that peer's manifests show up in
Pilot's NAC.describe() namespaced as `remote:<peer-id>:*`. The
underlying mechanism is the same v2.3 interop layer that ships
in the open NAC runtime (`pkuschnirof/rpaforce-crm:js/nac-mcp-interop.js`)
-- Pilot is "just" a polished consumer of it, plus the discovery,
disambiguation, macros, and license-gating.

## Look & feel

- Sumi-e ink aesthetic. Kanji watermark behind the chat log
  (different kanji per locale). Gold accent for the action verbs.
- Dark + light themes. Reduced-motion theme for accessibility.
- Push-to-talk by default; hands-free toggle per session.
- The chat log is itself a NAC plugin (`pilot.chat.*`) so an
  external agent can drive Pilot through the same contract Pilot
  uses to drive peers. Recursion is bounded (max 3 hops) to
  avoid amplification attacks.

## Day 0 commitments

- The Pilot binary is closed source. License-gated.
- The MCP scanner's wire protocol is documented as part of the
  NAC v2.3 spec (so any tool can implement compatible discovery).
- User audit logs stay local; only aggregated counts go to the
  parent for billing.
- No telemetry of utterance content or intent text.
- Per-app pause + revoke: the user can disconnect any peer at
  any moment, and the disconnect is enforced server-side too
  (the peer's MCP token is invalidated).

## Where this lives in the Yujin ecosystem

```
NAC (Apache-2.0)              -- rpaforce-crm + yujin.app/nac-spec
  spec + runtime + npm pkg + interop layer
       |
       | first commercial layer
       v
Yujin Forge (closed core)     -- pkuschnirof/yujin-forge
  React framework + Claude Code embedded for BUILDING NAC apps
       |
       | sibling commercial product
       v
Yujin Pilot (closed core)     -- pkuschnirof/yujin-pilot (this repo)
  Voice + chat cockpit for DRIVING NAC apps across origins
```

## Roadmap

| Milestone | Target | Status |
|-----------|--------|--------|
| Repo created | 2026-05-10 | DONE |
| Spec doc | 2026-05-11 | DONE (docs/SPEC.md) |
| MCP scanner POC | 2026-Q2 | not started |
| Desktop shell (Tauri) | 2026-Q3 | not started |
| Voice pipeline | 2026-Q3 | not started |
| Macros engine | 2026-Q4 | not started |
| v1.0 GA | 2026-Q4 | not started |

Implementation gated on NAC v2.3 GA (interop primitive must be
canonical first).

## See also

- [NAC spec](https://yujin.app/nac-spec/SPEC.md)
- [NAC interop spec (v2.3 preview)](https://github.com/pkuschnirof/rpaforce-crm/blob/feat/nac-interop-mcp/yujin.app/nac-spec/docs/NAC_INTEROP_MCP.md)
- [Yujin Forge](https://github.com/pkuschnirof/yujin-forge)
- [Yujin framework](https://github.com/pkuschnirof/yujin)

## License

See `LICENSE`. Apache-2.0 preamble until commercial license
lands.
