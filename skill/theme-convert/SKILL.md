---
name: theme-convert
description: Convert theme.ts to multiple output formats (MUI, Tailwind, SCSS, CSS, styled-components)
tags: [design-tokens, conversion, mui, tailwind, scss, css]
---

# Theme Convert - Format Conversion

Convert `theme.ts` design tokens to multiple framework-specific formats.

## Usage

```bash
/theme-convert theme.ts
/theme-convert theme.ts --format mui
/theme-convert theme.ts --format mui,tailwind,scss
/theme-convert theme.ts --format all
```

## What It Does

1. Reads theme.ts (TypeScript design tokens)
2. Auto-detects formats from package.json (or uses --format flag)
3. Generates format-specific files in parallel
4. Reports successes and failures independently

## Parameters

- `theme-file` (required): Path to theme.ts file
- `--format <formats>` (optional): Comma-separated format list or 'all'
  - `mui` - Material-UI (theme.mui.ts + theme.mui.json)
  - `mui-plugin` - theme JSON in the import format of the [MUI Figma plugin](https://github.com/mui/hackathon-figma-plugin) (theme.mui-plugin.json)
  - `tailwind` - Tailwind CSS (tailwind.config.js)
  - `scss` - Sass variables (theme.scss)
  - `css` - CSS custom properties (theme.css)
  - `styled` - styled-components (theme.styled.ts)
  - `all` - Generate all formats

## Output

Generates format files based on selection:

- **MUI:** `theme.mui.ts`, `theme.mui.json`
- **MUI Figma plugin:** `theme.mui-plugin.json`
- **Tailwind:** `tailwind.config.js`
- **SCSS:** `theme.scss`
- **CSS:** `theme.css`
- **styled-components:** `theme.styled.ts`

## Instructions for Claude

When a user invokes this skill:

### Step 1: Read Theme File

Load the module instead of regex-parsing it (nested objects break regexes, and never `eval` file contents):

```bash
npx tsx -e "import('./theme.ts').then(m => console.log(JSON.stringify({ colors: m.colors, typography: m.typography, spacing: m.spacing })))"
```

```typescript
const tokens = JSON.parse(stdout); // { colors, typography, spacing }

if (!tokens.colors) {
  throw new Error('Invalid theme.ts - missing colors export');
}
// typography/spacing may be absent (e.g. a palette-only Figma file): generate what exists
```

**`--from-mui-json <file>`** (reverse direction): read a theme JSON exported by the MUI Figma plugin (`{ palette, typography }`) and write a `theme.ts` in the token shape of `figma-extract`. Mapping: `palette.<group>.contrastText` -> `colors.<group>.contrast`; string palette entries (`divider`) stay strings; each `typography.<variant>` -> `typography.variants.<variant>`; other typography keys (`fontFamily`, `htmlFontSize`) stay at the root. Colours are kept exactly as exported (`rgba(r, g, b, a)` strings). Then continue with the normal flow.

### Step 2: Auto-Detect Formats

If no --format flag provided, check package.json:

```typescript
let formats = [];

if (hasFormatFlag) {
  formats = formatFlag.split(',');
} else {
  // Auto-detect
  const pkgJson = JSON.parse(await readFile('package.json', 'utf-8'));
  const deps = { ...pkgJson.dependencies, ...pkgJson.devDependencies };
  
  if (deps['@mui/material']) formats.push('mui');
  if (deps['tailwindcss']) formats.push('tailwind');
  if (deps['sass'] || deps['node-sass']) formats.push('scss');
  if (deps['styled-components']) formats.push('styled');
  
  // Always generate CSS as fallback
  if (formats.length === 0) formats.push('css');
}

if (formats.includes('all')) {
  formats = ['mui', 'mui-plugin', 'tailwind', 'scss', 'css', 'styled'];
}
```

### Step 3: Generate Formats in Parallel

Each format generator is isolated - failures don't block others:

```typescript
const generators = {
  mui: () => generateMuiTheme(tokens),
  'mui-plugin': () => generateMuiPlugin(tokens),
  tailwind: () => generateTailwind(tokens),
  scss: () => generateScss(tokens),
  css: () => generateCss(tokens),
  styled: () => generateStyled(tokens),
};

const results = await Promise.allSettled(
  formats.map(format => 
    generators[format]().then(content => ({ format, content }))
  )
);

const succeeded = results.filter(r => r.status === 'fulfilled');
const failed = results.filter(r => r.status === 'rejected');
```

### Step 4: Format Generators

#### MUI Generator

Follows the contract of the [MUI Figma plugin](https://github.com/mui/hackathon-figma-plugin): palette groups `primary secondary error warning info success common grey text divider background action`, `contrast` -> `contrastText`, and the 13 typography variants `h1-h6 subtitle1 subtitle2 body1 body2 button caption overline`. Rules:

- Emit every group/field present in `theme.ts`; never add values that are not there (no default `contrastText`). When `contrast`/`light`/`dark` are missing, MUI derives them in `createTheme` (`augmentColor`, `getContrastText`) - say so in the report.
- `colors.<group>.states` (Outlined Resting Border, Contained Hover Background, Outlined Hover Background) is **not** written to the palette: MUI computes these from `main`, `dark` and `action.hoverOpacity`. If Figma values differ from the derived ones, report the difference instead of silently overriding.
- Typography: root `fontFamily`/`htmlFontSize`/`fontSize`/`fontWeight*` pass through; variants come from `typography.variants` (legacy `heading` -> `h1`, `body` -> `body1` only when no `variants`). `rem` is converted with `htmlFontSize`, `em` letter spacing with the variant's own font size; plain-`em` font sizes are dropped (unknowable) and reported.
- `mode`, `contrastThreshold`, `tonalOffset` are not part of the plugin contract and are not carried by Figma; set `mode` yourself for dark themes.
- Do not emit `spacing`/`shape`/`breakpoints` (no invented `spacing: 8`): Figma tokens named `xs..xl` do not map onto MUI's numeric spacing unit; leave to MUI defaults.

Tested against the plugin's fixtures (light, dark, custom-theme1/2, typography test): every output passes `createTheme()` and `tsc --noEmit`.

```typescript
const PALETTE_GROUPS = ['primary', 'secondary', 'error', 'warning', 'info', 'success'];
const PALETTE_KEYS = [...PALETTE_GROUPS, 'common', 'grey', 'text', 'divider', 'background', 'action'];
const VARIANTS = ['h1','h2','h3','h4','h5','h6','subtitle1','subtitle2','body1','body2','button','caption','overline'];
const WEIGHT_NAMES: Record<string, number> = { Thin: 100, ExtraLight: 200, Light: 300, Regular: 400, Medium: 500, SemiBold: 600, Bold: 700, ExtraBold: 800, Black: 900 };

// "1.5rem" / "24px" / 24 -> "24px". Plain `em` needs the parent size, so it is not guessed.
function toPx(value: unknown, htmlFontSize = 16): string | undefined {
  if (typeof value === 'number') return `${value}px`;
  if (typeof value !== 'string') return undefined;
  if (value.endsWith('px')) return value;
  if (value.endsWith('rem')) return `${parseFloat(value) * htmlFontSize}px`;
  return undefined;
}

// letterSpacing: px and % pass through; em is relative to the variant's own font size.
function toLetterSpacing(value: unknown, fontSizePx?: string): string | undefined {
  if (typeof value !== 'string') return undefined;
  if (value.endsWith('px') || value.endsWith('%')) return value;
  if (value.endsWith('rem')) return `${parseFloat(value) * 16}px`;
  if (value.endsWith('em') && fontSizePx) return `${+(parseFloat(value) * parseFloat(fontSizePx)).toFixed(3)}px`;
  return undefined;
}

function toMuiPalette(colors: Record<string, any>) {
  const palette: Record<string, any> = {};
  for (const [group, value] of Object.entries(colors)) {
    if (!PALETTE_KEYS.includes(group)) continue; // plugin and MUI ignore anything else
    if (typeof value === 'string') { palette[group] = value; continue; }
    const out: Record<string, any> = {};
    for (const [field, color] of Object.entries(value)) {
      if (field === 'states') continue; // derived by MUI from main/dark/action.hoverOpacity
      if (typeof color !== 'string' && typeof color !== 'number') continue;
      out[field === 'contrast' ? 'contrastText' : field] = color;
    }
    palette[group] = out;
  }
  return palette;
}

function toMuiTypography(typography: Record<string, any> = {}) {
  const { variants = {}, heading, body, fontFamily, htmlFontSize = 16 } = typography;
  const out: Record<string, any> = {};
  if (fontFamily) out.fontFamily = fontFamily;
  for (const k of ['htmlFontSize', 'fontSize', 'fontWeightLight', 'fontWeightRegular', 'fontWeightMedium', 'fontWeightBold']) {
    if (typeof typography[k] === 'number') out[k] = typography[k]; // base values, kept as-is
  }
  const src: Record<string, any> = { ...variants };
  if (!Object.keys(variants).length) { // legacy heading/body tokens
    if (heading) src.h1 = heading;
    if (body) src.body1 = body;
  }
  for (const key of VARIANTS) {
    const v = src[key];
    if (!v) continue;
    const fontSize = toPx(v.fontSize, htmlFontSize);
    const weight = typeof v.fontWeight === 'string' ? WEIGHT_NAMES[v.fontWeight] ?? Number(v.fontWeight) : v.fontWeight;
    const entry = {
      fontFamily: v.fontFamily,
      fontSize,
      fontWeight: Number.isFinite(weight) ? weight : undefined,
      lineHeight: typeof v.lineHeight === 'string' && !isNaN(Number(v.lineHeight)) ? Number(v.lineHeight) : v.lineHeight,
      letterSpacing: toLetterSpacing(v.letterSpacing, fontSize),
    };
    out[key] = Object.fromEntries(Object.entries(entry).filter(([, x]) => x !== undefined));
  }
  return out;
}

const toPluginJson = (tokens: any) => ({ palette: toMuiPalette(tokens.colors ?? {}), typography: toMuiTypography(tokens.typography) });

// reverse: plugin export / fixture JSON -> token shape
function fromMuiJson(json: any) {
  const colors: Record<string, any> = {};
  for (const [g, v] of Object.entries<any>(json.palette ?? {})) {
    if (!PALETTE_KEYS.includes(g)) continue;
    if (typeof v === 'string') { colors[g] = v; continue; }
    colors[g] = Object.fromEntries(Object.entries(v).map(([k, c]) => [k === 'contrastText' ? 'contrast' : k, c]));
  }
  const typography: Record<string, any> = { variants: {} };
  for (const [k, v] of Object.entries<any>(json.typography ?? {})) {
    if (VARIANTS.includes(k)) typography.variants[k] = v; else typography[k] = v;
  }
  return { colors, typography };
}

function generateMuiTheme(tokens) {
  const themeOptions = {
    palette: toMuiPalette(tokens.colors ?? {}),
    typography: toMuiTypography(tokens.typography),
  };
  const json = JSON.stringify(themeOptions, null, 2);
  return {
    ts: `// theme.mui.ts - generated from theme.ts
import { createTheme, ThemeOptions } from '@mui/material/styles';

export const themeOptions: ThemeOptions = ${json};

export const theme = createTheme(themeOptions);
export default theme;
`,
    json,
  };
}
```

#### MUI Figma Plugin Generator (`--format mui-plugin`)

Writes `theme.mui-plugin.json` = `{ palette, typography }` from `toPluginJson(tokens)` above, ready for the plugin's **Upload theme** button (it takes a `.json` file, not pasted text). Plugin behaviour to respect:

- Palette import creates/updates paint styles (`Primary/Main`, `Primary/Contrast`, ...) and generates the three `States` styles itself.
- Typography import only **updates existing** `Typography/<Variant>` text styles (it does not create them) - the Figma file needs those 13 styles first or the typography section is a silent no-op.
- It needs fonts available in Figma and throws on function-valued typography, and on `em` font sizes - hence the generator emits `px` only.
- It ignores `lineHeight`, `mode`, `contrastThreshold`, `tonalOffset`, `spacing`, `shape`, `breakpoints`.
- Colours are parsed with MUI `decomposeColor` (hex, rgb(a), hsl(a)); anything else (named colours, `var(--x)`) fails per style.

```typescript
function generateMuiPlugin(tokens) {
  return JSON.stringify(toPluginJson(tokens), null, 2) + '\n';
}
```

**Round-trip check** (run after generating): `fromMuiJson(JSON.parse(output))` must equal `tokens` for palette keys, and a second pass through `toPluginJson` must be byte-identical (the first pass normalises `rem`/`em` to `px`).

#### Tailwind Generator

```typescript
function generateTailwind(tokens) {
  return `// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: {
          DEFAULT: '${tokens.colors.primary.main}',
          light: '${tokens.colors.primary.light}',
          dark: '${tokens.colors.primary.dark}',
        },
        secondary: {
          DEFAULT: '${tokens.colors.secondary.main}',
          light: '${tokens.colors.secondary.light}',
          dark: '${tokens.colors.secondary.dark}',
        },
      },
      fontFamily: {
        sans: ${JSON.stringify(tokens.typography.fontFamily.split(',').map(f => f.trim()))},
      },
      spacing: ${JSON.stringify(tokens.spacing, null, 8)},
    },
  },
  plugins: [],
};
`;
}
```

