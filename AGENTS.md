# Agent instructions (Codex and other coding agents)

**Read `CLAUDE.md` in full before any change.** It is the authoritative guide to this repository:
layout, multi-tenancy, configuration and secrets, building, deployment, testing and the known
traps. Claude Code reads `CLAUDE.md`; Codex reads this file. Both agents follow the same rules.
When the two files disagree, `CLAUDE.md` wins, so fix the conflict there and keep this file short.

## Rules repeated from CLAUDE.md (never skip these)

- **Tracked config is production config with live credentials.** Never read
  `webConfigFiles/`, `WebhookWebAPI/appsettings*.json` or `IDS.CRM.FTPServiceMonitor/App.config`.
  Never add, copy or echo credentials, keys, tokens or connection strings.
- **Never read the tracked PDFs under `*/resources/temp/`.** They may contain customer data.
- **Encoding:** CRLF, and keep each file's existing BOM state. Never rewrite a whole file just to
  make a small change.
- Do not copy the risky settings `CLAUDE.md` lists (disabled TLS validation,
  `enableViewStateMac="false"`, `CommandTimeout = int.MaxValue`, empty `catch` blocks,
  `.Result` under the ASP.NET context, concatenated SQL).

## No-disruption rules (AI Agents and voice work)

Production runs many tenants on shared databases and libraries. Everything new must be **added
alongside** the existing CRM and must never change it.

| Area | Rule |
|---|---|
| Existing databases | No changes to existing tables, columns, stored procedures or indexes. New data goes in its own schema (`aiAgents.*`) or in S3. Migration scripts only add, are reviewed, and run on DEV2 → QA → staging before prod. Keep large data out of prod RDS. |
| DAL / EDMX / `IDS.*` packages | No EDMX edits and no `IDS.*` version bumps. There is one version per package id across the estate. New services do not reference the DAL. |
| Shared libraries | Do not modify Inline Libraries, DAL Libraries, `IDS.Site.Helper` or `IDS.Shared.Helper`. Put new code in new projects, or in new files inside the CRM Site. |
| Business logic | Reuse existing CRM operations (for example `InsertQuote`, the rating code, `SendPaymentLink`). No direct SQL writes into tenant databases. |
| Existing pages | Leave existing `.aspx`/`.ascx` pages and their code-behind unchanged. Call shared methods from new files instead of refactoring pages. |
| CRM technology | .NET Framework 4.8.1, C# 7.3, `packages.config`. Add no NuGet packages that could conflict with existing versions or binding redirects. |
| Config and secrets | No edits to `webConfigFiles/`. New settings go through `deploy/transforms`; secrets go in AWS Secrets Manager. |
| Switching on | Every new capability sits behind a per-tenant setting that is **off by default**, so turning the setting off is the rollback. |
| Production safety | Per-tenant rate limits, timeouts and circuit breakers on every call into the CRM, so AI traffic cannot slow down normal CRM users. |
| Payments | Voice agents never collect card or bank data and never call `SellQuote` or `SellQuote2`. Payments go through the CRM Payment Link. On the development CRM the agent may save a verified open quote, and staff complete checkout. The paper-check sale desk (`services/AI.Agents.GptLiveRuntime/crm-tools/sale.mjs`, approved by Pavan on 2026-11-03 behind `AI_AGENTS_CRM_SALES=paper-check`) stays off: the CRM marks a paper check processed and settled as soon as a quote is sold (finding of 2026-11-03), so a sale would record money not yet received. |

If a task seems to require breaking one of these rules, stop and ask Pavan. Do not work around it.

## Coordination with Claude Code

Claude Code and Codex work in the same repositories.

1. **Check before you change.** Run `git status`, read the recent commits and check open branches
   first. Never overwrite, revert or reformat work you did not make in this task.
2. **One task, one branch or worktree.** Never edit the same files another agent is changing.
   Branch names are ticket-keyed (`ICRM-…`).
3. **Never commit or push without the user asking.** The default branch for pull requests is `staging`.
4. **Leave a handoff.** Record status, decisions and open questions in `Docs/AI-Agents/` (current
   plan: `Docs/AI-Agents/gpt-live-voice-agent-enterprise-plan.md`), so the other agent starts from the same facts.
5. **Review each other's work.** Use the same checklist: the no-disruption rules above, tests pass,
   encoding preserved, no secrets, and existing behaviour unchanged.
6. **Same tests.** Run the suites that cover your change before calling it done:
   - `node --test` under `tests/` for the AI Agents services (for example `tests/gpt-live-trial`, `tests/livekit-local`);
   - the .NET check projects in `tests/AI.Agents.*`;
   - `dotnet test ICRM811.Scripting.Tests` for Scripting.

   Report failures with their output; do not hide them.

## Where the AI Agents work lives

| Path | What |
|---|---|
| `services/AI.Agents.GptLiveTrial` | GPT-Live-1 speech-to-speech trial (chosen voice path) |
| `services/AI.Agents.LiveKitLocal` | Cascaded ASR → LLM → TTS lab, plus the DEV2 CRM voice tools (`crm-tools.cjs`) |
| `services/AI.Agents.Management` | .NET 10 voice control plane (tenants, numbers, agent versions, usage, transcripts) |
| `services/AI.Agents.VoiceRuntime` | Earlier runtimes, telephony POCs and the DEV2 tool implementations |
| `Inline CRM Site/secure/aiagents` | Client-facing AI Agents pages in the CRM |
| `Docs/AI-Agents` | Plans, decisions and handoffs |
