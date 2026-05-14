# Implementation Notes

**Exercise:** Customer Success Engineer — Postman  
**Services onboarded:** Payment Processing API - Refund Service, Lending Platform API - Loan Origination Service  
**Repos:** [payment-refund-service](https://github.com/mmorales84/payment-refund-service) · [loan-origination-service](https://github.com/mmorales84/loan-origination-service) · [cse-exercise](https://github.com/mmorales84/cse-exercise) (this repo — composite action + adaptation analysis)

---

## How I Built the Workflow

### Starting point: the starter asset

The exercise includes a starter composite action at `starter-assets/actions/onboard-service/action.yml` that wraps `postman-cs/postman-api-onboarding-action@v0`. I read it before using it and found a bug on line 35: `spec-url` was receiving `inputs.spec-path` instead of `inputs.spec-url`, meaning every run would attempt to fetch a local file path over HTTPS and fail at spec upload.

Rather than bypass the composite, I fixed the bug and extended it. The composite has real value as an abstraction layer: it exposes a reduced input surface (the 7 inputs any service team needs to know) and hides the 40+ input surface of the upstream action. For a platform team onboarding 50 services, that's the difference between a self-serve process and one that requires CSE involvement for every run. I also added `environments-json` as an explicit input with a default of `["prod"]`, and hardcoded `generate-ci-workflow: false` (explained below).

The fixed composite lives in this repo at `starter-assets/actions/onboard-service/action.yml` and is referenced by both service repos as:

```yaml
uses: mmorales84/cse-exercise/starter-assets/actions/onboard-service@main
```

### How the actions chain together

Calling the composite triggers this sequence:

1. **`postman-cs/postman-bootstrap-action@v0`** — reads the spec from the raw GitHub URL, creates a Postman workspace, uploads the spec to Spec Hub, generates three collections (Baseline, Smoke, Contract), and configures environments. Writes nothing to the repo — all output is Postman-side. Emits workspace ID, spec ID, and collection IDs as step outputs.

2. **`postman-cs/postman-repo-sync-action@v0`** — takes the IDs from bootstrap, exports the collections and environments as YAML/JSON files, commits them to the repo, and writes `.postman/resources.yaml` with the resource ID mappings. On first run this creates all assets; on rerun it uses the persisted IDs to update in place rather than creating duplicates.

On rerun, the action reads `.postman/resources.yaml` to find existing workspace, spec, and collection IDs. Collections are regenerated in `refresh` mode (the default), meaning they reflect the current spec state. This makes the workflow safe to rerun after a spec update — it won't create orphaned workspaces or duplicate collections.

### Topology decision: per-service repos

I chose one GitHub repo per service rather than a single monorepo. The `postman-repo-sync-action` commits generated artifacts back to the repo it runs in and links the Postman workspace to that specific repo. In a real engagement, generated CI artifacts (collections, environments, the test runner workflow) belong in the service's own codebase — not in a shared repo. The per-service topology mirrors how the platform team would actually maintain this.

The composite action lives in this central repo (`cse-exercise`) and is referenced remotely by both service repos. In a real engagement, this would be the platform team's shared infrastructure repo.

### The generated CI workflow

The `postman-repo-sync-action` generates a `ci.yml` test runner and commits it to the repo. I disabled auto-generation (`generate-ci-workflow: false` in the composite) after finding two bugs in the generated output:

1. **Literal `\n` escaping**: the action serialized the file with `\n` as a two-character escape sequence instead of actual newlines, producing a single-line string that fails YAML parsing.
2. **`secrets` in `if:` expressions**: the generated SSL decode step used `if: ${{ secrets.POSTMAN_SSL_CLIENT_CERT_B64 != '' }}`, which is invalid — GitHub Actions does not expose the `secrets` context in `if:` conditions.

I fixed both in a manually maintained `ci.yml` in `payment-refund-service`. The fix maps all SSL secrets to job-level env vars first, then the `if:` condition checks `env.POSTMAN_SSL_CLIENT_CERT_B64 != ''`. This makes the file service-agnostic: services with mTLS add the cert secrets and the SSL steps run automatically; services without mTLS leave the secrets unset and the steps are skipped. The same `ci.yml` template is copied to `loan-origination-service`.

Keeping the CI workflow decoupled from the onboarding action is the right architecture anyway — the platform team should own and version their test runner template independently of the onboarding automation.

---

## Universal vs. Per-Service

### What stays the same across every service

- The composite action call and its structure
- The three-action chain (bootstrap → repo-sync)
- The three collection types generated (Baseline, Smoke, Contract)
- The `.postman/resources.yaml` idempotency mechanism
- The CI workflow template
- The GitHub Secrets required (`POSTMAN_API_KEY`, `CSE_WORKFLOW_TOKEN`)
- The repo permissions required (`contents: write`, `actions: write`)

### What changes per service (inputs only)

| Input | Payment Refund | Loan Origination |
|-------|---------------|------------------|
| `project-name` | `payment-refund-api` | `loan-origination-api` |
| `domain` | `payments` | `lending` |
| `environments-json` | `["prod","uat","qa","dev"]` | `["prod","staging","dev"]` |
| `spec-url` / `spec-path` | refund spec | loan origination spec |

Four values. Everything else is identical.

### What requires post-onboarding configuration (not expressible as workflow inputs)

- **Base URL per environment**: the `env-runtime-urls-json` input maps environment slugs to real endpoint URLs. Without this, generated environments have the right structure but empty base URLs, and smoke tests fail with `Invalid URI`. In a real engagement, the platform team provides these from their AWS/ALB configuration.
- **Auth credentials for test execution**: real bearer tokens or service credentials must be set in the Postman environment variables for the smoke and contract tests to authenticate successfully against live endpoints.
- **mTLS certificates** (Loan Origination): the `POSTMAN_SSL_CLIENT_CERT_B64` and `POSTMAN_SSL_CLIENT_KEY_B64` secrets must be populated from the customer's PKI for the CI runner to authenticate to the ECS/ALB endpoint.
- **Test fixture data**: requests like `POST /applications` (Loan Origination) require valid `applicantId` values; `POST /applications/{id}/documents` requires a real file. These cannot be derived from the spec.

---

## What Generated Tests Prove vs. What They Don't

### What the generated tests establish from the spec alone

The bootstrap action generates three collections:

- **Baseline**: one request per operation, with example values from the spec. Proves the collection correctly represents the API surface.
- **Smoke**: lightweight execution checks — status code is 2xx, response time is within threshold, response body is not empty. Proves the endpoint is reachable and responding.
- **Contract**: validates response schemas, required fields, content types, and declared status codes against the OpenAPI definition. Proves the implementation matches the spec.

These are valid assertions derivable entirely from the spec. They give any engineer an immediate, structured view of the API's surface before writing a single custom test.

### Where generated tests stop

- **Real connectivity**: smoke tests fail if the base URL isn't configured with a real endpoint. This is expected — it's a configuration gap, not a test gap.
- **Auth**: contract tests cannot authenticate without real credentials. The spec defines the auth scheme; the credentials are a runtime concern.
- **Async workflows** (Loan Origination underwriting): the smoke test correctly expects `202 Accepted` but cannot poll for the async decision or validate the eventual result payload. Validating the full async lifecycle requires a custom polling step or webhook harness.
- **File uploads** (Loan Origination documents): the generated collection scaffolds the `multipart/form-data` request but cannot supply a real binary file. Meaningful smoke testing of this endpoint requires a test fixture.
- **Business logic**: the contract tests validate response shapes, not business rules. Whether a refund correctly reverses a payment, or whether a loan decision reflects the right risk model, is outside what any spec-derived test can verify.

**Why this distinction matters operationally:** the generated tests are a baseline, not a ceiling. They eliminate the "no tests at all" state and give the QA lead something to extend immediately. The platform team knows exactly where the automation stops and where service-specific work begins — which is the right conversation to have in the first working session.

---

## What the Platform Team Must Configure

These are items that require access I don't have in this exercise:

| Item | Required for | Who provides it |
|------|-------------|-----------------|
| Real endpoint URLs per environment | `env-runtime-urls-json` input or manual environment config in Postman | Platform/ops team (from AWS API Gateway / ALB configs) |
| Bearer tokens / service credentials for test execution | Postman environment variables | Service team |
| mTLS client cert + key (Loan Origination) | `POSTMAN_SSL_CLIENT_CERT_B64`, `POSTMAN_SSL_CLIENT_KEY_B64` secrets | Platform/ops team (from internal PKI) |
| GitLab CI integration (Claims Processing) | A separate integration pattern — the pre-built GitHub Actions cannot run in GitLab CI | Platform/ops team + GitLab admin |
| AWS account access for Insights agent | Enabling API traffic discovery via Postman Insights | Platform/ops team |
| Services without OpenAPI specs | Either generate specs from gateway traffic, write them from scratch, or use Postman's import-from-collection path | Service teams + CSE working session |

---

## Run Instructions

### Prerequisites

- Postman Enterprise account with API key (`POSTMAN_API_KEY`)
- GitHub PAT with `repo`, `workflow`, and `contents: write` scopes (`CSE_WORKFLOW_TOKEN`)
- Service repo is **public** (the bootstrap action fetches the spec via unauthenticated HTTPS from `raw.githubusercontent.com`)

### Running the onboarding workflow

```bash
# Payment Refund service
gh workflow run onboard-service.yml --repo mmorales84/payment-refund-service

# Loan Origination service
gh workflow run onboard-service.yml --repo mmorales84/loan-origination-service
```

Or trigger via GitHub Actions UI: Actions → "Onboard Service to Postman" → Run workflow.

### Validating the run

Check the job status is green, then verify:

**In Postman:**
- Workspace is visible under your account
- Spec is present in the APIs (Spec Hub) tab
- Three collections are present: `[Baseline]`, `[Smoke]`, `[Contract]`
- Environments match the `environments-json` input

**In the repo (committed by the action):**
- `postman/collections/` — YAML collection exports
- `postman/environments/` — environment JSON files
- `.postman/resources.yaml` — resource ID mappings (used for idempotent reruns)

### Rerunning

Reruns are safe. The action reads `.postman/resources.yaml` to find existing IDs and updates in place. Collections are refreshed from the current spec state. No duplicate workspaces or collections are created.

---

## Issues Encountered and How I Resolved Them

| Issue | Root cause | Resolution |
|-------|-----------|------------|
| Starter composite passes `spec-path` to `spec-url` | Bug in starter asset line 35 | Fixed: `inputs.spec-path` → `inputs.spec-url` |
| `spec-url` 404 on first run | Repo was created as private; `raw.githubusercontent.com` requires public access for unauthenticated fetches | Made repo public |
| Generated `ci.yml` fails YAML parse | Action serializes content with literal `\n` instead of newlines | Replaced with manually maintained template |
| `secrets` in `if:` expression invalid | GitHub Actions does not expose `secrets` context in `if:` conditions | Mapped secrets to job-level env vars; changed condition to check `env.*` |
| `GITHUB_TOKEN` cannot write `.github/workflows/ci.yml` | GitHub Actions App does not have `workflows` permission by default; `workflows` is not a valid scope in the `permissions:` block | Used GitHub PAT (`CSE_WORKFLOW_TOKEN`) with `repo` + `workflow` scopes as `github-token` |
| Smoke tests fail in CI | Generated environments have empty base URLs; specs point to `example.com` which doesn't exist | Expected and documented — requires real endpoint URLs via `env-runtime-urls-json` |

---

## Trade-offs

**Per-service repos vs. monorepo:** Per-service repos match real engagement topology and keep generated artifacts in the right codebase. A monorepo would be simpler for the exercise but harder to explain as a repeatable pattern to the platform team.

**Fixed composite vs. direct upstream action call:** The composite adds an abstraction layer the platform team owns. The cost is one additional moving part and a cross-repo dependency. The benefit is a stable, minimal input surface that service teams can use without understanding the upstream action.

**`generate-ci-workflow: false`:** Disabling auto-generation avoids regenerating a buggy file on every run. The cost is that the CI workflow is no longer automatically updated when the upstream action changes its template. The benefit is a predictable, platform-team-owned artifact.

---

## AI Usage

Claude Code (claude-sonnet-4-6 via the Claude Code VS Code extension) was used throughout this exercise for:

- Reading and comparing the three OpenAPI specs
- Generating the workflow YAML and composite action updates
- Diagnosing GitHub Actions errors from log output (private repo 404, `secrets` in `if:`, literal `\n` escaping)
- Writing this README and the adaptation analysis

Everything AI-generated was validated before committing: workflow files were read and reviewed, action logs were inspected for actual behavior rather than trusting green checkmarks, and bugs (both in the starter asset and the generated `ci.yml`) were caught by reading the files rather than assuming correctness.

The debugging decisions — why the repo needed to be public, what the `\n` encoding indicated, why `secrets` can't be used in `if:` conditions, how to fix the SSL step without removing mTLS support — were evaluated and confirmed before applying fixes.
