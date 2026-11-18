---
name: stash-scanning
description: Stash tab scanning and item pricing. Legacy stash endpoint works as of 2026-11-18 (verified: 32 tabs, 60 items) — undocumented endpoint, can change without notice; score_rare works for individual items.
when_to_use: Use for scanning or pricing stash contents — "what's in my stash worth selling", "price my dump tab", "is this item worth listing", "what should I vendor". Immediately tell the user bulk scanning is blocked (options - score_rare per item, OAuth registration, or wait); do not call list_tabs/get_tab — they 403.
---

This task uses the **Stash Scanning** playbook. The playbooks in `playbooks/` are the single source of truth — read them rather than working from memory, so you always get the current version.

1. Read `playbooks/README.md` first — the shared meta-framework (cursory-vs-detailed gate, context/pre-flight protocol, league reference, trust hierarchy, narration norms).
2. Then read `playbooks/stash-scanning.md` and follow it for this task.

**CRITICAL:** Read the "Current status: BLOCKED" section first before attempting any stash tools. `list_tabs`, `get_tab`, `scan_stash_tabs`, and `price_tab` all return HTTP 403. Use `score_rare` for individual item evaluation instead.

Do not summarize or duplicate the playbook content here.
