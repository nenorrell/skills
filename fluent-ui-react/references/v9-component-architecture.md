# Fluent UI React v9 Component Architecture

## Table of Contents
- [Hook-Based Architecture](#hook-based-architecture)
- [Component File Structure](#component-file-structure)
- [Slots System](#slots-system)
- [FluentProvider and Theming](#fluentprovider-and-theming)
- [Design Tokens](#design-tokens)
- [Custom Styling Mechanisms](#custom-styling-mechanisms)
- [Recomposition Pattern](#recomposition-pattern)
- [Props Best Practices](#props-best-practices)
- [Event Handler Convention](#event-handler-convention)
- [Accessibility Patterns](#accessibility-patterns)

## Hook-Based Architecture

v9 components (`@fluentui/react-components`) use a hooks-based architecture separating behavior, styling, and rendering:

```tsx
// Component.tsx - the composed component
export const Sample = React.forwardRef<HTMLElement, SampleProps>((props, ref) => {
  const state = useSample(props, ref);    // 1. State/behavior hook
  useSampleStyles(state);                  // 2. Styling hook
  return renderSample(state);              // 3. Render function
});
```

### Three Pillars

1. **`useSample(props, ref)`** - Maps props to state, handles internal state, consumes context, creates side effects
2. **`useSampleStyles(state)`** - Applies classNames to slots using Griffel (`makeStyles`/`mergeClasses`)
3. **`renderSample(state)`** - Pure function rendering JSX from state; no state mutation here

## Component File Structure

```
Component/
├── Component.tsx          # ForwardRef composition of hooks + render
├── useComponent.ts        # State/behavior hook
├── useComponentStyles.ts  # Styling hook (Griffel)
├── renderComponent.tsx    # Pure render function
└── Component.types.ts     # Props, State, Slots type definitions
```

### Types File Pattern

```ts
import { ComponentProps, ComponentState, Slot } from '@fluentui/react-utilities';

export type SampleSlots = {
  root: Slot<'div'>;
  icon?: Slot<'span'>;
};

export type SampleProps = ComponentProps<SampleSlots> & {
  size?: 'small' | 'medium' | 'large';
};

export type SampleState = ComponentState<SampleSlots> & {
  size: 'small' | 'medium' | 'large';
};
```

## Slots System

Slots allow consumers to replace or customize parts of a component. Every component has a `root` slot.

### Consuming Slots

```tsx
// Pass props to a slot
<Button icon={{ className: 'my-icon' }}>Click me</Button>

// Pass a custom element via children render function (escape hatch)
<Button icon={{ children: (Component, props) => <b>B</b> }}>Bold</Button>

// Disable a slot by passing null
<Button icon={null}>No icon</Button>
```

### Shorthand Syntax

Slots accept shorthand values:

```tsx
// String shorthand (for slots that accept text content)
<Tooltip content="Hello" />

// JSX shorthand
<Button icon={<MyCustomIcon />} />

// Object shorthand (pass props)
<Button icon={{ className: 'custom', 'aria-label': 'icon' }} />
```

## FluentProvider and Theming

Wrap your app with `FluentProvider` to provide theme tokens and text direction:

```tsx
import { FluentProvider, webLightTheme, webDarkTheme } from '@fluentui/react-components';

function App() {
  return (
    <FluentProvider theme={webLightTheme}>
      {/* All Fluent components inherit this theme */}
      <YourApp />
    </FluentProvider>
  );
}
```

### Nested Providers

```tsx
<FluentProvider theme={webLightTheme}>
  <MainContent />
  <FluentProvider theme={webDarkTheme}>
    <DarkSidebar />
  </FluentProvider>
</FluentProvider>
```

### RTL Support

```tsx
<FluentProvider dir="rtl">
  {/* Components automatically flip for RTL */}
</FluentProvider>
```

### Built-in Themes

- `webLightTheme` - Default light theme
- `webDarkTheme` - Dark theme
- `teamsLightTheme`, `teamsDarkTheme`, `teamsHighContrastTheme` - Teams themes

### Custom Themes

v9 themes are built from a **brand color ramp** — 16 shades (keys 10–160) derived from a primary brand color. Use the [Fluent UI Theme Designer](https://react.fluentui.dev/?path=/docs/themedesigner--docs) to generate these values from a single color.

```tsx
import {
  createLightTheme,
  createDarkTheme,
  BrandVariants,
  Theme,
} from '@fluentui/react-components';

// Full brand ramp: 16 shades from darkest (10) to lightest (160)
// Generate these with the Theme Designer tool or programmatically
const myBrand: BrandVariants = {
  10: '#020305',
  20: '#111723',
  30: '#16253D',
  40: '#193253',
  50: '#1B3F6A',
  60: '#1B4C82',
  70: '#18599B',
  80: '#1267B4',  // This is typically closest to your primary brand color
  90: '#3174C2',
  100: '#4F82C8',
  110: '#6790CF',
  120: '#7D9ED5',
  130: '#92ACDC',
  140: '#A6BBE2',
  150: '#BAC9E9',
  160: '#CDD8EF',
};

// Create both light and dark themes from the same brand ramp
const myLightTheme: Theme = createLightTheme(myBrand);
const myDarkTheme: Theme = createDarkTheme(myBrand);

// Dark theme needs a foreground override for readability
myDarkTheme.colorBrandForeground1 = myBrand[110];
myDarkTheme.colorBrandForeground2 = myBrand[120];
```

#### Applying the Custom Theme

```tsx
function App() {
  const [isDark, setIsDark] = React.useState(false);

  return (
    <FluentProvider theme={isDark ? myDarkTheme : myLightTheme}>
      <YourApp onToggleTheme={() => setIsDark(!isDark)} />
    </FluentProvider>
  );
}
```

### Partial Theme Overrides

Override specific token values without creating a full theme. Useful for tweaking a section of the UI:

```tsx
import { webLightTheme, FluentProvider } from '@fluentui/react-components';

// Override specific tokens — all other tokens inherit from the parent provider
const sidebarOverrides = {
  ...webLightTheme,
  colorNeutralBackground1: '#F5F5F5',
  colorNeutralBackground2: '#EBEBEB',
};

<FluentProvider theme={webLightTheme}>
  <MainContent />
  <FluentProvider theme={sidebarOverrides}>
    <Sidebar /> {/* Uses overridden background colors */}
  </FluentProvider>
</FluentProvider>
```

**When to use partial overrides vs. custom themes:**
- **Partial overrides**: Tweak a few tokens for a specific UI section (e.g., darker sidebar, branded header)
- **Custom theme**: Your add-in needs a fully branded look across all components (create from `BrandVariants`)
- **Built-in themes**: Use `webLightTheme`/`webDarkTheme` to match standard Office appearance

### Extended Design Tokens

Add custom CSS variables alongside Fluent tokens for app-specific styling needs:

```tsx
import { themeToTokensObject, FluentProvider, webLightTheme } from '@fluentui/react-components';

// Extend the theme with custom tokens
const customTheme = {
  ...webLightTheme,
  customHeaderHeight: '48px',
  customSidebarWidth: '280px',
};

// Convert to tokens object for type-safe CSS variable references
const customTokens = themeToTokensObject(customTheme);

// Use in makeStyles
const useStyles = makeStyles({
  header: { height: customTokens.customHeaderHeight },
  sidebar: { width: customTokens.customSidebarWidth },
});
```

**Caution**: Keep extended tokens minimal (~10-20 max). Too many CSS variables (300+) degrade performance. Prefer using existing Fluent tokens wherever possible.

## Design Tokens

v9 uses CSS custom properties (CSS variables) for theming. `FluentProvider` injects token values as CSS variables on a wrapper `<div>`, and components reference them via the type-safe `tokens` object.

### How Tokens Work

```
Theme Object                    CSS Variables                    Component Styles
{ colorBrandBackground:    →    --colorBrandBackground:     →    backgroundColor:
  '#0078D4' }                   #0078D4                          var(--colorBrandBackground)
```

The `tokens` object maps token names to `var(--tokenName)` strings:

```tsx
import { tokens } from '@fluentui/react-components';

// tokens.colorBrandBackground === 'var(--colorBrandBackground)'
// Use directly in makeStyles — the actual value comes from FluentProvider's theme
const useStyles = makeStyles({
  root: {
    backgroundColor: tokens.colorBrandBackground,  // resolves via CSS variable
  },
});
```

### Token Categories — Complete Reference

#### Color Tokens

**Neutral colors** (grays, backgrounds, foregrounds):
- Backgrounds: `colorNeutralBackground1` through `6`, plus `Hover`/`Pressed`/`Selected`/`Disabled` variants
  - Example: `colorNeutralBackground1`, `colorNeutralBackground1Hover`, `colorNeutralBackground2Selected`
- Foregrounds: `colorNeutralForeground1` through `4`, plus `Disabled`, `OnBrand`, `Inverted` variants
  - Example: `colorNeutralForeground1`, `colorNeutralForeground2`, `colorNeutralForegroundDisabled`
- Strokes (borders): `colorNeutralStroke1` through `3`, plus `Hover`/`Pressed`/`Disabled` variants
  - Example: `colorNeutralStroke1`, `colorNeutralStroke1Hover`, `colorNeutralStrokeAccessible`

**Brand colors** (primary/accent):
- `colorBrandBackground`, `colorBrandBackground2`, `colorBrandBackgroundHover`, `colorBrandBackgroundPressed`
- `colorBrandForeground1`, `colorBrandForeground2`, `colorBrandForegroundLink`, `colorBrandForegroundLinkHover`
- `colorBrandStroke1`, `colorBrandStroke2`
- `colorCompoundBrandBackground`, `colorCompoundBrandForeground1`, `colorCompoundBrandStroke`
  - Plus `Hover`/`Pressed` variants for interactive compound brand tokens

**Status colors**:
- Success: `colorStatusSuccessBackground1` through `3`, `colorStatusSuccessForeground1` through `3`, `colorStatusSuccessBorder1`/`2`
- Warning: `colorStatusWarningBackground1` through `3`, `colorStatusWarningForeground1` through `3`, `colorStatusWarningBorder1`/`2`
- Danger: `colorStatusDangerBackground1` through `3`, `colorStatusDangerForeground1` through `3`, `colorStatusDangerBorder1`/`2`
- Info: `colorStatusInfoBackground1` through `3`, `colorStatusInfoForeground1`/`2`

**Palette colors** (named colors for data visualization, avatars, etc.):
- `colorPaletteRedBackground1` through `3`, `colorPaletteRedForeground1` through `3`, `colorPaletteRedBorder1`/`2`
- Same pattern for: `Green`, `Blue`, `Yellow`, `Orange`, `Purple`, `Teal`, `Pink`, `Navy`, `Cornflower`, `Lilac`, `Marigold`, etc.

#### Typography Tokens

```tsx
// Font families
tokens.fontFamilyBase        // Segoe UI, system fonts
tokens.fontFamilyMonospace   // Consolas, monospace
tokens.fontFamilyNumeric     // Bahnschrift, numeric-optimized

// Font sizes (scale from 100=10px to 600=40px)
tokens.fontSizeBase100  // 10px
tokens.fontSizeBase200  // 12px
tokens.fontSizeBase300  // 14px  ← default body text
tokens.fontSizeBase400  // 16px
tokens.fontSizeBase500  // 20px
tokens.fontSizeBase600  // 24px
tokens.fontSizeHero700  // 28px
tokens.fontSizeHero800  // 32px
tokens.fontSizeHero900  // 40px
tokens.fontSizeHero1000 // 68px

// Font weights
tokens.fontWeightRegular    // 400
tokens.fontWeightMedium     // 500
tokens.fontWeightSemibold   // 600
tokens.fontWeightBold       // 700

// Line heights (match font size scale)
tokens.lineHeightBase100  // 14px
tokens.lineHeightBase200  // 16px
tokens.lineHeightBase300  // 20px  ← default body
tokens.lineHeightBase400  // 22px
tokens.lineHeightBase500  // 28px
tokens.lineHeightBase600  // 32px
tokens.lineHeightHero700  // 36px
tokens.lineHeightHero800  // 40px
tokens.lineHeightHero900  // 52px
tokens.lineHeightHero1000 // 92px
```

#### Spacing Tokens

```tsx
// Horizontal spacing
tokens.spacingHorizontalNone     // 0
tokens.spacingHorizontalXXS      // 2px
tokens.spacingHorizontalXS       // 4px
tokens.spacingHorizontalSNudge   // 6px
tokens.spacingHorizontalS        // 8px
tokens.spacingHorizontalMNudge   // 10px
tokens.spacingHorizontalM        // 12px
tokens.spacingHorizontalL        // 16px
tokens.spacingHorizontalXL       // 20px
tokens.spacingHorizontalXXL      // 24px
tokens.spacingHorizontalXXXL     // 32px

// Vertical spacing (same scale as horizontal)
tokens.spacingVerticalNone       // 0
tokens.spacingVerticalXXS        // 2px
tokens.spacingVerticalXS         // 4px
tokens.spacingVerticalSNudge     // 6px
tokens.spacingVerticalS          // 8px
tokens.spacingVerticalMNudge     // 10px
tokens.spacingVerticalM          // 12px
tokens.spacingVerticalL          // 16px
tokens.spacingVerticalXL         // 20px
tokens.spacingVerticalXXL        // 24px
tokens.spacingVerticalXXXL       // 32px
```

#### Border Radius Tokens

```tsx
tokens.borderRadiusNone      // 0
tokens.borderRadiusSmall     // 2px
tokens.borderRadiusMedium    // 4px
tokens.borderRadiusLarge     // 6px
tokens.borderRadiusXLarge    // 8px
tokens.borderRadiusCircular  // 10000px (fully rounded)
```

#### Shadow Tokens

```tsx
tokens.shadow2    // Subtle elevation (cards, dropdowns)
tokens.shadow4    // Low elevation
tokens.shadow8    // Medium elevation (popovers)
tokens.shadow16   // High elevation (dialogs)
tokens.shadow28   // Very high elevation
tokens.shadow64   // Maximum elevation
// Each has a brand variant: shadow2Brand, shadow4Brand, etc.
```

#### Stroke Width Tokens

```tsx
tokens.strokeWidthThin        // 1px
tokens.strokeWidthThick       // 2px
tokens.strokeWidthThicker     // 3px
tokens.strokeWidthThickest    // 4px
```

#### Animation Tokens

```tsx
// Durations
tokens.durationUltraFast   // 50ms
tokens.durationFaster      // 100ms
tokens.durationFast        // 150ms
tokens.durationNormal      // 200ms
tokens.durationGentle      // 250ms
tokens.durationSlow        // 300ms
tokens.durationSlower      // 400ms
tokens.durationUltraSlow   // 500ms

// Easing curves
tokens.curveAccelerateMax    // cubic-bezier(1, 0, 1, 1)
tokens.curveAccelerateMid    // cubic-bezier(0.7, 0, 1, 0.5)
tokens.curveAccelerateMin    // cubic-bezier(0.8, 0, 1, 1)
tokens.curveDecelerateMax    // cubic-bezier(0, 0, 0, 1)
tokens.curveDecelerateMid    // cubic-bezier(0.1, 0.9, 0.2, 1)
tokens.curveDecelerateMin    // cubic-bezier(0.33, 0, 0.1, 1)
tokens.curveEasyEase         // cubic-bezier(0.33, 0, 0.67, 1)
tokens.curveEasyEaseMax      // cubic-bezier(0.8, 0, 0.1, 1)
tokens.curveLinear           // cubic-bezier(0, 0, 1, 1)
```

### Token Naming Convention

Tokens follow the pattern: `{category}{Property}{Variant}{State}`

- `colorNeutralBackground1Hover` → color / Neutral / Background1 / Hover
- `colorBrandForegroundLinkPressed` → color / Brand / ForegroundLink / Pressed
- `fontSizeBase300` → fontSize / Base / 300

### Token Usage Rules

- **Never use raw color values** in component styles; always use tokens
- **Exception**: system colors (`ButtonText`, `Canvas`, etc.) inside `@media (forced-colors: active)` for high contrast
- **Prefer semantic tokens** (e.g., `colorNeutralForeground1`) over generic ones for theme portability
- **Use spacing tokens** for consistent rhythm: `spacingHorizontalM` over `'12px'`
- **Use typography tokens** for text: `fontSizeBase300` over `'14px'`
- **Tokens resolve at runtime** via CSS variables — changing the theme on `FluentProvider` instantly updates all components

## Custom Styling Mechanisms

### 1. className prop (instance-level)

```tsx
const useStyles = makeStyles({
  myButton: { backgroundColor: tokens.colorBrandBackground2 },
});

function MyComponent() {
  const styles = useStyles();
  return <Button className={styles.myButton}>Styled</Button>;
}
```

### 2. Slot className (slot-level)

```tsx
<Button icon={{ className: styles.myIcon }}>Click</Button>
```

### 3. Custom Style Hooks via FluentProvider (type-level)

```tsx
import { FluentProvider } from '@fluentui/react-components';

const useCustomButtonStyles = (state) => {
  const styles = useStyles();
  state.root.className = mergeClasses(state.root.className, styles.root);
};

<FluentProvider
  customStyleHooks_unstable={{
    useButtonStyles_unstable: useCustomButtonStyles,
  }}
>
  {/* All Buttons in this subtree get custom styles */}
</FluentProvider>
```

### 4. Recomposition (full control)

See [Recomposition Pattern](#recomposition-pattern) below.

## Recomposition Pattern

Create a new component by reassembling hooks from an existing component:

```tsx
import * as React from 'react';
import {
  useButton_unstable,
  useButtonStyles_unstable,
  renderButton_unstable,
} from '@fluentui/react-components';
import type { ButtonProps, ForwardRefComponent } from '@fluentui/react-components';

export const CustomButton: ForwardRefComponent<ButtonProps> = React.forwardRef((props, ref) => {
  // Override default props
  const state = useButton_unstable({ appearance: 'primary', ...props }, ref);

  // Apply custom styles THEN base styles (base wins on collision)
  useCustomButtonStyles(state);
  useButtonStyles_unstable(state);

  return renderButton_unstable(state);
});
CustomButton.displayName = 'CustomButton';
```

### Custom Styles in Recomposition

```tsx
const useCustomStyles = makeStyles({
  root: { flexGrow: 0, marginTop: '4px' },
});

const useCustomButtonStyles = (state) => {
  const styles = useCustomStyles();
  state.root.className = mergeClasses('fui-CustomButton', styles.root, state.root.className);
};
```

Key: Place custom styles BEFORE base styles in `mergeClasses` so base can override, or AFTER if custom should win.

## Props Best Practices

- Prefer `appearance` for visual variants, not `display`, `look`, etc.
- Sizes: `'extra-small' | 'small' | 'medium' | 'large' | 'extra-large'`
- Use discriminated unions over booleans for mutually exclusive options
- List properties alphabetically in type definitions
- Prefer `type` aliases over `interface` for props (prevents declaration merging)
- Avoid re-declaring native HTML attributes already inherited from `React.HTMLAttributes`
- Use the design spec part names for slot props

### Naming Conventions

| Pattern | Rule | Example |
|---------|------|---------|
| Boolean flags | Name for exceptional case | `disabled`, `hidden`, `checked` |
| Event callbacks | `on` + subject + event | `onClick`, `onInputChange` |
| Event params | `(ev, data)` pattern | `onChange: (ev, data: { value }) => void` |

## Event Handler Convention

v9 standardizes event handlers with the `(event, data)` pattern:

```tsx
// ✅ Correct v9 pattern
<Input onChange={(ev, data) => {
  console.log(data.value); // typed data object
}} />

<Checkbox onChange={(ev, data) => {
  console.log(data.checked); // boolean
}} />
```

The `data` object contains component-specific values relevant to the event. The `ev` is always the React synthetic event.

## Accessibility Patterns

### ARIA Attributes

- Use semantic HTML elements (`<button>`, `<input>`) over `<div>` with roles
- Forward all `aria-*` and `data-*` attributes to the root element
- Interactable elements must be keyboard, mouse, AND touch accessible
- Provide focus management with `tabIndex` and `useFocusableGroup`

### High Contrast

- Use `@media (forced-colors: active)` instead of vendor-prefixed `-ms-high-contrast`
- Only use system colors (`ButtonText`, `Canvas`, `LinkText`, etc.) inside forced-colors media queries
- Use transparent borders that become visible in high contrast mode for focus indicators
- Background images disappear in high contrast; use actual elements for functional icons

### Focus Indicators

Use `createCustomFocusIndicatorStyle` or `createFocusOutlineStyle` from `@fluentui/react-components`:

```tsx
import { createFocusOutlineStyle } from '@fluentui/react-components';

const useStyles = makeStyles({
  root: createFocusOutlineStyle(),
});
```

### Screen Readers

- Use `aria-label`, `aria-labelledby`, `aria-describedby` for non-visual descriptions
- Use visually hidden text (positioned off-screen) for screen reader-only content
- Follow WAI-ARIA patterns for composite widgets (menus, trees, grids)
