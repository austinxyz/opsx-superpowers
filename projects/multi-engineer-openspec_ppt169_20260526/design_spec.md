# Multi-Engineer OpenSpec Workflow - Design Spec

## I. Project Information

| Item | Value |
| ---- | ----- |
| **Project Name** | multi-engineer-openspec |
| **Canvas Format** | PPT 16:9 (1280×720) |
| **Page Count** | 9 |
| **Design Style** | Modern Tech / Dark Theme |
| **Target Audience** | Software engineers new to OpenSpec team workflow |
| **Use Case** | Team onboarding, internal knowledge sharing |
| **Created Date** | 2026-05-26 |

---

## II. Canvas Specification

| Property | Value |
| -------- | ----- |
| **Format** | PPT 16:9 |
| **Dimensions** | 1280×720 px |
| **viewBox** | `0 0 1280 720` |
| **Margins** | Left/right 64px, top/bottom 52px |
| **Content Area** | 1152×616 px |

---

## III. Visual Theme

### Theme Style

- **Style**: Modern Tech
- **Theme**: Dark theme
- **Tone**: Professional, precise, developer-focused

### Color Scheme

| Role | HEX | Purpose |
| ---- | --- | ------- |
| **Background** | `#0F1117` | Page background |
| **Secondary bg** | `#1C1F2E` | Card background, section panels |
| **Primary** | `#4F8EF7` | Titles, key labels, icons |
| **Accent** | `#34D399` | Success, eval pass, achieve |
| **Secondary accent** | `#818CF8` | Phase highlights, gradient |
| **Body text** | `#E2E8F0` | Main body text |
| **Secondary text** | `#94A3B8` | Captions, annotations |
| **Tertiary text** | `#64748B` | Footers, supplementary |
| **Border/divider** | `#2D3748` | Card borders, divider lines |
| **Success** | `#34D399` | Positive indicators |
| **Warning** | `#F87171` | Block markers, errors |

### Gradient Scheme

```xml
<linearGradient id="titleGradient" x1="0%" y1="0%" x2="100%" y2="0%">
  <stop offset="0%" stop-color="#4F8EF7"/>
  <stop offset="100%" stop-color="#818CF8"/>
</linearGradient>
<linearGradient id="accentGradient" x1="0%" y1="0%" x2="100%" y2="0%">
  <stop offset="0%" stop-color="#34D399"/>
  <stop offset="100%" stop-color="#4F8EF7"/>
</linearGradient>
```

---

## IV. Typography

| Element | Font | Size | Weight | Color |
| ------- | ---- | ---- | ------ | ----- |
| Slide title | Inter, system-ui | 40px | 700 | `#E2E8F0` |
| Section header | Inter, system-ui | 28px | 600 | `#4F8EF7` |
| Body text | Inter, system-ui | 20px | 400 | `#E2E8F0` |
| Caption/label | Inter, system-ui | 16px | 400 | `#94A3B8` |
| Code/monospace | JetBrains Mono, monospace | 16px | 400 | `#34D399` |
| Slide number | Inter, system-ui | 14px | 400 | `#64748B` |

---

## V. Layout Principles

- Dark card panels on dark background: use `#1C1F2E` cards with `#2D3748` border
- Left-aligned text, generous whitespace
- Flow diagrams: horizontal or top-down, bold connector arrows in `#4F8EF7`
- Phase labels: pill/badge style with primary color bg
- Max 5 bullet points per slide; prefer visual over text
- Bottom-right slide number, bottom-left phase label when relevant

---

## VI. Icon Usage

- Unicode emoji as inline icons: ✅ ⚙️ 📋 🎯 🔄 📝 👥 🚀 🌿 📦
- No external icon library
- Phase icons: 🔍 Explore · 📋 Propose · ⚙️ Apply · 📦 Archive

---

## VII. Visualization Plan

| Slide | Visual Type |
| ----- | ----------- |
| 3 (4-Phase flow) | Horizontal pipeline diagram with 4 phase boxes + arrows |
| 4 (Two-track) | Decision tree flowchart (small vs large track) |
| 5 (Achieve gate) | Vertical 4-layer gate diagram with retry loops |
| 6 (Parallel dev) | Side-by-side: conflict detection flow |
| 7 (Review model) | Two-column comparison: Spec review vs Code review |
| 8 (Shared files) | Before/after file structure diagram |

---

## VIII. Image Resources

No AI-generated images. All visuals are SVG diagrams and text cards.

---

## IX. Content Outline

### Slide 1 — Cover
- Title: "Multi-Engineer OpenSpec Workflow"
- Subtitle: "Spec quality = delivery quality"
- Visual: subtle grid/code background, gradient title

### Slide 2 — The Shift
- Title: "The Engineer's Role Has Changed"
- Left column: Before (coder → writes code → debug → ship)
- Right column: After (spec author → judge → Claude executes → verify)
- Key insight callout: "Speed is no longer the constraint. Judgment is."

### Slide 3 — 4-Phase Flow
- Title: "Four Phases, Hard Boundaries"
- Horizontal pipeline: 🔍 Explore → 📋 Propose → ⚙️ Apply → 📦 Archive
- Under each: one-line description + key output artifact
- Bottom note: "Cannot skip phases"

### Slide 4 — Two-Track Branch Model
- Title: "Two-Track Branch Strategy"
- Decision: Small feature (≤2 days, single module) vs Large feature (≥3 days, arch/cross-module)
- Small track: feat/topic → 1 PR → achieve
- Large track: spec/topic → PR1 Spec Review → feat/topic → PR2 Impl Review → achieve
- Visual: branching flow diagram

### Slide 5 — Achieve: Four-Layer Gate
- Title: "Apply ≠ Achieve"
- 4 layer cards stacked: Local Eval → CI → PR Review → Archive
- Each: icon + what passes + what happens on fail
- Bottom: "PR merge to main = Achieve"

### Slide 6 — Parallel Development
- Title: "Conflict Detection at Propose Time"
- Rule: "One engineer, one topic, end-to-end"
- Flow: propose → read specs + active branches → declare dependencies → sequence or parallelize
- Key rule: "Interface must lock before parallel implementation starts"

### Slide 7 — Review Model
- Title: "Humans Review Spec, CI Reviews Code"
- Two columns:
  - Spec Review (high bar): intent, decomposition, CONTRACT blocks — 15-30 min
  - Code Review (lightweight): CI green? eval pass? spec intent matched? — 5 min
- Bottom: "Evaluator subagent already did the code review"

### Slide 8 — Shared File Architecture
- Title: "Archive in Parallel, No Conflicts"
- Problem: CLAUDE.md / README.md = merge conflict at archive time
- Solution: per-capability pitfalls.md + CLAUDE.md as index + maintenance PR
- Before/after file structure

### Slide 9 — Getting Started
- Title: "Start Here"
- 3-step action guide:
  1. Run /opsx:explore on your next feature
  2. Open spec PR before writing code (for large features)
  3. Let the harness drive apply; you review spec fidelity
- Bottom: repo link + "Achieve = archive + CI green + PR merged"

---

## X. Speaker Notes Plan

Each slide: 2-3 sentences expanding on the visual for presenter context.

---

## XI. Technical Constraints

- All colors from spec_lock.md only
- No rgba() — use stop-opacity for gradients
- Font stack: Inter, system-ui, sans-serif
- viewBox: 0 0 1280 720, no exceptions
- Text in SVG: dominant-baseline="middle" for vertical centering
- Rounded rects: rx="8" standard, rx="16" for pill/badge shapes
