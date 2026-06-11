---
name: prism
description: "Structural analysis framework: 3 modes for deep code/artifact analysis. Prism-Scan (dynamic lenses, lightweight), Prism-3Way (WHERE/WHEN/WHY orthogonal ops), Prism-Full (maximum depth, adversarial self-correction). Load when user asks for deep analysis of code, architecture, text, or strategy."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [analysis, structural, prism, code-review, architecture, deep-dive, adversarial]
---

# Prism — Structural Analysis Framework

Three analysis modes, pick by depth and time available.

## Mode Selection

| Mode | When to Use | Time Cost | Instruction File |
|------|-------------|-----------|-----------------|
| **Prism-Scan** | Quick structural read; first look at unfamiliar code; find conservation laws | Low | `references/prism-scan.md` |
| **Prism-3Way** | Multi-domain artifacts; when you want WHERE/WHEN/WHY orthogonal perspectives; strategy, design, or architecture | Medium | `references/prism-3way.md` |
| **Prism-Full** | Maximum depth; important or critical artifacts; when you want adversarial self-correction of findings | High | `references/prism-full.md` |

---

## Quick Rule

- **First analysis on a repo → Prism-Scan** (fast, generates lenses specific to this artifact)
- **Code + architecture together → Prism-3Way** (WHERE/WHEN/WHY separates structural, temporal, causal)
- **High-stakes review (before a major decision or release) → Prism-Full** (multi-pass + adversarial)

---

## Running a Mode

Load the mode's reference file and follow its instructions exactly. The procedures are mandatory — each mode has an iron law about not skipping phases.

```
Prism-Scan:  read references/prism-scan.md → execute STEP 0 + STEP 1
Prism-3Way:  read references/prism-3way.md → execute OPERATIONS 1-3 + SYNTHESIS
Prism-Full:  read references/prism-full.md → execute PHASES 1-3 (design → run → attack)
```