# `docs/assets`

Static assets referenced by the documentation (exported diagram images,
screenshots for `docs/user-guide.md`, etc.). Currently empty — the
diagrams in `docs/architecture.md`, `docs/database.md`, and `roadmap.md`
are written as inline Mermaid code blocks (which render natively on
GitHub/GitLab) rather than pre-rendered images, so there is nothing to
store here yet.

Add exported `.png`/`.svg` versions here only when a diagram needs to be
embedded somewhere that doesn't render Mermaid (e.g., the printed FYP
report, or a PowerPoint deck) — keep the Mermaid source in the relevant
`docs/*.md` file as the source of truth, and treat anything in this folder
as a generated export.
