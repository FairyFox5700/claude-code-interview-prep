# Claude Code & AI-Assisted Developer Workflows — Interview Q&A

LinkedIn-style flashcards: **Q** → **A** → *why it's right*. Three objectives, 15+ pairs each, covering Claude Code config, CLAUDE.md, Skills, Subagents, MCP, Permissions, Hooks, Sandboxing, team config, CI/CD, and debugging.

---

## Objective 1: Configure Claude tools and environments for teams, including Claude Code

**Q1. Where does Claude Code look for project-level configuration, in order of precedence?**
A: Enterprise managed policy → command-line flags/env vars → local project settings (`.claude/settings.local.json`) → shared project settings (`.claude/settings.json`) → user settings (`~/.claude/settings.json`).
*Why: Claude Code merges settings with enterprise policy always winning, so teams can enforce guardrails that individual devs can't override, while still letting devs customize locally.*

**Q2. What is CLAUDE.md and where can it live?**
A: A plain-markdown file of project instructions/context that Claude Code automatically loads. It can live at the repo root, in subdirectories (loaded when Claude works in that folder), or in `~/.claude/CLAUDE.md` for user-global instructions.
*Why: It gives the model durable, versioned context (build commands, conventions, architecture notes) without re-explaining every session, and nested files let monorepos scope instructions per package.*

**Q3. How do you share a CLAUDE.md across a whole team?**
A: Commit it to the repo (e.g., root `CLAUDE.md`) so it's version-controlled and reviewed like code; everyone on the team gets the same instructions via git.
*Why: Treating it as code means PR review catches bad instructions, and history shows why conventions changed.*

