---
name: design2code
description: Design-to-code pipeline - extract design tokens from Figma, generate multiple output formats (MUI, Tailwind, SCSS, CSS, styled-components), create React icons, and build Storybook stories. Reads MUI Figma plugin styles (Primary/Main, Typography/H1) and can emit/import the plugin's theme JSON. Use when the user gives a Figma URL and wants tokens/theme/icons/stories generated end-to-end.
---

# Design2Code

Full orchestration instructions live in [SUPERPOWER.md](./SUPERPOWER.md) — read it in full before executing, then follow its pipeline: parse args → `figma-extract` → parallel `theme-convert` + `figma-icons` → `theme-stories` → summary report.

The four sub-skills it calls are not separately registered as top-level skills, so read their instructions directly from these files rather than invoking them by name:

- [figma-extract/SKILL.md](./figma-extract/SKILL.md)
- [theme-convert/SKILL.md](./theme-convert/SKILL.md)
- [figma-icons/SKILL.md](./figma-icons/SKILL.md)
- [theme-stories/SKILL.md](./theme-stories/SKILL.md)
