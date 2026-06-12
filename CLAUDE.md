# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

This repo **is itself a Claude Code skill** — not an application, library, or CLI. There is no build, no tests, no lint, no `package.json`. The "product" is the markdown content under `skill/` that gets copied into other projects as `.claude/skills/api2cli/`. When Claude in another project needs to wrap an API in a CLI, it loads `skill/SKILL.md` and follows the workflow encoded there.

Treat edits here as **prompt engineering**, not code changes. The audience for the files is a future Claude session, not a runtime.

## Layout

```
skill/
  SKILL.md                          # Main skill — loaded eagerly when triggered
  references/
    discovery-strategies.md         # Loaded on demand during Step 2 (endpoint discovery)
    api-client-template.md          # Loaded on demand during Step 4 (CLI generation)
    agent-first-patterns.md         # Loaded on demand during Step 4 (output design)
    commander-patterns.md           # Loaded on demand during Step 4 (CLI structure)
    build-and-deploy.md             # Loaded on demand during Step 4 (package.json / tsconfig / README templates, install modes)
```

The split is deliberate: `SKILL.md` stays small enough to keep in context every time the skill activates; the `references/` files are only pulled in when a specific phase needs them. **Don't inline reference content into SKILL.md** — it bloats every activation. Conversely, don't push critical workflow steps out of SKILL.md into references — they won't be read until too late.

## How The Skill Works (the 6-step workflow encoded in SKILL.md)

1. **Identify the API** — ask the user for a docs URL, base URL, or peek-api capture dir.
2. **Discover endpoints** — combine docs parsing (WebFetch), active probing (well-known spec paths, CRUD probes, GraphQL introspection), and/or peek-api catalog reads. See `references/discovery-strategies.md`.
3. **Build endpoint catalog** — normalize discoveries into the `EndpointCatalog` TypeScript shape defined in SKILL.md, then show counts to the user for review.
4. **Generate CLI** — scaffold a Commander.js + `tsx` CLI with dual-mode output. Two layouts: in-project (under `scripts/{service}/`) or standalone (`{service}-cli/`). Both layouts emit a full mini-package: `package.json` (with `tsc` build, `postbuild` shebang prepend, three install modes, and Bun cross-compile targets), `tsconfig.json`, `.gitignore`, and a `README.md` runbook documenting the build/install steps for the user. Verbatim templates live in `references/build-and-deploy.md`.
5. **Verify** — run `npx tsx bin/{service}.ts` with no args (must print self-documenting JSON tree) and exercise one simple GET. Source-only — no `npm install`, no build.
6. **Generate skill** — write `.claude/skills/{service}/SKILL.md` so future Claude sessions can use the generated CLI without reading its source. The skill points users at the generated `README.md` for install instructions.

When editing `SKILL.md`, preserve this 6-step spine. Steps map 1:1 to the user-facing experience; renumbering or merging steps will silently change how Claude paces the conversation. Do not add a build/install step — the agent's job ends at file emission (see the "agent does not execute build/install" non-negotiable below).

## Non-negotiable Patterns The Skill Teaches

These show up across multiple files — change them in one place, change them everywhere:

- **Dual-mode output via `process.stdout.isTTY`** — human-readable when attached to a terminal, JSON envelope (`{ ok, command, result, next_actions }`) when piped. Defined in SKILL.md Step 4 and `references/agent-first-patterns.md`.
- **HATEOAS `next_actions`** — every response (success or error) lists contextual follow-up commands. The agent never needs `--help`.
- **Self-documenting root** — running the entry point with no args prints the full command tree as JSON. This is the agent's discovery mechanism.
- **Errors include a `fix` field** — plain-language guidance for resolving the failure, not just an error code.
- **Env-var auth, name derived from `catalog.auth.envVar`** — never hardcode keys; never invent a different naming convention.
- **Source-entry shebang is `#!/usr/bin/env npx tsx`** — the TypeScript entry at `bin/{service}.ts` runs via `tsx` for dev iteration. The `dist/bin/{service}.js` artifact gets `#!/usr/bin/env node` injected by the `postbuild` hook in `package.json` (which prepends the line after `tsc` runs and `chmod +x`s the file). Two files, two shebangs, both load-bearing. Don't switch the source entry to `ts-node`, don't strip either shebang, and don't reshape the postbuild hook without updating `references/build-and-deploy.md` and the generated `package.json` template.
- **Agent does not execute build, install, or any package-manager commands** — the skill's job is file emission only. The agent never runs `npm install`, `npm run build`, `npm link`, `bun build`, `install -m 755`, or any `install:*` script during the workflow. Step 5's verification uses `npx tsx` against the source — that's the only execution. Build and install steps live in the **generated `README.md`** for the user to run themselves; this keeps the agent out of decisions about sudo, registry choice, target platform, and rebuild timing. If you find yourself adding a step that calls a build or install command, stop — that belongs in the generated README, not the skill workflow.

If you find yourself softening any of these in an edit, that's a signal to stop and reconsider — they're load-bearing for the agent UX the skill is selling.

## Discovery Strategy Priority

When multiple discovery paths apply, the skill expects them combined, not chosen between:

1. Well-known OpenAPI/Swagger paths first (`/openapi.json`, `/swagger.json`, etc.) — if a spec exists, it's authoritative.
2. Docs parsing via WebFetch for human-readable docs pages.
3. Active probing (CRUD on common resource names, OPTIONS for allowed methods, response-header inspection for rate limits and pagination style).
4. peek-api capture if the user has one — fastest path for sites that don't expose docs.

`references/discovery-strategies.md` enumerates the well-known paths, common resource names, and pagination/rate-limit/auth signals to look for. Keep that list curated — it's the skill's "API surface area" knowledge.

## Editing The Skill

- **Frontmatter `description` is the trigger** — Claude reads it to decide whether to activate. The phrase list ("wrap an API in a CLI", "generate a CLI from API docs", etc.) is what makes the skill discoverable. Edit it carefully; removing a phrase silently hides the skill from queries that used it.
- **Generated SKILL.md template** — Step 6 of SKILL.md contains a template for the per-service skill the workflow emits. The template's frontmatter `description` field follows the same trigger-phrase rules.
- **Reference files are loaded by name from SKILL.md** — if you rename or move a reference, update every cross-reference in `SKILL.md` and the README.

## Installing / Using

To install into another project (copying the skill is the only "deploy"):

```bash
cp -r skill/ /path/to/your/project/.claude/skills/api2cli/
```

There is nothing to run, test, or lint in this repo. Verification of changes happens by activating the skill in another project and walking through the workflow end-to-end against a real API.
