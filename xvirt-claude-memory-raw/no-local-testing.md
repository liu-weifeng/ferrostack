---
name: no-local-testing
description: User does not want autotest/validation runs executed from the local control box by default
metadata: 
  node_type: memory
  type: feedback
  originSessionId: dca19e25-4a8c-4fc3-928b-cb4ba9d1f85e
---

User said "不用在本地测试" (no need to test locally). Do NOT run the autotest
suite, dashboard, or ad-hoc validation runs from the local control machine
(192.168.2.112, where `python3 -m autotest.cli` lives) by default.

**Why:** they don't want me spending time/tokens on local test runs unless they
ask for it — landing/committing the code is enough; they'll drive validation
themselves or request it explicitly.

**How to apply:** after making changes, commit/push when asked and stop there —
don't proactively `autotest.cli run` or `serve` locally. Only run tests when the
user explicitly asks (e.g. "run the autotest suite"). Related: [[autotest-framework]].
