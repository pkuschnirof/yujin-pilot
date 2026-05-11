# CLAUDE.md

Context for AI-assisted contributions to `pkuschnirof/yujin-pilot`.

## What this repo is

Yujin Pilot -- the commercial NAC controller. One voice + chat
panel that drives every NAC-3 conformant app the user has open.
Built on NAC v2.3's interop primitive (from
`pkuschnirof/rpaforce-crm`).

**Day-0 status (2026-05-10):** planning + branding anchor only.
No production code. Implementation kickoff after NAC v2.3 GA.

## Strategic placement

```
NAC (Apache-2.0)              -- rpaforce-crm + yujin.app/nac-spec
       |
       v
Yujin Forge (closed core)     -- yujin-forge      (build NAC apps)
       |  sibling product
       v
Yujin Pilot (closed core)     -- yujin-pilot      (drive NAC apps)
       (this repo)
```

## Important: chat consolidation

As of 2026-05-11, every Yujin product going forward DOES NOT
ship its own chat panel. Internal chat surfaces are removed +
replaced by an affordance pointing the user to Yujin Pilot.

If you are working on a Yujin product and the spec says it
should have a chat panel, double-check the date: post-2026-05-11
the answer is "no internal chat -- adopt Pilot."

## Style + conventions

- ASCII-only markdown.
- Commit messages imperative, under 70 chars; body explains why.
  Co-author trailer welcome.
- License is in flux. Until commercial license lands, Apache-2.0.

## What you can rely on

- Same PAT as `pkuschnirof/rpaforce-crm`.
- NAC v2.3 interop preview lives at
  `feat/nac-interop-mcp` of rpaforce-crm. Pilot's MCP scanner +
  dispatcher are downstream consumers of that primitive.

## What you should NOT do without owner approval

- Do not write production code in this repo during day 0.
  Implementation kickoff gated on NAC v2.3 GA.
- Do not register a `@yujin/pilot` npm scope from this repo
  until v1.0 launch.
- Do not change LICENSE -- commercial license needs Pablo's
  sign-off.

## Useful starting points

- `README.md` -- pitch + ecosystem placement.
- `docs/SPEC.md` -- the full product spec.
- NAC interop primitive:
  https://github.com/pkuschnirof/rpaforce-crm/blob/feat/nac-interop-mcp/yujin.app/nac-spec/docs/NAC_INTEROP_MCP.md
- Yujin Forge sibling:
  https://github.com/pkuschnirof/yujin-forge
