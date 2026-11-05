---
name: design2code
type: superpower
description: Design-to-code pipeline - Extract design tokens from Figma, generate multiple output formats, create React icons, and build Storybook stories
tags: [figma, design-tokens, react, storybook, mui, tailwind]
---

# Design2Code - Design-to-Code Pipeline

Complete design-to-code pipeline from Figma Desktop MCP to production-ready code.

## Overview

This superpower orchestrates 4 focused sub-skills to transform Figma designs into usable code:

1. **figma-extract** - Figma (variables or MUI-plugin styles) → theme.ts (design tokens)
2. **theme-convert** - theme.ts → formats (MUI, Tailwind, SCSS, CSS, styled-components)
3. **figma-icons** - Figma → React icons (optional)
4. **theme-stories** - themes → Storybook stories (optional)

## Usage

```bash
# Full pipeline
/design2code <figma-url>

# With format selection
/design2code <figma-url> --format mui,tailwind

# Skip optional features
/design2code <figma-url> --no-icons
/design2code <figma-url> --no-stories

# With validation
/design2code <figma-url> --validate

# With API token fallback
/design2code <figma-url> --api-token <TOKEN>
```

## What It Does

**Complete pipeline:**

```
Figma File
    ↓
[figma-extract] → theme.ts
    ↓
    ├─→ [theme-convert] → theme.mui.ts, tailwind.config.js, theme.scss, etc.
    ├─→ [figma-icons] → icons/*.tsx, icons/index.ts
    └─→ [theme-stories] → Colors.stories.tsx, Typography.stories.tsx, etc.
```

**Output files:**