#### SCSS Generator

```typescript
function generateScss(tokens) {
  let scss = '// theme.scss - Sass variables\n\n';
  
  // Colors
  scss += '// Colors\n';
  for (const [key, value] of Object.entries(tokens.colors.primary || {})) {
    scss += `$primary-${key}: ${value};\n`;
  }
  // ... repeat for other color groups
  
  // Typography
  scss += '\n// Typography\n';
  scss += `$font-family: ${tokens.typography.fontFamily};\n`;
  // ...
  
  // Spacing
  scss += '\n// Spacing\n';
  for (const [key, value] of Object.entries(tokens.spacing || {})) {
    scss += `$spacing-${key}: ${value};\n`;
  }
  
  return scss;
}
```

#### CSS Generator

```typescript
function generateCss(tokens) {
  let css = '/* theme.css - CSS Custom Properties */\n:root {\n';
  
  // Colors
  css += '  /* Colors */\n';
  for (const [group, colors] of Object.entries(tokens.colors || {})) {
    for (const [key, value] of Object.entries(colors)) {
      css += `  --color-${group}-${key}: ${value};\n`;
    }
  }
  
  // Typography
  css += '\n  /* Typography */\n';
  css += `  --font-family: ${tokens.typography.fontFamily};\n`;
  // ...
  
  // Spacing
  css += '\n  /* Spacing */\n';
  for (const [key, value] of Object.entries(tokens.spacing || {})) {
    css += `  --spacing-${key}: ${value};\n`;
  }
  
  css += '}\n';
  return css;
}
```

