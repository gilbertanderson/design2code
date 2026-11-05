---
name: figma-extract
description: Extract design tokens (colors, typography, spacing) from Figma using Desktop MCP or REST API fallback
tags: [figma, design-tokens, mcp, extraction]
---

# Figma Extract - Design Token Extraction

Extract design tokens from Figma files using Figma Desktop MCP (preferred) or REST API fallback.

## Usage

```bash
/figma-extract <figma-url>
/figma-extract <figma-url> --api-token <TOKEN>
/figma-extract <figma-url> --output theme.ts
```

## What It Does

1. Parses Figma URL to extract file-id and node-id
2. Attempts connection to Figma Desktop MCP
3. Falls back to REST API if MCP unavailable (requires --api-token)
4. Extracts design variables: colors, typography, spacing
5. Generates TypeScript theme file with proper types
6. Never hallucinates values - all data from Figma

## Parameters

- `figma-url` (required): Full Figma URL with file and node-id
- `--api-token <TOKEN>` (optional): Figma personal access token for API fallback
- `--output <path>` (optional): Output file path (default: theme.ts)

## Output

Generates `theme.ts`:

```typescript
// theme.ts - Extracted from Figma
export const colors = {
  // MUI-plugin paint styles add `contrast`, `states`, and the groups error/warning/info/success/grey/common/background/action/divider
  primary: { main: '#0066CC', light: '#3385D6', dark: '#004C99', contrast: '#FFFFFF' },
  secondary: { main: '#6C757D', light: '#ADB5BD', dark: '#495057' },
  text: { primary: '#212529', secondary: '#6C757D', disabled: '#ADB5BD' },
};

export const typography = {
  fontFamily: 'Inter, sans-serif',
  heading: { fontSize: '24px', fontWeight: 600, lineHeight: '1.2' },
  body: { fontSize: '16px', fontWeight: 400, lineHeight: '1.5' },
  // Present when the file uses MUI-plugin text styles (Typography/H1 ... Overline), see Step 4b
  variants: {
    h1: { fontFamily: 'Inter', fontSize: '96px', fontWeight: 300, letterSpacing: '-1.5px' },
    body1: { fontFamily: 'Inter', fontSize: '16px', fontWeight: 400, letterSpacing: '0.15px' },
  },
};

export const spacing = {
  xs: '4px', sm: '8px', md: '16px', lg: '24px', xl: '32px',
};

export const theme = { colors, typography, spacing };
export default theme;
```

## Instructions for Claude

When a user invokes this skill:

### Step 1: Parse Figma URL

Extract file-id and node-id from URL:

```typescript
// Example: https://www.figma.com/file/ABC123/MyDesign?node-id=60257-25298
// file-id: ABC123
// node-id: 60257-25298
```

If URL is invalid or missing node-id, show error with correct format example.

### Step 2: Check MCP Connection

Try Figma Desktop MCP first:

```typescript
try {
  await mcp.get_metadata({ nodeId: extractedNodeId });
  // MCP available - use it
  useMcp = true;
} catch (error) {
  // MCP unavailable
  if (hasApiToken) {
    // Use REST API fallback
    useMcp = false;
  } else {
    throw new Error(
      `Figma Desktop MCP not available.\n\n` +
      `Resolution steps:\n` +
      `1. Ensure Figma Desktop app is running\n` +
      `2. Open your Figma file in the Desktop app\n` +
      `3. Verify MCP server configured: Settings → MCP Servers → figma-desktop\n\n` +
      `Alternative: Use --api-token flag with Figma personal access token\n` +
      `Get token: Figma Settings → Account → Personal Access Tokens`
    );
  }
}
```

### Step 3: Extract Variables

**If using MCP:**

```typescript
const variables = await mcp.get_variable_defs({
  nodeId: extractedNodeId,
  clientLanguages: 'typescript',
  clientFrameworks: 'react'
});
```

**If using REST API:**

