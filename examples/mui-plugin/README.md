Fixtures from https://github.com/mui/hackathon-figma-plugin (`fixtures/`), as exported/imported by the plugin.

- `light-theme.json` - minimal partial theme (primary, text, action, divider)
- `custom-theme1.json` - full theme incl. rem/em typography; `/theme-convert --from-mui-json` then `--format mui-plugin` normalises units to px

Check: `createTheme(JSON.parse(output))` succeeds and `tsc --noEmit` passes for the generated `theme.mui.ts`.
