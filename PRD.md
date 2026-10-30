# PRD: Repo Graph for Copilot — cross-repo graph, PR story, QA assist (2-day POC)

Status: draft, 2026-10-30. Owner: Deepak. Scope: 27 repos, one app.

## 1. Problem
- 27 repos form one application. Nobody holds the whole dependency picture.
- Copilot burns premium requests re-reading files to find callers, consumers and tests.
- PR reviewers cannot see what a change touches in other repos, so breakage is found late.
- QA writes and maintains tests without a map of what covers what.

## 2. Goals (POC, 2 days)
1. One merged code graph of all 27 repos that Copilot can query through MCP and a skill.
2. A "segment view": the changed area plus its 1-2 hop neighbours, rendered as a picture.
3. A PR comment that tells the change as a short story: what changed, what it affects, which repos, which tests.
4. A cross-repo warning when a change can break a consumer in another repo.
5. A graph lifecycle: main is the source of truth; local branches carry an overlay; merge refreshes main.
6. A QA skill: affected tests, coverage gaps, test-writing in the team's conventions.
7. Measurement: premium requests and file reads per task, with and without the graph.

## 3. Non-goals
- Training a model from scratch (infeasible on this data volume; Copilot cannot call a private model anyway).
- A new graph engine. Use Graphify as the base.
- Runtime/behavioural analysis beyond the contract edges in 5.3.
- Any export of source code, graphs or Confluence/Dynatrace content outside the bank's approved tooling.

## 4. Users
- Developers using Copilot day to day.
- PR reviewers.
- QA engineers.
- Tech leads checking cross-repo impact.

## 5. Solution

