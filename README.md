# Design2Code Superpower

> Transform Figma designs into production-ready code: design tokens → multiple formats → React icons → Storybook stories

## Overview

**design2code** is a superpower (collection of focused skills) that automates the design-to-code pipeline:

```
Figma File → Design Tokens → Multi-Format Output → React Icons → Storybook Stories
```

**What it generates:**
- Design tokens (colors, typography, spacing)
- Framework-specific themes (MUI, Tailwind, SCSS, CSS, styled-components)
- React icon components with proper props and accessibility
- Storybook stories showcasing your design system

## Quick Start

```bash
# Full pipeline
/design2code https://www.figma.com/file/YOUR_FILE_ID/YourDesign?node-id=NODE_ID

# With format selection
/design2code https://figma.com/file/... --format mui,tailwind

# Skip optional features
/design2code https://figma.com/file/... --no-icons --no-stories
```

## Installation

Symlink the `skill/` directory of this repo into Claude Code, for example `ln -s <path-to-repo>/skill ~/.claude/skills/design2code`.

**Prerequisites:**
- **Figma Desktop MCP** (preferred) OR Figma API token
- Node.js 16+ (for TypeScript validation)
- Optional: Storybook 7+ (for story viewing)

**Figma Desktop MCP Setup:**
1. Install Figma Desktop app
2. Configure MCP server in Claude Code settings
3. Open your Figma file in Desktop app

**Figma API Token (fallback):**
1. Go to Figma Settings → Account → Personal Access Tokens
2. Generate new token
3. Use with `--api-token` flag

## Sub-Skills

This superpower orchestrates 4 independent sub-skills:

| Skill | Purpose | Can Use Standalone |
|-------|---------|-------------------|
| `/figma-extract` | Figma → theme.ts | ✅ Yes |
| `/theme-convert` | theme.ts → formats | ✅ Yes |
| `/figma-icons` | Figma → React icons | ✅ Yes |
| `/theme-stories` | themes → Storybook | ✅ Yes |

## Usage

### Full Pipeline

```bash
/design2code <figma-url>
```

Runs complete pipeline:
1. Extracts design tokens → `theme.ts`
2. Generates format files (auto-detected from package.json)
3. Extracts icons → `icons/*.tsx`
4. Generates Storybook stories

### Format Selection

```bash
# MUI only
/design2code <figma-url> --format mui

# Multiple formats
/design2code <figma-url> --format mui,tailwind,scss

# All formats
/design2code <figma-url> --format all
```

**Available formats:**
- `mui` - Material-UI (theme.mui.ts + theme.mui.json)
- `mui-plugin` - MUI Figma plugin import JSON (theme.mui-plugin.json)
- `tailwind` - Tailwind CSS (tailwind.config.js)
- `scss` - Sass variables (theme.scss)
- `css` - CSS custom properties (theme.css)
- `styled` - styled-components (theme.styled.ts)

### Optional Features

```bash
# Skip icons
/design2code <figma-url> --no-icons

# Skip stories
/design2code <figma-url> --no-stories

# Design tokens and formats only
/design2code <figma-url> --no-icons --no-stories
```

### With Validation

```bash
# Run Storybook MCP validation
/design2code <figma-url> --validate
```

Requires:
- Storybook v10.3+
- @storybook/addon-mcp installed
- Storybook dev server running

### API Token Fallback

```bash
# Use REST API instead of MCP
/design2code <figma-url> --api-token figd_abc123...
```

Use when:
- Figma Desktop MCP unavailable
- Working in headless environment
- Desktop app not installed

## MUI Figma Plugin Support

