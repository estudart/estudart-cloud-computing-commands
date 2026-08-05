# Cloud & Dev Cheatsheet

Personal lookup reference for infra commands I forget between tasks. This is not a tutorial — it's a set of narrative runbooks organized by topic, meant to be grepped mid-task, not read start to finish.

Every command uses `<PLACEHOLDER>` values, followed by one fully filled example using a consistent fake company (Nike / `nike.com` / project `nike-retail-prod`) so the shape is still readable without any real data ever appearing here.

## How to use this

- Grep for the tool or error you're stuck on, not the file name.
- Every entry ends with **"Why it bites"** — that section is the actual point of this repo. The command itself you could google in ten seconds; the failure mode you can't re-derive later.
- `<!-- verified: YYYY-MM -->` under a title means it was actually run and the failure mode actually happened. No tag = untested, treat with suspicion.

See [CONTRIBUTING.md](CONTRIBUTING.md) before adding an entry.

## Structure

- [`github/`](github/) — SSH key setup & everyday use, GitHub UI steps
- [`gcp/deploy/`](gcp/deploy/) — Cloud Run deploy + IAP
- [`gcp/iam/`](gcp/iam/) — identity vs access tokens, service-to-service auth, IAM bindings
- [`gcp/cloud-sql/`](gcp/cloud-sql/) — Cloud SQL connectivity paths
- [`gcp/org-policies/`](gcp/org-policies/) — org policies that force architecture decisions
- [`docker/`](docker/) — Artifact Registry create/auth/push
- [`node/`](node/) — proxy timeouts, corporate TLS interception
- [`git/`](git/) — subtree split and other git recipes
- [`dev-tools/`](dev-tools/) — prettier, pre-commit
- [`bash/ssh-auth/`](bash/ssh-auth/) — Raspberry Pi / generic Wi-Fi + SSH
- `aws/` — reserved for future AWS entries
