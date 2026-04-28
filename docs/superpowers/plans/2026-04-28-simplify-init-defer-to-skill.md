# Simplify `init` and defer configuration to `sdd` skill — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Linear ticket:** MME-69 — "Mejorar claude-sdd"

**Goal:** Eliminate interactive prompts in `init` when running inside an existing project. Defer stack and extension detection to the `sdd` skill, which reads the actual codebase. Keep the interactive flow for empty/new projects, and keep all current flags as escape hatches.

**Architecture:**

1. **Auto-detect "existing project"** in the CLI: if the cwd contains any of `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, or `.git/`, switch to a non-interactive default (project name = directory name, stack = `other`, no extensions). Show one confirmation prompt before writing files. Empty directories keep the current 5-question flow.
2. **Make extension fragments part of the install payload.** Today fragments under `templates/specs/templates/extensions/<id>.md` are filtered out of the destination because the merge happens at install time. To let the skill activate extensions later, we copy them to the destination (`specs/templates/extensions/<id>.md`). The skill then reads them and merges the relevant ones into `feature.spec.md`.
3. **Skill extension** in `templates/.claude/skills/sdd/SKILL.md`: add a "detect & propose extensions" step inside first-time setup. The skill reads the codebase, picks candidate extensions (multi-tenant, persistent-data, etc.) using explicit heuristics, asks the user to confirm, then merges fragments into `feature.spec.md`.

**Tech Stack:** Node ≥18, ESM, JS+JSDoc, vitest. No new dependencies.

**Out of scope:**
- An `update` command (still deferred to v0.2 per README roadmap).
- Auto-applying extensions without user confirmation (the skill always proposes, user approves).
- Removing the `--extensions` CLI flag (kept as a power-user override).

---

## File map

| File | Change |
|---|---|
| `lib/install.js` | Export `isExistingProject(cwd)`; remove fragment filter so `*.md` fragments are copied. |
| `bin/cli.js` | New default flow: detect existing project → skip prompts + show confirmation. Add `--ask` flag for forced interactive mode. |
| `lib/prompts.js` | Add `askConfirmInstall(cwd, fileCount)` confirmation prompt for the new default. |
| `tests/install.test.js` | Add tests for `isExistingProject`. Invert assertion that fragments are NOT copied → ARE copied. |
| `templates/.claude/skills/sdd/SKILL.md` | Add "detect & propose extensions" step in first-time setup with heuristics + merge instructions. |
| `README.md` | Update Quickstart, "What gets installed", "Extensions", "Install options" sections. |
| `.github/workflows/ci.yml` | Invert smoke assertion: `test ! -f .../multi-tenant.md` → `test -f`. |
| `CLAUDE.md` | Note in "Conventions" that fragments now ship to destination (was source-only). |

---

## Task 1: `isExistingProject(cwd)` helper

**Files:**
- Modify: `lib/install.js` (add + export new function)
- Test: `tests/install.test.js` (new `describe('isExistingProject')` block)

- [ ] **Step 1: Write failing tests**

Add to `tests/install.test.js` (top-level, after existing imports add `isExistingProject` to the import):

```js
import { installTemplates, appendGitignore, isExistingProject, EXTENSIONS, isValidExtensionId } from '../lib/install.js';
```

Then add a new `describe` block after `appendGitignore`:

```js
describe('isExistingProject', () => {
  it('returns false for an empty directory', async () => {
    expect(await isExistingProject(tmpDir)).toBe(false);
  });

  it('returns true when package.json exists', async () => {
    await fs.writeFile(path.join(tmpDir, 'package.json'), '{}');
    expect(await isExistingProject(tmpDir)).toBe(true);
  });

  it('returns true when pyproject.toml exists', async () => {
    await fs.writeFile(path.join(tmpDir, 'pyproject.toml'), '');
    expect(await isExistingProject(tmpDir)).toBe(true);
  });

  it('returns true when Cargo.toml exists', async () => {
    await fs.writeFile(path.join(tmpDir, 'Cargo.toml'), '');
    expect(await isExistingProject(tmpDir)).toBe(true);
  });

  it('returns true when go.mod exists', async () => {
    await fs.writeFile(path.join(tmpDir, 'go.mod'), '');
    expect(await isExistingProject(tmpDir)).toBe(true);
  });

  it('returns true when .git directory exists', async () => {
    await fs.mkdir(path.join(tmpDir, '.git'));
    expect(await isExistingProject(tmpDir)).toBe(true);
  });
});
```

- [ ] **Step 2: Run tests, confirm failure**

```bash
npx vitest run tests/install.test.js -t "isExistingProject"
```

Expected: 6 failures with `isExistingProject is not a function` or `not exported`.

- [ ] **Step 3: Implement**

Add to `lib/install.js` (after `isValidExtensionId`):

```js
const EXISTING_PROJECT_MARKERS = ['package.json', 'pyproject.toml', 'Cargo.toml', 'go.mod', '.git'];

