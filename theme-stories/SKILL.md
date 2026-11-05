---
name: theme-stories
description: Generate Storybook stories from theme files (colors, typography, spacing, icons) with optional MCP validation
tags: [storybook, stories, csf, design-tokens]
---

# Theme Stories - Storybook Generation

Generate Storybook stories showcasing design tokens and components.

## Usage

```bash
/theme-stories theme.ts
/theme-stories theme.ts --validate
/theme-stories theme.ts icons/
/theme-stories theme.ts --output stories/
```

## What It Does

1. Reads theme file(s) and icon components
2. Generates CSF 3.0 stories for colors, typography, spacing, icons
3. Stories reference theme files (no hardcoded values)
4. Optionally validates via Storybook MCP
5. Compatible with Storybook v7+

## Parameters

- `theme-file` (required): Path to theme.ts
- `icons-dir` (optional): Path to icons directory
- `--validate` (optional): Run Storybook MCP validation
- `--output <dir>` (optional): Output directory (default: current dir)

## Output

Generates story files:

- `Colors.stories.tsx` - Color palette showcase
- `Typography.stories.tsx` - Font styles showcase
- `Spacing.stories.tsx` - Spacing scale showcase
- `Icons.stories.tsx` - Icon gallery (if icons provided)

## Instructions for Claude

When a user invokes this skill:

### Step 1: Read Theme File

```typescript
const themeContent = await readFile(themePath, 'utf-8');

// Parse exports (same as theme-convert)
const tokens = {
  colors: extractColors(themeContent),
  typography: extractTypography(themeContent),
  spacing: extractSpacing(themeContent),
};
```

**Token shape note (MUI-plugin styles):** `theme.ts` may carry every MUI palette group (`primary ... success`, `grey`, `common`, `background`, `action`, `divider`) and `typography.variants` (h1-h6, subtitle1/2, body1/2, button, caption, overline). When present:

- `Colors.stories.tsx`: one `ColorSwatch` per field of every group that exists (iterate `Object.entries(theme.colors)`, skipping `states`), titled `<Group> <Field>`; show `contrast` swatches with the group's `main` as background. Render `states` as extra swatches only if defined.
- `Typography.stories.tsx`: one sample per entry in `typography.variants` (`h1` sample text "Heading 1", etc.) using that variant's own fontSize/fontWeight/lineHeight/letterSpacing. Fall back to `typography.heading`/`typography.body` when `variants` is absent.
- Never assume `primary`/`secondary`/`text` exist; skip missing groups instead of referencing undefined.

### Step 2: Check for Icons

```typescript
let iconComponents = [];

if (iconsDir && await exists(iconsDir)) {
  const iconFiles = await readdir(iconsDir);
  iconComponents = iconFiles
    .filter(f => f.endsWith('.tsx') && f !== 'index.ts')
    .map(f => f.replace('.tsx', ''));
}
```

### Step 3: Generate Colors Story

**Critical: Stories MUST import from theme file - never hardcode values**

```typescript
function generateColorsStory(themePath, tokens) {
  return `// Colors.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { theme } from '${relativePath(themePath)}';

interface ColorSwatchProps {
  name: string;
  value: string;
}

