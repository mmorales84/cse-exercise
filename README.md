# Customer Success Engineer Implementation Exercise

**Repository:** https://github.com/postman-cs/cse-exercise

**Role:** Customer Success Engineer at Postman
**Timeline:** 2-3 hours (use AI assistants)
**Final Session:** 45 minutes (30-min presentation + 15-min live discussion)

**What this role actually looks like:** CSEs spend roughly 75% of their time working in customer environments -- their cloud infrastructure, Git repos, CI/CD pipelines. Postman itself is about 25% of the day. You're an engineer who works alongside customer engineers, not someone who demos features.

**Why This Exercise Matters:** CSEs solve infrastructure problems for enterprise customers -- sometimes building from scratch, sometimes leveraging existing tooling. This exercise tests whether you can take production-grade automation, apply it to a realistic customer environment, adapt it when things don't fit, and present a credible scaling and handoff plan.

## The Situation

**Customer Profile:** Large financial services company, high six-figure ARR, ~2,000 Postman seats across the engineering org

The company has hundreds of APIs across the organization. Your engagement is scoped to one platform engineering team that supports roughly 50 services across several product lines (payments, lending, insurance). They're the pilot -- if this works, it expands org-wide.

**Current State (key facts):**
- User activity is decent on paper, but systemic engineering value is low
- Hundreds of shared workspaces with inconsistent structure and ownership
- Thousands of collections, many ad-hoc single requests with only basic status checks
- It can take hours to discover and successfully call an internal API
- No API catalog, no centralized governance, no automated contract testing
- Infrastructure is mixed: most services run on AWS Lambda behind API Gateway, some on ECS behind ALBs, a handful on Kubernetes. CI/CD is mixed too, and the surrounding delivery patterns are not uniform across teams.
- Most teams use GitHub Actions for CI/CD, but one group runs GitLab CI and has no plans to switch
- Some services have OpenAPI specs in their repos. Many don't. Some have outdated specs that have drifted from the actual implementation.

**Baseline Problem:** Engineers routinely lose significant time just trying to discover and successfully call internal APIs -- jumping across systems, hitting dead ends, and piecing together tribal knowledge. Writing tests against internal endpoints is equally painful: there's often no spec to reference, no collection to start from, and no environment config to plug in. The team knows these problems are widespread but doesn't know where to start fixing them. APIs exist in production but aren't discoverable, documented, or testable through Postman.

**The Platform Team (your counterpart):** A 4-person platform/ops team spans all the product lines. They manage the AWS accounts, CI/CD templates, and shared infrastructure. They don't write application code but they own the deployment pipelines and would maintain any cross-cutting automation after CSE leaves. They care about: working within their existing CI/CD patterns (they won't adopt a new tool just for Postman), being able to onboard new services themselves without calling you, and keeping maintenance low. They're stretched thin.

**Key contacts:**
- VP of Platform Engineering (your primary stakeholder -- cares about the value and needs to present progress to the CIO in 60 days)
- A senior DevOps engineer on the platform team (your hands-on partner)
- An engineering manager on the payments side who's willing to pilot first
- A QA lead on the lending side who previously used Postman heavily at a prior company

**What you have vs what you don't:**
- You have: a Postman API key and workspaces you control, three OpenAPI specs representing services with different characteristics, and pre-built GitHub Actions with public documentation. Review the specs carefully -- they represent real-world documentation, complete with the kinds of inconsistencies you'd find in any enterprise environment.
- You don't have: access to the customer's AWS account, GitLab instance, internal repos, gateway configs, or CI/CD pipelines

**Your Mission:** Demonstrate how you'd onboard services into the API catalog with automated test coverage -- using pre-built tooling where it fits, and building where it doesn't -- and present a credible plan for scaling this across the pilot team's ~50 services and eventually org-wide.

## Your Assignment

This exercise has two parts: **working implementation** (against the Postman API -- no AWS account required) and **consulting design** (how you'd adapt the pattern for the customer's real infrastructure).

### Part 1: Working Implementation (Build It)

