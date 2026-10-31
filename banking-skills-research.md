# Banking skills for Claude Code: search findings and gaps

## What the search found
- Public banking/fintech skills exist, but their quality is unverified. The main ones are pci-compliance and banking-expert from the wshobson/agents collection, banking-expert (ACH, SWIFT, SEPA, KYC, AML, Basel), and managing-payment-compliance (PCI, MTL, BSA). The only SOX skill I found was a finance-sox-testing listing with almost no detail. I found no official Anthropic PCI or SOX plugin.
- Treat community skills as reference material, not as installs. Third-party skills are a supply-chain risk in a bank. Read one, borrow its checklist, and write your own version.
- Token cost comes mostly from re-sent context. Several guides say a bloated CLAUDE.md reloads every turn. They also say skills load on demand, and that subagents only pay off for heavy reading (vantaige). The "caveman" skill claims 65% savings, but only on prose output, not code. The percentages in these posts are author claims.

## What you already have
- Review and security: owasp-security, security-scan, review-code, code-reviewer
- Debugging: systematic-debugging
- Delivery: build-feature, ship-it, canary, benchmark, generate-prp, execute-prp, ralph, grill-me
- Context and memory: context-check, memory-search, recall
- Your own Chiron project: a self-healing bank SDLC with evidence-gated agents and a compiled org/domain/repo harness. That is your strongest unicorn asset, and no public skill matches it.

## Gaps to fill, in priority order
1. bank-compliance-gate (custom). One skill that runs a PCI-DSS, SOX, GLBA, and data-residency checklist on every diff and outputs an evidence receipt. It extends your build-evidence skill, and an auditor can read the receipts.
2. payments-domain (custom). Idempotency keys, double-entry ledgers, reconciliation, ISO 20022 and ACH formats, and money-as-integers rules. Write it as a lean reference file that loads on demand. Borrow structure from banking-expert.
3. threat-model (custom, STRIDE per feature). Run it before code, in the generate-prp step.
4. audit-trail and model-risk doc generators. Model-risk documentation under SR 11-7 or OSFI E-23 is what actually blocks AI in banks. A skill that drafts it automatically would be a real differentiator.
5. A Postgres/Supabase migration reviewer for row-level security, PII columns, your stack.

## Token and speed changes
- Keep ~/.claude/CLAUDE.md under about 80 lines. Yours is already close, but the long model-routing and API-code sections could become skills that load on demand.
- Run /context-check regularly. Set MCP servers you rarely use (rea, Higgsfield) with mcp-toggle when doing banking work.
- Send exploration to Haiku subagents and keep implementation in the main thread. Use Sonnet for build work, and Opus only for threat models and architecture, as your routing rules already say.

If you want, I can write the first two skills (bank-compliance-gate and payments-domain) on your Zod/Supabase/Next.js stack and tie them into ship-it so the gate runs automatically.

## Sources
- pci-compliance (wshobson/agents)
- banking-expert
- managing-payment-compliance
- finance-sox-testing
- 9 Ways to Cut Token Consumption in Claude Code
- Claude Code Subagents That Save Context
- Caveman Skill and Proxy