/**
 * Returns true if the directory looks like an existing project (has a manifest
 * or is a git repo). Used by the CLI to decide whether to skip interactive prompts.
 * @param {string} cwd
 * @returns {Promise<boolean>}
 */
export async function isExistingProject(cwd) {
  for (const marker of EXISTING_PROJECT_MARKERS) {
    try {
      await fs.access(path.join(cwd, marker));
      return true;
    } catch {
      // not present, continue
    }
  }
  return false;
}
```

- [ ] **Step 4: Run tests, confirm pass**

```bash
npx vitest run tests/install.test.js -t "isExistingProject"
```

Expected: 6 PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/install.js tests/install.test.js
git commit -m "feat(install): add isExistingProject helper

Detects manifests (package.json, pyproject.toml, Cargo.toml, go.mod) and
.git/ as a signal that the cwd is an existing project. CLI will use this
to skip interactive prompts in the next commit. Refs MME-69."
```

---

## Task 2: Copy extension fragments to destination

Today the install filter drops fragments. We need them at the destination so the skill can read and merge them later.

**Files:**
- Modify: `lib/install.js:80-91` (remove fragment filter)
- Modify: `tests/install.test.js:151-167` (invert assertion)
- Modify: `.github/workflows/ci.yml` (invert smoke assertion)

- [ ] **Step 1: Update test to expect fragments DO copy**

Replace the `it('does NOT copy individual extension fragments to destination', ...)` block at `tests/install.test.js:151-167` with:

```js
it('DOES copy individual extension fragments to destination', async () => {
  await installTemplates({ cwd: tmpDir, vars: VARS });
  // README.md is the catalog — should be copied
  const catalogExists = await fs
    .access(path.join(tmpDir, 'specs/templates/extensions/README.md'))
    .then(() => true)
    .catch(() => false);
  expect(catalogExists, 'extensions/README.md (catalog) should exist').toBe(true);
  // Individual fragments SHOULD be copied so the sdd skill can merge them on-demand
  for (const ext of EXTENSIONS) {
    const fragExists = await fs
      .access(path.join(tmpDir, `specs/templates/extensions/${ext.id}.md`))
      .then(() => true)
      .catch(() => false);
    expect(fragExists, `extensions/${ext.id}.md fragment should exist at destination`).toBe(true);
  }
});
```

- [ ] **Step 2: Run, confirm fail**

```bash
npx vitest run tests/install.test.js -t "DOES copy individual extension fragments"
```

Expected: FAIL — fragments are filtered out.

- [ ] **Step 3: Remove the fragment filter in `lib/install.js`**

Find this block (around line 84-91):

```js
  const filtered = skillOnly
    ? allFiles.filter((f) => f.startsWith(`.claude${path.sep}skills${path.sep}sdd${path.sep}`))
    : allFiles.filter(
        (f) =>
          !SOURCE_ONLY_FILES.has(f) &&
          // Keep extensions/README.md (catalog docs), drop individual fragments
          !(f.startsWith(EXT_PREFIX) && !f.endsWith(`${path.sep}README.md`))
      );
```

Replace with:

```js
  const filtered = skillOnly
    ? allFiles.filter((f) => f.startsWith(`.claude${path.sep}skills${path.sep}sdd${path.sep}`))
    : allFiles.filter((f) => !SOURCE_ONLY_FILES.has(f));
```

The `EXT_PREFIX` constant declared a few lines above is now unused — remove it too.

- [ ] **Step 4: Run all tests, confirm pass**

```bash
npm test
```

Expected: all tests pass. The pre-existing tests for the merge logic in `feature.spec.md` (`with multi-tenant extension`, `with multiple extensions`, etc.) still pass because the merge happens before write regardless of whether fragments are also copied.

- [ ] **Step 5: Update CI smoke assertion**

In `.github/workflows/ci.yml`, find the lean smoke step and replace:

```yaml
          test ! -f specs/templates/extensions/multi-tenant.md
```

with:

```yaml
          test -f specs/templates/extensions/multi-tenant.md
          test -f specs/templates/extensions/persistent-data.md
```

- [ ] **Step 6: Commit**

```bash
git add lib/install.js tests/install.test.js .github/workflows/ci.yml
git commit -m "feat(install): ship extension fragments to destination

Fragments under specs/templates/extensions/ are now copied at install
time so the sdd skill can read and merge them when it detects the
project needs them. The init-time merge into feature.spec.md still
happens when --extensions is passed; this is purely additive. Refs MME-69."
```

