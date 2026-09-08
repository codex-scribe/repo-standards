# Repo Standards

Conventions for team project repos

This file is the reference for making a repo conform. It is written to be read by a person
in a hurry or by an AI agent with no prior context. If you are an agent asked to "conform
this repo to standards," this document is the target state.

---

## 1. The four commands

Every repo answers the same four questions the same way. This is the whole point of the
standard — you should never open `package.json` to find out how to run something.

| Task                 | Dev                   | Prod                   |
| -------------------- | --------------------- | ---------------------- |
| Run the server       | `npm start`           | `npm run deploy`       |
| Migrate the database | `npm run migrate:dev` | `npm run migrate:prod` |

Rules:

- **`npm start` always means the dev server**, in APIs and UIs alike. It never means
  production. It works without `run`, which is why it gets the short name.
- **`npm run deploy` always means production**: build the code and restart the running
  process. It does not touch the database.
- **An env suffix (`:dev` / `:prod`) appears only where dev and prod genuinely differ.**
  Everything else is unsuffixed. If you find yourself wondering whether `generate:prod`
  exists, the standard has failed — it does not, because generate is identical in both.
- Migrations are a **separate, deliberate step** from deploying code. Schema first, then code.

## 2. Full script set

The scripts below are the **required floor**: every name in the tables that apply to a repo must
exist there, and must mean what the table says. `start:dev` exists as an alias of `start` so that
the dev server is greppable — searching a repo for `start:dev` finds it even though the command
you actually type is `npm start`.

**Additional script names are allowed.** This is a floor, not a ceiling. A repo may add:

- **Aliases for muscle memory.** `"dev": "npm start"` is welcome — plenty of people type
  `npm run dev` by reflex from other stacks, and there is no reason to retrain them. Any number of
  aliases beats an argument about which name is canonical.
- **Repo-specific scripts** for work the repo actually has — `screenshots`, `studio`, a one-off
  data fix. These need no blessing from this document.

Two constraints on the extras:

1. **An alias delegates; it does not duplicate.** Write `"dev": "npm start"`, never
   `"dev": "vite"`. A copied command line is a second definition, and it will drift from the first
   without anyone noticing.
2. **An extra name must never be the only way — or the documented way — to do a standard thing.**
   The docs in §9, the `.cmds` file in §8, and whatever you tell a teammate all name the standard
   command. An alias is a convenience at the keyboard, not a second interface: someone who knows
   only this document must be able to work in any repo.

### All repos

| Script         | Runs                        | Notes                            |
| -------------- | --------------------------- | -------------------------------- |
| `start`        | dev server, watch mode      | see §3 for the per-stack runner  |
| `start:dev`    | alias of `start`            | for greppability only            |
| `build`        | compile to `dist/`          | APIs run `prisma generate` first |
| `deploy`       | production: build + restart | see §6                           |
| `lint`         | `eslint .`                  |                                  |
| `lint:fix`     | `eslint . --fix`            |                                  |
| `prettier`     | `prettier --check .`        | see §7                           |
| `prettier:fix` | `prettier --write .`        | see §7                           |
| `typecheck`    | `tsc --noEmit`              | TS repos only                    |

### APIs additionally

| Script         | Runs                                       | Notes                                                     |
| -------------- | ------------------------------------------ | --------------------------------------------------------- |
| `migrate:dev`  | `prisma migrate dev`                       | generates the client automatically                        |
| `migrate:prod` | `prisma migrate deploy && prisma generate` | `migrate deploy` does **not** generate — see §5           |
| `seed`         | `prisma db seed`                           | plumbing differs per repo, see §5                         |
| `generate`     | `prisma generate`                          | needed after a schema pull, or after `npm ci` on Prisma 7 |
| `studio`       | `prisma studio`                            | local DB browser at `localhost:5555`                      |

### UIs additionally

| Script    | Runs           | Notes                                                         |
| --------- | -------------- | ------------------------------------------------------------- |
| `preview` | `vite preview` | **local check of the built bundle. Not a production server.** |

UIs have no production start command. A Vite build emits a static bundle; nginx serves it.
There is no long-running process to start, so `npm start` is dev-only there and that is correct.

## 3. Dev runner per stack

The command _name_ is uniform. The runner underneath is dictated by the framework, and that
is fine — you never type it.

| Stack                    | `start` runs                                          |
| ------------------------ | ----------------------------------------------------- |
| Express + TS             | `node --watch --require ts-node/register src/main.ts` |
| NestJS, no CLI installed | `node --watch --require ts-node/register src/main.ts` |
| NestJS + `@nestjs/cli`   | `nest start --watch`                                  |
| Vite                     | `vite`                                                |

