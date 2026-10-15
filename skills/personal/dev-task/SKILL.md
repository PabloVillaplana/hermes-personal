---
name: dev-task
description: Implement a feature, fix or refactor in one of Villa's own repos (mainly pitmio) by delegating the coding to Claude Code — brief, read-only plan, Villa's approval, implementation, verification with the repo's checks, commit message. Use when Villa asks "hacé X en pitmio", "arreglá…", "agregá…" in his own projects.
version: 1.0.0
platforms: [macos]
metadata:
  hermes:
    tags: [claude-code, pitmio, laravel, workflow]
    category: personal
    related_skills: [claude-code, project-memory, session-handoff]
    requires_toolsets: [terminal]
---

# Dev task → Claude Code

You (Hermes) own the task, the plan approval, the verification and the communication. **Claude Code** edits code inside one repo. Read the bundled `claude-code` skill for CLI details.

- Repo (default): `/Users/pablovillaplana/Proyectos/pitmio`
- Task workspace: `$HERMES_HOME/workspace/tasks/<slug>/` (never inside the repo)
- `SKILL_DIR`: `~/.hermes/profiles/personal/skills/personal/dev-task` (holds `claude-rules.md`)
- `CC_MODEL`: `opus` unless Villa names another model for this task; store it in `state.json`.

## 0. Preconditions

```bash
claude --version && claude auth status --text
cd <repo> && git status --porcelain     # must be empty — if not, STOP and ask Villa
./vendor/bin/sail ps                     # pitmio: Sail must be up; if not, ask Villa to run `sail up -d`
```

## 1. Understand and brief

1. Read `CLAUDE.md` (and the parts of `docs/PRODUCT.md` / `docs/ARCHITECTURE.md` the task touches).
2. Restate the goal in 2–4 bullets; list open questions. Ask the blocking ones before planning.
3. Create a branch only if Villa wants one (pitmio works on `main` by default): `git switch -c feat/<slug>`.
4. Write `tasks/<slug>/brief.md`: goal, acceptance criteria, modules involved, constraints from `CLAUDE.md`, out of scope, open questions.

## 2. Plan (read-only)

```bash
claude -p "$(cat "$T/brief.md")

Produce an implementation plan only. Do not edit files." \
  --permission-mode plan --model "$CC_MODEL" \
  --append-system-prompt-file "$SKILL_DIR/claude-rules.md" \
  --output-format json --max-turns 25 > "$T/plan.json"
```

Save `session_id` to `tasks/<slug>/state.json`. **Show Villa the plan** (files, approach, migrations, risks) and wait for his OK.

## 3. Implement

```bash
claude -p "Villa approved the plan. Implement it now. Notes from Villa: <notes or 'none'>." \
  --resume "<session_id>" --permission-mode acceptEdits --model "$CC_MODEL" \
  --allowedTools "Read,Edit,Write,Grep,Glob,mcp__laravel-boost__search-docs,mcp__laravel-boost__database-schema,mcp__laravel-boost__database-query,mcp__laravel-boost__browser-logs,Bash(git status:*),Bash(git diff:*),Bash(git log:*),Bash(./vendor/bin/sail composer lint:*),Bash(./vendor/bin/sail composer lint\:fix:*),Bash(./vendor/bin/sail composer types:*),Bash(./vendor/bin/sail composer test:*),Bash(./vendor/bin/sail artisan test:*),Bash(./vendor/bin/sail artisan make\:*),Bash(./vendor/bin/sail npm run test:*),Bash(./vendor/bin/sail npm run types\:check:*),Bash(./vendor/bin/sail npm run build:*),Bash(./vendor/bin/sail npx vp check:*),Bash(./vendor/bin/sail composer types\:generate:*)" \
  --disallowedTools "Bash(git push:*),Bash(git commit:*),Bash(git reset:*),Bash(git checkout:*),Bash(git switch:*),Bash(rm:*),Bash(./vendor/bin/sail artisan migrate\:fresh:*),Bash(./vendor/bin/sail artisan db\:wipe:*),Bash(./vendor/bin/sail composer require:*),Bash(./vendor/bin/sail npm install:*),Bash(curl:*),Bash(wget:*),Bash(open:*),WebFetch" \
  --append-system-prompt-file "$SKILL_DIR/claude-rules.md" \
  --output-format json --max-turns 60 > "$T/implement.json"
```

Terminal timeout ~1800 s. On `error_max_turns`, resume once with "Continue where you left off"; if it fails again, stop and report.

## 4. Verify (you)

```bash
git status && git diff --stat
./vendor/bin/sail composer lint && ./vendor/bin/sail composer types && ./vendor/bin/sail composer test
./vendor/bin/sail npm run test && ./vendor/bin/sail npx vp check && ./vendor/bin/sail npm run types:check
```

Also check: diff matches the plan; new UI strings exist in both `lang/es.json` and `lang/en.json`; `composer types:generate` was run if Data/enums changed; new workshop-private models use `BelongsToWorkshop` and have cross-workshop tests; no real data outside `database/seeders/Local/`; no secrets; no AI attribution. Fix through another `--resume` round if needed.

## 5. Hand off

Reply in Spanish: what changed, checks run and results, the commit message(s) in Conventional Commits English (`feat(scope): …`), and anything Villa must do (migrations to run, `sail npm run build`, env vars). Commit only if he asks. **Never push or deploy.**

If the task produced a decision worth keeping, record it with the `project-memory` skill (pitmio keeps project knowledge in `CLAUDE.md` and `docs/`; ask before editing `CLAUDE.md`).