---

## Task 3: CLI auto-detect existing project + confirmation prompt

**Files:**
- Modify: `lib/prompts.js` (add `askConfirmInstall`)
- Modify: `bin/cli.js` (branch on `isExistingProject`)

- [ ] **Step 1: Add `askConfirmInstall` to `lib/prompts.js`**

Append at the end of `lib/prompts.js`:

```js
/**
 * Single-prompt confirmation for the auto-detected existing-project flow.
 * @param {{cwd: string}} opts
 * @returns {Promise<boolean>}
 */
export async function askConfirmInstall(opts) {
  const cancelled = { cancelled: false };
  const answer = await promptsLib(
    {
      type: 'confirm',
      name: 'proceed',
      message: `Install claude-sdd into ${path.basename(opts.cwd)}? (~15 files; existing files are kept)`,
      initial: true,
    },
    {
      onCancel: () => {
        cancelled.cancelled = true;
        return false;
      },
    }
  );
  if (cancelled.cancelled) return false;
  return Boolean(answer.proceed);
}
```

- [ ] **Step 2: Update `bin/cli.js` to branch on detection**

In `bin/cli.js`, update the import:

```js
import { askInteractive, askConfirmInstall } from '../lib/prompts.js';
import { installTemplates, appendGitignore, reportFile, EXTENSIONS, isExistingProject, isValidExtensionId } from '../lib/install.js';
```

Then replace the answer-gathering block (currently `if (opts.yes || opts.skillOnly) { ... } else { const interactive = await askInteractive(...) ... }`) with:

```js
    let answers;
    const autoDetect = !opts.yes && !opts.skillOnly && !opts.ask && (await isExistingProject(cwd));

    if (opts.yes || opts.skillOnly) {
      const projectName = opts.projectName || cwd.split('/').pop() || 'my-project';
      answers = {
        projectName,
        stack: opts.stack,
        extensions: parsedExtensions,
        addToGitignore: opts.gitignore !== false,
        onConflict: /** @type {'overwrite'|'skip'} */ (opts.overwrite ? 'overwrite' : 'skip'),
      };
    } else if (autoDetect) {
      console.log(kleur.dim('Detected existing project — deferring stack/extension detection to the `sdd` skill.'));
      const proceed = await askConfirmInstall({ cwd });
      if (!proceed) {
        console.log(kleur.red('Cancelled.'));
        process.exit(1);
      }
      const projectName = opts.projectName || cwd.split('/').pop() || 'my-project';
      answers = {
        projectName,
        stack: opts.stack, // 'other' by default; skill will refine
        extensions: parsedExtensions,
        addToGitignore: opts.gitignore !== false,
        onConflict: /** @type {'overwrite'|'skip'} */ (opts.overwrite ? 'overwrite' : 'skip'),
      };
    } else {
      const interactive = await askInteractive({ cwd });
      if (!interactive) {
        console.log(kleur.red('Cancelled.'));
        process.exit(1);
      }
      answers = interactive;
      if (parsedExtensions.length) answers.extensions = parsedExtensions;
    }
```

- [ ] **Step 3: Manual smoke — empty directory**

```bash
TMP=$(mktemp -d)
cd "$TMP"
node /home/miguemercedes/projects/claude-sdd/bin/cli.js init --yes --project-name "smoke-empty"
cd - && rm -rf "$TMP"
```

Expected: runs to completion, files created.

- [ ] **Step 4: Manual smoke — existing project (auto-detect)**

```bash
TMP=$(mktemp -d)
cd "$TMP"
echo '{"name":"smoke-existing"}' > package.json
echo "yes" | node /home/miguemercedes/projects/claude-sdd/bin/cli.js init
ls CLAUDE.md SPEC_PIPELINE.md specs/templates/extensions/multi-tenant.md
grep "Detected existing project" /tmp/log 2>/dev/null || echo "(stdout check above)"
cd - && rm -rf "$TMP"
```

Expected: a single confirmation prompt, then files written, fragments present at destination.

- [ ] **Step 5: Run vitest, confirm pass**

```bash
npm test
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add bin/cli.js lib/prompts.js
git commit -m "feat(cli): auto-detect existing project, skip prompts

When the cwd has a known manifest or .git/, init defers stack/extension
detection to the sdd skill and only asks for confirmation. The 5-question
flow remains for empty directories. Existing flags (--yes, --skill-only,
--extensions) preserved. Refs MME-69."
```

---

## Task 4: `--ask` flag to force the interactive flow

