Installed via auto-skill-picker on 2026-07-20.

- Source: https://github.com/alchaincyf/huashu-design (verified owner/repo match, default branch `master`)
- What's included: `SKILL.md` only (core workflow, anti-slop rules, 3-direction Fallback mode, deck/PPT routing table).
- What's NOT included: the repo's `references/` (25+ deep-dive docs), `assets/` (JSX components, 37 SFX files, starter templates), and `scripts/` (video/PDF/PPTX export, image fetching, verification). The repo is a large multi-file structure, so these were intentionally left out rather than auto-copied wholesale.
- If a task needs one of those (e.g. PPTX export via `scripts/html2pptx.js`, or a specific `references/slide-decks.md` deep-dive), fetch that individual file from the repo on demand, or clone the repo directly.
- Content was reviewed before install: no credential exfiltration, authority-impersonation, or safety-bypass instructions found. One notable default behavior: video/animation outputs get a "Created by Huashu-Design" watermark unless the user asks to remove it.
