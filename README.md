# dmv-scanner-runner

Public runner for the DMV road-test availability scanner.

This repo contains **only** a GitHub Actions workflow. The scanner's actual
source code is **not stored here** — it lives in a private repository
(`David-Cito/dmv-monitor-v2`) and is cloned at runtime by the workflow using a
scoped, read-only deploy token. This keeps the scanning/booking logic and DMV
endpoint details private while allowing the scheduled job to run on GitHub's
free public-repo Actions minutes.

## How it works

`.github/workflows/scan.yml` runs on a `*/5 * * * *` cron (plus manual
`workflow_dispatch`). Each run:

1. Clones the private `dmv-monitor-v2` repo (shallow, depth 1) into `scanner/`
   using `PRIVATE_REPO_TOKEN`.
2. Installs dependencies with pnpm inside `scanner/`.
3. Runs `apps/scanner/src/rt-scanner.ts` in a continuous loop for up to ~5
   hours (bypassing GitHub Actions' scheduled-workflow queue delays), with a
   30s night-mode throttle between 1am-5am Hawaii time.

This is a byte-for-byte port of production's `rt-scanner.yml` loop, with only
the private-repo clone and `working-directory: scanner` changes needed to run
someone else's checked-out code.

## Required Actions secrets

| Secret | Purpose |
|---|---|
| `PRIVATE_REPO_TOKEN` | Fine-grained PAT, **read-only**, scoped to the single `dmv-monitor-v2` repo's contents. Used only to `git clone` the scanner source. |
| `SUPABASE_URL` | Supabase project URL the scanner reads/writes availability to. |
| `SUPABASE_SERVICE_KEY` | Supabase service-role key used by the scanner. |
| `DISCORD_WEBHOOK_URL` | Webhook the scanner posts alerts to. |

## Security notes

- `scan.yml` has **no `pull_request` trigger**. Workflows in public repos that
  do trigger on `pull_request` from forks can be tricked into printing or
  exfiltrating secrets via a malicious PR diff (a well-known GitHub Actions
  supply-chain risk) — this workflow avoids that class of attack entirely by
  only running on `schedule` and `workflow_dispatch`.
- GitHub additionally requires maintainer approval before Actions runs
  triggered by outside collaborators/forks execute in a repo with secrets, so
  even a fork of this repo cannot run with real secrets without an explicit
  approval from a maintainer.
- `PRIVATE_REPO_TOKEN` should be a fine-grained PAT scoped to **read-only
  contents access on `dmv-monitor-v2` only** — it cannot write to the private
  repo, cannot access any other repo, and clones only what's needed to run
  the scanner.
