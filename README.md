# Renovate #45269 reproduction

Minimal reproduction for [renovatebot/renovate#45269](https://github.com/renovatebot/renovate/issues/45269): repeated `Skipping ... because no currentDigest or pinDigests` DEBUG messages for equivalent dependencies.

## Fixture

- `Dockerfile`: `node:alpine`
- `Dockerfile.worker`: `node:alpine`
- `compose.yaml`: `node:alpine` and `redis:alpine`
- `renovate.json`: enables only Dockerfile and Docker Compose managers and disables digest pinning.

Docker versioning treats `alpine` as unversioned. There are no current digests, so all four references reach the unsupported-value skip path. Redis is the distinct-dependency control.

## Run from a Renovate source checkout

Prerequisites: Git, Node.js satisfying the source checkout's engines (Node 24.22.1 was used), and installed Renovate source dependencies. Use that checkout's pnpm version and frozen lockfile. Do not use a separately installed Renovate package.

Clone this repository; the fixture files must be tracked because the local platform uses `git ls-files`. From this repository, run (POSIX shell):

```sh
LOG_LEVEL=debug LOG_FORMAT=json node /absolute/path/to/renovate/lib/renovate.ts --platform=local --dry-run=lookup --onboarding=false > /tmp/renovate-45269.jsonl
```

PowerShell equivalent (ensure `node` is the compatible Node 24 executable):

```powershell
$env:LOG_LEVEL = 'debug'
$env:LOG_FORMAT = 'json'
node D:/Code/renovate/lib/renovate.ts --platform=local --dry-run=lookup --onboarding=false > "$env:TEMP/renovate-45269.jsonl"
```

Keep logs outside this repository. Use lookup mode, not extraction-only mode. No Docker daemon or container execution is needed.

## Verified local before-and-after results

Tested with identical fixtures and separate source checkouts at baseline `78734cb84` and patched `1d64077a2`. The patched checkout was run twice independently.

| Exact DEBUG message | Baseline | Patched run 1 | Patched run 2 |
| --- | ---: | ---: | ---: |
| `Skipping node because no currentDigest or pinDigests` | 3 | 1 | 1 |
| `Skipping redis because no currentDigest or pinDigests` | 1 | 1 | 1 |

All runs extracted four dependencies from three files, retained `skipReason: invalid-value` for all four, generated zero updates, and emitted no WARN/ERROR logs. The adjacent `Dependency node has unsupported/unversioned value alpine (versioning=docker)` message appeared three times in every run, confirming that each node occurrence still reached lookup. That adjacent message is deliberately outside the fix's scope.

Count exact JSON `msg` values; do not count substring matches elsewhere in configuration dumps. These results demonstrate this unversioned-tag path, not valid-version update generation.

For hosted GitHub testing, supply a suitable token through `RENOVATE_TOKEN` without printing it, and run the same source with `--platform=github --dry-run=lookup --onboarding=false ahmedbektic/renovate-45269-repro`. Never put credentials in command arguments or repository files.

## Verified hosted GitHub results

The actual source entry points at the same baseline and patched revisions were also run against this public repository using `--platform=github --dry-run=lookup --onboarding=false ahmedbektic/renovate-45269-repro`. Authentication was supplied only through the process environment. Each run used a fresh `RENOVATE_BASE_DIR` outside the repositories.

| Exact DEBUG message | GitHub baseline | GitHub patched run 1 | GitHub patched run 2 |
| --- | ---: | ---: | ---: |
| `Skipping node because no currentDigest or pinDigests` | 3 | 1 | 1 |
| `Skipping redis because no currentDigest or pinDigests` | 1 | 1 | 1 |

The hosted runs cloned the repository, extracted the same four dependencies, and produced dependency-result objects identical to the local baseline: four `invalid-value` skips, no updates, and no dependency warnings. Each run had zero ERROR logs and one unrelated global WARN about Renovate's default Git author email. That warning occurred on both baseline and patched source.

Git refs and the complete issue/PR listing were identical before and after all hosted dry runs. Publishing this fixture and documentation was a separate, intentional Git operation; Renovate created no branches, commits, issues, or PRs.

The baseline is the direct parent of the fix, not a different released Renovate version. No source code was overwritten to obtain these comparisons. Raw logs and caches are deliberately not committed.