design2code follows the conventions of the [MUI Importer & Exporter](https://github.com/mui/hackathon-figma-plugin) Figma plugin, so a design file prepared with the plugin can feed the pipeline and a generated theme can be loaded back into Figma.

**What the plugin expects in the Figma file**
- Palette: local *paint styles* named `Group/Field` (`Primary/Main`, `Primary/Contrast`, `Text/Primary`, `Divider`, ...) for `primary secondary error warning info success common grey text divider background action`. Importing a palette creates these styles, plus `Group/States/*` for the six palette groups.
- Typography: local *text styles* named `Typography/H1` ... `Typography/Overline` (13 variants: h1-h6, subtitle1/2, body1/2, button, caption, overline). **These must already exist**: importing typography only updates them and does nothing if they are missing.

**One-time plugin setup** (the plugin is a hackathon prototype with no published release; these steps come from its repo layout, `package.json`, and `manifest.json`, and were not run end to end here)
1. `git clone https://github.com/mui/hackathon-figma-plugin.git && cd hackathon-figma-plugin`
   (A fork, [gilbertanderson/mui-figma-sync-plugin](https://github.com/gilbertanderson/mui-figma-sync-plugin), exists for AI-assisted improvements; at the time of writing its `master` has the same code as upstream, so use the MUI repo until the fork diverges.)
2. `yarn install && yarn build` (webpack writes `dist/code.js` and `dist/ui.html`)
3. In Figma Desktop: Plugins → Development → Import plugin from manifest... → choose `manifest.json`
4. Run it from Plugins → Development → MUI Importer & Exporter. It shows two buttons: **Upload theme** (takes a `.json` file) and **Export theme** (downloads `data.json`). The plugin closes after each action, so reopen it for the next one.

**Workflow A: Figma file → code**
1. In the plugin, click **Export theme**; save the download, for example as `exported-theme.json`.
2. `/design2code --from-mui-json exported-theme.json --format mui,tailwind` (no Figma access needed), or run `/design2code <figma-url>` and the extractor reads the same paint/text styles directly.

**Workflow B: code/tokens → Figma file**
1. `/design2code <figma-url> --format mui,mui-plugin` (or `/theme-convert theme.ts --format mui-plugin`) writes `theme.mui-plugin.json`.
2. Make sure the 13 `Typography/*` text styles exist in the target file.
3. Open the plugin, click **Upload theme**, choose `theme.mui-plugin.json`. A notification confirms the import.

**Limits and troubleshooting**
- Typography import ignores `lineHeight`, `spacing`, `shape`, `breakpoints`, and `palette.mode`; they never round-trip through Figma. design2code does still read line height from the text styles.
- The plugin fails on function-valued typography and on plain `em` font sizes, so generated JSON uses `px` only.
- Fonts used in the theme must be available in Figma, or typography import errors.
- Colours must be hex, `rgb(a)`, or `hsl(a)`; named colours and `var(--x)` fail for that style.
- Palette keys outside the supported list, and style names deeper than two levels, are ignored.
- Examples: `skill/examples/mui-plugin/` (fixtures from the plugin repo). See `SUPERPOWER.md` for pipeline flags.

## Output Files

After running `/design2code`, you'll have:

```
project/
├── theme.ts                    # Base TypeScript design tokens
├── theme.mui.ts                # Material-UI theme
├── theme.mui.json              # Material-UI JSON
├── theme.mui-plugin.json       # Import file for the MUI Figma plugin
├── tailwind.config.js          # Tailwind config
├── theme.scss                  # Sass variables
├── theme.css                   # CSS custom properties
├── theme.styled.ts             # styled-components
├── icons/
│   ├── SearchIcon.tsx          # Icon components
│   ├── CloseIcon.tsx
│   └── index.ts                # Barrel export
├── Colors.stories.tsx          # Storybook stories
├── Typography.stories.tsx
├── Spacing.stories.tsx
└── Icons.stories.tsx
```

## Using Generated Files

### TypeScript Theme

```typescript
import { theme } from './theme';

function MyComponent() {
  return (
    <button style={{
      backgroundColor: theme.colors.primary.main,
      padding: theme.spacing.md,
      fontFamily: theme.typography.fontFamily,
    }}>
      Click me
    </button>
  );
}
```

### Material-UI

```typescript
import { ThemeProvider } from '@mui/material/styles';
import { theme } from './theme.mui';

function App() {
  return (
    <ThemeProvider theme={theme}>
      <Button color="primary">MUI Button</Button>
    </ThemeProvider>
  );
}
```

### Tailwind CSS

```jsx
// tailwind.config.js is auto-generated
// Use utility classes:
<button className="bg-primary hover:bg-primary-dark text-white px-md py-sm">
  Tailwind Button
</button>
```

### Icons

```typescript
import { SearchIcon, CloseIcon } from './icons';

<SearchIcon size={24} color={theme.colors.primary.main} aria-label="Search" />
<CloseIcon size={16} color="currentColor" />
```

### Storybook

```bash
# View generated stories
npm run storybook

# Navigate to "Design Tokens" section
# See: Colors, Typography, Spacing, Icons
```

## Standalone Sub-Skills

Use individual skills for specific tasks:

### Extract Design Tokens Only

```bash
/figma-extract https://figma.com/file/...
```

Generates: `theme.ts`

### Convert Existing Theme

```bash
/theme-convert theme.ts --format tailwind
```

Works on any `theme.ts` file (not just from Figma).

### Extract Icons Only

```bash
/figma-icons https://figma.com/file/...
```

Generates: `icons/*.tsx`

### Generate Stories

```bash
/theme-stories theme.ts icons/
```

Generates: `*.stories.tsx`

## Error Handling

The superpower includes comprehensive error handling:

### MCP Unavailable

```
❌ Figma Desktop MCP not available

Resolution:
  1. Ensure Figma Desktop app is running
  2. Open your Figma file
  3. Check MCP config in Claude Code settings
  
Fallback: /design2code <url> --api-token YOUR_TOKEN
```

### Partial Failures

```
⚠️  Pipeline partially completed

✓ Design tokens extracted
✓ Formats generated (2/3)
✗ Stories failed - Storybook not installed

Partial success - review generated files
```

### Format-Specific Errors

```
⚠️  Generated 2/3 formats

✓ theme.mui.ts
✓ tailwind.config.js
✗ theme.scss - Parse error

2 formats ready to use
```

## Troubleshooting

### Figma Connection Issues

**Problem:** MCP connection fails

**Solutions:**
1. Verify Figma Desktop is running
2. Check file is open in Desktop app
3. Review MCP config: `~/.config/claude/mcp.json`
4. Try API token fallback: `--api-token`

### No Formats Generated

**Problem:** Only theme.ts created, no format files

**Solutions:**
1. Check package.json has framework dependencies
2. Specify formats explicitly: `--format mui,tailwind`
3. Use `--format all` to generate everything

### Stories Don't Load

**Problem:** Stories not visible in Storybook

**Solutions:**
1. Install Storybook: `npx storybook@latest init`
2. Check `.storybook/main.js` includes `*.stories.tsx`
3. Restart dev server: `npm run storybook`
4. Verify imports are correct

### Icons Look Broken

**Problem:** Icon SVGs not rendering correctly

**Solutions:**
1. Check Figma URL points to icon frames/components
2. Verify viewBox is correct in generated components
3. Test with simple color: `<Icon color="#FF0000" />`
4. Report persistent issues

### Validation Fails

**Problem:** `--validate` flag doesn't work

**Solutions:**
1. Ensure Storybook v10.3+: `npm list storybook`
2. Install MCP addon: `npx storybook add @storybook/addon-mcp`
3. Start dev server: `npm run storybook`
4. Check server running on port 6006

## Performance

**Small files (≤10 variables, ≤5 icons):**
- Execution time: 10-20 seconds
- No subagent batching

**Large files (>50 variables, >15 icons):**
- Execution time: 60-120 seconds
- Parallel subagent batching
- Progress updates during execution

**Tips for faster execution:**
- Extract specific nodes instead of entire file
- Skip optional features if not needed
- Use --format flag to limit output formats

## Examples

### Design System Extraction

```bash
# Extract complete design system
/design2code https://figma.com/file/ABC123/DesignSystem?node-id=1-2 --format all

# Generates all formats + icons + stories
# ~30 seconds for typical design system
```

### Component Library

```bash
# Extract design tokens + MUI theme + icons
/design2code https://figma.com/file/ABC123/Components --format mui --no-stories

# Skip stories for component-only library
# Focus on theme and icon extraction
```

### Tailwind Project

```bash
# Extract for Tailwind project
/design2code https://figma.com/file/ABC123/TailwindDesign --format tailwind,css

# Generates tailwind.config.js + CSS fallback
# No MUI or styled-components
```

### Design Token Update Workflow

```bash
# 1. Extract latest design tokens
/figma-extract https://figma.com/file/ABC123/Tokens

# 2. Convert to formats
/theme-convert theme.ts --format mui,tailwind

# 3. Regenerate stories
/theme-stories theme.ts icons/ --validate

# Incremental updates without full pipeline
```

## Contributing

Found a bug or want to improve a sub-skill?

1. Identify affected sub-skill
2. Edit relevant `SKILL.md` file:
   - `figma-extract/SKILL.md`
   - `theme-convert/SKILL.md`
   - `figma-icons/SKILL.md`
   - `theme-stories/SKILL.md`
3. Test sub-skill independently
4. Test full pipeline

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for technical details.

## Version Compatibility

| Tool | Minimum Version | Recommended |
|------|----------------|-------------|
| Figma Desktop MCP | Any | Latest |
| Storybook | v7.0 (CSF 3.0) | v10.3+ (for MCP) |
| Material-UI | v5.0 | v5.latest |
| Tailwind CSS | v3.0 | v3.latest |
| React | v16.8 (hooks) | v18+ |
| TypeScript | v4.0 | v5+ |
| Node.js | v16 | v20+ |

## License

Part of Claude Code skills - see main repository for license.

## Support

- **Documentation:** This README + sub-skill SKILL.md files
- **Issues:** Report via Claude Code issue tracker
- **Examples:** See `examples/` directory

---

**Built with Claude Code** | [GitHub](https://github.com/anthropics/claude-code)
