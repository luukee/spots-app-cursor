# .cursor guide

This folder stores Cursor-specific collaboration assets for the SPOTS repo.

## Folder overview

### `plans/`
Task-specific implementation plans created during AI sessions.

- **Purpose:** break one feature/refactor into steps, tradeoffs, and file-by-file changes.
- **Lifecycle:** temporary/iterative; many are historical records.
- **When to use:** when work is non-trivial and you want a scoped plan before or during coding.
- **Good examples:** large auth flow refactor, session/logout behavior changes, multi-file symptom flow updates.

### `rules/`
Persistent project guardrails and conventions.

- **Purpose:** always-on repo behavior and quality constraints.
- **Lifecycle:** long-lived and version-controlled.
- **When to use:** standards that should apply repeatedly across tasks (security, architecture, workflows).
- **Current SPOTS themes:** healthcare/privacy expectations, Next.js/App Router patterns, Amplify/Auth guidance, no silent REDCap fallbacks.

### `skills/`
Reusable task playbooks the agent can load on demand.

- **Purpose:** focused instructions for a recurring task type.
- **Lifecycle:** long-lived; update as patterns evolve.
- **When to use:** repeated work categories where shared context improves output quality.
- **Current SPOTS skills:**
  - `spots-security-privacy`
  - `spots-api-redcap-integration`
  - `spots-pro-ctcae-business`
  - `spots-testing-qa`

## Which one should I use?

- **Use a plan** for **one concrete change** (feature/refactor) with clear scope.
- **Use a rule** for **global policy** that should apply broadly.
- **Use a skill** for **repeatable workflow guidance** (e.g., API proxy patterns, PRO-CTCAE logic, testing strategy).

## Avoid overlap

- Put **policy** in `rules/`, not in each skill.
- Put **task walkthroughs** in `plans/`, not in `rules/`.
- Keep **skills concise and practical** (what to do + when to use), and link to docs/rules for deep details.

## Suggested maintenance

- Keep `rules/` small and stable; remove duplicates.
- Keep `skills/` specific (one skill = one domain/workflow).
- Archive or prune stale `plans/` periodically so active plans are easier to find.
- When adding a skill, use clear trigger terms in the YAML `description` so it is discoverable.

## Quick team workflow

1. Start with an issue/task.
2. If complex, create/update a file in `plans/`.
3. Implement with relevant `rules/` and `skills/`.
4. If a pattern repeats across tasks, promote it into a `skill`.
5. If a behavior should always apply repo-wide, promote it into a `rule`.
