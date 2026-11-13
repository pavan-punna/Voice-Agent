# Shared local work — current working folder

Dashboard: http://127.0.0.1:59530/

Pavan requested one common implementation. From November 4, 2026, both agents make new source changes in **`C:\dev\Inline-AI-Agents-Shared`**, branch `ICRM-ai-agents-shared-20261104`. Read `AGENTS.md` and `CLAUDE.md` before editing. The older Codex and Claude worktrees are preserved backups; do not continue features there or copy them over the newer shared files.

Use `C:\dev\Inline-AI-Coordination-Codex\tools\AgentCoordination\cli.mjs` for status, claims, progress and handoff. Existing logical path ownership still applies in the common folder: one writer per file, with an explicit handoff before editing another agent's scope. Log work before starting and post results and review requests when finished. Agent identity is self-reported. Never share account credentials.

The running coordinator still uses its original three-workspace configuration. Its Git cards monitor the older folders, not this common working tree. A `codex` or `claude` workspace label currently identifies the claim owner; include this actual shared path in task notes. Check Git state in this folder directly. Do not remove locks, release another agent's stale claims, restart the coordinator or change its 60-second scan interval to hide that limitation.

Full guide: `C:\dev\Inline-AI-Coordination-Codex\Docs\AI-Agents\local-agent-coordination-2026-10-28.md`

The original checkout and its voice trial stay at `C:\dev\Inline-AI-Agents-QA-Review`. Keep its unfinished work safe. This common folder contains the reviewed merged source, not provider credentials, recordings or runtime databases. Private local data remains in its existing location under the current user's LocalAppData.

Current task: align the existing 61030 studio and 61031 phone center through one management profile and call-history store. Claude owns management/runtime/local launcher; Codex owns the phone-center adapter and dashboard UI under their existing board reservations. The phone worker is not restarted. Do not commit, push, deploy, change production or restart 59520 without the user's instruction. Current CRM is `dev.inlinecrm.com`; DEV2 is retired. DEV open quotes and staff checkout only: no payment settlement or policy issuance.