Rules:

- **Do not use `nodemon`.** `node --watch` is built into Node and does the same job, so
  nodemon is a dependency earning nothing. (Still flagged experimental on Node 20; stable
  as of Node 22. It is reliable in practice.)
- **Do not use `tsx` for a NestJS app.** `tsx` is esbuild-based, and esbuild does not
  implement `emitDecoratorMetadata`, which Nest's dependency injection reads at runtime.
  Nest needs `ts-node` or the Nest CLI. `tsx` is fine for standalone scripts such as seeds.
- `cross-env NODE_ENV=development` is unnecessary on Linux/macOS but harmless; keep it
  where it already exists rather than churning.

## 4. Layout

| Convention   | Value                       |
| ------------ | --------------------------- |
| Entry point  | `src/main.ts`               |
| Build output | `dist/` — always gitignored |
| Built entry  | `dist/main.js`              |

`build/` as an output directory is non-conforming; it comes from the
`prisma-express-typescript-boilerplate` scaffold. Rename to `dist/`.

Uniform output paths are what let the pm2 configs in §6 be identical across repos.

## 5. Prisma

### The generate rules

`prisma generate` reads `schema.prisma`. **It never connects to the database.** Two
consequences that cause real outages:

1. **A successful build proves nothing about the database.** The generated client's types
   describe your schema file, so `tsc` compiles fine against a schema the production
   database has never seen. You find out at runtime, when a query hits a missing column.
   This is why `deploy` guards on `prisma migrate status` (§6) instead of trusting that a
   green build means a migrated database.
2. **`prisma migrate deploy` does not generate the client.** This has always been true, in
   every version. Only `prisma migrate dev` generates automatically.

### The Prisma 7 install change

`@prisma/client` v6 had a `postinstall` hook that auto-generated the client. **v7 removed
it.** So:

| Prisma version | Does `npm ci` generate the client?                |
| -------------- | ------------------------------------------------- |
| 6.x            | Yes, via `postinstall`                            |
| 7.x            | **No.** You must run `prisma generate` explicitly |

Because of this, `prisma generate` belongs in **both** `build` and `migrate:prod`. Both are
idempotent and cheap, and it makes it impossible for either path to leave a stale client.

### Seeding

The command is always `npm run seed` → `prisma db seed`. The plumbing differs by version and
that is acceptable, because the command you type does not:

- **Prisma 7**: declare the seed in `prisma.config.ts` under `migrations.seed`.
- **Prisma 6**: declare it in `package.json` under the `prisma.seed` key.

Do not invent seed data to make a repo have a seed. A repo with nothing to seed has no
`seed` script, and that is fine.

### Generated client location

Prefer generating into a gitignored directory rather than committing the client. Committed
clients go stale silently and produce confusing diffs.

## 6. Production and pm2

### Why a committed `ecosystem.config.js`

Restarting by name (`pm2 restart <repo-name>`) assumes a process that someone created by
hand, once. Its definition — script path, cwd, name, `NODE_ENV`, instance count, log paths,
owning user — lives only in pm2's own state file (`~/.pm2/dump.pm2`) for that user. It is not
in git, not reviewable, and not reproducible. If the box is rebuilt or someone runs
`pm2 delete`, the knowledge is gone and `pm2 restart` simply fails. Nobody can tell from the
repo what production is actually running.

A committed ecosystem file makes that definition code.

Use `pm2 startOrReload`, not `pm2 start`: it creates the process if missing and reloads it if
running. Plain `pm2 start` on an already-running app errors with "already launched."

```js
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: "<repo-name>", // MUST match the existing live process name
      script: "dist/main.js",
      instances: 1,
      autorestart: true,
      watch: false,
      time: true,
      env: { NODE_ENV: "production" },
    },
  ],
};
```

**The name must match the process already running on that server.** A mismatch does not
error — pm2 starts a _second_ process that fights for the port, which surfaces as
`EADDRINUSE` if you are lucky and as a silently duplicated app if you are not.

### The deploy script

**What is standardized is the command you type, not how it is implemented.** The point of the
standard is that `npm run deploy` deploys in every repo — never that every repo deploys the
same way. An inline one-liner in `package.json` is the default and is entirely fine:

```json
"deploy": "npm ci && npm run build"
```

Promote it to a committed `scripts/deploy.sh` only when the steps genuinely outgrow one line —
multiple guarded stages, comments that need to survive, or a non-obvious ownership dance. That
is why an API that guards on migration status and hands off between users may warrant a script
file, while most repos do not:

