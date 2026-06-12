# Build & Deploy

This is the **source of truth** for the build/install layer the api2cli skill scaffolds. It defines:

- The exact `package.json`, `tsconfig.json`, `.gitignore`, and `README.md` content the agent emits during Step 4.
- The runbook end users execute themselves (via `npm` / `bun`) — **the agent never runs these commands during the skill workflow**.

The pattern mirrors `gongfengcli` (`/Users/chenyang/source/gongfengcli`): TypeScript source under `bin/` and `src/`, `tsc` build into `dist/`, a `postbuild` hook that prepends a Node shebang and `chmod +x`, plus optional `bun build --compile` for self-contained native binaries.

## Shebang Mechanics

Two shebangs, one source file path, one build-output path:

| File | Shebang | When |
|------|---------|------|
| `bin/{service}.ts` (source) | `#!/usr/bin/env npx tsx` | Dev mode — `npm start` / `tsx bin/{service}.ts` |
| `dist/bin/{service}.js` (post-tsc) | `#!/usr/bin/env node` | Injected by `postbuild` script after `tsc` runs |

`tsc` does not emit shebangs. The `postbuild` hook prepends `#!/usr/bin/env node` to `dist/bin/{service}.js` and `chmod +x`. The source shebang is only there to make `bin/{service}.ts` directly runnable during development.

