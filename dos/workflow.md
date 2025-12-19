## Goal
Move from a single-branch repo (main) to a fully structured workflow with:
- main → Production
  - Source of truth
  - All feature branches must start from here
  - Only receives PRs from staging
  - Auto-tagged on merge
- staging → QA / Preview
  - Temporary integration branch
  - Receives PRs from feature branches
  - Gets periodically reset/refreshed from main
  - Used for preview deployments + QA
- feature branches → pull requests → staging
  - Always created from main
  - Merged into staging for QA
  - Eventually released to main through a staging → main promotion PR
- GitHub Actions → tests + auto-tag releases

## One-Time Setup (Run Once)
### Step 1 — Create the staging branch
``` bash
# Ensure you're on main and clean
git checkout main
git pull origin main

# Create staging from latest main
git checkout -b staging
git push -u origin staging
```
### Step 2 — Protect both branches
In GitHub Settings → Branch protection rules:

main (production)
- Require PR before merging
- Require status checks to pass
- Require linear history
- No direct pushes

staging (QA)
- Same rules as main
- Allow auto-deploy previews

### Step 3 — Add a GitHub Actions workflow for tests + tagging
``` bash
.github/workflows/ci.yml
``` 
This does:
- Runs tests on PR → staging
- Runs tests on pushes to staging + main
- Automatically tags releases on successful main deployment

## Day-to-Day Development Workflow
### Step 1 — Create a feature branch
feature branches should ALWAYS be created from main
``` bash
git checkout main
git pull origin main
git checkout -b feature/my-feature
```
Work normally. Commit normally.
``` bash
git add .
git commit -m "Improve my feature"
git push -u origin feature/my-feature
```
### Step 3 - Create Your Pull Request

PR always goes → staging

On GitHub → New Pull Request

- base branch: staging

- compare branch: feature/my-feature

GitHub Actions runs tests automatically.

If all tests pass → Merge.

This updates staging, triggering:

- Amplify → staging build

- Preview link for QA

- Zero impact on production

### Step 4 - Promoting to Production (main)
After QA approves staging:

Create PR from staging → main
- base: main
- compare: staging

This is your production deployment PR.

GitHub Actions will:
- run full test suite
- production deploy
- when merged → tag a release
- release notes
- Amplify deploys to production

### Step 5 -  Sync staging after main deploy

After deploying to main, update staging so future PRs hit a clean branch:

Whenever main receives production fixes:

``` bash
git checkout staging
git pull origin staging
git pull --rebase origin main
git push --force-with-lease
git push
```

This ensures staging is always a clean, short-lived recreation of main. It resets staging to be a perfect reflection of main.

```yaml
CREATE FEATURE BRANCH
           |
           v
--- main ------------------------- (stable)
       \ 
        \ feature/*
         \___________ PR → staging
                         |
                         v
------------------ staging ------------------- (QA / preview)
                         |
                         v
                PR → main  (release)
                         |
                         v
------------------ main ---------------------- (new baseline)
                         |
                         v
           Reset staging = main

```

## GitHub Actions

- auto-pr-main.yml -> When staging is updated, auto-open a PR to main
- sync-staging.yml -> To ensure staging always matches main automatically right after deployment.
- release-tag.yml  -> Automatically tags releases on every merge into main
- ci.yml -> Auto-run tests for PRs into staging
- preview-build -> Creates preview builds for PRs → staging
- release-notes.yml -> Auto-generate GitHub Release Notes for each tag