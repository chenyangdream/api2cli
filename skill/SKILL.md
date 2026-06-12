---
name: api2cli
description: Generate a working CLI from any API, then wrap it in a Claude Code skill. Point it at API docs, a live URL, or a peek-api capture and get a dual-mode Commander.js CLI (human + agent output) plus a ready-to-use skill folder. Use when user wants to wrap an API in a CLI, generate a CLI from API docs, turn an API into a command-line tool, scaffold a CLI from discovered endpoints, or create a skill for an API.
---

# api2cli

Generate a working Node.js CLI from any API, then wrap it in a Claude Code skill. Discovers endpoints, scaffolds a dual-mode Commander.js CLI with a full-featured API client, and creates a skill folder so Claude knows how to use it.

## Workflow

1. **Identify the API** -- user provides a docs URL, a live API base URL, or a peek-api capture
2. **Discover endpoints** -- parse docs, probe the API, or read a peek-api catalog
3. **Build endpoint catalog** -- normalize all discovered endpoints into a standard format
4. **Generate CLI** -- scaffold Commander.js CLI from the catalog, including a `package.json` with build/install scripts and a `README.md` runbook
5. **User chooses destination** -- scaffold into current project or create standalone project
6. **Generate skill** -- create a SKILL.md that teaches Claude how to use the generated CLI

**The agent does not run `npm install`, `npm run build`, `npm link`, `bun compile`, or any install command during this workflow.** Build and install are documented in the generated `README.md` for the user to run themselves. Step 5's verification uses `npx tsx` against the source — no build step required.

## Step 1: Identify the API

Ask the user:
- "What API do you want to wrap? Share a docs URL, a base URL, or point me at a peek-api capture."

Determine which discovery paths to use based on what they provide:

| Input | Discovery Path |
|-------|---------------|
| Docs URL (e.g., `https://docs.stripe.com/api`) | Docs parsing + active probing |
| Base URL (e.g., `https://api.example.com/v1`) | Active probing |
| peek-api capture dir (e.g., `./peek-api-linkedin/`) | Read existing catalog |
| Live website URL | Suggest running peek-api first, then active probing |

Also ask:
- "What auth does this API use?" (API key, Bearer token, cookies, OAuth, none)
- "Do you want this CLI in your current project or as a standalone project?"

## Step 2: Discover Endpoints

Use all applicable discovery paths. Combine results into a single catalog.

### Path A: Docs Parsing

1. Fetch the docs URL with WebFetch
2. Extract endpoint information: method, path, description, parameters, request/response examples
3. Look for pagination patterns, auth requirements, rate limit info
4. Follow links to sub-pages for individual endpoint docs if the main page is an index

### Path B: Active Probing

1. Check well-known paths for API specs:
   - `/.well-known/openapi.json`, `/.well-known/openapi.yaml`
   - `/openapi.json`, `/openapi.yaml`, `/swagger.json`, `/swagger.yaml`
   - `/api-docs`, `/docs`, `/api/docs`
   - `/graphql` (with introspection query)
2. Try `OPTIONS` on the base URL and common resource paths
3. Probe common REST patterns: `/api/v1/`, `/api/v2/`, `/v1/`, `/v2/`
4. For each discovered resource, try standard CRUD: `GET /resources`, `GET /resources/:id`, `POST /resources`, etc.
5. Parse response shapes to understand data models
6. Check response headers for rate limit info (`X-RateLimit-*`, `Retry-After`)
7. Check for pagination patterns in responses (`next`, `cursor`, `page`, `offset`)

See `references/discovery-strategies.md` for detailed probing patterns.

### Path C: peek-api Capture

1. Read the capture directory: `endpoints.json`, `auth.json`, `CAPTURE.md`
2. Parse endpoints into the standard catalog format
3. Extract auth headers and cookies from `auth.json`

If peek-api is not installed or no capture exists, tell the user:
```
To capture endpoints from a live site, install peek-api:
  git clone https://github.com/alexknowshtml/peek-api
  cd peek-api && npm install
  node bin/cli.js https://example.com
```

