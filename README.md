# Fork Instructions:

## Installation:
Clone the repository, install dependencies, and build the CLI:

`git clone https://github.com/TheChronoTimer/onedrive-sync.git && cd onedrive-sync && npm ci && npm run build`

## Configuration:
Run the one-time authentication/setup commands:

`node dist/cli.js --host YOUR_TENANT.sharepoint.com login`

Then verify the session:

`node dist/cli.js --host YOUR_TENANT.sharepoint.com auth-check`

If the session expires, renew it with:

`node dist/cli.js --host YOUR_TENANT.sharepoint.com auth-renew`

## Storage:
Edit the values on the file `./onedrive-sync`.

The SharePoint address:

`HOST="limerickandclareetb-my.sharepoint.com"`

The user's remote path (**DO NOT** remove `/Documents` unless you know what you're doing):

`REMOTE="/personal/09_your_name_learner_lcetb_ie/Documents"`

The user's local path (where your files will be synchronized):

`LOCAL="/home/user/09.name.name@learner.lcetb.ie"`

## Run:
To synchronize your files, run:

`./onedrive-sync`

**IMPORTANT:** The order of the operations is highly relevant for synchronization.
### IS STRONGLY RECOMENDED A BACKUP BEFORE SYNCHRONIZING THE FIRST TIME
> **Note:** The script `./onedrive-sync` script can be moved or copied freely. It can therefore be executed from another location, as long as its configuration contains the correct paths.

## Uninstall:

Remove the project directory and the sync script if they are no longer needed.

Then remove the stored authentication and synchronization state:

`rm -rf ~/.sharepoint-cli`

---
---
---

# sharepoint-access (`sharepoint-cli`)

TypeScript CLI for SharePoint Online, driven by a browser-captured session.
Third member of the set alongside [`outlook-access`](https://github.com/weirdapps/outlook-access)
(`outlook-cli`) and [`teams-access`](https://github.com/weirdapps/teams-access)
(`teams-cli`), following the same one-surface-per-repo pattern: its own CLI, its
own `~/.sharepoint-cli/` directory, its own Playwright profile, its own login.

> **Status: in production since 2026-08-08.** All ten commands are implemented,
> 255 tests pass, and the read and write surfaces are verified against a live
> tenant on both the team-sites and OneDrive-for-Business hosts, from the Mac
> and from the VPS. See [`docs/design/project-design.md`](docs/design/project-design.md).

## Why a browser-captured session

The target tenant blocks OAuth device-code and third-party app flows, and issues
**no SharePoint Bearer token at all**. Authentication is cookie-based
(`FedAuth` + `rtFa`), captured by driving a real Chrome window through Playwright
and reading the session out of the browser context.

That constraint propagates:

- **Microsoft Graph is unusable here.** `graph.microsoft.com` needs a
  Graph-audience Bearer, and cookies do not authorise it. The tenant also returns
  403 on the SharePoint-hosted `/_api/v2.0` mirror of the same object model.
- **Classic SPO REST is the only viable surface** (`/_api/web`, `/_api/search`),
  and it works.
- **Writes need CSRF handling.** Cookie-authenticated POSTs are rejected without
  an `X-RequestDigest` from `/_api/contextinfo`. Bearer callers are exempt, which
  is why the requirement is easy to miss in Graph-oriented documentation.

All of the above was measured against the live tenant, not assumed. The evidence
table is in the design doc.

## Commands

| Command        | Purpose                                                        |
| -------------- | -------------------------------------------------------------- |
| `login`        | Interactive sign-in, capture and persist the session           |
| `auth-renew`   | Headless silent re-capture via `ESTSAUTHPERSISTENT` (~90 days) |
| `auth-check`   | Probe the read, write and search surfaces                      |
| `health-check` | Same probes with timings, for cron                             |
| `ls`           | List a folder                                                  |
| `get`          | Download a file by server-relative path or absolute URL        |
| `search`       | Search files and sites                                         |
| `libraries`    | List document libraries in a site                              |
| `mkdir`        | Create one folder (parents must exist)                         |
| `put`          | Upload a file, auto-chunked above 10 MB                        |

```bash
CLI="node ~/SourceCode/sharepoint-access/dist/cli.js"

# Team sites
$CLI --host contoso.sharepoint.com ls "/sites/finance/Shared Documents"

# OneDrive for Business needs the -my host. One session covers both.
$CLI --host contoso-my.sharepoint.com ls "/personal/jane_doe_contoso_com/Documents"

$CLI --host contoso.sharepoint.com get "/sites/finance/Shared Documents/q3.xlsx" --out ./q3.xlsx
$CLI --host contoso.sharepoint.com put ./report.docx "/sites/finance/Shared Documents"
$CLI --host contoso.sharepoint.com search "contentclass:STS_Site" --rows 5
```

### Three things that will bite you

**`--host` selects the wrong content silently if you get it wrong.** Anything
under `/personal/...` lives on `<tenant>-my.sharepoint.com`.

**Paths are server-relative**, not URLs. Sub-site scoping is derived from the
path automatically; only a web nested deeper than two segments needs `--site`.

**An unexpected empty listing used to mean "wrong web".** SharePoint answers
`200` with empty collections for a path the queried web does not own, and even
for a path that does not exist. The CLI now requests `Exists` and raises
`not_found`, but stay suspicious of a surprising empty result.

## Build / Run

```bash
npm ci
npm run build              # tsc + chmod dist/cli.js
npx tsc --noEmit           # type-check only
node dist/cli.js --help    # run compiled CLI
npm run cli                # dev run via ts-node (no compile)
```

The binary is **not** on PATH. Invoke as
`node ~/SourceCode/sharepoint-access/dist/cli.js`.

## Test / Lint

```bash
npm test                   # vitest run
npm run test:coverage
npm run lint
npm run format
./scripts/pii-gauntlet.sh  # tracked-file PII scan; pre-commit hook AND required CI job
```

Test files use the `.spec.ts` suffix, matching `outlook-access`. Note
`teams-access` uses `.test.ts`; the two existing repos are already inconsistent
and this one enforces its choice in `vitest.config.ts`.

## Runtime state

`~/.sharepoint-cli/` holds `playwright-profile/`, `session.json`, and
`.browser.lock`. Nothing here is committed, and the session file holds live
credentials.

## Licence

MIT