```typescript
const response = await fetch(
  `https://api.figma.com/v1/files/${fileId}/variables/local`,
  { headers: { 'X-Figma-Token': apiToken } }
);
const variables = await response.json();
```

**Retry logic (3 attempts with exponential backoff):**

```typescript
let attempts = 0;
while (attempts < 3) {
  try {
    const variables = await extractVariables();
    break;
  } catch (error) {
    attempts++;
    if (attempts === 3) throw error;
    
    const delay = Math.pow(2, attempts - 1) * 1000; // 1s, 2s
    console.log(`Extraction failed, retrying in ${delay/1000}s... (${attempts}/3)`);
    await sleep(delay);
  }
}
```

### Step 4: Parse and Organize Design Tokens

Group variables by collection/category:

```typescript
const tokens = {
  colors: {},
  typography: {},
  spacing: {}
};

for (const variable of variables) {
  const collection = variable.collectionName.toLowerCase();
  const name = variable.name;
  const value = variable.resolvedValue;
  
  // Map to semantic structure
  if (collection.includes('color')) {
    // Parse name like "primary/main" → colors.primary.main
    tokens.colors[...] = value;
  } else if (collection.includes('typography') || collection.includes('font')) {
    tokens.typography[...] = value;
  } else if (collection.includes('spacing') || collection.includes('space')) {
    tokens.spacing[...] = value;
  }
}
```

**Critical: Use actual values from Figma - never generate placeholders.**

### Step 4b: Detect MUI-plugin style layout (paint/text styles)

Files prepared with the [MUI Importer & Exporter plugin](https://github.com/mui/hackathon-figma-plugin) carry the theme in **local paint styles and text styles**, not variables. `get_variable_defs` returns nothing useful for them, so also read local styles (load the `figma:figma-use` skill first, then use `use_figma` to call `figma.getLocalPaintStyles()` / `figma.getLocalTextStyles()`; REST fallback: `GET /v1/files/:key/styles` + node lookups).

**Paint styles** - name is `Group/Field`, at most 2 levels (names are Capitalised; match case-insensitively, store lowercase):

| Figma style name | Token |
| --- | --- |
| `Primary/Main`, `Light`, `Dark` (also `Secondary`, `Error`, `Warning`, `Info`, `Success`) | `colors.primary.main` etc. |
| `Primary/Contrast` | `colors.primary.contrast` (maps to MUI `contrastText`) |
| `Primary/States/Outlined Resting Border`, `Contained Hover Background`, `Outlined Hover Background` | `colors.primary.states.outlinedRestingBorder` / `containedHoverBackground` / `outlinedHoverBackground` |
| `Text/Primary`, `Secondary`, `Disabled` | `colors.text.*` |
| `Background/Paper`, `Default` | `colors.background.*` |
| `Action/Active`, `Hover`, `Selected`, `Disabled`, `DisabledBackground`, `Focus` | `colors.action.*` |
| `Grey/50` ... `Grey/900`, `Grey/A100` ... | `colors.grey.*` |
| `Common/Black`, `Common/White` | `colors.common.*` |
| `Divider` (single level) | `colors.divider` (string) |

Only the first paint of a style is used, and only `SOLID` paints. Convert to hex, or `rgba(r, g, b, a)` when opacity < 1 (rgb channels are 0-1 floats in Figma: `Math.round(c * 255)`). Skip other groups and styles with more than 2 levels (the plugin ignores them too).

**Text styles** - name is `Typography/<Variant>` for the 13 MUI variants: `H1`-`H6`, `Subtitle1`, `Subtitle2`, `Body1`, `Body2`, `Button`, `Caption`, `Overline`. Store under `typography.variants.<lowercased variant>`:

- `fontFamily` = `fontName.family`
- `fontWeight` from `fontName.style`: Thin 100, ExtraLight 200, Light 300, Regular 400, Medium 500, SemiBold 600, Bold 700, ExtraBold 800, Black 900 (an unrecognised style such as `Italic` is kept as a note, weight left undefined)
- `fontSize` = `${fontSize}px`
- `letterSpacing` = `${value}px` for `PIXELS`, `${value}%` for `PERCENT`
- `lineHeight`: the plugin does **not** carry it, but Figma text styles do. Read it (`PIXELS` -> unitless ratio `value / fontSize`, `PERCENT` -> `value / 100`, `AUTO` -> omit) and include it when present.

**Precedence:** variables win when both sources define the same token; styles fill the gaps. Never invent states, contrast colours or missing variants: leave them out. Set the header comment `Source:` to `variables`, `mui-plugin-styles`, or `variables + mui-plugin-styles`.

**Import direction:** a theme JSON exported by the plugin (`{ palette, typography }`) can be fed back in with `/theme-convert --from-mui-json <file>`; it maps onto the same token shape.

### Step 5: Generate TypeScript Theme

Create type-safe TypeScript file:

```typescript
const themeContent = `
// theme.ts - Extracted from Figma
// File: ${figmaUrl}
// Extracted: ${new Date().toISOString()}

export const colors = ${JSON.stringify(tokens.colors, null, 2)};

export const typography = ${JSON.stringify(tokens.typography, null, 2)};

export const spacing = ${JSON.stringify(tokens.spacing, null, 2)};

export const theme = { colors, typography, spacing };
export default theme;
`;

