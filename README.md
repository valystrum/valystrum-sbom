# Valystrum SBOM Action

Generates a **CycloneDX** SBOM with the official CycloneDX tooling and ingests it into your
Valystrum software inventory, where components are matched against CVE feeds and surfaced
under **SBOM & Licences**. This action is licensed under the Apache License 2.0
([LICENSE](LICENSE)); the Valystrum platform it calls is not.

Same shape as [`valystrum/valystrum-pipeline`](https://github.com/valystrum/valystrum-pipeline):
one step, one repository secret, HMAC-signed.

Not on GitHub? The same SBOM upload, with the same generators, pins and request, comes for GitLab
CI/CD ([`gitlab.com/valystrum/valystrum-gitlab-ci`](https://gitlab.com/valystrum/valystrum-gitlab-ci)),
Azure Pipelines ([`github.com/valystrum/valystrum-azure-pipelines`](https://github.com/valystrum/valystrum-azure-pipelines)),
AWS CodeBuild and CodePipeline ([`github.com/valystrum/valystrum-aws-codebuild`](https://github.com/valystrum/valystrum-aws-codebuild))
and Google Cloud Build ([`github.com/valystrum/valystrum-cloud-build`](https://github.com/valystrum/valystrum-cloud-build)).

SBOM ingest needs the Gold plan or higher. Below it, issuing a key and every signed upload are
refused with HTTP 403 `FeatureLocked`.


```yaml
- uses: valystrum/valystrum-sbom@v1
  with:
    ingest-url: https://app.valystrum.io/api/v1/<org>/sbom/ingest
    ingest-secret: ${{ secrets.CS_WEBHOOK_SECRET }}
```

## Setup

**1. Get the CI secret** (once per repository; if you already run the pipeline action for
this repo, you have it and can skip to step 2). An organisation admin issues it from
**Settings → CI pipeline keys** in the dashboard, entering the repository as `owner/repo`, or
with the same call over the API from a signed-in admin session (an API key is read-only and
cannot issue CI keys):

```bash
curl -X POST https://app.valystrum.io/api/v1/<org>/pipeline/config \
  -b "<your signed-in session cookie>" \
  -H "content-type: application/json" -d '{"repo": "acme/web-api"}'
```

Issue it for the repository name exactly as the action sends it as `source`: by default
`$GITHUB_REPOSITORY` (`owner/repo`). The ingest picks the key by that label, matched exactly,
case included. Each new repository key takes one of the plan's repository slots; rotating an
existing key does not. Gold includes 100 repositories and Enterprise is custom. From Gold, 25 more
repositories are an add-on at €100 a month or €1,000 a year. Issuing a key for a new repository
over the allowance is refused with HTTP 409 `QuotaExceeded` (`limitKey` `repositories`): revoke a
key you no longer use, or add the pack.

The response contains `webhookSecret` (`cs_wh_…`), scoped to that one repository. It is shown
**once**. Re-running the call for the same `repo` does not rotate it unless you pass
`{"repo": "acme/web-api", "regenerateSecret": true}`, which would 401 that repo's pipeline until
it's updated with the new secret. Other repos' keys are unaffected either way. The exception is a
revoked key: issuing again for that repository, in the dashboard or with the call above, brings
its key back with a new secret (shown once), without `regenerateSecret`. The old secret stays
refused, so put the new one in `CS_WEBHOOK_SECRET`.

Each repo's key can be revoked independently (`DELETE /api/v1/<org>/pipeline/config/<keyId>`)
without touching any other repo, and `GET /api/v1/<org>/pipeline/config` lists every active key
for the org. Revoking a key returns its repository slot. **Settings → CI pipeline keys** lists,
issues and revokes them too. Issuing again there for a repository that already has a key rotates
its secret, unlike the API call above.

If the repository is renamed or moved to another owner, `$GITHUB_REPOSITORY` changes and the old
key no longer matches. Rebind the key instead of issuing a new one:
`PATCH /api/v1/<org>/pipeline/config/<keyId>` with `{"repo": "owner/new-name"}`, from a signed-in
admin session. The secret stays the same, so `CS_WEBHOOK_SECRET` needs no change. The dashboard
cannot rebind a key. If the new name ever had a key of its own, revoked or not, the rebind fails:
issue a key for the new name in step 1 instead and put its secret in `CS_WEBHOOK_SECRET`. Later
uploads go under the new name; the components uploaded under the old `source` stay under that
label (see [Removing what is gone](#removing-what-is-gone)).

Always pass `repo`: a call without it is refused with HTTP 400. Every key signs for one
repository, and an upload is checked against its own repository's key alone. An older key issued
without a repository shows under **Settings → CI pipeline keys** as "organisation-wide (retired: no
scan accepts it, revoke it)": issue a key for each repository that used it (step 1, one repository
of the allowance each), put its secret in that repository's `CS_WEBHOOK_SECRET` (the pipeline
action signs with the same secret), then revoke the organisation-wide key.

**2. Add it as a repository secret** (`Settings → Secrets and variables → Actions`),
named `CS_WEBHOOK_SECRET`. Store it per repository, not as a GitHub organisation secret shared by
several repositories. Each repository signs with its own key, so another repository's secret is
refused with HTTP 401 `Invalid signature`.

**3. Copy [`examples/sbom.yml`](examples/sbom.yml)** to `.github/workflows/sbom.yml`, replace
`acme` in the `ingest-url` with your org slug, and, if your default branch is not `main`, replace
`main` with its name in both places: the `push` trigger and the `complete` condition. The example
installs npm dependencies, which the npm generator needs. For Python, Go or Maven, replace the
Node steps with your project's own setup (see [What the runner needs](#what-the-runner-needs)).

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `ingest-url` | yes | none | `https://app.valystrum.io/api/v1/<org>/sbom/ingest` |
| `ingest-secret` | yes | none | CI secret (`cs_wh_…`) from step 1 |
| `path` | no | `.` | Directory to scan for manifests |
| `ecosystem` | no | `auto` | `auto`, or a space-separated subset of `npm python go maven` |
| `source` | no | `owner/repo` (`GITHUB_REPOSITORY`) | Label the components are grouped under in the inventory. It also selects the CI key: it must equal the `repo` the key was issued for, and no other key is tried |
| `asset` | no | none | Valystrum Asset name/hostname these components belong to (see [Linking to an asset](#linking-to-an-asset)) |
| `complete` | no | `false` | `true` when this step's SBOM is the whole of the source (see [Removing what is gone](#removing-what-is-gone)). Any value other than `true` counts as `false` |

## Outputs

| Output | Description |
|---|---|
| `sbom-path` | Path to the CycloneDX JSON document that was uploaded |
| `components` | Number of components Valystrum ingested |
| `removed` | Number of the source's components Valystrum removed because the SBOM no longer lists them (`0` unless `complete` is `true`). It counts component versions, not packages: upgrading `lodash@4.17.20` to `lodash@4.17.21` removes the old version (1 in `removed`) and ingests the new one (counted in `components`) |

The step also writes the counts to the run's job summary: "Ingested N components as source
`owner/repo`; removed M the SBOM no longer lists".

## Ecosystems

Detection is by manifest in `path` itself (not its subdirectories), and every generator whose
manifest is present runs. For Python, the first of `poetry.lock`, `Pipfile.lock` and
`requirements.txt` found picks the one mode that runs.

| Manifest | Generator (pinned version, see [Pinning](#pinning)) |
|---|---|
| `package-lock.json` | `@cyclonedx/cyclonedx-npm` 6.0.1 |
| `requirements.txt` / `poetry.lock` / `Pipfile.lock` | `cyclonedx-bom` (`cyclonedx-py`) 7.6.0 |
| `go.mod` | `cyclonedx-gomod` 1.12.0 |
| `pom.xml` | `cyclonedx-maven-plugin` 2.9.3 |

Not yet covered: Yarn and pnpm (npm detection needs `package-lock.json`, so a project with only
`npm-shrinkwrap.json` is not detected either), Python projects with only `pyproject.toml` or
`uv.lock` (Python detection needs `poetry.lock`, `Pipfile.lock` or `requirements.txt`), Gradle,
.NET, Cargo, Composer, RubyGems. Each is one more branch in the `generate` step of
[`action.yml`](action.yml): the shape is the same for all of them.

**For npm, install dependencies first.** `cyclonedx-npm` reads the installed `node_modules`
tree, not the lockfile alone, so `npm ci` has to run before this action. The Python generator
reads its lockfile or requirements file directly, and the Go and Maven generators resolve
modules themselves. An SBOM with 0 components fails the action loudly rather than reporting an empty
success.

The npm bill includes `devDependencies` (the generator's default), so their CVEs and licenses
count as well. To leave them out, set `NODE_ENV: production` in the job's `env`: the generator
then omits the development tree, and `npm ci` installs none. For Python, `Pipfile.lock` mode reads
only the default packages, and `poetry.lock` mode reads every group that is not optional.

**Known issue: npm packages nested under another package are not inventoried.** When npm cannot
hoist a version to the top of `node_modules` (two versions of one package in the tree),
`cyclonedx-npm` lists the nested copy inside its parent's entry, and Valystrum reads only the
bill's top-level components. That version is not in the inventory: its CVEs are not matched, its
license is not checked, and a `complete` upload neither lists nor removes it. `npm ls --all` shows
which versions your tree nests.

When more than one ecosystem is detected, the per-tool documents are merged into a single
component list, deduplicated by `purl`.

### What the runner needs

The action runs in `bash` and calls `jq`, `curl` and `openssl`, which it does not install, plus
the toolchain of each ecosystem it scans: Node.js and npm for npm, `python3` with `venv` for
Python, Go for Go (see below for its version), and Maven with a JDK for Maven. GitHub's Ubuntu
runners have all of them. A self-hosted runner or a container job may not. The locked npm
generator asks for Node.js 20.18 or later (22.9 or later on the 22 line) with npm 9 or later. The
Python lock holds Linux wheels only (x86_64 and aarch64, glibc or musl, CPython 3.9 to 3.15), so
on macOS or Windows runners, or with a CPython newer than 3.15, set `ecosystem` to leave out
`python`. The generators are fetched when the step runs, not shipped inside the action, so the
runner must reach app.valystrum.io and the package source of each ecosystem it scans: your npm
registry, your pip index, your Go module proxy and sum.golang.org, your Maven repositories.

`cyclonedx-gomod` 1.12.0 declares `go 1.26.0`, so building it needs Go 1.26 or later. With Go 1.21
to 1.25 on the runner, `go run` first downloads a Go 1.26 toolchain through your module proxy
(checked against sum.golang.org), and fails instead when `GOTOOLCHAIN` is `local` ("requires go >=
1.26.0"). Go before 1.21 cannot build it at all. When you scan a Go module, put Go 1.26 or later on
the runner with `actions/setup-go`.

## Linking to an asset

License and vulnerability findings are far more useful when they name the thing that
ships the dependency. Pass `asset` with the name (or hostname) of the Valystrum Asset:

```yaml
- uses: valystrum/valystrum-sbom@v1
  with:
    ingest-url: https://app.valystrum.io/api/v1/acme/sbom/ingest
    ingest-secret: ${{ secrets.CS_WEBHOOK_SECRET }}
    asset: Checkout API
```

Without it, Valystrum tries a case-insensitive exact match of the `source` label against
your asset names and hostnames. Anything that matches nothing stays unlinked: the
components are still inventoried and still scored, they just are not attached to a guessed
asset.

When an asset resolves, its components are linked to that asset and to the service
the asset belongs to, and both links appear as edges on the security graph.

## What gets sent

```json
{ "source": "acme/web-api", "assetRef": "Checkout API", "complete": true, "sbom": { "bomFormat": "CycloneDX", "components": [ … ] } }
```

(`assetRef` only when `asset` is set, `complete` only when the input is `true`.)

POSTed to `ingest-url` with `X-CS-Signature: t=<unix seconds>,v1=<hex>`: HMAC-SHA256 over
`"<t>.<raw request body>"` under your CI secret, the form the pipeline action signs
too. The platform accepts a stamp within five minutes of its clock, and once, so every attempt,
retries included, is signed afresh. The earlier `sha256=<hex>` form (the body alone, no stamp)
is refused with HTTP 401. The secret authorizes SBOM ingest and the
pipeline action's scan upload for its repository, and nothing else: it opens no route that lists
or reads the organisation's records. An upload's answer does carry a little of them: an SBOM upload
returns the id of the asset it linked, and a scan returns a blast-radius summary (the id of an
asset or cloud resource whose name, or a cloud resource's external id, contains the resource name
of the scan's top infrastructure-as-code finding, how many relations it has, and, unrelated to the
finding, a count of the organisation's failing controls, read from at most ten failing checks, and
the codes of up to five of them).
Both go to whoever holds the secret.

Valystrum keeps `name`, `version`, `purl`-derived ecosystem and license per component. Ingest
upserts on `(org, name, version, source)`, so re-running a workflow is idempotent and retries
are safe.

Each upload's components are matched at once against the CVE records Valystrum already holds, so
a vulnerable component shows as soon as the upload lands, with or without `complete`. The nightly
CVE match adds the records Valystrum does not hold yet. It also closes a match, with or without
`complete`, when OSV's advisory no longer lists the package as affected (withdrawn or corrected):
its finding is noted "Resolved by the SBOM scan of &lt;day&gt;: OSV no longer lists this package
as affected by this CVE: the advisory was withdrawn or corrected", and the component stays.

## Licences

Each component's declared license (the first, when it lists several; an SPDX expression such
as `MIT OR Apache-2.0` counts as one) is resolved to an SPDX id and classified (public domain,
permissive, weak/strong/network copyleft, proprietary, or unknown). Licenses your organization's
policy flags raise a Finding (one per `source` per license, not one per package), which flows into the
risk register and remediation like any other finding. The default policy scores an unrecognised
license high rather than waving it through, so it surfaces for a decision.

Only the npm and Maven generators report licenses. `cyclonedx-gomod` detects none unless it is
asked to (the action does not pass `-licenses`), and `cyclonedx-py` reading `requirements.txt`,
`poetry.lock` or `Pipfile.lock` has no package metadata to read them from. Go and Python
components therefore arrive with no license, which counts as unknown. Under the default policy,
each such source raises one high finding, "An unrecognised license in &lt;source&gt; (N
components)", marked for legal review. An allow list cannot clear it, because there is no license
id to list. If you accept that, lower the risk of the unknown category in your license policy (it
then applies to every unrecognised license, npm and Maven included), or review those components
by hand.

## Removing what is gone

By default an upload only adds and updates: a component the SBOM no longer lists stays in the
inventory with its findings. Set `complete: true` when the step's SBOM is the whole of its
`source` (one workflow, one step per source). Valystrum then removes the source's components
the SBOM no longer lists and resolves their CVE matches (their register findings close with
them), and closes the licence findings of every licence the SBOM no longer carries, each noted
with the upload's day in Europe/Amsterdam (for example "Fixed: not found by the SBOM scan of
28 Oct 2026"). A CVE that still affects the version the SBOM now lists (for example after a
minor upgrade) stays open as the same finding, its title then naming the new version.

A person's verdict wins: a finding marked accepted risk or false positive stays as it is, and a
finding a person reopened stays open until the problem is seen again and a later upload no
longer lists it (a CVE finding whose package left the bill is never seen again under it, so it
stays open until a person closes it). When a finding closes, its open remediation items move to
ready for verification, never to closed (a person verifies and closes them), and on a plan with
remediation their Jira issue or ServiceNow record gets the same note as a comment. A flagged
licence that comes back on a later upload reopens its finding and takes its items waiting for
verification back to in progress. A package that comes back is a new component, and its CVEs
raise new findings at once, from the CVE records Valystrum already holds (the nightly CVE match
adds the ones it does not hold yet).

Leave `complete` off when several steps or workflows upload parts of one source: every upload
would remove the others' components. That holds for any split of one `source`: modules each
scanned with its own `path`, ecosystems each scanned in a step of its own with its own
`ecosystem` (or a step whose `ecosystem` is narrower than earlier uploads), or a step plus a
paste under the same label in **SBOM & Licences**, **Upload an SBOM**, whose tick box sends
`complete` too. Detection looks only in `path` itself, not in its subdirectories, so one step at
the repository root does not see a module's manifest. A monorepo has two ways to remove what is
gone. Give each module its own `source` (for example `acme/mono/billing`) and issue a CI key for
exactly that label in step 1 (each label takes a repository slot and has its own secret), then
set `complete: true` on each module's step. Store each module's secret as a repository secret of
its own (for example `CS_WEBHOOK_SECRET_BILLING`), and pass it as `ingest-secret` on that module's
step together with its `source`. A step signs with the key of the label it sends, so a module step
that uses the repository's `CS_WEBHOOK_SECRET` is refused with HTTP 401 `Invalid signature`. Or
scan once from a manifest that covers every module (a Maven aggregator `pom.xml`, which the action
builds with `makeAggregateBom`) in a single complete step.

`complete` is honoured only when the upload names its `source` (the action always does) and
every named component in the SBOM carries a version. Otherwise the upload only adds, and nothing
says so: it is accepted as usual and `removed` reads `0`. The usual cause is Python's requirements
mode. A line in `requirements.txt` that is not pinned with `==` (for example `requests>=2`, or
`-e .`) becomes a component without a version, and that one line makes the whole upload add-only.
Pin every line, as `pip freeze` or `pip-compile` writes the file, or scan from `poetry.lock` or
`Pipfile.lock`, which carry versions.

With `complete`, the bill is what is installed. The npm generator runs with `--ignore-npm-errors`,
so a package missing from `node_modules` is left out without failing the step. That covers an
install that skipped it, and development packages that `npm ci --omit=dev` left out while
`NODE_ENV` is not `production`. A complete upload then removes the package and closes its CVE
findings as fixed, though your project still depends on it. Send `complete` only from a job whose
full install ran before this step.

With `complete`, the last upload wins: the platform does not know which branch or commit a bill
comes from. Send the complete upload from the default branch only. A pull request or
feature-branch run under the same `source` replaces the inventory with that branch's
dependencies, and the next default-branch run puts them back, closing and raising findings on
the way. Keep runs in order with a `concurrency` group (for example
`concurrency: { group: "sbom-${{ github.repository }}", cancel-in-progress: false }`; the quotes
keep YAML from reading the braces as the mapping's end), so an older run that finishes late
cannot put an older set back.

Besides a complete upload, only a person removes components. A source no longer uploaded (a
renamed `source`, an archived repository, a module that left the monorepo) keeps its components
and their open findings, and shows as stale under **Sources by age** on **SBOM & Licences** after
30 days, with a link to its page. There, the **Retire this source** panel's **Retire source**
button (confirmed with **Yes, retire**) removes its components, resolves their CVE matches, and
resolves their open CVE and licence findings noted "Resolved: the SBOM source &lt;name&gt; was
retired, so its components are no longer tracked"; their remediation items move to ready for
verification as above, and accepted risks and false positives stay. Any member who may upload an
SBOM may retire one, in the dashboard or with `POST /api/v1/<org>/sbom/retire` and
`{ "source": "<the label>" }` from a signed-in session; the retire is on the audit trail.
Retiring a source leaves its CI key active, so a later upload under the same `source` starts it
again. For an archived repository, also revoke its key under **Settings → CI pipeline keys**,
which returns its repository slot. Revoking a key removes nothing from the inventory: retire the
source for that. A complete upload cannot empty a source either: an SBOM with no components fails
the action, and the platform refuses one with no versioned component with HTTP 400 `No valid
components found in SBOM payload`.

## Pinning

The action pins each generator to an exact release and checks it before it runs:

| Generator | Version | How it and everything it installs are locked |
|---|---|---|
| `@cyclonedx/cyclonedx-npm` | 6.0.1 | its whole tree from [`locks/cyclonedx-npm/`](locks/cyclonedx-npm/package-lock.json) (`npm ci --ignore-scripts`: no install script runs), each package by its SHA-512, fetched through your npm registry; nothing is resolved |
| `cyclonedx-bom` | 7.6.0 | its whole tree from [`locks/cyclonedx-py/requirements.txt`](locks/cyclonedx-py/requirements.txt) (`pip --require-hashes --no-deps --only-binary=:all:`: wheels only, so no package's build code runs), each wheel by its SHA-256, in a virtual environment; nothing is resolved |
| `cyclonedx-gomod` | 1.12.0 | `go run` at that version builds from the module's own `go.sum`, every module checked against Go's checksum database (sum.golang.org); it declares `go 1.26.0`, so it needs Go 1.26 or later (see [What the runner needs](#what-the-runner-needs)) |
| `cyclonedx-maven-plugin` | 2.9.3 | the plugin's version fixes every dependency's version, each checked against the repository's checksums (`--strict-checksums`) |

The Python lock's native wheels go no further than CPython 3.15, so a newer CPython needs a
release of this action with a newer lock. The npm generator runs without its optional schema
validators, which Valystrum does not need because it reads the components itself.

The npm generator installs through the registry `npm config get registry` reports in `path` (your
project's `.npmrc` included). Because it installs in a folder of its own, a registry that needs a
token takes it from the user-level npm config (as `actions/setup-node` writes it with
`registry-url` and `NODE_AUTH_TOKEN`), not from the project's `.npmrc`. The Python lock installs
from the index pip is configured with (PyPI unless `PIP_INDEX_URL` or `pip.conf` names another),
and a mirror has to serve PyPI's own wheels, or the hash check refuses them. Go's check against
sum.golang.org holds unless the runner turns it off (`GOSUMDB=off`, or a `GONOSUMDB` or
`GOPRIVATE` pattern that covers `github.com/CycloneDX`). `--strict-checksums` covers the whole
Maven run, your project's dependencies included: an artifact a repository serves without a
checksum, or with a wrong one, fails the step ("Checksum validation failed").

The versions and the lock files are part of the action. The step reads `locks/` from the action's
own folder, and for npm and Python the lock file, not a version setting, decides what installs.
No input chooses another generator version. A newer generator, or a fix in anything it installs,
comes with a newer release of this action, so pinning the action by commit SHA pins the
generators and their whole trees too. If you copy the action into your own repository, keep
`locks/` beside `action.yml`: without it, the npm and Python branches fail. The lock files are
generated from the pinned versions and are not edited by hand: npm's with
`npm install --package-lock-only --ignore-scripts`, without the optional validators, and
Python's with `uv pip compile --universal --generate-hashes --only-binary :all:`, keeping the
Linux wheels' hashes. Pin this action itself by commit SHA rather than `@v1` if you need it to
stay the same across runs.

### Upgrading from v1.1

Up to v1.1 this action was published as `valystrum/sbom-action`; from v1.2 it is `valystrum/valystrum-sbom`. To move,
change `uses: valystrum/sbom-action@v1` to `uses: valystrum/valystrum-sbom@v1`: the inputs are the same, plus
`complete`. `@v1` follows the newest v1 release. v1.2 is for Valystrum v0.21.0 and later, which app.valystrum.io runs.
What a workflow still on `valystrum/sbom-action@v1` (v1.1) sees:

- No `complete` input and no `removed` output. A workflow that sets `complete`, as
  [`examples/sbom.yml`](examples/sbom.yml) does, gets an "Unexpected input(s) 'complete'" warning,
  and the upload only adds and updates.
- The generators float to their latest releases (`npx --yes`, `pipx run` with no version,
  `@latest`, the latest Maven plugin release).
- The Python branch fails: any Python manifest in `path` fails the step, even next to an npm, Go
  or Maven manifest. Set `ecosystem` to the others (for example `npm go maven`) to skip Python.
- Python runs through `pipx`. v1.2 needs `python3` with `venv` instead, which GitHub's runners
  have; check a self-hosted runner before you move.

[CHANGELOG.md](CHANGELOG.md) lists what each release changes.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `No supported manifest found` | No recognised manifest in `path`: check `path` (subdirectories are not searched), or that `actions/checkout` ran |
| `SBOM has 0 components` | Dependencies were not installed before the action |
| `ingest-secret is empty` | The run cannot read the secret the workflow names: check its name (`CS_WEBHOOK_SECRET`). Runs for pull requests from forks, and runs Dependabot starts, get no Actions secrets |
| `npm-ls exited with errors` | `node_modules` has extraneous/invalid packages. The action passes `--ignore-npm-errors`; if you see this, you are on an older tag; re-pin to `@v1` |
| `requires go >= 1.26.0` | The runner's Go is older than 1.26 and `GOTOOLCHAIN` is `local`, so `cyclonedx-gomod` 1.12.0 cannot build: put Go 1.26 or later on the runner (see [What the runner needs](#what-the-runner-needs)) |
| HTTP 409 `QuotaExceeded` when issuing a key | Every repository slot of the plan is taken (Gold: 100). Revoke a key you no longer use, or add the 25 extra repositories add-on. Rotating or rebinding a key never needs a slot |
| HTTP 400 `No valid components found in SBOM payload` | No component in the SBOM has both a name and a version, so nothing could be stored (the action's own 0-component check counts components with or without a version) |
| HTTP 400 `Invalid SBOM body (complete must be true or false)` | The body's `complete` is not a boolean. The action sends only `true` or nothing |
| HTTP 401 `Invalid signature: sign with this repository's CI key (POST /api/v1/<org>/pipeline/config creates one).` | One answer for each of these, so it tells nobody which organisations exist or which repositories have a key. The `<org>` in `ingest-url` is not your organisation's slug. Or there is no active key for the `source` label: step 1 was never run for this repo, its key was revoked (issue it again in step 1 for a new secret), or `source` names something other than the repo the key was issued for (case included), or the repository was renamed or transferred: rebind its key with `PATCH /api/v1/<org>/pipeline/config/<keyId>` and `{"repo": "owner/new-name"}` (no organisation-wide key stands in). Or a wrong or rotated secret in `CS_WEBHOOK_SECRET`, or another repository's (for example one shared as a GitHub organisation secret); `source` names another repository or monorepo module with a key of its own, so that key was checked (a module step needs its own module's secret); or a commit pinned from before the stamped signature, which sends the refused `sha256=` form (re-pin to `@v1`) |
| HTTP 401 `Signature timestamp outside the five-minute window` | The runner's clock is off by more than five minutes; the signature is stamped with `date +%s` |
| HTTP 401 `Signature already used` | The same signed request was sent twice; each attempt, retries included, signs afresh, so this only happens on a manual replay |
| HTTP 403 `FeatureLocked` | The organisation is below the Gold plan |
| HTTP 403 `Organization is suspended` | The organisation is suspended; machine ingest stops with the dashboard |
| HTTP 413 | SBOM over 8 MB: narrow `path`, or split the workflow per module (and leave `complete` off for the split uploads, or give each module its own `source`) |
| Components keep disappearing and coming back | Several uploads of one source set `complete: true` (split by `path`, by `ecosystem`, or a dashboard upload under the same label), so each removes the others' components: set it only on a single upload that carries the whole source. Runs from another branch, or an older run that finishes late, do the same (see [Removing what is gone](#removing-what-is-gone)) |
| `complete: true` is set, but dropped packages stay listed and `removed` reads `0` (or is empty) | The platform refuses nothing when it does not honour `complete`: the upload is ingested as add-only, with no warning from it. Either a component in the SBOM has no version (the whole upload then only adds; in Python's requirements mode, a `requirements.txt` line not pinned with `==`), or the action in use is v1.1, which has no `complete` input or `removed` output (the log shows "Unexpected input(s) 'complete'"; see [Upgrading from v1.1](#upgrading-from-v11)) |
| HTTP 429 | More than 30 uploads for the organisation in one minute, counted only once an upload's signature checks out, or more than 150 requests a minute from one address, signed or not (runners behind one NAT share an address); the action's retries back off, then the step fails |
