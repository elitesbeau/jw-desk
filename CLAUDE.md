# JW Desk — Claude Instructions

## Publishing
After every change, automatically publish to `main` via a PR (branch → PR → squash merge). Do not wait for the user to ask. Only skip publishing if the user explicitly says not to.

**Publish flow:**
1. Commit changes to the feature branch (`claude/jw-desk-changes-*` or `publish-fresh` from `origin/main`)
2. Push the branch
3. Create a PR targeting `main`
4. Squash merge immediately

If there are merge conflicts, create a clean branch from `origin/main`, cherry-pick only the new commits, and PR from that.

## Project
Single-file app: `/home/user/jw-desk/index.html` — all CSS, HTML, and JS are inline.