writeFile(outputPath, themeContent);
```

### Step 6: Partial Extraction Recovery

If extraction fails mid-process, save what we have:

```typescript
try {
  const colors = await extractColors();
  const typography = await extractTypography();
  const spacing = await extractSpacing(); // Fails here
} catch (error) {
  // Save partial results
  if (colors || typography || spacing) {
    console.log('⚠️  Partial extraction - saving progress');
    writePartialTheme({ colors, typography, spacing });
    throw new Error(
      `Extraction partially failed. Saved partial results to ${outputPath}.\n` +
      `Re-run to complete extraction.`
    );
  }
  throw error;
}
```

### Step 7: Validation

Validate generated TypeScript:

```bash
npx tsc --noEmit ${outputPath}
```

If validation fails, report errors.

### Step 8: Report Results

```
✓ Extracted 32 design tokens from Figma
  - Colors: 12 design tokens
  - Typography: 8 design tokens
  - Spacing: 5 design tokens
  
✓ Generated: theme.ts (847 bytes)
✓ TypeScript validation: passed

Next steps:
  - Review theme.ts
  - Use /theme-convert to generate format files (MUI, Tailwind, etc.)
  - Use /theme-stories to generate Storybook stories
```

## Error Handling

### MCP Unavailable

```
❌ Figma Desktop MCP not available

Resolution steps:
1. Ensure Figma Desktop app is running
2. Open your Figma file: <figma-url>
3. Check MCP config: ~/.config/claude/mcp.json
4. Try fallback: /figma-extract <url> --api-token YOUR_TOKEN

Get API token: Figma Settings → Account → Personal Access Tokens
```

### Invalid URL

```
❌ Invalid Figma URL

Expected format:
https://www.figma.com/file/<FILE_ID>/<NAME>?node-id=<NODE_ID>

Example:
https://www.figma.com/file/ABC123/MyDesign?node-id=60257-25298
```

### Network Timeout

```
⚠️  Network timeout during extraction

Retrying (attempt 2/3)...
```

### Partial Extraction

```
⚠️  Partial extraction - saving progress

Extracted:
  ✓ Colors: 12 design tokens
  ✓ Typography: 8 design tokens
  ✗ Spacing: failed

Partial theme saved to: theme.ts
Re-run /figma-extract to complete extraction
```

## Examples

```bash
# Using MCP (Figma Desktop app running)
/figma-extract https://www.figma.com/file/ABC123/MyDesign?node-id=60257-25298

# Using REST API fallback
/figma-extract https://www.figma.com/file/ABC123/MyDesign?node-id=60257-25298 --api-token figd_abc123...

# Custom output path
/figma-extract https://www.figma.com/file/ABC123/MyDesign?node-id=60257-25298 --output src/theme.ts
```

## Integration

This sub-skill is part of the `/design2code` superpower. It can be used:

1. **Standalone:** Extract design tokens only
2. **Pipeline:** Called by `/design2code` orchestrator

Output `theme.ts` is consumed by:
- `/theme-convert` - Generate format files
- `/theme-stories` - Generate Storybook stories

## Troubleshooting

**Q: MCP connection fails**
A: Check Figma Desktop is running and file is open. Verify MCP server in Claude settings. Use --api-token as fallback.

**Q: No design tokens extracted**
A: Verify node-id points to a frame/page with design variables. Check Figma file has variables defined.

**Q: TypeScript validation errors**
A: Report the issue - generated code should always be valid. This is a bug.

**Q: Extraction timeout**
A: Large files may take longer. Skill retries automatically. If it fails 3 times, try extracting a specific node/frame instead of entire file.