## Step 3: Build Endpoint Catalog

Normalize all discovered endpoints into this format:

```typescript
interface EndpointCatalog {
  service: string;           // e.g., "stripe", "nexudus"
  baseUrl: string;
  auth: {
    type: 'api-key' | 'bearer' | 'cookies' | 'oauth' | 'none';
    headerName?: string;     // e.g., "Authorization", "X-API-Key"
    envVar: string;          // e.g., "STRIPE_API_KEY"
  };
  pagination?: {
    style: 'cursor' | 'offset' | 'page' | 'link-header';
    paramName: string;       // e.g., "starting_after", "offset", "page"
    responseField: string;   // e.g., "has_more", "next", "next_page_url"
  };
  rateLimit?: {
    requests: number;
    window: string;          // e.g., "1m", "1h"
  };
  resources: ResourceGroup[];
}

interface ResourceGroup {
  name: string;              // e.g., "customers", "invoices"
  description: string;
  endpoints: Endpoint[];
}

interface Endpoint {
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  path: string;              // e.g., "/v1/customers/:id"
  description: string;
  parameters: Parameter[];
  requestBody?: object;      // JSON schema or example
  responseExample?: object;
}

interface Parameter {
  name: string;
  in: 'path' | 'query' | 'header';
  required: boolean;
  type: string;
  description: string;
}
```

Present the catalog to the user for review before generating:
```
Found 24 endpoints across 5 resources:
  customers (6 endpoints): list, get, create, update, delete, search
  invoices (5 endpoints): list, get, create, send, void
  ...
Ready to generate the CLI?
```

## Step 4: Generate CLI

Generate a dual-mode CLI using Commander.js. The CLI auto-detects human vs agent output via `process.stdout.isTTY`.

### File Structure

**In-project scaffold (self-contained mini-package nested in host repo):**
```
scripts/{service}/
  package.json                    # mini-package — does NOT modify host's package.json
  tsconfig.json
  README.md                       # build & install runbook (user runs these themselves)
  .gitignore                      # ignores node_modules/ and dist/
  bin/
    {service}.ts                  # source entry, #!/usr/bin/env npx tsx
  src/
    lib/
      client.ts                   # API client (auth, pagination, retry, caching)
      envelope.ts                 # Agent JSON envelope helpers
    commands/
      {resource}.ts               # One file per resource group
  dist/                           # tsc output, gitignored
    bin/
      {service}.js                # post-build, #!/usr/bin/env node prepended
```

**Standalone project:**
```
{service}-cli/
  package.json                    # full package, bin field → dist/bin/{service}.js
  tsconfig.json
  README.md                       # build & install runbook
  .gitignore
  bin/
    {service}.ts                  # source entry, #!/usr/bin/env npx tsx
  src/
    lib/
      client.ts
      envelope.ts
    commands/
      {resource}.ts
  dist/                           # tsc output, gitignored
    bin/
      {service}.js                # post-build, #!/usr/bin/env node prepended
```

Both layouts use **the same `package.json` script set** — `start` / `build` / `postbuild` / `install:local` / `install:bin` / `install:global` / `compile:*`. The in-project layout differs only in where it lives; users still `cd scripts/{service}/` and run npm/bun commands the same way they would in standalone.

### Code Generation Patterns

See these references for the patterns to apply during generation:

- `references/api-client-template.md` -- API client class with pagination, retry, rate limiting, caching
- `references/agent-first-patterns.md` -- JSON envelope, HATEOAS next_actions, context-safe output, error fix suggestions
- `references/commander-patterns.md` -- Commander.js subcommands, global options, interactive prompts, colored output

### Key Generation Rules

**Entry point (`{service}.ts`):**
- Shebang: `#!/usr/bin/env npx tsx`
- Self-documenting root command (no args → prints full command tree as JSON)
- Global options: `--json` (force JSON output), `--verbose`, `--config <path>`