### 5.1 Graph build
- Per repo: `graphify extract . --code-only` (tree-sitter AST, no LLM, no cost).
- Combine: `graphify merge-graphs` over each repo's `graphify-out/graph.json`.
- Known risk: merge can collapse same-named nodes across repos into false edges (Graphify issue #1729). Mitigation: namespace every node ID with the repo name and run a collision report as part of the build; fail the build on unflagged collisions.

### 5.2 Graph lifecycle
| Stage | Where | What |
|---|---|---|
| Baseline | Central location, rebuilt by CI on every merge to main | Full merged graph, versioned by commit SHA |
| Local | Developer machine | Baseline plus an incremental `graphify update` overlay for the working branch |
| PR | CI on the pull request | Delta vs baseline: added/removed/changed nodes and edges |
| Merge | CI on main | Rebuild or union-merge (Graphify's `graph.json` git merge driver), publish new baseline |
- Parent (main) keeps the original; the branch graph never overwrites it.
- Previous versions are kept so a PR story can show before/after.

### 5.3 Contracts layer (the cross-repo warning)
AST alone sees imports, not the HTTP and package boundaries between repos. Add edges for:
- `consumes_api` / `exposes_api` from OpenAPI specs and client code.
- `depends_on_version` from `package.json` and lockfiles for shared libraries.
- `reads_config` for shared environment/config keys.
A change to an exposed API, a shared library export or a config key triggers a warning listing consumer repos, with the edge that links them.

### 5.4 Segment view and PR story
- Input: PR diff -> changed symbols -> `affected --depth 2` on the merged graph.
- Output rendered as Mermaid (GitHub renders it natively in PR comments; no image hosting).
- Comment sections: story (about 5 sentences), picture, affected repos and owners, affected tests, warnings.
- Delivered by a GitHub Action plus a local skill (`pr-blast-radius`) so a developer sees the same view before pushing.

### 5.5 Copilot integration
- Graph served by `graphify-mcp --graph <path>`; configured in `.vscode/mcp.json` (`servers`) and `.github/mcp.json` (`mcpServers`).
- Allow-list only the few tools used (explain, affected, path); MCP tool definitions cost tokens on every request.
- Short `.github/copilot-instructions.md` (about 5 lines: query the graph before grepping). Longer guidance lives in skills under `.github/skills/`.

### 5.6 QA assist
- Skill flow: changed code -> `affected` -> tests covering it by edge -> gaps -> propose or write tests.
- A "test conventions" card distilled from the 27 repos: framework, naming, mocks, fixtures, data builders. This replaces the idea of a from-scratch test model.
- Output: list of affected tests to run, list of uncovered changed symbols, drafted tests for the gaps.

### 5.7 Context cards (Confluence and Dynatrace)
- Use the existing MCP servers (Atlassian Rovo for Confluence, Dynatrace MCP) subject to security approval.
- Distil to short cards per service: owner, runbook link, known error signatures, key dashboards. Cards are small files in a skill, not live queries on every request.
- Verify Dynatrace MCP status (GA vs preview) for the bank's tenant before relying on it.

## 6. Model question
- From scratch: not recommended. Estimated $85k-$500k compute and millions of samples; vendor-blog numbers, order of magnitude only.
- Fine-tune (stretch): LoRA/QLoRA on a code-specialised ~7B model, tens to low hundreds of dollars per run, feasible on a single high-end GPU. Only after retrieval and skills show a ceiling.
- Token savings come from retrieval, narrow tool answers and short skills, not custom weights.

## 7. Plan

### Day 1: graph and PR story
1. Extract all 27 repos; merge; namespace; collision report.
2. `affected` wrapper and Mermaid segment renderer.
3. `pr-blast-radius` skill and PR-comment GitHub Action.
4. Replay 3 real past PRs; compare the affected list with what reviewers know.

### Day 2: contracts, QA, demo
1. OpenAPI and npm-version edges; cross-repo warning.
2. QA skill and test conventions card.
3. Copilot MCP allow-list install; instructions file.
4. Measure and demo.

## 8. Success metrics
- Affected-set precision/recall on the 3 replayed PRs, judged by the reviewer who knows the change.
- Cross-repo warning catches a known past breakage (or a seeded one) with no more than 1 false positive per PR.
- Premium requests and file reads per task, with vs without the graph, on 5 fixed tasks. Prior single-repo result: -24% fresh tokens, flat cost; expect larger gains only on large repos.
- Time to produce a PR story: under 2 minutes in CI.
- QA: share of changed symbols with a covering test identified automatically.

## 9. Risks
| Risk | Mitigation |
|---|---|
| Source or derived data leaves approved tooling | Everything runs inside the bank's environment; no personal accounts or external services; security sign-off before enabling Confluence/Dynatrace MCP |
| Graph merge creates false cross-repo edges | Namespacing, collision report, fail on unflagged collisions |
| AST misses runtime coupling | Contracts layer; state the limits in every warning |
| Small-repo token savings are flat | Pitch certainty and blast radius; claim token savings only where measured |
| Stale graph | CI rebuild on merge; show graph commit SHA in every answer |
| Tool names/features unverified | `get_pr_impact` MCP tool not confirmed in docs; check against the installed version |

## 10. Open questions
- Where does the central baseline live (artifact store, a graph repo, a shared drive)?
- Which CI system and GitHub org settings allow the PR Action?
- Are the OpenAPI specs in-repo or in a registry?
- Which credit pool is the "30k": Copilot premium requests, or something else?
- Who approves the Rovo and Dynatrace MCP servers?

## 11. References
- Graphify: https://github.com/Graphify-Labs/graphify (merge-graphs in v0.5.0, git merge driver in v0.7.0, collision issue #1729)
- Cross-repo impact tools (inspiration only): Riftmap, Synopsis, Codegraph
- GitHub Copilot customization: https://docs.github.com/en/copilot/how-tos/copilot-in-your-ide/customize-copilot
- Atlassian Rovo MCP: https://www.atlassian.com/blog/announcements/atlassian-rovo-mcp-ga
- Dynatrace MCP for Rovo: https://www.dynatrace.com/news/blog/dynatrace-mcp-server-for-atlassian-rovo-investigate-production-problems-without-leaving-jira-or-jsm/
