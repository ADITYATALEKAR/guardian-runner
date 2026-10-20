# guardian-runner

A **public** repo that provides free, unlimited GitHub Actions minutes (4-vCPU Linux
runners) to run Guardian external scans. It contains **no engine code** — only two
workflows and a public encryption key. The private engine is checked out at run time
with a read-only deploy key, runs one scan, and the only thing that leaves the runner
is an encrypted result that just the private-key holder can open.

## Why a public repo is safe here

| Risk of a public repo | How this repo avoids it |
|---|---|
| Open-sourcing the engine | Engine never lives here. It is checked out at run time via `ENGINE_DEPLOY_KEY` (read-only, scoped to the engine repo) and deleted when the job ends. |
| Findings in public build logs | The engine forces its sensitive loggers to WARNING; the launcher prints only coarse progress and byte counts. |
| Findings in downloadable artifacts | The result is encrypted (X25519 + AES-256-GCM) to `ci_result_public.pem` **before** upload. Anyone can download the artifact and get ciphertext. |
| Who-you-scanned visible on the run page | The target is **not** a dispatch input. It is read from a repo secret you name (e.g. `TARGET_1`), and masked in logs. |
| Forks running the workflow / reading secrets | No `pull_request` trigger. Manual dispatch (and selftest on push) only. |

## Setup (already done for this repo)

- Engine repo deploy key (read-only) ↔ secret `ENGINE_DEPLOY_KEY`
- Variables `ENGINE_REPO` (`owner/engine-repo`) and `ENGINE_REF` (branch to scan with)
- `ci_result_public.pem` — the recipient key. The matching private key is at
  `~/.guardian/ci_result_private.pem` on the operator's machine and is **never** uploaded.

To add a scan target, create a repo secret whose value is `Institution Name|example.com`:

```
gh secret set TARGET_1 -R <owner>/guardian-runner --body "Example Bank|example.com"
```

## Running a scan

Actions → **guardian-scan** → *Run workflow*:
- `target_secret` — name of the secret holding the target (e.g. `TARGET_1`)
- `seconds` — cycle budget (default 1200 = 20 min)
- `max_endpoints` — observation cap (default 1000; engine ceiling 5000)
- `max_workers` — observation threads (default 32; engine cap 75; scans are network-bound so 32–48 is a good range on a 4-vCPU runner)

When it finishes, download the artifact `scan-result-<run id>` and import it into your
local console from the engine repo:

```
# a dedicated login for that institution (how the bank's own operator would see it):
python scripts/ci_result_import.py --in scan_result.enc --link new-operator \
    --operator-email ops@example-bank.com --operator-password '<at least 12 chars>'

# or point an existing operator at it (its previous tenant stays on disk):
python scripts/ci_result_import.py --in scan_result.enc --link replace --operator-id usr_…
```

Guardian links one institution per login, so pick one of the two. Sign in and the
tenant appears with its atoms, findings and signed ledger intact.
Delete the artifact afterwards (Actions → run → artifact → delete) — it is ciphertext,
but there is no reason to leave it around.

## Self-test

**guardian-selftest** runs on every push and on demand. It starts a local HTTPS server
on the runner seeded with chatbot / internal-exposure signals, scans `localhost` with
strict TLS (via a throwaway CA installed in the runner's trust store), and asserts that
observation saved endpoints, the `ai_chatbot` and `internal_exposure` probes fired, and
the artifact is ciphertext. No third party is contacted.

## Scaling

One run = one institution = one fully independent tenant (own storage, atom store and
transparency ledger), so runs never share state and there is no ledger-merge problem.
Launch several dispatches to scan several institutions at once (Free plan: 20
concurrent jobs). Within a run, raise `max_workers` to speed up observation.

## Authorization

Only scan domains you are authorized to assess. Scans run from GitHub's IP ranges;
make sure the target's security team expects that source.