#### styled-components Generator

```typescript
function generateStyled(tokens) {
  return `// theme.styled.ts - styled-components theme
import { DefaultTheme } from 'styled-components';

export const theme: DefaultTheme = ${JSON.stringify(tokens, null, 2)};

export default theme;
`;
}
```

### Step 5: Write Files

```typescript
for (const result of succeeded) {
  const { format, content } = result.value;
  
  if (format === 'mui') {
    writeFile('theme.mui.ts', content.ts);
    writeFile('theme.mui.json', content.json);
  } else if (format === 'mui-plugin') {
    writeFile('theme.mui-plugin.json', content);
  } else if (format === 'tailwind') {
    writeFile('tailwind.config.js', content);
  } else if (format === 'scss') {
    writeFile('theme.scss', content);
  } else if (format === 'css') {
    writeFile('theme.css', content);
  } else if (format === 'styled') {
    writeFile('theme.styled.ts', content);
  }
}
```

### Step 6: Report Results

```
✓ Generated 3/4 formats successfully

Generated:
  ✓ theme.mui.ts (1.2 KB)
  ✓ theme.mui.json (847 B)
  ✓ tailwind.config.js (623 B)

Failed:
  ✗ theme.scss - Parse error: Invalid color value

Next steps:
  - Review generated files
  - Import in your application
  - Use /theme-stories to generate Storybook stories
```