**API client (`lib/client.ts`):**
- Constructor takes base URL + auth config
- Auth from env var (name based on `catalog.auth.envVar`)
- Built-in pagination matching the API's pattern
- Retry with exponential backoff for 5xx and 429 errors
- Rate limiting based on discovered limits
- Optional response caching

**Envelope helpers (`lib/envelope.ts`):**
```typescript
const isAgent = !process.stdout.isTTY;

function respond(command: string, result: any, nextActions: Action[] = []) {
  if (isAgent) {
    console.log(JSON.stringify({ ok: true, command, result, next_actions: nextActions }));
  } else {
    return result; // caller handles human rendering
  }
}

function respondError(command: string, message: string, code: string, fix: string, nextActions: Action[] = []) {
  if (isAgent) {
    console.log(JSON.stringify({ ok: false, command, error: { message, code }, fix, next_actions: nextActions }));
  } else {
    console.error(`Error: ${message}`);
    console.error(`Fix: ${fix}`);
  }
  process.exit(1);
}
```

**Command files (`commands/{resource}.ts`):**
- One file per resource group
- Each endpoint becomes a subcommand: `mycli customers list`, `mycli customers get <id>`
- `list` commands: support `--limit`, `--offset`/`--cursor`, `--status` (if filterable)
- `get` commands: take ID as argument
- `create`/`update` commands: accept `--data <json>` or individual `--field` flags
- Every command includes contextual `next_actions` for agent mode
- Errors include `fix` suggestions

**Project files (both layouts):**

- `package.json` — `"type": "module"`, `bin` field points at `dist/bin/{service}.js`, scripts include `start` (tsx), `build` (tsc), `postbuild` (prepends `#!/usr/bin/env node` and `chmod +x`), `install:local` (`npm link`), `install:bin` (`bun build --compile` + `install -m 755 dist/{service} /usr/local/bin/{service}`), `install:global` (`npm install -g`), and `compile:*` cross-platform targets.
- `tsconfig.json` — ES2022, strict, `outDir: dist`, includes `bin/**/*.ts` and `src/**/*.ts`.
- `.gitignore` — `node_modules/`, `dist/`, `.env`.
- `README.md` — **generated runbook documenting how to install** (npm registry, npm link, bun-compiled binary). The agent does not run any of these commands; the user does, after the agent finishes.
- `.env.example` — the required env var name from `catalog.auth.envVar`.

**Verbatim templates for `package.json`, `tsconfig.json`, and `README.md` are in `references/build-and-deploy.md`.** Copy them and substitute `{service}`, `{Service}`, `{description}`, `{ENV_VAR}`, `{primary-resource}`, and (optionally) `{scope}` / `{registry}`.

**No host-project pollution (in-project layout):** the in-project scaffold lives entirely under `scripts/{service}/` as a self-contained mini-package with its own `package.json` and `node_modules`. The host repo's `package.json`, dependencies, and tsconfig are untouched.

## Step 5: Verify

After generating the CLI, verify the **source** runs — do not run the build, install, or any npm/bun command. Step 5 only exercises `tsx` against the source.

1. **Verify it runs:** `npx tsx bin/{service}.ts` with no args — confirm the self-documenting JSON tree prints.
2. **Test one endpoint:** Pick a simple GET endpoint and run it through `tsx` (e.g. `npx tsx bin/{service}.ts {resource} list`). Confirm the output shape matches the catalog.
3. Move on to Step 6.

The user runs `npm install`, `npm run build`, and any install command (`install:local` / `install:bin` / `install:global`) themselves, following the generated `README.md`.

## Step 6: Generate Skill

Create a Claude Code skill folder that teaches Claude how to use the generated CLI. This is the final step -- it turns the CLI into something any Claude session can pick up and use without reading the code.

### Skill Structure

```
.claude/skills/{service}/
  SKILL.md                    # Skill instructions
```

### SKILL.md Template

Generate a SKILL.md with this structure:

