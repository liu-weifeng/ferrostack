---
name: git-commit-workflow
description: User prefers committing directly to main (no feature branch + merge) in the petrel repo and the xvirt lab repos
metadata: 
  node_type: memory
  type: feedback
  originSessionId: dca19e25-4a8c-4fc3-928b-cb4ba9d1f85e
---

User wants commits made **directly to `main`** — NOT the branch-off-main + `--no-ff` merge
flow. Stated 2026-06-18 for the petrel repo (`ssh://git@192.168.2.107:2222/liuwfng/petrel.git`)
and "后边也直接提交到主分支" = going forward too.

**Why:** these are the user's own lab repos (sole owner, internal GitLab on 192.168.2.107);
the branch+MR ceremony is unwanted overhead. (Note: the repo's *historical* `perf-*` commits
used branch→merge, but the user has since overridden that.)

**How to apply:** commit straight to `main` and push. Do NOT `git checkout -b` a feature
branch first, even though the global default is "if on the default branch, branch first" —
this preference overrides it here. End commit messages with the
`Co-Authored-By: Claude Opus 4.8 (1M context)` trailer.

**2026-06-19 update — auto commit/push granted:** user said "你可以自动commit 和push"
(you may auto commit and push). For multi-component feature work in these lab repos I no
longer need to wait for an explicit "commit"/"push" each time — commit + push proactively at
sensible checkpoints (per-component, after it builds/works). Still write clear messages with
the trailer; group changes logically per repo. Applies to the xvirt component repos generally.
Related: [[xvirt-components-and-xvirtctl]]