## Error Handling

### Invalid Theme File

```
❌ Invalid theme.ts

Expected exports:
  export const colors = { ... };
  export const typography = { ... };
  export const spacing = { ... };

Found in theme.ts:
  ✓ colors
  ✗ typography (missing)
  ✗ spacing (missing)
```

### Format Generation Failure

```
⚠️  Generated 2/3 formats (1 failed)

✓ theme.mui.ts
✓ theme.mui.json
✗ tailwind.config.js - Error: Invalid spacing value '16' (expected '16px')

Partial success - review generated files
```

### No Formats Detected

```
ℹ️  No frameworks detected in package.json

Auto-detected formats: none
Generating CSS as fallback

To specify formats:
  /theme-convert theme.ts --format mui,tailwind
```

## Examples

```bash
# Auto-detect from package.json
/theme-convert theme.ts

# MUI only
/theme-convert theme.ts --format mui

# Multiple formats
/theme-convert theme.ts --format mui,tailwind,scss

# All formats
/theme-convert theme.ts --format all

# From custom path
/theme-convert src/design/theme.ts --format mui
```

## Integration

Part of `/design2code` superpower. Can be used:

1. **Standalone:** Convert existing theme.ts
2. **Pipeline:** Called by `/design2code` after `/figma-extract`

Output files are consumed by:
- Your application (import theme)
- `/theme-stories` (generate Storybook stories)

## Troubleshooting

**Q: Format not generated**
A: Check package.json has the framework dependency. Use --format flag to specify explicitly.

**Q: Invalid TypeScript in generated file**
A: Report this - generated code should always be valid. Run `npx tsc --noEmit theme.mui.ts` to see errors.

**Q: Tailwind config not working**
A: Ensure it's in project root or referenced in tailwind.config.js. Check Tailwind documentation for setup.

**Q: Can I convert existing theme file?**
A: Yes! This skill works on any theme.ts file, not just from /figma-extract.
