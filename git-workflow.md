# Git Workflow & Branch Protection

Branch strategy and how to work with pull requests / code review.

---

## 🌿 Branch structure

| Branch | Purpose | Who can push | Protection |
|---|---|---|---|
| `main` / `master` | Production — deployed code only | No one directly | PR + review required |
| `develop` | Integration / test — features merge here | No one directly | PR + review required |
| `feature/*` | Individual dev work | Author | PR from feature → develop |

---

## 🔄 How to work (PR workflow)

### Step 1: Start from develop

```bash
# Make sure you're on develop and up to date
git checkout develop
git pull origin develop

# Create a feature branch
git checkout -b feature/your-change-name
```

### Step 2: Make changes

Edit code, run tests, verify:

```bash
# KB repo
python3 -m py_compile kb_manager/web/routes/search.py

# Run KB tests
pytest kb_manager/ -x -q

# Check lint (if configured)
```

### Step 3: Commit

```bash
git add <changed-files>
git commit -m "feat(kb): add supplement entity-list tool path"

# Conventional commit prefixes:
# feat:     new feature
# fix:      bug fix
# docs:     documentation only
# chore:    maintenance / submodule pins
# refactor: code restructuring, no behavior change
```

### Step 4: Push feature branch

```bash
git push origin feature/your-change-name
```

### Step 5: Open a pull request

On GitHub, open a PR:
- **From:** `feature/your-change-name`
- **To:** `develop`
- **Title:** descriptive, matches commit style
- **Body:** what changed, why, how to test

Request at least **one code review**. Address review comments, then merge.

### Step 6: Merge to develop

After approval, merge the PR via GitHub UI or:
```bash
git checkout develop
git pull origin develop
git merge --no-ff feature/your-change-name
git push origin develop
```

### Step 7: Release to production (main/master)

When develop is stable:
```bash
git checkout main
git pull origin main
git merge --no-ff develop
git push origin main
```

Then deploy:
```bash
bash deploy/vast/stop.sh
bash deploy/vast/start.sh
bash deploy/vast/health.sh
```

---

## 🛡 Branch protection rules (configure on GitHub)

Go to **Settings → Branches → Add branch protection rule** for each repo:

### For `main` / `master`:

| Setting | Value |
|---|---|
| Require a pull request before merging | ✅ Enabled |
| Required approving reviews | 1 |
| Dismiss stale approvals when new commits are pushed | ✅ |
| Require review from code owners | ✅ (if CODEOWNERS exists) |
| Require status checks to pass before merging | ✅ (CI: tests, lint, docs-toc) |
| Require branches to be up to date before merging | ✅ |
| Require conversation resolution before merging | ✅ |
| Restrict pushes to matching branches | ✅ (no direct pushes to main) |
| Allow force pushes | ❌ Disabled |
| Allow deletions | ❌ Disabled |
| Include administrators | ✅ |

### For `develop`:

Same as `main`, but can allow more flexibility if desired.

---

## 🔗 Submodule workflow

Component repos (`Work_RAG-KB`, `Work_RAG-Orchestrator`, `Work_RAG-Guardrails`) are submodules of `Work_Credit-RAG_Phase1`.

**To fix a component:**

```bash
# 1. Work in the component repo
cd components/knowledgebase   # or the component repo directly
git checkout -b fix/issue-name
# ... make changes ...
git commit -m "fix(kb): resolve asyncpg loop bug"
git push origin fix/issue-name
# Open PR on that component repo → main/master
# After merge, advance the gitlink in the parent:
cd ..  # back to Work_Credit-RAG_Phase1
git add components/knowledgebase
git commit -m "chore: advance knowledgebase submodule"
git push origin develop
```

**Never** commit component source changes directly in the parent repo — always go through the submodule.

---

## 🚫 Blocked actions (enforced by protection)

- Direct push to `main`/`master` → rejected
- Direct push to `develop` → rejected
- Force push on any protected branch → rejected
- Deleting protected branches → rejected
- Merging without approval → rejected
- Merging with failing status checks → rejected
