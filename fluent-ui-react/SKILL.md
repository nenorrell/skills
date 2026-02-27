---
name: fluent-ui-react
description: "Build modern, responsive, and accessible UIs with Microsoft Fluent UI React v9 (@fluentui/react-components) for Word add-ins and Office extensions. Use when: (1) Creating or modifying Word add-in task pane UI, (2) Writing React components using Fluent UI v9 components, (3) Styling with Griffel (makeStyles/makeResetStyles/mergeClasses), (4) Working with Fluent design tokens and theming, (5) Building accessible Office add-in interfaces, (6) Implementing the v9 hooks-based component architecture (useComponent/useComponentStyles/renderComponent pattern), or (7) Any task involving @fluentui/react-components imports."
---

# Fluent UI React v9 for Word Add-ins

## Quick Start

Wrap the app root with `FluentProvider` and a theme:

```tsx
import { FluentProvider, webLightTheme } from '@fluentui/react-components';

Office.onReady(() => {
  createRoot(document.getElementById('container')!).render(
    <FluentProvider theme={webLightTheme}>
      <App />
    </FluentProvider>
  );
});
```

## Core Concepts

### Component Architecture

v9 components separate behavior, styling, and rendering into hooks:

```tsx
const MyComponent = React.forwardRef((props, ref) => {
  const state = useMyComponent(props, ref);  // behavior
  useMyComponentStyles(state);               // styling
  return renderMyComponent(state);           // render
});
```

This enables **recomposition** - reassemble hooks to create custom variants without forking.

### Styling with Griffel

```tsx
import { makeStyles, makeResetStyles, mergeClasses, shorthands, tokens } from '@fluentui/react-components';

// Base styles (single monolithic class - use for base/default state)
const useBaseClassName = makeResetStyles({
  display: 'flex',
  padding: '8px 12px',  // CSS shorthands OK in makeResetStyles
  color: tokens.colorNeutralForeground1,
  ':hover': { backgroundColor: tokens.colorNeutralBackground1Hover },
});

// Variant styles (atomic classes - use for conditional overrides)
const useStyles = makeStyles({
  primary: {
    backgroundColor: tokens.colorBrandBackground,
    color: tokens.colorNeutralForegroundOnBrand,
  },
  small: { ...shorthands.padding('4px', '8px') },  // Must use shorthands.* in makeStyles
});

function MyButton({ appearance, size, className }) {
  const base = useBaseClassName();
  const styles = useStyles();
  return (
    <button className={mergeClasses(
      base,
      appearance === 'primary' && styles.primary,
      size === 'small' && styles.small,
      className,  // consumer override always last
    )} />
  );
}
```

**Critical rules:**
- Never concatenate Griffel classes with `+`; always use `mergeClasses()`
- Call `mergeClasses()` once per element, not nested
- Use `tokens.*` for all colors, spacing, typography; never raw values
- `makeResetStyles` allows CSS shorthands; `makeStyles` requires `shorthands.*` helpers

### Slots

Slots let consumers customize component parts:

```tsx
// Props to a slot
<Button icon={{ className: styles.icon, 'aria-hidden': true }}>Click</Button>

// JSX element as slot content
<Button icon={<MyIcon />}>Click</Button>

// Disable a slot
<Button icon={null}>No icon</Button>

// Render function (escape hatch - replaces the slot element entirely)
<Button icon={{ children: (Component, props) => <b>!</b> }}>Alert</Button>
```

### Event Handlers

v9 uses the `(event, data)` convention:

```tsx
<Input onChange={(ev, data) => console.log(data.value)} />
<Checkbox onChange={(ev, data) => console.log(data.checked)} />
<Dropdown onOptionSelect={(ev, data) => console.log(data.selectedOptions)} />
```

## Reference Files

Consult these for detailed guidance:

- **[v9-component-architecture.md](references/v9-component-architecture.md)** - Complete v9 architecture docs: hooks pattern, slots system, FluentProvider, theming, design tokens, recomposition, custom styling hooks, props conventions, accessibility. Read when building custom components or needing deeper architectural understanding.

- **[griffel-styling.md](references/griffel-styling.md)** - Full Griffel API reference: `makeStyles`, `makeResetStyles`, `mergeClasses`, `shorthands.*`, RTL handling, `@noflip`, nested selectors, performance rules, `tokens` usage. Read when styling components or troubleshooting CSS issues.

- **[word-addin-patterns.md](references/word-addin-patterns.md)** - Word add-in specific patterns: task pane layout, responsive design (300-600px), Office theme detection, forms, lists, toolbars, dialogs, message bars, progress states, Office JS API integration, accessibility and focus management. Read when building Word add-in UI.

## Key Component Reference

### Layout
`FluentProvider`, `Card`, `Divider`, `Drawer`, `TabList`/`Tab`

### Input
`Button`, `Input`, `Textarea`, `Select`, `Dropdown`/`Option`, `Checkbox`, `Switch`, `RadioGroup`/`Radio`, `Slider`, `SpinButton`, `DatePicker`, `Combobox`

### Data Display
`Avatar`, `Badge`, `Tag`, `DataGrid`, `Table`, `Tree`, `Accordion`, `Text`, `Title*`, `Subtitle*`, `Body*`, `Caption*`

### Feedback
`Spinner`, `ProgressBar`, `MessageBar`, `Toast`, `Dialog`, `Popover`, `Tooltip`, `Alert`

### Navigation
`Toolbar`/`ToolbarButton`, `Menu`/`MenuItem`, `Breadcrumb`, `Link`, `Nav`/`NavItem`

### Icons
Import from `@fluentui/react-icons`: `import { AddRegular, DeleteRegular } from '@fluentui/react-icons'`

Icon naming: `{Name}{Style}` where Style is `Regular`, `Filled`, or size variants like `20Regular`.

## Word Add-in Guidelines

### Task Pane Defaults
- Width: ~320px (resizable 300-600px); design mobile-first
- Use `height: 100vh` with flex column layout
- Scrollable content area with fixed header/footer
- Match Office host theme via `FluentProvider`

### Accessibility Requirements
- All interactive elements must be keyboard accessible
- Use `role="status"` with `aria-live="polite"` for operation feedback
- Return focus to the task pane after Word API operations
- Support high contrast via `@media (forced-colors: active)` with system colors
- Use semantic HTML (`<button>`, `<input>`) over styled `<div>`s

### Performance
- Define `makeStyles`/`makeResetStyles` at module scope, not inside components
- Use `React.lazy` for heavy panels to keep initial load fast
- Tree-shake icons with direct imports: `import { AddRegular } from '@fluentui/react-icons'`
- Memoize callbacks that interact with the Word API
