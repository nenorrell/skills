# Griffel Styling System (Fluent UI v9)

## Table of Contents
- [Overview](#overview)
- [makeStyles](#makestyles)
- [makeResetStyles](#makeresetstyles)
- [mergeClasses](#mergeclasses)
- [Hybrid Approach](#hybrid-approach)
- [Shorthands](#shorthands)
- [Design Tokens Usage](#design-tokens-usage)
- [RTL Support](#rtl-support)
- [Nested Selectors](#nested-selectors)
- [Performance Guidelines](#performance-guidelines)
- [Common Patterns](#common-patterns)

## Overview

Griffel is the CSS-in-JS engine for Fluent UI React v9. It uses **Atomic CSS** where each property-value pair becomes a single CSS rule, enabling deduplication and small bundle sizes.

Import from `@fluentui/react-components` (not `@griffel/react` directly) when building with Fluent UI:

```tsx
import { makeStyles, makeResetStyles, mergeClasses, shorthands, tokens } from '@fluentui/react-components';
```

## makeStyles

Define style permutations. Returns a React hook:

```tsx
const useStyles = makeStyles({
  root: {
    display: 'flex',
    alignItems: 'center',
    color: tokens.colorNeutralForeground1,
  },
  primary: {
    backgroundColor: tokens.colorBrandBackground,
    color: tokens.colorNeutralForegroundOnBrand,
  },
  small: { fontSize: tokens.fontSizeBase200 },
  medium: { fontSize: tokens.fontSizeBase300 },
  large: { fontSize: tokens.fontSizeBase400 },
});

function Component({ size = 'medium', primary }) {
  const styles = useStyles();
  const className = mergeClasses(
    styles.root,
    primary && styles.primary,
    styles[size],
  );
  return <div className={className} />;
}
```

### CSS Shorthand Limitation

Griffel does NOT support CSS shorthands directly. Use `shorthands.*` helpers:

```tsx
const useStyles = makeStyles({
  root: {
    // WRONG: padding: '2px 4px 8px 16px',
    // CORRECT:
    ...shorthands.padding('2px', '4px', '8px', '16px'),
    ...shorthands.borderRadius('4px'),
    ...shorthands.margin('0', 'auto'),
  },
});
```

Each value MUST be a separate argument:
```tsx
// WRONG - produces incorrect output:
shorthands.padding('2px 4px');
// CORRECT:
shorthands.padding('2px', '4px');
```

## makeResetStyles

Generate a single monolithic class to avoid "CSS rule explosion" from Atomic CSS:

```tsx
const useBaseClassName = makeResetStyles({
  display: 'flex',
  padding: '4px',   // CSS shorthands ARE allowed in makeResetStyles
  margin: '4px',
  ':hover': { color: tokens.colorBrandForeground1 },
  '::before': { display: 'block', content: "' '" },
  '@media (forced-colors: active)': { color: 'ButtonText' },
});
```

Only ONE `makeResetStyles` class can be applied per element. Combining multiple is non-deterministic.

## mergeClasses

Merge Griffel classes together. Deduplicates atomic classes and resolves conflicts by argument order (last wins):

```tsx
const className = mergeClasses(
  baseClassName,                    // base styles
  props.primary && styles.primary,  // conditional styles
  props.className,                  // consumer overrides (last = highest priority)
);
```

### Critical Rules

```tsx
// NEVER concatenate class strings:
const wrong = classes.rootA + ' ' + classes.rootB;  // Non-deterministic!

// ALWAYS use mergeClasses:
const correct = mergeClasses(classes.rootA, classes.rootB);  // Proper deduplication

// Call mergeClasses ONCE per element, not nested:
// BAD:
const bad = mergeClasses(baseClass, mergeClasses(a, b), mergeClasses(c, d));
// GOOD:
const good = mergeClasses(baseClass, condA && a, condB && b, condC && c);
```

## Hybrid Approach

Recommended pattern: `makeResetStyles` for base + `makeStyles` for variants:

```tsx
const useBaseClassName = makeResetStyles({
  display: 'flex',
  alignItems: 'center',
  padding: '8px 12px',
  color: tokens.colorNeutralForeground1,
  backgroundColor: tokens.colorNeutralBackground1,
  border: `${tokens.strokeWidthThin} solid ${tokens.colorNeutralStroke1}`,
  borderRadius: tokens.borderRadiusMedium,
  ':hover': { backgroundColor: tokens.colorNeutralBackground1Hover },
  ':active': { backgroundColor: tokens.colorNeutralBackground1Pressed },
});

const useStyles = makeStyles({
  // Only define DIFFERENCES from base, avoid duplicating base styles
  primary: {
    backgroundColor: tokens.colorBrandBackground,
    color: tokens.colorNeutralForegroundOnBrand,
  },
  small: { ...shorthands.padding('4px', '8px'), fontSize: tokens.fontSizeBase200 },
  large: { ...shorthands.padding('12px', '16px'), fontSize: tokens.fontSizeBase400 },
});

function MyComponent({ appearance = 'default', size = 'medium', className }) {
  const baseClassName = useBaseClassName();
  const styles = useStyles();

  return (
    <div className={mergeClasses(
      baseClassName,
      appearance === 'primary' && styles.primary,
      size !== 'medium' && styles[size],
      className,  // always last for consumer overrides
    )} />
  );
}
```

## Shorthands

Available shorthand functions:

```tsx
import { shorthands } from '@fluentui/react-components';

shorthands.padding('top', 'right', 'bottom', 'left')
shorthands.margin('top', 'right', 'bottom', 'left')
shorthands.border('width', 'style', 'color')
shorthands.borderTop('width', 'style', 'color')
shorthands.borderRight('width', 'style', 'color')  // auto-flipped for RTL
shorthands.borderBottom('width', 'style', 'color')
shorthands.borderLeft('width', 'style', 'color')    // auto-flipped for RTL
shorthands.borderRadius('topLeft', 'topRight', 'bottomRight', 'bottomLeft')
shorthands.borderColor('top', 'right', 'bottom', 'left')
shorthands.borderStyle('top', 'right', 'bottom', 'left')
shorthands.borderWidth('top', 'right', 'bottom', 'left')
shorthands.gap('row', 'column')
shorthands.overflow('x', 'y')
shorthands.flex('grow', 'shrink', 'basis')
shorthands.gridArea('rowStart', 'columnStart', 'rowEnd', 'columnEnd')
shorthands.inset('top', 'right', 'bottom', 'left')
shorthands.outline('width', 'style', 'color')
shorthands.transition('property', 'duration', 'delay', 'timingFunction')
shorthands.textDecoration('line', 'style', 'color', 'thickness')
```

## Design Tokens Usage

```tsx
import { tokens } from '@fluentui/react-components';

const useStyles = makeStyles({
  root: {
    // Colors - always use tokens, never raw values
    color: tokens.colorNeutralForeground1,
    backgroundColor: tokens.colorNeutralBackground1,

    // Typography
    fontFamily: tokens.fontFamilyBase,
    fontSize: tokens.fontSizeBase300,
    fontWeight: tokens.fontWeightRegular,
    lineHeight: tokens.lineHeightBase300,

    // Spacing
    columnGap: tokens.spacingHorizontalM,
    rowGap: tokens.spacingVerticalS,

    // Borders
    borderRadius: tokens.borderRadiusMedium,
    ...shorthands.borderWidth(tokens.strokeWidthThin),

    // Shadows
    boxShadow: tokens.shadow4,

    // Transitions
    transitionDuration: tokens.durationNormal,
    transitionTimingFunction: tokens.curveEasyEase,
  },
});
```

## RTL Support

Griffel automatically flips directional properties for RTL:

```tsx
makeStyles({
  root: {
    paddingLeft: '10px',
    // LTR: padding-left: 10px
    // RTL: padding-right: 10px (auto-flipped)
  },
});
```

### Disable Flipping

Use `/* @noflip */` comment:

```tsx
makeStyles({
  root: {
    paddingLeft: '10px /* @noflip */',
    // Always padding-left in both LTR and RTL
  },
});
```

### CSS Variables Caveat

Values containing CSS variables may not flip automatically:

```tsx
// boxShadow with a CSS variable won't flip
// Handle manually:
const useStyles = makeStyles({
  root: { boxShadow: 'var(--shadow)' },
  rtl: { boxShadow: 'var(--shadow-rtl)' },
});

function App() {
  const styles = useStyles();
  const { dir } = useFluent();
  return <div className={mergeClasses(styles.root, dir === 'rtl' && styles.rtl)} />;
}
```

## Nested Selectors

### Pseudo-classes (Recommended)

```tsx
makeResetStyles({
  color: tokens.colorNeutralForeground1,
  ':hover': { color: tokens.colorBrandForeground1 },
  ':focus-visible': { outlineColor: tokens.colorBrandStroke1 },
  ':active': { color: tokens.colorBrandForeground1Pressed },
});
```

### Avoid Complex Selectors

```tsx
// BAD - complex nesting, hard to override, poor perf
makeResetStyles({
  '> .foo': { '> .bar': { '> .baz': { display: 'flex' } } },
});

// GOOD - apply classes directly to elements
const useStyles = makeStyles({
  baz: { display: 'flex' },
});
```

### Avoid `> *` Selectors

`> *` matches all elements on the page and causes performance issues. Prefer targeting specific classNames.

### Avoid Input Pseudo-classes

Prefer JS state over CSS pseudo-classes like `:checked`, `:disabled`:

```tsx
// BAD
makeResetStyles({
  ':checked': { color: tokens.colorNeutralForeground2 },
});

// GOOD
const useStyles = makeStyles({
  checked: { color: tokens.colorNeutralForeground2 },
});
// Apply with: mergeClasses(base, checked && styles.checked)
```

## Performance Guidelines

1. **Use `makeResetStyles` for base styles** to avoid atomic class explosion from pseudo-selectors and media queries
2. **Use `makeStyles` for conditional variants** that are applied additively
3. **Call `mergeClasses` once per element**, not nested
4. **Avoid rule duplication** - don't repeat base styles in variant definitions
5. **Avoid `!important`** - Griffel's mergeClasses handles specificity correctly
6. **Use structured styles** - map variant keys to style names for clean lookups:
   ```tsx
   const className = mergeClasses(baseClassName, styles[props.size]);
   ```
7. **Prefer pseudo-classes over wide selectors** - `:hover` on the element itself rather than parent `> *`

## Common Patterns

### Conditional Styles

```tsx
const useStyles = makeStyles({
  disabled: { opacity: 0.5, pointerEvents: 'none' },
  selected: { backgroundColor: tokens.colorNeutralBackground1Selected },
});

mergeClasses(base, props.disabled && styles.disabled, props.selected && styles.selected);
```

### Responsive Styles

```tsx
makeResetStyles({
  display: 'flex',
  flexDirection: 'column',
  '@media (min-width: 640px)': {
    flexDirection: 'row',
  },
});
```

### High Contrast

```tsx
makeResetStyles({
  backgroundColor: tokens.colorNeutralBackground1,
  '@media (forced-colors: active)': {
    backgroundColor: 'Canvas',
    color: 'CanvasText',
    borderColor: 'ButtonText',
  },
});
```

### Pseudo-elements

```tsx
makeResetStyles({
  position: 'relative',
  '::after': {
    content: "''",
    position: 'absolute',
    inset: '0',
    borderRadius: 'inherit',
  },
});
```