**Q4. What's the difference between `.claude/settings.json` and `.claude/settings.local.json`?**
A: `settings.json` is committed and shared with the team; `settings.local.json` is gitignored and holds personal overrides (e.g., a dev's own permission preferences or API keys).
*Why: Separates team-wide policy from individual preference, preventing personal config from leaking into the shared repo.*

**Q5. How do you configure a team-wide "managed settings" policy that users cannot override?**
A: Use the enterprise managed policy file (deployed via MDM/system config, e.g. `/Library/Application Support/ClaudeCode/managed-settings.json` on macOS or equivalent on Linux/Windows), which sits above all other settings layers.
*Why: Centralized IT/security teams need a non-bypassable layer to enforce things like disabled tools or mandatory permission modes across the org.*

**Q6. What environment variables commonly configure Claude Code behavior in CI or team setups?**
A: Examples include `ANTHROPIC_API_KEY`, `CLAUDE_CODE_USE_BEDROCK`/`CLAUDE_CODE_USE_VERTEX` (for cloud routing), and model override vars.
*Why: Teams often route through a cloud provider's enterprise billing/compliance (Bedrock/Vertex) rather than direct API keys, and env vars make this swappable per environment without code changes.*

**Q7. How do you give Claude Code consistent "house style" across every repo an engineer touches?**
A: Put general preferences in the user-level `~/.claude/CLAUDE.md`, and repo-specific conventions in each project's `CLAUDE.md`; both get combined into context.
*Why: Separates "how I like to work" (global) from "how this project works" (local), avoiding duplication across repos.*

**Q8. What's a practical way to bootstrap CLAUDE.md for an existing codebase?**
A: Run `/init` in Claude Code — it scans the repo and generates a draft CLAUDE.md with build/test commands and structure, which you then review and refine.
*Why: Saves time versus writing from scratch, and forces a human review pass so the generated instructions are accurate, not hallucinated.*

**Q9. How should secrets (API keys, tokens) be handled in team Claude Code configs?**
A: Never commit them in `settings.json`/CLAUDE.md; use environment variables, `.env` files excluded via `.gitignore`, or a secrets manager referenced indirectly.
*Why: Config files are often committed and reviewed by the whole team/CI; secrets belong in a layer with proper access control and rotation.*

**Q10. What is the purpose of the `/permissions` and `/config` slash commands inside Claude Code?**
A: `/config` lets you view/edit settings interactively; permission-related commands/UI let you review and adjust which tools Claude can use and under what approval mode.
*Why: Gives an in-session, discoverable way to audit/change config without hand-editing JSON files.*

**Q11. How do you onboard a new team member to a consistent Claude Code setup quickly?**
A: Commit `.claude/settings.json`, `CLAUDE.md`, shared `.claude/skills/`, and `.mcp.json` to the repo; the new dev just clones and runs Claude Code — most of the environment is already defined.
*Why: Config-as-code means onboarding is "clone repo" rather than manually replicating a teammate's local setup.*

**Q12. What's the risk of letting every engineer maintain their own divergent CLAUDE.md per project?**
A: Inconsistent conventions, duplicated/contradictory instructions, and Claude giving different quality output per developer.
*Why: A shared, reviewed CLAUDE.md in version control is the single source of truth; personal drift undermines reproducibility.*

**Q13. How can you scope different instructions to different parts of a monorepo?**
A: Place a `CLAUDE.md` in each subproject directory; Claude Code loads the nearest one(s) based on working directory/context, layering with the root file.
*Why: A monorepo's frontend and backend may need very different build/test instructions — nesting avoids one bloated root file.*

**Q14. What should a well-written CLAUDE.md typically include?**
A: Build/test/lint commands, code style conventions, architecture overview, "do not touch" areas, and any repo-specific gotchas.
*Why: These are exactly the things a new engineer would need explained once — codifying them lets Claude act like an onboarded teammate.*

**Q15. How do you keep CLAUDE.md from becoming stale?**
A: Review/update it as part of normal PR review when conventions change, and periodically prune outdated instructions (treat it like code, not a one-time doc).
*Why: Stale instructions actively mislead the model, which is worse than having none.*

**Q16. What configuration enables Claude Code to use a specific model version for a team by default?**
A: Set the model field/env var (e.g. `ANTHROPIC_MODEL` or a `model` key in settings.json) so everyone defaults to the approved model version.
*Why: Ensures reproducible behavior and cost control instead of each dev picking arbitrary models.*

---

## Objective 2: Improve developer workflows using AI-assisted tooling

**Q1. What is a "Skill" in Claude Code?**
A: A reusable, packaged capability — a folder with a `SKILL.md` (instructions/metadata) plus optional scripts/templates/references — that Claude loads on demand when relevant to the task.
*Why: Skills turn one-off prompt engineering into durable, shareable, version-controlled procedures instead of re-explaining a workflow every session.*

**Q2. How are Skills discovered and loaded at runtime?**
A: Claude Code scans available skills' short descriptions/triggers; when a task matches, it loads the full `SKILL.md` content (and linked files on demand) rather than preloading everything.
*Why: Keeps context usage low — only relevant procedural knowledge enters the context window when needed.*

**Q3. What's the difference between a Skill and a Subagent?**
A: A Skill is a packaged "how-to" (instructions + resources) the main agent follows in its own context; a Subagent is a separate agent instance with its own isolated context/conversation that the main agent delegates a task to and gets a final result back from.
*Why: Skills add knowledge/procedure to the current agent; subagents add parallelism and context isolation for complex or independent sub-tasks.*

**Q4. When should you use a Subagent instead of just continuing in the main context?**
A: For reasoning-heavy work, tasks that would flood the main context with noise, or independent parallel workstreams (e.g., spin up 3 subagents to investigate 3 unrelated bugs at once).
*Why: Subagents keep the primary conversation's context clean and let independent work run concurrently rather than serially.*

**Q5. What is MCP and why does it matter for developer tooling?**
A: Model Context Protocol — an open standard letting Claude Code connect to external tools/data sources (databases, APIs, internal services) via a standardized server interface declared in `.mcp.json`.
*Why: Instead of bespoke integrations per tool, MCP gives one protocol so any compliant server (GitHub, Slack, a company's internal API) plugs into Claude Code the same way.*

**Q6. How do you configure an MCP server for a team project?**
A: Add it to a project-level `.mcp.json` (committed to the repo) specifying the server command/args/env; team members get the same tool connections when they open the project.
*Why: Makes tool access reproducible and reviewable, instead of each developer manually wiring up local MCP servers.*

**Q7. What are the three MCP server scopes in Claude Code and how do they differ?**
A: Local (per-user, per-project, not shared), Project (`.mcp.json`, committed, shared with the team), and User/Global (applies across all of a user's projects).
*Why: Lets you choose the right sharing boundary — e.g., a personal API key integration stays local, while a shared Jira MCP server goes in the project scope.*

**Q8. How can Claude Code help automate repetitive refactors across a codebase?**
A: Write/invoke a Skill or give explicit instructions describing the pattern, let Claude search the codebase, propose and apply changes, and review via diff before committing — optionally parallelized with subagents per module.
*Why: AI-assisted tooling shines at mechanical-but-context-sensitive changes (e.g., renaming an API across 200 files) faster than manual find/replace while still needing human review.*

**Q9. What is a practical workflow for using Claude Code in code review assistance?**
A: Have Claude run `git diff`/`gh pr view`, summarize the change, flag risks (security, performance, missed tests), and optionally leave review comments — a human still approves/merges.
*Why: Speeds up review triage without removing the human-in-the-loop decision, which is essential for trust and accountability.*

**Q10. How do hooks improve developer workflow automation?**
A: Hooks are user-defined shell commands that fire automatically on specific Claude Code lifecycle events (e.g., before/after a tool call, on file edit, on session start), letting you enforce linting, formatting, or notifications without manual intervention.
*Why: Automates guardrails (e.g., auto-run `prettier` after every file write) so quality checks happen deterministically regardless of what the model "remembers" to do.*

**Q11. Give an example of a useful PostToolUse hook.**
A: After every `Edit`/`Write` tool call, automatically run the project's linter/formatter and auto-fix or surface errors back to Claude.
*Why: Keeps code style consistent and catches syntax errors immediately, closing the loop faster than waiting for CI.*

**Q12. How can teams standardize "slash commands" for common workflows?**
A: Define custom slash commands as markdown files in `.claude/commands/` (project-scoped) describing a reusable prompt/workflow (e.g., `/deploy-staging`), committed so the whole team can invoke them identically.
*Why: Turns tribal knowledge ("the steps to cut a release") into a one-command, reviewable, shareable action.*

**Q13. What role does context window management play in effective AI-assisted workflows?**
A: Use Skills/subagents/`/compact` and selective file reads instead of dumping entire large files or histories into context, keeping the model focused and reducing cost/latency.
*Why: Irrelevant or excessive context dilutes the model's attention and increases the chance of it losing track of the actual task.*

**Q14. How should a team decide what becomes a Skill vs. inline CLAUDE.md instructions?**
A: Put short, always-relevant facts (build commands, conventions) in CLAUDE.md; put larger, situational, or infrequently-needed procedures (a complex deployment runbook, a niche API's auth flow) in a Skill that loads only when relevant.
*Why: CLAUDE.md loads every session (cost to always include), while Skills load on demand — matching storage location to usage frequency keeps context lean.*

**Q15. How can AI-assisted tooling be integrated into pair/mob programming sessions?**
A: Use Claude Code live to draft/explain changes while a human drives review and decisions, with the AI acting as an accelerant for boilerplate, test-writing, and exploring alternatives.
*Why: Keeps human judgment central for design decisions while offloading mechanical typing/lookup work, improving throughput without sacrificing ownership.*

**Q16. What's a good practice for keeping prompts/workflows reusable across the team rather than siloed in one dev's head?**
A: Externalize them as Skills, slash commands, or CLAUDE.md sections committed to the repo instead of ad hoc chat prompts that only live in one person's session history.
*Why: Institutional knowledge about "how we use AI here" should be as durable and shared as the codebase itself.*

---

## Objective 3: Support debugging and operational issue resolution

**Q1. What are "permission modes" in Claude Code and why do they matter for safe debugging?**
A: Modes like `default` (ask before risky actions), `acceptEdits` (auto-approve file edits), `plan` (read-only planning), and `bypassPermissions` (no prompts) control how much autonomy Claude has before it touches your system.
*Why: When debugging production issues, you want tight control (ask before destructive commands) but can loosen it for low-risk read-only investigation — matching autonomy to blast radius.*

**Q2. How do you configure fine-grained tool permissions (e.g., allow `git diff` but require approval for `git push`)?**
A: Define `allow`/`deny`/`ask` rules per-tool (and per-command pattern, e.g. `Bash(git push:*)`) in `.claude/settings.json`.
*Why: Lets you safely give broad read/investigate access while gating anything that mutates state or deploys, which is critical during incident response.*

**Q3. What is sandboxing in the context of Claude Code, and why use it during debugging?**
A: Running Claude Code's shell/file tool execution inside an isolated environment (container, VM, or OS-level sandbox) so commands it runs can't affect the host system or production directly.
*Why: During debugging, you often let the agent run exploratory or even risky commands (to reproduce a bug) — sandboxing contains any accidental damage (e.g., `rm -rf`, hitting prod APIs).*

**Q4. How should Claude Code be permissioned differently in a CI pipeline vs. an interactive dev session?**
A: In CI, run with minimal/explicit allowed tools and no interactive approval (since no human is present to click "approve") — often a restricted allowlist plus sandboxing; interactively, a dev can use `default`/ask mode with broader tool access.
*Why: CI has no human to catch a bad action in real time, so it needs stricter upfront guardrails than a supervised session.*

**Q5. What's a systematic way to have Claude Code help root-cause a production bug (vs. guessing)?**
A: Have it gather evidence first — reproduce the issue, read logs/stack traces, check recent diffs/commits, form a hypothesis, test the hypothesis narrowly — before proposing a fix.
*Why: Mirrors real debugging discipline (akin to the "4-phase" understand-before-fixing approach); jumping straight to a fix risks patching symptoms, not causes.*

**Q6. How can hooks help catch problems during an agent-assisted debugging session?**
A: A `PreToolUse` hook can block or flag dangerous commands (e.g., matching `rm -rf`, prod DB writes) before they execute; a `PostToolUse` hook can auto-run tests after each fix attempt.
*Why: Automates safety nets and fast feedback loops so mistakes are caught deterministically rather than relying on the model to self-police.*

**Q7. What does "approval mode" trade off, and how do you pick one for an incident response scenario?**
A: Trades speed/autonomy against safety; for incidents, a common choice is `default` (ask) for anything mutating state, while allowing unrestricted read-only investigation (logs, grep, diff) to move fast without risking further outage.
*Why: Incidents need both speed (to resolve) and caution (not to make things worse) — scoping autonomy by action type balances both.*

**Q8. How do you debug Claude Code itself when it's misbehaving (e.g., ignoring CLAUDE.md, wrong tool calls)?**
A: Check config layering (is a managed/user setting overriding project settings?), inspect which CLAUDE.md files are actually loaded, review recent session transcript for where it diverged, and test with a minimal reproduction prompt.
*Why: Agent "bugs" are often config/context issues (wrong file loaded, conflicting settings) rather than model issues, so debugging config first is usually faster than re-prompting repeatedly.*

**Q9. What's the value of an MCP server for operational/observability tooling during debugging?**
A: An MCP server can connect Claude Code directly to logging/metrics/tracing systems (e.g., Datadog, Sentry, a custom internal dashboard) so it can query real operational data instead of relying on pasted logs.
*Why: Live, structured access to telemetry lets the agent correlate errors with deploys/metrics far more reliably than manual copy-paste of log snippets.*

**Q10. How should secrets/production credentials be handled when Claude Code is used for live debugging?**
A: Use scoped, short-lived, read-only credentials where possible, injected via environment/secrets manager, never pasted into chat or committed to settings; pair with sandboxing and deny-rules for write/delete actions.
*Why: Debugging access is a common vector for privilege escalation; least-privilege plus sandboxing limits damage if the agent (or a prompt injection) misuses access.*

**Q11. What's a good CI/CD integration pattern for using Claude Code to triage failing builds?**
A: A pipeline step invokes Claude Code (headless/non-interactive mode) with the failing build log, scoped to a restricted tool allowlist and sandbox, to produce a root-cause summary or suggested patch as a PR comment — not auto-merge.
*Why: Keeps humans as the merge gatekeeper while automating the tedious "read 2000 lines of CI log" triage step.*

**Q12. Why is a "read-only investigation first" pattern recommended for agent-assisted ops debugging?**
A: It lets Claude gather full context (logs, diffs, configs) without risk before any mutating action is proposed/approved, reducing the chance of acting on an incomplete/incorrect hypothesis.
*Why: Debugging-before-fixing (understand, then act) reduces false fixes and accidental new incidents — same principle as disciplined human incident response.*

**Q13. How can subagents help during a complex multi-service outage investigation?**
A: Spin up parallel subagents to independently investigate each candidate service/log source, then have the main agent synthesize their findings into one root-cause hypothesis.
*Why: Parallelizes evidence-gathering across independent workstreams, cutting time-to-diagnosis versus one agent serially checking each service.*

**Q14. What should a Skill for "incident response runbook" typically contain?**
A: Step-by-step triage procedure (where to check logs/metrics, who to page, rollback commands), explicit "do not do X without approval" guardrails, and links/scripts for common diagnostic commands.
*Why: Encodes institutional incident knowledge so any engineer (or the agent on their behalf) follows the same tested procedure instead of improvising under pressure.*

**Q15. How do you verify that a hook or permission rule actually behaves as intended before relying on it in production debugging?**
A: Test it in a sandboxed/non-production environment first — trigger the exact command patterns it should allow/block/ask, and confirm via session logs/transcript rather than assuming the config is correct.
*Why: Permission/hook config bugs (wrong regex, wrong scope) are silent until the exact scenario occurs — verifying proactively avoids a nasty surprise mid-incident.*

**Q16. What's the benefit of keeping a transcript/audit log of agent actions during operational debugging?**
A: Full traceability of what commands/tools the agent ran and when, enabling post-incident review, compliance audits, and faster debugging of the agent's own behavior if something went wrong.
*Why: Just like human on-call actions are logged, agent actions touching production need an audit trail for accountability and learning.*