**Files:**
- Modify: `bin/cli.js` (add option)

- [ ] **Step 1: Add option declaration**

In `bin/cli.js`, in the `program.command('init')` chain, add after `--skill-only`:

```js
  .option('--ask', 'Force the interactive prompts even when an existing project is detected', false)
```

(The branching logic added in Task 3 already reads `opts.ask`, so no other change is needed.)

- [ ] **Step 2: Manual smoke — `--ask` overrides auto-detect**

```bash
TMP=$(mktemp -d)
cd "$TMP"
echo '{"name":"smoke-ask"}' > package.json
# Should show the 5-question flow despite package.json being present.
# Cancel with Ctrl+C — we are just verifying the prompt appears.
node /home/miguemercedes/projects/claude-sdd/bin/cli.js init --ask < /dev/null || true
cd - && rm -rf "$TMP"
```

Expected: the interactive flow starts (visible "Project name" prompt), not the single confirmation.

- [ ] **Step 3: Commit**

```bash
git add bin/cli.js
git commit -m "feat(cli): add --ask flag to force interactive flow

Escape hatch when auto-detect picks the deferred flow but the user
wants the original 5-question prompts. Refs MME-69."
```

---

## Task 5: Skill — detect & propose extensions in first-time setup

**Files:**
- Modify: `templates/.claude/skills/sdd/SKILL.md`

- [ ] **Step 1: Insert new step into first-time setup**

In `templates/.claude/skills/sdd/SKILL.md`, find step 3 of the first-time setup section (currently starts with `3. **Confirm spec extensions enabled**.`). Replace that step with:

```markdown
3. **Detect & propose extensions.** Look at `specs/templates/feature.spec.md`:
   - Check the `<!-- Extensions enabled: ... -->` comment near the top, if any.
   - Run the heuristics below against the codebase. For each extension whose heuristic matches, propose it to the user.

   | Extension | Heuristic — propose if any are true |
   |---|---|
   | `multi-tenant` | Code or schemas reference `tenant_id`, `business_id`, `account_id`, `workspace_id`, or `org_id` as a foreign key / scope. |
   | `persistent-data` | Repo has `migrations/`, `alembic/`, `prisma/`, `drizzle/`, `schema.sql`, or an ORM dependency (sequelize, typeorm, prisma, sqlalchemy, alembic, gorm, diesel). |
   | `production-rollout` | Code references feature flags (`launchdarkly`, `growthbook`, `unleash`, env-var gated `FEATURE_*`). |
   | `operational` | Project has `Dockerfile` + observability deps (datadog, sentry, opentelemetry, prom_client) or a `runbooks/` directory. |
   | `external-deps` | Code calls third-party APIs (stripe, paddle, twilio, sendgrid) or has webhook handlers. |
   | `public-api` | Project is a library (no top-level app entry, has `main`/`exports` in `package.json`, or publishes to npm/PyPI). |

   - Show the user the proposed list with one-line rationale per match: "Propose `persistent-data` because alembic/ exists." Ask which to enable. Default = all matches.
   - For each confirmed extension, append the fragment from `specs/templates/extensions/<id>.md` into `specs/templates/feature.spec.md` immediately before the `## Review notes` heading. If a marker comment `<!-- Extensions enabled: ... -->` already exists, update its list; otherwise add it after the front-matter.
   - If no extensions match, say so and skip — `feature.spec.md` stays lean.
```

- [ ] **Step 2: Verify the SKILL.md still has correct numbering**

Read the file from "First-time setup" through the end of that section and confirm step 4 (the ADR placeholder) follows naturally. No code change expected — just sanity check.

- [ ] **Step 3: Commit**

```bash
git add templates/.claude/skills/sdd/SKILL.md
git commit -m "feat(skill): detect and propose extensions in first-time setup

The skill now reads the codebase for explicit signals (tenant_id, ORM
deps, feature-flag libs, webhook handlers, library publish config) and
proposes the matching extensions, then merges fragments into
feature.spec.md. Replaces the init-time question with detection that
sees the actual code. Refs MME-69."
```

---

## Task 6: Update README and CLAUDE.md

**Files:**
- Modify: `README.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Update README Quickstart**

In `README.md`, in the `## Quickstart` section, after the install line, replace the description of the 4 questions with:

```markdown
If you run `init` inside an existing project (manifest or `.git/` detected),
the CLI skips the questions and only asks for one confirmation — stack and
extensions are detected by the `sdd` skill on first invocation. In an empty
directory the original 5-question flow runs. Use `--ask` to force the
interactive flow regardless of detection.
```

- [ ] **Step 2: Update README "What gets installed"**