```sh
#!/usr/bin/env bash
set -euo pipefail

npm ci --include=dev
npx prisma migrate status      # aborts if migrations are pending
npm run build
pm2 startOrReload ecosystem.config.js --update-env
pm2 save
```

Order matters:

- **`npm ci` comes before `prisma migrate status`.** The Prisma CLI is a devDependency, so
  checking status first can fail because the CLI is missing rather than because migrations
  are pending.
- **`pm2 save` is not optional.** Without it, pm2's process list is not persisted and the
  process definition is lost on reboot.
- **`git pull` is not part of `deploy`.** Pulling stays a deliberate, separate step you run
  first, so a deploy never silently changes which commit is live.

`prisma migrate status` reads `prisma/migrations/*` and the `_prisma_migrations` table and
compares _migration history_ — not schema-versus-database structure (that is `migrate diff`).
Since Prisma 4.3.0 it exits 1 when there are unapplied migration files or a connection error,
which is what makes it usable as a guard. Run `migrate:prod` first, then `deploy`.

UIs: `npm ci && npm run build`. nginx serves `dist/` directly out of the repo checkout, so
there is nothing to copy — which is short enough that a UI's `deploy` should stay inline in
`package.json`.

### Changing the build output path is a production cutover

If conforming a repo changes its build output directory or entry filename, the live pm2
process still points at the **old** path. The old artifact stays on disk frozen at the last
build, and pm2 keeps serving it with no error anywhere. Perform the cutover explicitly:

```
pm2 delete <name>
git pull && npm ci --include=dev && npm run build
pm2 startOrReload ecosystem.config.js --update-env
pm2 save
rm -rf <old-output-dir>          # or it silently lingers
```

### Do not let tsc nest the output

With `outDir: ./dist` and no `rootDir`, tsc mirrors the common ancestor of all included
files. If anything outside `src/` is in the include set (a root-level `prisma.config.ts`, a
`test/` dir), the entry lands at `dist/src/main.js` rather than `dist/main.js`. Either pin
`rootDir: "./src"` and exclude root-level files from the build tsconfig, or accept the nested
path — but whichever you choose, `start:prod` and `ecosystem.config.js` must state the path
that actually exists. Verify with `ls dist`, not by assumption.

## 7. Prettier and lint

Prettier is being rolled out across all company repos. Also a version of ESLint must be implemented and configured as a pre-commit hook. It is fine if prettier and ESLint are configured together in one of those combined packages (like eslint-plugin-prettier) or as two independent packages.

## 8. `.cmds`

Every repo has a `.cmds` file at its root. A `cd` hook in `~/.zshrc` cats it on entry, so the
available commands announce themselves when you walk into the directory.

Keep it to the commands you actually reach for, plus the facts you always have to look up:

```
<repo-name> — NestJS + Prisma. Dev port 3000.

dev server     npm start
build          npm run build
migrate dev    npm run migrate:dev
migrate prod   npm run migrate:prod
deploy prod    npm run deploy

```

**`.cmds` is part of a tool used by one team member; a cd hook to display common commands when cding into a folder ** — it is listed in `~/.gitignore_global`, so it
exists only on the machine that created it. That is deliberate: it pairs with a personal shell
hook.

The consequence matters: a fresh clone, a teammate, or an agent working in CI gets **nothing**
from `.cmds`. It is a convenience layer, not documentation. The canonical, portable command
reference is the table in each repo's `CLAUDE.md` and `README.md` (§9) — `.cmds` duplicates a
subset of that for the person at the keyboard. Never let it be the only place a command is
written down.

## 9. Docs

Each repo's `CLAUDE.md` and `README.md` must state, correctly:

- The four commands from §1
- The port the dev server listens on
- Required environment variables, matching a real `.env.example`
- For APIs: the production host, the pm2 process name, and the deploy user

## 10. Ports

The dangerous port bug is not two dev servers colliding — that is loud and easy to fix. It is
**drift**, where a UI and its API quietly stop agreeing on a port number, or a repo's
`.env.example` disagrees with what production actually runs.

Two rules:

1. **Every Vite config sets an explicit `port` and `strictPort: true`.** Without `strictPort`,
   a taken port makes Vite slide silently to the next one, which then fails CORS against an
   allowlist that only names the original — a failure that looks like an auth or network bug
   and wastes an afternoon. Fail loudly instead.
2. **The port appears in exactly three places and they must agree:** the Vite config or API
   env default, the repo's `.env.example`, and the repo docs. If nginx proxies to it, that is
   a fourth.

## 11. What this standard does not attempt

- **Unifying frameworks.**
- **Unifying Prisma major versions.**
- **Adding CI.** These are small projects; deploys are manual by choice.