You're provided with:
- Three OpenAPI specs representing services with different characteristics (see [specs/](https://github.com/postman-cs/cse-exercise/tree/main/specs))
- Three pre-built Postman GitHub Actions with public READMEs (see "Provided Materials")
- An optional repo-local starter asset under `starter-assets/` that you may adapt or ignore

**What you need to do:**

1. **Build a working workflow** for a first service of your choice
   - Choose one of the provided specs, wire together a GitHub Actions workflow, and get it running end-to-end against a real Postman workspace you control
   - You may adapt the included starter asset or wire the immutable upstream actions directly
   - The result should be: workspace created, spec uploaded to Spec Hub, collections generated (baseline, smoke, contract), environments configured, collections exported to your repo

2. **Adaptation Analysis for a second spec** that has meaningfully different characteristics from the first
   - Produce a documented analysis of what would change in your workflow to handle the second service
   - Explain what stays the same, what changes, and what you would need from the customer's platform team to make it work

3. **README with:**
   - How you built the workflow and what decisions you made
   - What's universal in the onboarding pattern vs what changes per service
   - What the customer's ops/platform team would need to configure on their side (things you can't do without their AWS account, GitLab access, etc.)
   - Run instructions, validation notes, and trade-offs

**Non-negotiable requirement:**
- This must work end-to-end against a real Postman workspace using a real API key you control.
- No mocked API responses, fake outputs, placeholder screenshots, or pseudocode presented as a working solution.
- You should be able to explain what the workflow is doing, how the actions chain together, what would happen on rerun, and what you ran into while building it.

### Part 2: Consulting Design (Present It)

The working implementation proves you can execute. The presentation proves you can consult.

**30 minutes to present, 15 minutes live discussion. The session is 45 minutes total.**

The build is table stakes, but the real signal is in how you explain what you're proposing and how you handle questions you didn't prepare for. Expect the discussion portion to include scenario-based questions about operational realities -- token expiry, rerun behavior, scope management, customer coordination -- not just clarifications on your slides. We are not expecting you to debug the workflow live during the call; we are evaluating how well you explain the debugging and decisions that already happened during the build.

Assume your audience is made up of engineers and platform practitioners who work in these systems every day. Do not spend time narrating every low-level implementation detail. Focus on what the automation enables, what changes operationally for the team, what still requires human coordination, and why the pattern matters.

**Presentation (30 min). Required sections:**

1. **Problem and Value:** The current discovery/integration friction, the manual steps eliminated, and the estimated engineering time saved per execution. Focus on the operational value unlocked by the automation rather than deep financial ROI math. State your assumptions clearly.

2. **Solution Demo:** Demo the workflow live for your onboarded service using the working implementation you already built. You may run or rerun it if that helps your presentation, but the expectation is a working demo -- not live debugging. Show the workspace structure, collections, spec in Spec Hub, environments, and monitors.

3. **Adaptation and Scaling:** This is the consulting piece. The customer's environment isn't uniform. Explain:
   - What in the onboarding workflow stays the same across services and what changes in the inputs, environment configuration, and platform-team coordination required for a service with different infrastructure or CI/CD
   - How you scale from this pilot to the team's ~50 services in 60 days
   - Define your repeatable action template—which parts are fully repeatable per service vs what still depends on platform-team coordination, repo hygiene, or CI/CD access

4. **Handoff and Ownership:** What artifacts do you leave behind? What does the platform team need to maintain this independently? What would require ongoing CSE support vs what's self-serve after Day 90? What would a working session with their team look like?

**Discussion (15 min).** We'll ask scenario-based questions that test your operational awareness and consulting instinct. These may cover topics like what happens on Day 2 when the happy-path demo is over, how you handle requests that push beyond the engagement scope, or how you would reason about a workflow that's silently degrading based on what you already built and learned. Come prepared to think on your feet.

## Evaluation Criteria

Your submission is evaluated across four areas. Each area carries equal importance. The build is the entry ticket. The signal comes from how you explain, adapt, and respond to questions.

**Infrastructure Consulting** -- Can you diagnose a customer environment with mixed infrastructure, adapt tooling to different stacks, and clearly articulate what changes per service vs what's universal? Can you scope a realistic working session and identify what the customer needs to provide? Do you understand what happens on Day 2 -- what breaks when tokens expire, when workflows are rerun, when specs drift from implementation? Can you reason about operational failure modes (silent degradation, environment duplication, auth expiry) without being prompted?

**Value Articulation** -- Can you make the value concrete for an engineering and platform audience? Generic statements like "things are faster" or "improves consistency" don't land. The strongest candidates connect their automation to specific operational pain from the customer scenario -- the kind of pain that costs real engineering hours, causes real incidents (what even is an "incident"? Hint: they're rarely production issues in a large enterprise), or blocks real work. Lightweight assumptions and simple ROI framing can help, but operational value matters more than polished business math.