Update the tree to show that fragments are now copied:

```
│   │   └── extensions/             # 6 fragments + catalog README
```

(Replace the line that previously said "Catalog (README + 6 fragments not auto-copied)".)

- [ ] **Step 3: Update README "Extensions" → "Two ways to use them"**

Reword the "Project-wide" paragraph:

```markdown
**Auto-detected (default for existing projects):** the `sdd` skill reads
your codebase on first invocation, proposes extensions whose heuristics
match (e.g. `persistent-data` if it sees `alembic/` or a Prisma schema),
and merges the chosen fragments into `feature.spec.md`. You confirm.

**CLI flag (manual override):** pass `--extensions` to lock in choices at
install time without waiting for the skill.

```bash
npx github:MigueMercedes/claude-sdd init --extensions multi-tenant,persistent-data
```
```

- [ ] **Step 4: Update README "Install options"**

Add `--ask` after `--no-gitignore`:

```bash
# Force the 5-question interactive flow even in an existing project
npx github:MigueMercedes/claude-sdd init --ask
```

- [ ] **Step 5: Update repo CLAUDE.md "Install pipeline" note**

In the project's `CLAUDE.md` at "### Install pipeline (`lib/install.js`)", update the second bullet of step 2:

```markdown
   - Source-only files (`.gitignore.additions` — consumed by `appendGitignore()` instead).
```

(Remove the second bullet about extension fragments; they're no longer filtered.)

Also update the file map line about fragments under "What lives where" if needed (re-read the section to verify).

- [ ] **Step 6: Commit**

```bash
git add README.md CLAUDE.md
git commit -m "docs: reflect deferred-config flow + fragment ship behavior

- README Quickstart: explain auto-detect + --ask escape hatch.
- README Extensions: reword 'auto-detected vs manual override'.
- CLAUDE.md: drop stale note about fragments being source-only. Refs MME-69."
```

---

## Task 7: End-to-end smoke

- [ ] **Step 1: Run full test suite**

```bash
npm test
```

Expected: all green.

- [ ] **Step 2: Smoke — empty directory still gets interactive prompts**

```bash
TMP=$(mktemp -d) && cd "$TMP"
node /home/miguemercedes/projects/claude-sdd/bin/cli.js init --yes --project-name "smoke-empty" --stack node
test -f CLAUDE.md && test -f specs/templates/extensions/multi-tenant.md && echo "OK empty"
cd - && rm -rf "$TMP"
```

Expected: `OK empty`.

- [ ] **Step 3: Smoke — existing project auto-detects, no prompts**

```bash
TMP=$(mktemp -d) && cd "$TMP"
echo '{"name":"smoke-auto","dependencies":{"prisma":"5"}}' > package.json
echo "y" | node /home/miguemercedes/projects/claude-sdd/bin/cli.js init
test -f CLAUDE.md && test -f specs/templates/extensions/persistent-data.md && echo "OK auto"
cd - && rm -rf "$TMP"
```

Expected: `OK auto`. (The skill, run later inside Claude Code, would propose `persistent-data` because of the `prisma` dependency.)

- [ ] **Step 4: Smoke — `--ask` forces interactive even with package.json**

```bash
TMP=$(mktemp -d) && cd "$TMP"
echo '{}' > package.json
node /home/miguemercedes/projects/claude-sdd/bin/cli.js init --ask < /dev/null 2>&1 | grep -q "Project name" && echo "OK ask"
cd - && rm -rf "$TMP"
```

Expected: `OK ask`.

- [ ] **Step 5: No commit needed** — this is verification only.

---

## Self-review checklist

- [ ] Spec coverage: ticket asks "eliminar selección de project name, primary-stack, which spec… que el agente identifique". Tasks 1+3+5 cover this. Empty-project flow preserved (Task 3 condition).
- [ ] Placeholders: none ("TBD", "TODO", "implement later", or steps without code).
- [ ] Type/symbol consistency: `isExistingProject` (declared Task 1, used Task 3); `askConfirmInstall` (declared Task 3 step 1, used Task 3 step 2); `--ask` flag (declared Task 4, read in branch logic from Task 3).
- [ ] Tests: invariant changes (`it('does NOT copy ...') → it('DOES copy ...')`) updated in same task that changes behavior.
- [ ] CI: smoke YAML asserts updated alongside the test in Task 2 step 5.
- [ ] Public docs: README and CLAUDE.md updated in Task 6 to match new behavior.
- [ ] Backwards compatibility: all existing flags (`--yes`, `--skill-only`, `--extensions`, `--overwrite`, `--no-gitignore`) preserved. Auto-detect kicks in only when none of those are passed and there is no `--ask`.
