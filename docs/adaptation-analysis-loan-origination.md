# Adaptation Analysis: Loan Origination API

**Service:** Lending Platform API - Loan Origination Service  
**Spec:** `specs/loan-origination-api-openapi.yaml` (v1.4.0)  
**Infrastructure:** ECS / Application Load Balancer  
**CI/CD:** GitHub Actions  
**Compared against:** Payment Processing API - Refund Service (working implementation)

---

## What Stays the Same

The onboarding workflow is identical in structure. The same composite action call works unchanged:

```yaml
- uses: mmorales84/cse-exercise/starter-assets/actions/onboard-service@main
  with:
    project-name: loan-origination-api
    domain: lending
    spec-url: https://raw.githubusercontent.com/<owner>/loan-origination-service/main/specs/loan-origination-api-openapi.yaml
    spec-path: specs/loan-origination-api-openapi.yaml
    environments-json: '["prod","staging","dev"]'
    postman-api-key: ${{ secrets.POSTMAN_API_KEY }}
    postman-access-token: ${{ secrets.POSTMAN_ACCESS_TOKEN }}
    github-token: ${{ secrets.CSE_WORKFLOW_TOKEN }}
```

Everything the bootstrap action does is driven by the spec and inputs — workspace creation, spec upload to Spec Hub, generation of all three collections (baseline, smoke, contract), environment scaffolding, and resource ID persistence in `.postman/resources.yaml`. None of that logic changes.

The CI workflow template (`ci.yml`) is also identical — the SSL certificate block already handles mTLS conditionally via job-level env vars. For this service, the platform team adds the cert secrets; the workflow logic doesn't change.

---

## What Changes

### 1. Workflow inputs (minimal)

| Input | Payment Refund | Loan Origination |
|-------|---------------|------------------|
| `project-name` | `payment-refund-api` | `loan-origination-api` |
| `domain` | `payments` | `lending` |
| `environments-json` | `["prod","uat","qa","dev"]` | `["prod","staging","dev"]` |
| `spec-url` / `spec-path` | points to refund spec | points to loan origination spec |

The environment list shrinks from 4 to 3. The Lending team uses `staging` instead of `uat`/`qa` — a naming convention difference with no workflow impact.

### 2. mTLS authentication requires additional secrets

The spec describes two auth patterns: mTLS for service-to-service calls, JWT for internal gateway calls. The CI workflow already has the SSL cert handling built in — it's gated on the presence of secrets. The platform team needs to add three secrets to the service repo:

| Secret | Purpose |
|--------|---------|
| `POSTMAN_SSL_CLIENT_CERT_B64` | Base64-encoded client certificate |
| `POSTMAN_SSL_CLIENT_KEY_B64` | Base64-encoded private key |
| `POSTMAN_SSL_CLIENT_PASSPHRASE` | Passphrase if the key is encrypted (optional) |

Without these, the generated smoke and contract tests will run using JWT only — which works if the ALB accepts JWT, but will fail for endpoints that require mTLS at the network layer.

### 3. Spec inconsistency: mTLS described but not formally defined

The spec's `info.description` documents mTLS as the service-to-service auth method, but the `securitySchemes` section only defines `JWT`, and the global `security:` block applies JWT only. mTLS is not represented as a security scheme.

**Operational impact:** The generated contract tests will validate JWT auth coverage but will have no awareness of mTLS requirements. The bootstrap action can only generate tests for what is formally declared in the spec. This is a spec drift issue — the platform team and lending service owners need to either:
- Add an `mTLS` security scheme to the spec and mark the appropriate endpoints, or
- Accept that mTLS validation is out of scope for generated tests and covered by network-layer controls

This should be surfaced in the working session with the platform team as a governance issue, not a tooling gap.

### 4. Async underwriting workflow

`POST /applications/{applicationId}/underwriting` returns `202 Accepted` — the decision arrives asynchronously. The generated smoke test will correctly assert the `202` status, but it cannot poll for the underwriting result or validate the eventual decision payload.

**What the generated test proves:** the endpoint accepts the request and queues the job.  
**What it cannot prove without service-specific knowledge:** whether the async decision arrives correctly, what the decision payload looks like, or how long processing takes.

For complete coverage, the platform team would need to add a polling step or webhook listener to the collection — that work cannot be generated from the spec alone.

### 5. Document upload uses multipart/form-data

`POST /applications/{applicationId}/documents` uses `multipart/form-data` with a binary file field. The generated baseline collection will scaffold this request, but the smoke test will not have a real file to upload. Expect the smoke test for this endpoint to fail in CI until the platform team supplies test fixture data.

This is the same class of problem as the base URL gap in Payment Refund — the generated test establishes the right request structure; real execution requires service-specific test data the spec cannot provide.

---

## What the Platform Team Must Supply

These cannot be derived from the spec. They require access that only the platform team has:

| Item | Why it's needed | Who provides it |
|------|----------------|-----------------|
| Internal ALB endpoint URLs (prod, staging, dev) | `env-runtime-urls-json` input, or manually set in Postman environments post-onboarding | Platform/ops team |
| mTLS client certificate + key | Issued by the customer's internal CA, scoped to CI service account | Platform/ops team (from PKI/secrets management) |
| JWT test token or service account credentials | For smoke/contract test execution against real endpoints | Lending service team |
| Test applicant IDs safe for automated test use | `POST /applications` requires a real `applicantId` — using production IDs in CI is a compliance risk | Lending service team |
| Document test fixtures | Binary files for the document upload endpoint | Lending service team |

---

## What the Generated Tests Establish Immediately

From the spec alone, the bootstrap action generates tests that validate:

- All HTTP methods and paths are reachable at the configured base URL
- Request schemas are enforced (required fields, enum values, type constraints)
- Response status codes match the spec (201, 202, 400, 404, 409)
- Response schemas match declared shapes
- JWT auth is applied to all secured endpoints
- The health endpoint (`GET /health`) responds with the declared schema

These are valid from day one of onboarding, before the platform team adds any service-specific configuration. They give the QA lead on the lending side an immediate baseline to build on.

---

## Summary: What the Pattern Proves

The onboarding workflow for Loan Origination is four input changes from Payment Refund. The pattern is not a series of custom implementations — it's a parameterized system. The meaningful differences (mTLS, async workflows, multipart uploads, environment naming) are all handled through configuration and documented handoff items, not by changing the workflow.

The spec inconsistency around mTLS and the async underwriting pattern are the two areas where generated coverage stops short of full confidence. Both are worth a working session with the lending team to resolve — one as a spec governance issue, one as a test design question.
