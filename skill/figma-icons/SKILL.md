---
name: figma-icons
description: Extract icons from Figma as React components with standardized props and accessibility
tags: [figma, icons, react, svg, components]
---

# Figma Icons - Icon Extraction

Extract icon assets from Figma and generate React icon components.

## Usage

```bash
/figma-icons <figma-url>
/figma-icons <figma-url> --api-token <TOKEN>
/figma-icons <figma-url> --output icons/
```

## What It Does

1. Discovers icon nodes in Figma file
2. Extracts SVG data for each icon
3. Generates React components with IconProps interface
4. Adds accessibility attributes
5. Creates barrel export (index.ts)

## Parameters

- `figma-url` (required): Figma URL pointing to icon frame/page
- `--api-token <TOKEN>` (optional): Figma API token for REST fallback
- `--output <dir>` (optional): Output directory (default: icons/)

## Output

Generates React icon components:

```typescript
// icons/SearchIcon.tsx
import React from 'react';

export interface IconProps {
  size?: number;
  color?: string;
  className?: string;
  'aria-label'?: string;
}

export const SearchIcon: React.FC<IconProps> = ({
  size = 24,
  color = 'currentColor',
  className,
  'aria-label': ariaLabel,
}) => {
  return (
    <svg
      width={size}
      height={size}
      viewBox="0 0 24 24"
      fill="none"
      xmlns="http://www.w3.org/2000/svg"
      className={className}
      aria-hidden={!ariaLabel}
      aria-label={ariaLabel}
      role={ariaLabel ? 'img' : undefined}
    >
      <path
        d="M21 21L15 15M17 10C17 13.866..."
        stroke={color}
        strokeWidth="2"
        strokeLinecap="round"
        strokeLinejoin="round"
      />
    </svg>
  );
};
```

Plus barrel export:

```typescript
// icons/index.ts
export { SearchIcon } from './SearchIcon';
export { CloseIcon } from './CloseIcon';
export { MenuIcon } from './MenuIcon';
export type { IconProps } from './SearchIcon';
```

## Instructions for Claude

When a user invokes this skill:

### Step 1: Check MCP Connection

Same as /figma-extract - try MCP first, fall back to API if unavailable.

### Step 2: Discover Icon Nodes

```typescript
const metadata = await mcp.get_metadata({
  nodeId: extractedNodeId,
  clientLanguages: 'typescript',
  clientFrameworks: 'react'
});

// Filter for icon nodes (typically COMPONENT or FRAME with SVG content)
const iconNodes = metadata.children.filter(node => 
  node.type === 'COMPONENT' || 
  (node.type === 'FRAME' && node.name.toLowerCase().includes('icon'))
);

if (iconNodes.length === 0) {
  throw new Error('No icon nodes found. Ensure URL points to icon frames or components.');
}
```

### Step 3: Batch Icon Extraction

If >5 icons, use parallel subagents:

```typescript
if (iconNodes.length > 5) {
  // Batch into groups of 5
  const batches = [];
  for (let i = 0; i < iconNodes.length; i += 5) {
    batches.push(iconNodes.slice(i, i + 5));
  }
  
  // Spawn subagents in parallel (single message, multiple Agent calls)
  const results = await Promise.all(
    batches.map((batch, i) => 
      Agent({
        description: `Icons batch ${i+1}/${batches.length}`,
        prompt: `Extract these ${batch.length} icons from Figma and generate React components:\n${batch.map(n => n.name).join('\n')}\n\nFor each icon:\n1. Call get_screenshot with nodeId\n2. Extract SVG paths and viewBox\n3. Generate React component with IconProps\n4. Return component code\n\nNode IDs: ${JSON.stringify(batch.map(n => n.id))}`
      })
    )
  );
} else {
  // Direct processing
  const components = [];
  for (const node of iconNodes) {
    const svg = await mcp.get_screenshot({
      nodeId: node.id,
      contentsOnly: true
    });
    components.push(generateIconComponent(node.name, svg));
  }
}
```

### Step 4: Generate Icon Component

