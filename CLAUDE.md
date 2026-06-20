# JW Desk — Claude Instructions

## Publishing
After every change, automatically publish to `main` via a PR (branch → PR → squash merge). Do not wait for the user to ask. Only skip publishing if the user explicitly says not to.

**Publish flow:**
1. `git fetch origin main`
2. `git checkout -b pub-<short-slug> origin/main` — always start from latest main to avoid squash-merge conflicts
3. Cherry-pick or apply changes onto that branch
4. Push, create PR targeting `main`, squash merge immediately
5. Never reuse old branches like `publish-fresh` — they diverge after squash merges

## Project
Single-file app: `/home/user/jw-desk/index.html` — all CSS, HTML, and JS are inline.