- `theme.ts` - TypeScript design tokens (base)
- `theme.mui.ts`, `theme.mui.json` - Material-UI theme (full palette + 13 typography variants)
- `theme.mui-plugin.json` - theme JSON importable by the [MUI Figma plugin](https://github.com/mui/hackathon-figma-plugin)
- `tailwind.config.js` - Tailwind CSS config
- `theme.scss` - Sass variables
- `theme.css` - CSS custom properties
- `theme.styled.ts` - styled-components theme
- `icons/*.tsx` - React icon components
- `*.stories.tsx` - Storybook stories

## MUI Figma plugin round-trip

Files set up with the MUI Importer & Exporter plugin store the theme as paint styles (`Primary/Main`, `Primary/Contrast`, `Primary/States/...`, `Text/Primary`, `Divider`, ...) and text styles (`Typography/H1` ... `Typography/Overline`). `figma-extract` reads these in addition to variables (variables win on conflict).

```bash
# Figma (plugin styles) -> theme.ts -> theme.mui.ts + theme.mui-plugin.json
/design2code <figma-url> --format mui,mui-plugin

# Plugin export JSON -> theme.ts -> any format (no Figma access needed)
/design2code --from-mui-json exported-theme.json --format mui,tailwind
```

Caveats of the plugin (a hackathon prototype): typography import only updates text styles that already exist, ignores `lineHeight`/`spacing`/`shape`/`breakpoints`, and fails on function-valued typography or plain `em` font sizes. Generated JSON uses `px` only for that reason. Target: MUI v5+ `createTheme`.

## Parameters

- `figma-url` (required): Figma file URL with node-id
- `--format <formats>` (optional): Comma-separated format list (mui, mui-plugin, tailwind, scss, css, styled, all)
- `--from-mui-json <file>` (optional): Skip Figma extraction; start from a theme JSON exported by the MUI Figma plugin (`{ palette, typography }`)
- `--no-icons` (optional): Skip icon extraction
- `--no-stories` (optional): Skip Storybook story generation
- `--validate` (optional): Run Storybook MCP validation
- `--api-token <TOKEN>` (optional): Figma API token for REST fallback
- `--output <dir>` (optional): Output directory (default: current dir)

## Instructions for Claude

When a user invokes this superpower:

### Step 1: Parse Arguments

```typescript
const {
  figmaUrl,
  formats,
  skipIcons,
  skipStories,
  validate,
  apiToken,
  outputDir
} = parseArgs(userInput);

// Validate Figma URL
if (!figmaUrl || !figmaUrl.includes('figma.com')) {
  throw new Error('Invalid Figma URL. Expected: https://www.figma.com/file/...');
}
```

### Step 2: Extract Design Tokens

Call `/figma-extract` sub-skill:

```typescript
console.log('Step 1/4: Extracting design tokens from Figma...');

const extractArgs = [figmaUrl];
if (apiToken) extractArgs.push('--api-token', apiToken);
if (outputDir) extractArgs.push('--output', `${outputDir}/theme.ts`);

const themeFile = await invokeSkill('figma-extract', extractArgs.join(' '));

if (!themeFile || !await exists(themeFile)) {
  throw new Error('Design token extraction failed - no theme.ts generated');
}

console.log(`✓ Extracted design tokens → ${themeFile}`);
```

### Step 3: Parallel: Icons + Format Conversion

Run these in parallel (independent operations):

```typescript
console.log('Step 2/4: Generating formats and extracting icons...');

const [formatResults, iconResults] = await Promise.allSettled([
  // Format conversion
  (async () => {
    const formatArgs = [themeFile];
    if (formats) formatArgs.push('--format', formats);
    return await invokeSkill('theme-convert', formatArgs.join(' '));
  })(),
  
  // Icon extraction (unless skipped)
  (async () => {
    if (skipIcons) {
      console.log('  ℹ️  Icon extraction skipped (--no-icons)');
      return null;
    }
    
    const iconArgs = [figmaUrl];
    if (apiToken) iconArgs.push('--api-token', apiToken);
    if (outputDir) iconArgs.push('--output', `${outputDir}/icons/`);
    
    try {
      return await invokeSkill('figma-icons', iconArgs.join(' '));
    } catch (error) {
      console.warn('  ⚠️  Icon extraction failed (non-blocking):', error.message);
      return null;
    }
  })(),
]);

// Report results
if (formatResults.status === 'fulfilled') {
  console.log(`✓ Generated ${formatResults.value.count} format(s)`);
} else {
  console.error('✗ Format conversion failed:', formatResults.reason);
}

if (!skipIcons && iconResults.status === 'fulfilled' && iconResults.value) {
  console.log(`✓ Extracted ${iconResults.value.count} icons`);
}
```

### Step 4: Generate Storybook Stories

Call `/theme-stories` sub-skill:

```typescript
console.log('Step 3/4: Generating Storybook stories...');

if (skipStories) {
  console.log('  ℹ️  Story generation skipped (--no-stories)');
} else {
  const storyArgs = [themeFile];
  
  // Include icons if extracted
  const iconsDir = outputDir ? `${outputDir}/icons` : 'icons';
  if (!skipIcons && await exists(iconsDir)) {
    storyArgs.push(iconsDir);
  }
  
  if (validate) storyArgs.push('--validate');
  if (outputDir) storyArgs.push('--output', outputDir);
  
  try {
    const storyResults = await invokeSkill('theme-stories', storyArgs.join(' '));
    console.log(`✓ Generated ${storyResults.count} stories`);
  } catch (error) {
    console.warn('  ⚠️  Story generation failed (non-blocking):', error.message);
  }
}
```

### Step 5: Summary Report

```typescript
console.log('\n' + '='.repeat(60));
console.log('✓ Design2Code Pipeline Complete');
console.log('='.repeat(60));

// List generated files
const files = await listGeneratedFiles(outputDir || '.');

console.log('\nGenerated Files:');
console.log('  Design Tokens:');
console.log(`    ✓ theme.ts (${files.theme.size})`);

if (files.formats.length > 0) {
  console.log('  Formats:');
  files.formats.forEach(f => console.log(`    ✓ ${f.name} (${f.size})`));
}

if (files.icons.length > 0) {
  console.log('  Icons:');
  console.log(`    ✓ ${files.icons.length} icon components`);
}

if (files.stories.length > 0) {
  console.log('  Stories:');
  files.stories.forEach(f => console.log(`    ✓ ${f.name}`));
}

console.log('\nNext Steps:');
console.log('  1. Review generated files');
console.log('  2. Import theme in your application');
console.log('  3. Run Storybook to view stories: npm run storybook');
console.log('  4. Check accessibility: npm run test-storybook');

console.log('\nUsage Examples:');
console.log('  // Import theme');
console.log(`  import { theme } from './${files.theme.name}';`);
console.log('');
console.log('  // Use in React');
console.log('  <button style={{');
console.log('    backgroundColor: theme.colors.primary.main,');
console.log('    padding: theme.spacing.md');
console.log('  }}>');
console.log('    Click me');
console.log('  </button>');

if (files.icons.length > 0) {
  const firstIcon = files.icons[0].replace('.tsx', '');
  console.log('');
  console.log('  // Use icons');
  console.log(`  import { ${firstIcon} } from './icons';`);
  console.log(`  <${firstIcon} size={24} color={theme.colors.primary.main} />`);
}
```

### Error Handling

**Figma MCP Unavailable:**

```
❌ Figma Desktop MCP not available

Resolution steps:
1. Ensure Figma Desktop app is running
2. Open your Figma file
3. Verify MCP server configured in Claude Code settings

Alternative: Use API token fallback
  /design2code <url> --api-token YOUR_TOKEN

Get token: Figma Settings → Account → Personal Access Tokens
```

**Partial Pipeline Failure:**

```
⚠️  Pipeline partially completed (1/3 steps failed)

✓ Step 1: Design token extraction
✓ Step 2: Format conversion (3 formats)
✗ Step 3: Story generation - Storybook not installed

Partial success - review generated files
Stories can be generated later: /theme-stories theme.ts
```

**MUI plugin format issues:**

```
⚠️  theme.mui-plugin.json: 2 typography variants dropped
  - subtitle2: fontSize "1em" (plain em cannot be resolved to px)
  - body1: function-valued typography is not supported by the plugin
Palette exported in full.
```

**Format-Specific Failures:**

```
⚠️  Some formats failed to generate

✓ theme.mui.ts, theme.mui.json (MUI)
✓ theme.scss (SCSS)
✗ tailwind.config.js - Invalid spacing value

2/3 formats generated successfully
```

### Progress Reporting

Show clear progress through pipeline:

```
┌─────────────────────────────────────────────────────────┐
│ Design2Code Pipeline                                    │
│ Source: https://figma.com/file/ABC123/MyDesign          │
└─────────────────────────────────────────────────────────┘

[1/4] Extracting design tokens...
  ├─ Connecting to Figma Desktop MCP
  ├─ Extracting variables (colors, typography, spacing)
  └─ ✓ Generated theme.ts (847 B)

[2/4] Converting formats and extracting icons...
  ├─ Format conversion:
  │  ├─ ✓ theme.mui.ts (1.2 KB)
  │  ├─ ✓ theme.mui.json (847 B)
  │  └─ ✓ tailwind.config.js (623 B)
  └─ Icon extraction:
     ├─ Found 12 icon nodes
     ├─ Batching (3 batches of 4-5 icons)
     └─ ✓ Generated 12 icon components

[3/4] Generating Storybook stories...
  ├─ ✓ Colors.stories.tsx (12 swatches)
  ├─ ✓ Typography.stories.tsx (2 styles)
  ├─ ✓ Spacing.stories.tsx (5 values)
  └─ ✓ Icons.stories.tsx (12 icons)

[4/4] Validation...
  ├─ Storybook MCP available: Yes
  ├─ Running story tests...
  └─ ✓ All stories passed (0 accessibility warnings)

✓ Pipeline complete
```

## Examples

```bash
# Full pipeline with auto-detected formats
/design2code https://www.figma.com/file/ABC123/MyDesign?node-id=60257-25298

# MUI + Tailwind only
/design2code https://www.figma.com/file/ABC123/MyDesign --format mui,tailwind

# Design tokens and formats only (skip icons and stories)
/design2code https://www.figma.com/file/ABC123/MyDesign --no-icons --no-stories

# With validation
/design2code https://www.figma.com/file/ABC123/MyDesign --validate

# Using API token fallback
/design2code https://www.figma.com/file/ABC123/MyDesign --api-token figd_abc123...

# Custom output directory
/design2code https://www.figma.com/file/ABC123/MyDesign --output src/design-system/
```

## Sub-Skills Reference

Each sub-skill can be used independently:

```bash
# Extract design tokens only
/figma-extract <figma-url>

# Convert existing theme.ts
/theme-convert theme.ts --format tailwind

# Extract icons only
/figma-icons <figma-url>

# Generate stories from existing theme
/theme-stories theme.ts icons/
```

## Integration

**In React applications:**

```typescript
// theme.ts or theme.mui.ts
import { theme } from './theme';

function App() {
  return (
    <ThemeProvider theme={theme}>
      <Button color="primary">Click me</Button>
    </ThemeProvider>
  );
}
```

**In Tailwind projects:**

```javascript
// tailwind.config.js is auto-generated
// Use in components:
<button className="bg-primary hover:bg-primary-dark text-white px-md py-sm">
  Click me
</button>
```

**In Sass projects:**

```scss
// Import generated variables
@import './theme.scss';

.button {
  background-color: $primary-main;
  padding: $spacing-md;
}
```

## Troubleshooting

**Q: Pipeline fails at extraction step**
A: Check Figma Desktop is running and file is open. Try --api-token fallback. Verify URL is correct.

**Q: No formats generated**
A: Check package.json has framework dependencies (e.g., @mui/material for MUI). Use --format flag to specify explicitly.

**Q: Icons look broken**
A: Verify Figma URL points to icon frames. Check generated SVG paths. Report if icons consistently fail.

**Q: Stories don't show in Storybook**
A: Install Storybook if not present. Check .storybook/main.js includes .stories.tsx files. Restart dev server.

**Q: MCP validation fails**
A: Ensure Storybook v10.3+ and @storybook/addon-mcp installed. Start dev server before running with --validate.

**Q: Can I customize output?**
A: Yes! Sub-skills can be run independently with custom options. Edit generated files as needed.

**Q: Pipeline takes too long**
A: Large files (50+ variables, 20+ icons) may take 1-2 minutes. Check network connection. Consider extracting specific nodes instead of entire file.

## Architecture

**Sub-Skill Dependency Graph:**

```
figma-extract (independent)
    ↓
    ├─→ theme-convert (uses theme.ts)
    ├─→ figma-icons (independent)
    └─→ theme-stories (uses theme.ts + icons)
```

**Execution Strategy:**
- Extract runs first (produces theme.ts)
- Convert + Icons run in parallel (independent)
- Stories run last (needs theme.ts, optionally icons)

**Error Isolation:**
- Each sub-skill has its own error handling
- Failures are non-blocking where possible
- Pipeline reports partial successes

## Version Compatibility

- **Figma Desktop MCP:** Any version
- **Storybook:** v7+ (CSF 3.0), v10.3+ for MCP validation
- **Material-UI:** v5+
- **Tailwind CSS:** v3+
- **React:** v16.8+ (hooks)
- **TypeScript:** v4+

## Contributing

This superpower is composed of 4 independent sub-skills. To improve:

1. Identify which sub-skill needs changes
2. Edit the relevant SKILL.md file
3. Test sub-skill independently
4. Test full pipeline via this orchestrator

**Sub-skill locations:**
- `design2code/figma-extract/SKILL.md`
- `design2code/theme-convert/SKILL.md`
- `design2code/figma-icons/SKILL.md`
- `design2code/theme-stories/SKILL.md`