The `bun build --compile` path produces a fully self-contained executable at `dist/{service}` (or `dist/{service}-{platform}-{arch}`) — no shebang needed (it's a real ELF/Mach-O/PE binary).

## Three Install Modes

Three completely separate install paths. The generated README documents all three; users pick the one that fits.

### Mode 1: `install:local` — npm link (developers)

Symlinks the package into npm's global bin, so `{service}` resolves to the local working tree's `dist/bin/{service}.js`. Useful for iterating on the CLI itself.

```bash
npm install
npm run build
npm run install:local        # npm link
npm run uninstall:local      # npm unlink -g {service}
```

After source changes, re-run `npm run build` — the link picks up the new `dist/`.

### Mode 2: `install:bin` — bun compile + /usr/local/bin (system-wide standalone)

Compiles to a self-contained native binary via `bun build --compile`, then `install -m 755` to `/usr/local/bin`. Target machines need **nothing** at runtime — no Node, no Bun. Requires Bun on the build machine only.

```bash
curl -fsSL https://bun.sh/install | bash    # one-time
npm run install:bin                         # compile + install
npm run uninstall:bin                       # rm /usr/local/bin/{service}
```

`install:bin` may need `sudo` depending on `/usr/local/bin` permissions. The generated README mentions this.

### Mode 3: `install:global` — npm registry (end users)

Once published, end users install via npm:

```bash
npm install -g @scope/{service}
```

The package's `bin` field maps `{service}` to `dist/bin/{service}.js`, so npm wires it onto `PATH` automatically. Requires `node ≥ 18` at runtime.

For internal Tencent registries (or any custom registry), include the `--registry` flag — see the gongfengcli pattern that uses `https://mirrors.tencent.com/npm/`.

## Cross-Platform Compile Targets (Bun)

`bun build --compile` accepts `--target=` to cross-compile. Standard targets:

| Script | Target | Output |
|--------|--------|--------|
| `compile` | host platform | `dist/{service}` |
| `compile:macos-arm64` | `bun-darwin-arm64` | `dist/{service}-macos-arm64` |
| `compile:macos-x64` | `bun-darwin-x64` | `dist/{service}-macos-x64` |
| `compile:linux-x64` | `bun-linux-x64` | `dist/{service}-linux-x64` |
| `compile:linux-arm64` | `bun-linux-arm64` | `dist/{service}-linux-arm64` |
| `compile:win-x64` | `bun-windows-x64` | `dist/{service}-win-x64.exe` |
| `compile:all` | all five | all of the above |

Only emit the targets the user cares about. For Tencent-internal tools it's typically `macos-arm64`, `macos-x64`, and `linux-x64`.

## Files To Generate

The agent writes these files during Step 4. **The agent does not execute any build/install command.**

### `package.json` (template)

Substitute `{service}`, `{description}`, `{scope}` (optional, e.g. `@tencent/`), and `{registry}` (optional).

```json
{
  "name": "{scope}{service}",
  "version": "0.1.0",
  "description": "{description}",
  "type": "module",
  "bin": {
    "{service}": "./dist/bin/{service}.js"
  },
  "files": [
    "dist/"
  ],
  "scripts": {
    "start": "tsx bin/{service}.ts",
    "build": "tsc",
    "postbuild": "printf '#!/usr/bin/env node\\n' | cat - dist/bin/{service}.js > /tmp/_{service}_shebang && mv /tmp/_{service}_shebang dist/bin/{service}.js && chmod +x dist/bin/{service}.js",
    "prepack": "npm run build",
    "lint": "tsc --noEmit",
    "compile": "bun build ./bin/{service}.ts --compile --outfile dist/{service}",
    "compile:macos-arm64": "bun build ./bin/{service}.ts --compile --target=bun-darwin-arm64 --outfile dist/{service}-macos-arm64",
    "compile:macos-x64": "bun build ./bin/{service}.ts --compile --target=bun-darwin-x64 --outfile dist/{service}-macos-x64",
    "compile:linux-x64": "bun build ./bin/{service}.ts --compile --target=bun-linux-x64 --outfile dist/{service}-linux-x64",
    "compile:linux-arm64": "bun build ./bin/{service}.ts --compile --target=bun-linux-arm64 --outfile dist/{service}-linux-arm64",
    "compile:win-x64": "bun build ./bin/{service}.ts --compile --target=bun-windows-x64 --outfile dist/{service}-win-x64.exe",
    "compile:all": "npm run compile:macos-arm64 && npm run compile:macos-x64 && npm run compile:linux-x64 && npm run compile:linux-arm64 && npm run compile:win-x64",
    "install:local": "npm link",
    "uninstall:local": "npm unlink -g {service}",
    "install:bin": "npm run compile && install -m 755 dist/{service} /usr/local/bin/{service}",
    "uninstall:bin": "rm -f /usr/local/bin/{service}",
    "install:global": "npm install -g {scope}{service}{registry-flag}"
  },
  "engines": {
    "node": ">=18"
  },
  "dependencies": {
    "commander": "^12.1.0"
  },
  "devDependencies": {
    "@types/node": "^22.10.0",
    "tsx": "^4.19.2",
    "typescript": "^5.7.2"
  },
  "license": "MIT"
}
```

If a custom registry is required, also add (mirroring gongfengcli):

```json
"publishConfig": { "registry": "{registry-url}" },
```

…and set `{registry-flag}` to ` --registry {registry-url}` in `install:global`.

### `tsconfig.json` (template, copy verbatim)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "outDir": "dist",
    "declaration": false,
    "sourceMap": true,
    "lib": ["ES2022"]
  },
  "include": ["bin/**/*.ts", "src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### `.gitignore`

```
node_modules/
dist/
.env
```

### `README.md` (template — this is what carries the build/install runbook)

Substitute `{service}`, `{Service}`, `{description}`, `{ENV_VAR}` from the catalog, and `{primary-resource}` examples.

```markdown
# {service}

A dual-mode (human + agent) CLI for the {Service} API.

## Install

### For users

Requires Node.js ≥ 18.

```bash
npm install -g {scope}{service}{registry-flag-or-empty}
```

If `{service}` is not found after install, add npm's global bin to PATH:

```bash
export PATH="$(npm root -g)/../bin:$PATH"
```

### For developers

**Run from source** (no build step required):

```bash
git clone {repo-url}
cd {service}
npm install
npm start -- --help                    # alias for `tsx bin/{service}.ts`
```

**Local global install via `npm link`** (code changes take effect after each `npm run build`):

```bash
npm install
npm run build
npm run install:local       # npm link
npm run uninstall:local     # remove the link
```

**Standalone binary** (no Node required at runtime — uses [Bun](https://bun.sh)):

```bash
curl -fsSL https://bun.sh/install | bash       # one-time

npm run compile                # current platform → dist/{service}
npm run compile:macos-arm64
npm run compile:macos-x64
npm run compile:linux-x64
npm run compile:linux-arm64
npm run compile:win-x64
npm run compile:all            # build all targets

npm run install:bin            # compile + install to /usr/local/bin/{service}
npm run uninstall:bin
```

`install:bin` may require `sudo` depending on `/usr/local/bin` permissions.

## Auth

Set the `{ENV_VAR}` environment variable:

```bash
export {ENV_VAR}=your-token-here
```

## Usage

```bash
# See all commands
{service}

# Example commands
{service} {primary-resource} list
{service} {primary-resource} get <id>
```

When piped, output flips to a JSON envelope automatically:

```bash
{service} {primary-resource} list | jq
```

## Development

```bash
npm run lint            # tsc --noEmit (type-check without building)
npm start               # alias for `tsx bin/{service}.ts`
```

## License

MIT
```

## In-Project Layout Specifics

When the user picks the in-project layout, scaffold under `scripts/{service}/` as a **self-contained mini-package** — same shape as standalone, just nested inside another repo:

```
scripts/{service}/
  package.json           # mini-package, NOT modifying host's package.json
  tsconfig.json
  README.md
  .gitignore             # ignores node_modules/ and dist/
  bin/{service}.ts
  src/lib/client.ts
  src/lib/envelope.ts
  src/commands/{resource}.ts
```

The host project's `package.json`, `node_modules`, and TypeScript config are untouched. Users `cd scripts/{service}/` then run `npm install`, `npm run build`, `npm run install:local`, etc. — exactly the same commands as standalone.

## What The Agent Does Not Do

The agent's job ends at file emission. The agent must **never**:

- Run `npm install`
- Run `npm run build`
- Run `npm link` / `install:local`
- Run `bun build` / `compile` / `install:bin`
- Run `npm install -g` / `install:global`
- `cp` anything into `/usr/local/bin`

These all require user judgment (sudo, registry choice, target platform, when to rebuild) and may have side effects on the user's environment. Step 5's `tsx`-mode verification is the only execution the agent performs against the generated CLI; everything else lives in the README runbook for the user to run themselves.
