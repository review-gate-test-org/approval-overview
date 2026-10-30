# approval-overview

This repo compares three ways to let an AI review approve pull requests. Each way has its own demo repo. This repo has no code, only this comparison.

**The rule we want:** a PR can merge when its code owners approve it, **or** when Copilot code review approves it (🟢) and path rules allow an agent to approve every changed file.

Each demo repo ran the same three PRs:

- **A**: backend-only, harmless. Expected: mergeable with no human.
- **B**: backend-only, with a bug. Expected: blocked.
- **C**: backend plus `README.md`. Expected: blocked even if Copilot approves.

Status key: ✅ **verified**, which means checked through the GitHub API on 2026-10-29. ⚠️ **unverified**, which means taken from docs, a repo README, or code, and not observed directly. Copilot settings that exist only in the UI cannot be read through the API. For those, this page states only what the API behaviour shows.

## Comparison

| | [approval-builtin-copilot](https://github.com/review-gate-test-org/approval-builtin-copilot) | [approval-review-gate](https://github.com/review-gate-test-org/approval-review-gate) | [approval-machine-user](https://github.com/review-gate-test-org/approval-machine-user) |
|---|---|---|---|
| **Mechanism** | GitHub's built-in Copilot approvals (GA 2026-10-28). Copilot's approval is the required approval. | A GitHub App posts a required `review-gate` status. Custom code ([`gate.js`](https://github.com/review-gate-test-org/approval-review-gate/blob/main/.github/review-gate/gate.js)) applies `CODEOWNERS` + [`AGENTOWNERS`](https://github.com/review-gate-test-org/approval-review-gate/blob/main/.github/AGENTOWNERS). | The bot account `venetrius-review-bot` approves once Copilot has approved. GitHub's own "code-owner review required" rule decides. |
| **Ruleset** ✅ | [26113860](https://github.com/review-gate-test-org/approval-builtin-copilot/rules/26113860): 1 approval, **code-owner review off**, checks `backend`/`client` | [26114641](https://github.com/review-gate-test-org/approval-review-gate/rules/26114641): 1 approval, **code-owner review off**, checks `backend`/`client`/`review-gate` (pinned to App 5441748) | [26113907](https://github.com/review-gate-test-org/approval-machine-user/rules/26113907): 1 approval, **code-owner review on**, checks `backend`/`client` |
| **Other setup** | UI-only settings (see the next row). Nothing else. | GitHub App and its private key, environment `review-gate`, two workflows, `AGENTOWNERS` | Bot account with an org seat and Write access, fine-grained PAT in environment `machine-user-approval`, workflow + [`approve.js`](https://github.com/review-gate-test-org/approval-machine-user/blob/main/.github/machine-user-approval/approve.js), bot listed in `CODEOWNERS` for `/backend/` only |
| **"Allow Copilot to approve pull requests"** (UI only) | On ✅. Copilot's reviews are `APPROVED`. | On ✅. Before it was turned on, Copilot posted 🟢 only as `COMMENTED`. | On ✅. Before it was turned on, Copilot posted 🟢 only as `COMMENTED`. |
| **"Count Copilot approvals toward merge requirements"** | On ✅ (from behaviour: Copilot's approval alone gives `reviewDecision=APPROVED` on A). **This is the whole mechanism.** | On ✅ (same evidence, on A and C). **It matters here:** without it, A would also need a human approval. C counted too, which means the Copilot path limit here is empty or covers `README.md` (⚠️ the UI value was not checked). So `review-gate` is the only path control. | ⚠️ Unknown. The code-owner rule hides it. **It does not matter today:** Copilot cannot satisfy code-owner review, and the bot's approval satisfies both rules. It would matter if code-owner review were turned off. |
| **A**: backend, harmless | [#1](https://github.com/review-gate-test-org/approval-builtin-copilot/pull/1) 🟢 → **mergeable**, no human ✅ | [#1](https://github.com/review-gate-test-org/approval-review-gate/pull/1) 🟢 → **mergeable**, no human ✅. But the green status came from a **manual dispatch**. | [#2](https://github.com/review-gate-test-org/approval-machine-user/pull/2) 🟢 → bot approved 29 s later → **mergeable** ✅ |
| **B**: backend, bug | [#2](https://github.com/review-gate-test-org/approval-builtin-copilot/pull/2) 🟡 → **blocked** ✅ | [#2](https://github.com/review-gate-test-org/approval-review-gate/pull/2) 🟡 then 🔵 → **blocked** ✅ | [#3](https://github.com/review-gate-test-org/approval-machine-user/pull/3) 🟡 then 🔵 → **blocked** ✅ |
| **C**: backend + README | [#3](https://github.com/review-gate-test-org/approval-builtin-copilot/pull/3) 🟢 → **blocked** (Copilot's approval does not count) ✅ | [#3](https://github.com/review-gate-test-org/approval-review-gate/pull/3) 🟢 → **blocked** (gate pending). It also now has a merge conflict ✅ | [#4](https://github.com/review-gate-test-org/approval-machine-user/pull/4) 🟢 → **blocked** (no code-owner approval) ✅ |
| **Human click needed for the automatic path?** | No ✅ | **Yes** ✅. All 7 relay runs triggered by Copilot reviews were held as `action_required` ("Approve workflows to run"). | No ✅. 0 held runs. There is no review trigger: the push-triggered run waits up to 15 min for Copilot. |
| **Path scoping** | One list of globs for the whole repo, under Copilot settings (`backend/**`). Every changed file must match. ✅ **An empty list means all paths** (C was briefly mergeable). The scope is checked at merge time ✅. | `AGENTOWNERS` per path (`/backend/ copilot`), read from `main`, plus a hard-coded deny list in `gate.js` | `CODEOWNERS` (bot owns `/backend/` only), read from `main`, plus a hard-coded deny list in `approve.js` |
| **Prompt-injection exposure** (see below) | Protected **only by the path scope**. Every file location that flipped Copilot's verdict in the lab is outside `backend/**`. ⚠️ No injection PR was run in this repo, and an empty scope removes the protection. | Protected by the deny list (the [lab PR #17](https://github.com/review-gate-test-org/copilot-review-harness-playground/pull/17) list) and by `AGENTOWNERS`. ✅ Code read, 28/28 unit tests pass. ⚠️ No injection PR was run in this repo. | Protected by the deny list (the PR #17 list plus `README.md`) and by `CODEOWNERS`. ✅ Code read, 10/10 unit tests pass. ⚠️ No injection PR was run in this repo. |
| **Secrets kept outside GitHub** | None | The App private key. Anyone holding it can post a green `review-gate` on every repo where the App is installed. | The bot account's credentials (including 2FA) and a long-lived PAT (30-day expiry, rotated by hand) |
| **Strengths / best when** | No custom code, workflows or secrets: only settings ✅. No human click and no held runs ✅. Supported by GitHub (GA 2026-10-28, documented) ⚠️ docs. **Best when** a repo can drop code-owner review and one repo-wide path list is enough. | Code owners stay the human path (enforced by the gate, not by the ruleset): C stays pending with *Needs code owner approval* ✅. Per-path agent policy in `AGENTOWNERS`, read from `main` ✅ code read. The status description says why a PR is blocked ✅. Built for more than one agent, but only `copilot` is configured ⚠️. Needs no user account or seat. **Best when** per-path agent rules are needed and a trigger that is not held (for example a webhook service) can be run. | GitHub's own code-owner rule stays on ✅. The bot's approval counts only where `CODEOWNERS` lists it (C blocked) ✅. No held runs ✅. Each bot approval is an ordinary review that names the commit Copilot approved and the scope ✅. Re-approved about 80 s after a new push ✅. **Best when** code-owner review must stay on and the org accepts a machine account. |
| **Known weaknesses** | Code-owner review must be off for the whole branch. One path list for the whole repo, with no per-team policy. Copilot is the only possible approver. | Held runs (above). The push-triggered run crashed on an HTTP 503 ✅. The 15-min schedule ran only 4 times in about 20 h, the first about 6.5 h after the workflow landed ✅. Custom code re-implements CODEOWNERS. The PR page no longer says "code owner review required" ⚠️. | Needs a seat for a non-human user: it worked on this free org ✅, but seat cost and policy in a paid or enterprise org are not checked ⚠️. The 15-min schedule fallback ran only 4 times in about 20 h, the first about 6.5 h after the workflow landed ✅. Self-approval protection is covered only by unit tests ⚠️. Each waiting run occupies a runner. |

All three rulesets have no bypass actors, dismiss stale approvals on push, and request Copilot review on every push ✅.

A fourth option was considered and not built: a **merge bot** that may bypass the rules and merges when Copilot and the path rules agree. It was dropped because the bot could merge anything, and the result is a merge, not an approval.

The PRs in `approval-review-gate` have different content from the other two repos: there, B makes `list()` return the backing list, and C adds `contains()` plus a README change. The outcomes have the same shape.

## Prompt injection: instruction files on the PR branch

Copilot code review reads its instruction files from the **PR's head branch**. In the lab, adding one to a buggy PR flipped Copilot to 🟢 **4 out of 4 times** for each of these files: `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md`, and root `AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `REVIEW.md`. Nested copies, a comment in code, and the PR description did not flip it, but only one instruction wording was tried. See [lab issue #97](https://github.com/review-gate-test-org/copilot-review-harness-playground/issues/97).

None of the three paths stops Copilot from being fooled. What they do is keep a fooled 🟢 from counting when the PR touches those files:

- **builtin-copilot**: a file outside `backend/**` makes Copilot's approval not count. The same mechanism blocked C ✅. Nothing in the repo enforces this. It depends on a UI field that fails open when empty.
- **review-gate** and **machine-user**: a hard-coded deny list (`.github/`, `CODEOWNERS`, the root instruction files, root `.claude/skills/` and `.agents/skills/`), plus path rules read from `main`.
- **The lab repo itself is not protected for root files.** The deny-list fix, [PR #17](https://github.com/review-gate-test-org/copilot-review-harness-playground/pull/17), is **still unmerged** ✅. The lab gate is covered only because its `AGENTOWNERS` happens to exclude the repo root.

Not covered by any of the three: injection through ordinary code or the PR description (not ruled out). Other wordings, upper/lower-case file names, and `.github/agents` and `.github/prompts` are untested. The two `.github/` folders are covered by all three paths anyway.

## What is solved

- In all three repos, A was mergeable without a human, while B (bug) and C (a path outside the scope) stayed blocked ✅.
- In builtin-copilot and machine-user, the whole flow ran with no human click ✅.
- A new push removes the AI approval within seconds. Copilot then re-reviews: in builtin-copilot it re-approved about 75 s after the push ([#4](https://github.com/review-gate-test-org/approval-builtin-copilot/pull/4)), and in machine-user the bot re-approved about 80 s after the push ([#5](https://github.com/review-gate-test-org/approval-machine-user/pull/5)) ✅.
- In review-gate and machine-user, the path rules are read from `main`, so a PR cannot loosen its own rules ✅ (code read).

## What is not solved yet

- **Code owners + Copilot together** (builtin-copilot). The built-in setting cannot satisfy code-owner review, so code-owner review has to be turned off for the whole branch ✅.
- **Held workflow runs** (review-gate). Runs triggered by a Copilot review need a human to approve them ✅. A green result needed a manual dispatch.
- **Reliability**. The 15-min schedule fallbacks fired only 4 times in about 20 h in each repo, with gaps of 3.5 to 7 h ✅, so a late Copilot review can wait hours. Transient 503 errors crashed runs ✅. machine-user now retries API calls 3 times ([PR #6](https://github.com/review-gate-test-org/approval-machine-user/pull/6), merged 2026-10-30 ✅), but this has not been tested against a real outage ⚠️. review-gate has no retries.
- **Self-approval** by the bot (machine-user). It is blocked in code and in unit tests only ⚠️.
- **Fine-grained allow/block lists** (machine-user). Tracked in [issue #1](https://github.com/review-gate-test-org/approval-machine-user/issues/1).
- **Determinism**. Copilot gave B 🟡 and then 🔵 on the same kind of change ✅. Each scenario was run only once or twice.
- **Pushing without review**. A backend-only PR can keep changing and stay mergeable, with Copilot alone approving each push ✅.

## What is needed for production

1. **A decision on code-owner review.** builtin-copilot works only with code-owner review off. The other two keep code owners as the human path, but we have to run them ourselves.
2. **Ownership of the secret** for the custom paths: the App key (review-gate) or the bot's credentials and PAT (machine-user). This includes custody, rotation, and audit.
3. **For review-gate**: a trigger that does not get held as "Approve workflows to run" (for example, a webhook service in place of an Actions relay ⚠️ untested), plus error handling for API errors.
4. **For machine-user**: org and terms-of-service approval for a non-human account with a seat in a paid or enterprise org ⚠️, and a fallback for late Copilot reviews that runs reliably (the schedule does not).
5. **Injection defence**: a deny list (or a path scope) that excludes instruction files on every path, and lab [PR #17](https://github.com/review-gate-test-org/copilot-review-harness-playground/pull/17) merged. Also a policy for the vectors no list can cover (code and PR description).
6. **Locking the UI-only settings.** "Allow Copilot to approve" and the path limit have no API to read or set them. An emptied path limit silently allows every path ✅. Something must check these settings regularly.

## Unverified items

- The UI values of the Copilot settings, which have no API. Above, they are inferred from behaviour.
- Whether "Allow Copilot to approve" and "count toward merge requirements" are one setting or two. machine-user's [README on `main`](https://github.com/review-gate-test-org/approval-machine-user#how-its-configured) and its setup guide (both since PR #6) describe one setting that does both, while builtin-copilot's README lists two separate settings. The UI was not re-checked.
- That injection PRs are blocked in these three repos. This is inferred from code, unit tests, and the lab experiment; no injection PR was opened here.
- Bot self-approval, machine-user seat and policy in a paid or enterprise org, and the 15-glob limit (from the docs).

## History

The lab repo is [copilot-review-harness-playground](https://github.com/review-gate-test-org/copilot-review-harness-playground). It contains the draft options document [#16](https://github.com/review-gate-test-org/copilot-review-harness-playground/pull/16), the injection experiment [#97](https://github.com/review-gate-test-org/copilot-review-harness-playground/issues/97), and the deny-list fix [#17](https://github.com/review-gate-test-org/copilot-review-harness-playground/pull/17).