const ColorSwatch: React.FC<ColorSwatchProps> = ({ name, value }) => (
  <div style={{ 
    display: 'flex', 
    alignItems: 'center', 
    gap: theme.spacing.sm, // From theme, not hardcoded
    marginBottom: theme.spacing.sm 
  }}>
    <div
      style={{
        width: '64px',
        height: '64px',
        backgroundColor: value,
        border: \`1px solid \${theme.colors.text.disabled}\`, // From theme
        borderRadius: theme.spacing.sm,
        boxShadow: '0 2px 4px rgba(0,0,0,0.1)',
      }}
      data-testid="color-swatch"
    />
    <div>
      <div style={{ 
        fontWeight: theme.typography.heading.fontWeight, // From theme
        fontSize: '14px', 
        marginBottom: '4px',
        color: theme.colors.text.primary // From theme
      }}>
        {name}
      </div>
      <div style={{ 
        fontFamily: 'monospace', 
        fontSize: '12px', 
        color: theme.colors.text.secondary, // From theme
        textTransform: 'uppercase' 
      }}>
        {value}
      </div>
    </div>
  </div>
);

const meta: Meta = {
  title: 'Design Tokens/Colors',
  parameters: {
    layout: 'padded',
  },
};

export default meta;

export const Primary: StoryObj = {
  render: () => (
    <div>
      <h2 style={{ marginBottom: theme.spacing.md }}>Primary Colors</h2>
      <ColorSwatch name="Primary" value={theme.colors.primary.main} />
      <ColorSwatch name="Primary Light" value={theme.colors.primary.light} />
      <ColorSwatch name="Primary Dark" value={theme.colors.primary.dark} />
    </div>
  ),
};

export const AllColors: StoryObj = {
  render: () => (
    <div>
      <h2 style={{ marginBottom: theme.spacing.md }}>All Colors</h2>
      
      <h3 style={{ 
        marginTop: theme.spacing.lg, 
        marginBottom: theme.spacing.sm,
        fontFamily: theme.typography.fontFamily,
      }}>
        Primary
      </h3>
      <ColorSwatch name="Primary" value={theme.colors.primary.main} />
      <ColorSwatch name="Primary Light" value={theme.colors.primary.light} />
      <ColorSwatch name="Primary Dark" value={theme.colors.primary.dark} />
      
      <h3 style={{ 
        marginTop: theme.spacing.lg, 
        marginBottom: theme.spacing.sm,
        fontFamily: theme.typography.fontFamily,
      }}>
        Secondary
      </h3>
      <ColorSwatch name="Secondary" value={theme.colors.secondary.main} />
      <ColorSwatch name="Secondary Light" value={theme.colors.secondary.light} />
      <ColorSwatch name="Secondary Dark" value={theme.colors.secondary.dark} />
      
      <h3 style={{ 
        marginTop: theme.spacing.lg, 
        marginBottom: theme.spacing.sm,
        fontFamily: theme.typography.fontFamily,
      }}>
        Text
      </h3>
      <ColorSwatch name="Text Primary" value={theme.colors.text.primary} />
      <ColorSwatch name="Text Secondary" value={theme.colors.text.secondary} />
      <ColorSwatch name="Text Disabled" value={theme.colors.text.disabled} />
    </div>
  ),
};
`;
}
```

### Step 4: Generate Typography Story

```typescript
function generateTypographyStory(themePath, tokens) {
  return `// Typography.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { theme } from '${relativePath(themePath)}';

interface TypographyExampleProps {
  label: string;
  style: React.CSSProperties;
}

const TypographyExample: React.FC<TypographyExampleProps> = ({ label, style }) => (
  <div style={{ marginBottom: theme.spacing.xl }}>
    <div style={{ 
      fontSize: '12px', 
      color: theme.colors.text.secondary, // From theme
      marginBottom: theme.spacing.sm,
      textTransform: 'uppercase',
      letterSpacing: '0.5px',
    }}>
      {label}
    </div>
    <div style={style}>
      The quick brown fox jumps over the lazy dog
    </div>
    <div style={{ 
      fontSize: '11px', 
      color: theme.colors.text.disabled, // From theme
      marginTop: theme.spacing.xs,
      fontFamily: 'monospace',
    }}>
      {style.fontSize} / {style.fontWeight} / {style.lineHeight}
    </div>
  </div>
);

const meta: Meta = {
  title: 'Design Tokens/Typography',
  parameters: {
    layout: 'padded',
  },
};

export default meta;

export const AllTypography: StoryObj = {
  render: () => (
    <div>
      <h2 style={{ marginBottom: theme.spacing.lg }}>Typography Styles</h2>
      <TypographyExample
        label="Heading"
        style={{
          fontFamily: theme.typography.fontFamily,
          fontSize: theme.typography.heading.fontSize,
          fontWeight: theme.typography.heading.fontWeight,
          lineHeight: theme.typography.heading.lineHeight,
          color: theme.colors.text.primary,
        }}
      />
      <TypographyExample
        label="Body"
        style={{
          fontFamily: theme.typography.fontFamily,
          fontSize: theme.typography.body.fontSize,
          fontWeight: theme.typography.body.fontWeight,
          lineHeight: theme.typography.body.lineHeight,
          color: theme.colors.text.primary,
        }}
      />
    </div>
  ),
};
`;
}
```

### Step 5: Generate Spacing Story

```typescript
function generateSpacingStory(themePath, tokens) {
  return `// Spacing.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { theme } from '${relativePath(themePath)}';

interface SpacingExampleProps {
  name: string;
  value: string;
}

const SpacingExample: React.FC<SpacingExampleProps> = ({ name, value }) => (
  <div style={{ marginBottom: theme.spacing.md }}>
    <div style={{ 
      fontSize: '12px', 
      color: theme.colors.text.secondary,
      marginBottom: theme.spacing.xs 
    }}>
      {name} ({value})
    </div>
    <div style={{ display: 'flex', alignItems: 'center', gap: theme.spacing.sm }}>
      <div style={{ 
        width: value, 
        height: '32px', 
        backgroundColor: theme.colors.primary.main 
      }} />
      <div style={{ 
        fontFamily: 'monospace', 
        fontSize: '12px',
        color: theme.colors.text.primary,
      }}>
        {value}
      </div>
    </div>
  </div>
);

const meta: Meta = {
  title: 'Design Tokens/Spacing',
  parameters: {
    layout: 'padded',
  },
};

export default meta;

export const AllSpacing: StoryObj = {
  render: () => (
    <div>
      <h2 style={{ marginBottom: theme.spacing.lg }}>Spacing Scale</h2>
      ${Object.entries(tokens.spacing || {}).map(([key, value]) => 
        `<SpacingExample name="${key}" value="${value}" />`
      ).join('\n      ')}
    </div>
  ),
};
`;
}
```

### Step 6: Generate Icons Story (if icons exist)

```typescript
function generateIconsStory(iconsDir, iconComponents) {
  return `// Icons.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import * as Icons from '${relativePath(iconsDir)}';
import { theme } from '${relativePath(themePath)}';

const IconGrid: React.FC<{ icons: Record<string, React.ComponentType<any>> }> = ({ icons }) => (
  <div style={{ 
    display: 'grid', 
    gridTemplateColumns: 'repeat(auto-fill, minmax(120px, 1fr))', 
    gap: theme.spacing.md 
  }}>
    {Object.entries(icons)
      .filter(([name]) => name !== 'IconProps')
      .map(([name, Icon]) => (
        <div key={name} style={{ 
          textAlign: 'center', 
          padding: theme.spacing.md,
          border: \`1px solid \${theme.colors.text.disabled}\`,
          borderRadius: theme.spacing.sm,
        }}>
          <Icon size={32} color={theme.colors.primary.main} />
          <div style={{ 
            fontSize: '12px', 
            marginTop: theme.spacing.sm,
            color: theme.colors.text.secondary,
          }}>
            {name}
          </div>
        </div>
      ))}
  </div>
);

const meta: Meta = {
  title: 'Design Tokens/Icons',
  parameters: {
    layout: 'padded',
  },
};

export default meta;

export const AllIcons: StoryObj = {
  render: () => (
    <div>
      <h2 style={{ marginBottom: theme.spacing.lg }}>Icon Library</h2>
      <IconGrid icons={Icons as any} />
    </div>
  ),
};

export const IconSizes: StoryObj = {
  render: () => {
    const ExampleIcon = Icons.${iconComponents[0]};
    return (
      <div>
        <h2 style={{ marginBottom: theme.spacing.lg }}>Icon Sizes</h2>
        <div style={{ display: 'flex', alignItems: 'center', gap: theme.spacing.lg }}>
          <ExampleIcon size={16} color={theme.colors.primary.main} />
          <ExampleIcon size={24} color={theme.colors.primary.main} />
          <ExampleIcon size={32} color={theme.colors.primary.main} />
          <ExampleIcon size={48} color={theme.colors.primary.main} />
        </div>
      </div>
    );
  },
};
`;
}
```

### Step 7: Write Stories

```typescript
writeFile(`${outputDir}/Colors.stories.tsx`, colorsStory);
writeFile(`${outputDir}/Typography.stories.tsx`, typographyStory);
writeFile(`${outputDir}/Spacing.stories.tsx`, spacingStory);

if (iconComponents.length > 0) {
  writeFile(`${outputDir}/Icons.stories.tsx`, iconsStory);
}
```

### Step 8: Optional MCP Validation

```typescript
if (validateFlag) {
  const storybookMcp = await checkStorybookMcp();
  
  if (storybookMcp.available) {
    try {
      const storyIds = [
        'design-tokens-colors--all-colors',
        'design-tokens-typography--all-typography',
        'design-tokens-spacing--all-spacing',
      ];
      
      if (iconComponents.length > 0) {
        storyIds.push('design-tokens-icons--all-icons');
      }
      
      const results = await mcp.run_story_tests({ storyIds });
      
      console.log('✓ Story validation passed');
      if (results.accessibility?.warnings > 0) {
        console.warn(`⚠️  ${results.accessibility.warnings} accessibility warnings`);
      }
    } catch (error) {
      console.warn('⚠️  Story validation failed (non-blocking):', error.message);
    }
  } else {
    console.log('ℹ️  Storybook MCP unavailable - validation skipped');
    console.log('   To enable: Start Storybook dev server and re-run with --validate');
  }
}
```

### Step 9: Report Results

```
✓ Generated 4 Storybook stories

Generated:
  ✓ Colors.stories.tsx (12 color swatches)
  ✓ Typography.stories.tsx (2 type styles)
  ✓ Spacing.stories.tsx (5 spacing values)
  ✓ Icons.stories.tsx (18 icons)

Stories use CSF 3.0 format (Storybook v7+ compatible)
All values reference theme files (no hardcoded values)

View in Storybook:
  npm run storybook

Next steps:
  - Review generated stories
  - Run Storybook dev server to view
  - Use --validate flag for accessibility checks
```

## Error Handling

### Storybook Not Installed

```
ℹ️  Storybook not detected in package.json

Stories generated with CSF 3.0 format (forward-compatible)

To view stories:
  1. Install Storybook: npx storybook@latest init
  2. Start dev server: npm run storybook
  3. Navigate to Design Tokens section
```

### MCP Validation Unavailable

```
ℹ️  Storybook MCP validation skipped

Reason: Storybook dev server not running on http://localhost:6006

To enable validation:
  1. Start Storybook: npm run storybook
  2. Re-run: /theme-stories theme.ts --validate
```

### Version Warning

```
⚠️  Storybook v8.2 detected

MCP requires v10.3+
Stories generated with CSF 3.0 (compatible with v7+)
MCP validation unavailable

To enable MCP: npm install storybook@latest
```

## Examples

```bash
# Generate stories from theme
/theme-stories theme.ts

# Include icons
/theme-stories theme.ts icons/

# With MCP validation
/theme-stories theme.ts --validate

# Custom output directory
/theme-stories theme.ts icons/ --output src/stories/
```

## Integration

Part of `/design2code` superpower. Can be used:

1. **Standalone:** Generate stories from existing theme
2. **Pipeline:** Called by `/design2code` (optional feature)

## Troubleshooting

**Q: Stories don't load in Storybook**
A: Check Storybook is configured for .stories.tsx files. Verify import paths are correct.

**Q: Theme values not updating**
A: Stories import from theme file - changes to theme.ts automatically reflect. Restart Storybook dev server.

**Q: TypeScript errors in stories**
A: Verify theme.ts exports match story imports. Check @storybook/react types installed.

**Q: MCP validation not working**
A: Ensure Storybook v10.3+, dev server running on port 6006, and @storybook/addon-mcp installed.