```markdown
---
name: {service}
description: Interact with the {Service} API via CLI. Use when user wants to
  {list of actions based on discovered resources, e.g., "list customers,
  create invoices, check order status"}. Commands: {service} {resource} {action}.
---

# {Service} CLI

CLI wrapper for the {Service} API.

## Setup

The CLI must be installed before use. See `{path/to/cli}/README.md` for the full install runbook — three options are documented:

1. **npm registry** — `npm install -g {scope}{service}` (end users; requires the package is published)
2. **`npm link`** — `cd {path/to/cli} && npm install && npm run build && npm run install:local` (developers iterating on the CLI)
3. **Standalone binary** — `cd {path/to/cli} && npm run install:bin` (compiles via Bun and installs to `/usr/local/bin/{service}`; no Node needed at runtime)

After install, set the `{ENV_VAR}` environment variable:

\`\`\`bash
export {ENV_VAR}=your-token-here
\`\`\`

Verify:

\`\`\`bash
{service}                      # prints self-documenting JSON command tree
\`\`\`

## Commands

{For each resource group, list commands with examples:}

### {Resource}

\`\`\`bash
# List {resources}
{service} {resource} list

# Get a specific {resource}
{service} {resource} get <id>

# Create a {resource}
{service} {resource} create --field value
\`\`\`

## Common Workflows

{Generate 2-3 practical workflows combining multiple commands:}

### Example: {Workflow name}
\`\`\`bash
# Step 1: Find the customer
{service} customers list --status=active

# Step 2: Get their invoices
{service} invoices list --customer-id=abc123
\`\`\`

## Agent Usage

When piped, all commands return JSON with `next_actions`:

\`\`\`bash
{service} {resource} list | cat
\`\`\`

## Development Mode

If the CLI is not installed yet (or you're iterating on its source), run via `tsx` instead:

\`\`\`bash
cd {path/to/cli}
npx tsx bin/{service}.ts {resource} list
\`\`\`

Output is identical to the installed binary in agent (piped) mode.

## Uninstall

See `{path/to/cli}/README.md` — the uninstall command depends on which install path was used (`npm uninstall -g`, `npm run uninstall:local`, or `npm run uninstall:bin`).
```

### Key Rules for Skill Generation

1. **Description is critical** -- include specific trigger phrases and list the actions the CLI supports. This is what Claude reads to decide when to use the skill.
2. **Include real command examples** -- use the actual CLI path and real subcommand names from the generated CLI.
3. **Generate practical workflows** -- combine multiple commands into realistic multi-step scenarios based on how the API's resources relate to each other.
4. **Keep it lean** -- the skill should be a quick reference, not a restatement of `--help`. Focus on what Claude needs to know that it can't infer.

### Tell the User

After generating the CLI, the README, and the skill, print this summary. **Do not run any of the commands** — list them so the user can run whichever install path fits.

```
CLI generated at:    {cli_path}
README at:           {cli_path}/README.md   ← build & install runbook
Skill generated at:  .claude/skills/{service}/SKILL.md

Try the source directly (no install required):
  cd {cli_path}
  npm install                            # one-time, installs commander, tsx, typescript
  npx tsx bin/{service}.ts               # see all commands
  npx tsx bin/{service}.ts {resource} list

To install as a system command, see {cli_path}/README.md. Three options:
  - npm registry:        npm install -g {scope}{service}
  - npm link (dev):      npm run install:local
  - Standalone binary:   npm run install:bin   (uses Bun, no Node needed at runtime)

Claude will automatically use this skill when you ask about {service}.
```

## Reference Files

- `references/discovery-strategies.md` -- Detailed probing patterns, well-known paths, GraphQL introspection, response parsing
- `references/api-client-template.md` -- Full API client class with pagination, retry, rate limiting, caching
- `references/agent-first-patterns.md` -- Agent JSON envelope, HATEOAS, context-safe output, error handling
- `references/commander-patterns.md` -- Commander.js subcommands, nested commands, interactive prompts, colored output, config files, testing
- `references/build-and-deploy.md` -- Verbatim `package.json` / `tsconfig.json` / `README.md` templates, shebang prepend mechanics, three install modes (npm registry, npm link, bun-compiled binary), cross-platform compile targets