**Pattern Thinking & Scaling** -- Does your approach prove that the onboarding pattern is a repeatable system, not a series of custom projects? The strongest signal is when your second-spec analysis demonstrates how little actually changes. Beyond that: how do you handle the parts of the environment where GitHub Actions don't apply? Is your 90-day roadmap built on real dependency chains and parallelization opportunities, or is it just "repeat for each service"?

**Co-Execution & Handoff** -- How does the customer team participate and take ownership? What does enablement look like for their platform/ops team? Is independent operation by Day 90 credible based on your plan?

## Provided Materials

**OpenAPI Specifications:**

- [`payment-refund-api-openapi.yaml`](https://github.com/postman-cs/cse-exercise/blob/main/specs/payment-refund-api-openapi.yaml) -- Payment processing service (Lambda/API Gateway)
- [`loan-origination-api-openapi.yaml`](https://github.com/postman-cs/cse-exercise/blob/main/specs/loan-origination-api-openapi.yaml) -- Loan origination service (ECS/ALB)
- [`claims-processing-api-openapi.yaml`](https://github.com/postman-cs/cse-exercise/blob/main/specs/claims-processing-api-openapi.yaml) -- Claims processing service (mixed Lambda/ECS, GitLab CI deployments)

**Pre-Built GitHub Actions:**

These are production actions maintained by the CSE team. You may call them directly or adapt the optional starter asset in this repo.

- [postman-api-onboarding-action](https://github.com/postman-cs/postman-api-onboarding-action) -- End-to-end orchestrator that chains bootstrap + repo-sync
- [postman-bootstrap-action](https://github.com/postman-cs/postman-bootstrap-action) -- Creates workspace, uploads spec to Spec Hub, generates collections (baseline, smoke, contract), configures environments
- [postman-repo-sync-action](https://github.com/postman-cs/postman-repo-sync-action) -- Exports collections to repo, generates CI workflow, creates mock servers and monitors, links workspace to repo

**Practical Setup Notes:**

- You'll need a Postman Enterprise trial for this exercise (the actions use features not available on free plans). Here's a walkthrough: [How to Activate a Postman Enterprise Trial](https://www.loom.com/share/f7daa6e93845489fa2b7b55dfde95676)
- Generate your own Postman API key (PMAK) from your Postman account settings. Set it as a GitHub secret called `POSTMAN_API_KEY` on your repos.
- The pre-built actions also require a **Postman access token** for workspace linking, governance, and environment association features. To obtain it:
  1. Install the [Postman CLI](https://learning.postman.com/docs/postman-cli/postman-cli-overview/) if you haven't already.
  2. Run `postman login` and complete the browser-based authentication.
  3. Extract the token: `cat ~/.postman/postmanrc | jq -r '.login._profiles[].accessToken'`
  4. Set it as a GitHub secret called `POSTMAN_ACCESS_TOKEN` on your repos.
  > **Note:** This token is session-scoped and will expire. Without it, the actions still run but governance assignment, workspace linking, and system environment associations are silently skipped.
- The actions' `spec-url` input expects a URL the runner can fetch at build time. Since you're checking the OpenAPI spec files into your repo, use the raw GitHub URL (e.g., `https://raw.githubusercontent.com/<you>/<repo>/main/specs/payment-refund-api-openapi.yaml`).
- The actions persist state across runs using GitHub repository variables and may commit generated workflow files. The default `GITHUB_TOKEN` may not have sufficient permissions for these operations — review the action READMEs for guidance on authentication inputs.
- The repo includes optional starter material under `starter-assets/`. You may adapt it or ignore it, but be prepared to explain what you kept, what you changed, and why.
- You do NOT need an AWS account. The specs represent the customer's APIs but you're working against the Postman API, not deploying cloud infrastructure.
- Organize your repo or repos however you think best supports repeatable onboarding. Be prepared to justify the topology you chose.

**Documentation & References:**

- Learning Center: [https://learning.postman.com/](https://learning.postman.com/)
- Postman API authentication: [https://learning.postman.com/docs/developer/postman-api/authentication/](https://learning.postman.com/docs/developer/postman-api/authentication/)
- Spec Hub: [https://learning.postman.com/docs/designing-and-developing-your-api/managing-apis/](https://learning.postman.com/docs/designing-and-developing-your-api/managing-apis/)
- Postman CLI: [https://learning.postman.com/docs/postman-cli/postman-cli-overview/](https://learning.postman.com/docs/postman-cli/postman-cli-overview/)

## Submission Requirements

**Format:**

- Presentation deck (PDF or PPTX)
- One or more GitHub repos containing your working implementation for one service and adaptation analysis for a second service, with links to anything needed for review:
  - The service OpenAPI specs
  - The workflow or workflows you actually ran
  - Generated Postman collections (JSON exports)
  - Any additional configuration
- A README (in either repo or separate) with: how you built it, what's universal vs what changes per service, customer ops requirements, run instructions, validation notes, and how you used AI

**What we expect to see in the repo and demo:**

- A real end-to-end run against a real Postman workspace, not a theoretical design
- The pattern proven on 1 service and evaluated on a second service with meaningfully different characteristics, showing your approach transfers via documented analysis
- Clear explanation of how the actions chain together and why you structured the workflow the way you did
- Clear explanation of why you used, fixed, or bypassed the starter material
- Honest documentation of issues encountered and how you resolved them

**Clarifications:**

- Focus your working implementation on 1 of the 3 provided specs. Focus your adaptation analysis on a 2nd provided spec. The broader environment's variance (GitLab CI, services without specs, etc.) is a design exercise covered in your presentation.
- You don't need an AWS account. The provided specs represent the customer's APIs. You're onboarding them into Postman, not deploying them.
- You may start from the included starter asset or build directly from the action documentation. Either way, be prepared to explain why you chose that path and what you changed.
- Use whatever AI tools you want. Document what AI generated vs what you wrote or validated yourself.

**Time Guidance:**

- Phase 1 (review specs, build workflow, onboard first spec): ~30-45 minutes with AI assistance -- the starter material may not work as-is. Debugging is part of the build portion of the exercise. Take time to understand what you're onboarding before automating it, and what you're debugging before fixing it. If you're spending more than an hour here, step back and reassess your approach.
- Phase 2 (adaptation analysis + presentation + scenario prep): ~1.5 hours -- this is where the real evaluation weight lives. Invest here. Your ability to explain what you built, why it works, what breaks, and how you'd adapt it matters more than the build itself.
- Practice/refinement: ~0.5 hours

**AI Assistance Policy:** Strongly encouraged. We want your strategy, consulting instinct, and communication -- not manual YAML debugging. Use Claude Code, Cursor, Gemini CLI, or ChatGPT to accelerate.

## What We're Looking For

We want to see that you can take existing tooling, make it work in a realistic scenario, and present a credible plan for a customer engagement. The build should work; the presentation and discussion are where we learn how you think. Specifically:

- The workflow runs end-to-end against a real workspace for 1 service, and your adaptation analysis proves the pattern transfers to a second service with meaningfully different characteristics
- You can explain how you built the workflow, what decisions you made, and what you ran into
- You clearly articulate what's universal in the onboarding pattern vs what changes per service
- Your scaling plan distinguishes between what becomes repeatable per service and what still depends on customer/platform coordination
- You clearly communicate the practical value of the pattern to an engineering/platform audience, with lightweight assumptions where helpful, and explain why it matters beyond a one-time demo
- Your handoff plan addresses the platform/ops team specifically -- what do they need to maintain this without CSE involvement?
- You can think on your feet when asked about failure modes, operational edge cases, and scope management -- not just recite your slides or treat the discussion like a scripted status update
- You're honest about what AI generated vs what you wrote, validated, or changed yourself
