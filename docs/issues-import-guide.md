# GitHub Issues Import Guide (Patient Service)

This folder contains an importable CSV backlog sized for 1–2 day tasks. Use this to bulk‑create issues in your GitHub repo and then assign them between you and your partner.

## Files
- `docs/issues-backlog.csv` — Issues with Titles, Bodies, Labels, and Milestones.

## Steps (GitHub UI)
- Create (or open) your GitHub repository.
- Create the following milestones (optional but recommended):
  - `Phase 1 — Spec & Design`
  - `Phase 1 — Local Dev`
  - `Phase 1 — API MVP`
  - `Phase 1 — AWS Dev (optional)`
  - `Phase 1 — QA & Docs`
- Go to `Issues` → `Import` (or visit `https://github.com/<owner>/<repo>/import/issues`).
- Upload `docs/issues-backlog.csv` (Python + AWS only) and map columns (Title, Body, Labels, Milestone).
- Complete the import and then assign issues to collaborators.

## Notes
- Labels listed in the CSV will be created automatically if they don’t exist.
- Assignees are left blank for you to allocate; you can bulk‑assign by label/milestone after import.
- Time estimates are included in each issue body for planning.