```typescript
function generateIconComponent(name: string, svgData: string) {
  // Parse SVG to extract viewBox and paths
  const viewBoxMatch = svgData.match(/viewBox="([^"]*)"/);
  const viewBox = viewBoxMatch ? viewBoxMatch[1] : '0 0 24 24';
  
  // Extract paths (keep all path/circle/rect/etc elements)
  const pathsMatch = svgData.match(/<(path|circle|rect|line|polyline|polygon)[^>]*>/g);
  const paths = pathsMatch ? pathsMatch.join('\n      ') : '';
  
  // Convert name to PascalCase (e.g., "search-icon" → "SearchIcon")
  const componentName = name
    .split(/[-_\s]/)
    .map(word => word.charAt(0).toUpperCase() + word.slice(1).toLowerCase())
    .join('') + (name.toLowerCase().includes('icon') ? '' : 'Icon');
  
  return `// ${componentName}.tsx
import React from 'react';

export interface IconProps {
  size?: number;
  color?: string;
  className?: string;
  'aria-label'?: string;
}

export const ${componentName}: React.FC<IconProps> = ({
  size = 24,
  color = 'currentColor',
  className,
  'aria-label': ariaLabel,
}) => {
  return (
    <svg
      width={size}
      height={size}
      viewBox="${viewBox}"
      fill="none"
      xmlns="http://www.w3.org/2000/svg"
      className={className}
      aria-hidden={!ariaLabel}
      aria-label={ariaLabel}
      role={ariaLabel ? 'img' : undefined}
    >
      ${paths.replace(/stroke="[^"]*"/g, `stroke={color}`).replace(/fill="[^"]*"/g, 'fill="none"')}
    </svg>
  );
};
`;
}
```

### Step 5: Create Barrel Export

```typescript
function generateBarrelExport(componentNames: string[]) {
  return `// index.ts - Icon exports
${componentNames.map(name => `export { ${name} } from './${name}';`).join('\n')}
export type { IconProps } from './${componentNames[0]}';
`;
}
```

### Step 6: Write Files

```typescript
for (const component of components) {
  writeFile(`${outputDir}/${component.name}.tsx`, component.code);
}
writeFile(`${outputDir}/index.ts`, barrelExport);
```

### Step 7: Report Results

```
✓ Extracted 12 icons from Figma

Generated:
  ✓ icons/SearchIcon.tsx
  ✓ icons/CloseIcon.tsx
  ✓ icons/MenuIcon.tsx
  ✓ icons/UserIcon.tsx
  ... (8 more)
  ✓ icons/index.ts

Usage:
  import { SearchIcon, CloseIcon } from './icons';
  
  <SearchIcon size={20} color="#0066CC" aria-label="Search" />

Next steps:
  - Review generated components
  - Use /theme-stories to generate icon gallery story
```

## Error Handling

### No Icons Found

```
❌ No icon nodes found in Figma file

Expected: Frames or Components named with "icon" or SVG content
Found: 0 matching nodes

Tips:
  - Ensure URL points to icon frame/page
  - Icons should be Components or Frames with vector content
  - Check node-id is correct
```

### SVG Parse Error

```
⚠️  Generated 11/12 icons (1 failed)

✓ SearchIcon, CloseIcon, MenuIcon... (10 more)
✗ BrokenIcon - SVG parse error: No paths found

Partial success - review generated icons
```

### Batch Failure

```
⚠️  Icon extraction partially failed

Batches:
  ✓ Batch 1/3 (5 icons)
  ✓ Batch 2/3 (5 icons)
  ✗ Batch 3/3 (2 icons) - Network timeout

10 icons generated successfully, 2 failed
Re-run to retry failed icons
```

## Examples

```bash
# Extract all icons from frame
/figma-icons https://www.figma.com/file/ABC123/Icons?node-id=60257-25298

# With API token
/figma-icons https://www.figma.com/file/ABC123/Icons?node-id=60257-25298 --api-token figd_...

# Custom output directory
/figma-icons https://www.figma.com/file/ABC123/Icons --output src/components/icons/
```

## Integration

Part of `/design2code` superpower. Can be used:

1. **Standalone:** Extract icons only
2. **Pipeline:** Called by `/design2code` (optional feature)

Output components are consumed by:
- Your application (import and use icons)
- `/theme-stories` (generate icon gallery story)

## Troubleshooting

**Q: Icons look broken**
A: Check viewBox matches icon dimensions. Verify stroke/fill attributes correctly use `color` prop.

**Q: Icons not rendering**
A: Ensure React is imported. Check for TypeScript errors. Verify SVG paths are valid.

**Q: Accessibility warnings**
A: Always provide aria-label when icon has semantic meaning. Decorative icons should omit it (aria-hidden=true by default).

**Q: Can I customize IconProps?**
A: Yes! Edit the generated interface. Common additions: `onClick`, `style`, `title`.
