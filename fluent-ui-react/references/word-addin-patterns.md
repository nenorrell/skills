# Fluent UI React for Word Add-ins

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Project Setup](#project-setup)
- [Task Pane Layout](#task-pane-layout)
- [Common Component Patterns](#common-component-patterns)
- [Responsive Design for Task Panes](#responsive-design-for-task-panes)
- [Theme Integration](#theme-integration)
- [Accessibility in Add-ins](#accessibility-in-add-ins)
- [Performance Considerations](#performance-considerations)
- [Common UI Patterns](#common-ui-patterns)

## Architecture Overview

Word add-ins render in a **task pane** (typically 320px wide, resizable) or as **dialogs**. The UI is a web app hosted in an iframe. Fluent UI React v9 (`@fluentui/react-components`) is the recommended library to ensure visual consistency with the Office host.

### Key Constraints
- Task pane default width: ~320px (min ~300px, user-resizable)
- Runs in an iframe - no direct DOM access to Word
- Must handle light/dark/high-contrast themes matching the Office host
- Office JS API (`@types/office-js`) for Word document interaction
- Must be keyboard accessible for Office accessibility requirements

## Project Setup

### Dependencies

```json
{
  "dependencies": {
    "@fluentui/react-components": "^9.x",
    "@fluentui/react-icons": "^2.x",
    "react": "^18.x",
    "react-dom": "^18.x"
  },
  "devDependencies": {
    "@types/office-js": "^1.x"
  }
}
```

### App Root Pattern

```tsx
import { FluentProvider, webLightTheme } from '@fluentui/react-components';

// Office.onReady ensures the Office JS runtime is initialized
Office.onReady(() => {
  const root = createRoot(document.getElementById('container')!);
  root.render(
    <FluentProvider theme={webLightTheme}>
      <App />
    </FluentProvider>
  );
});
```

## Task Pane Layout

### Standard Task Pane Structure

```tsx
import {
  makeResetStyles,
  makeStyles,
  mergeClasses,
  tokens,
  shorthands,
} from '@fluentui/react-components';

const useTaskPaneBase = makeResetStyles({
  display: 'flex',
  flexDirection: 'column',
  height: '100vh',
  overflow: 'hidden',
  backgroundColor: tokens.colorNeutralBackground1,
  color: tokens.colorNeutralForeground1,
  fontFamily: tokens.fontFamilyBase,
});

const useStyles = makeStyles({
  header: {
    ...shorthands.padding(tokens.spacingVerticalL, tokens.spacingHorizontalL),
    ...shorthands.borderBottom(tokens.strokeWidthThin, 'solid', tokens.colorNeutralStroke1),
    flexShrink: 0,
  },
  content: {
    ...shorthands.padding(tokens.spacingVerticalM, tokens.spacingHorizontalL),
    overflowY: 'auto',
    flexGrow: 1,
  },
  footer: {
    ...shorthands.padding(tokens.spacingVerticalM, tokens.spacingHorizontalL),
    ...shorthands.borderTop(tokens.strokeWidthThin, 'solid', tokens.colorNeutralStroke1),
    flexShrink: 0,
  },
});

function TaskPane({ children }) {
  const baseCls = useTaskPaneBase();
  const styles = useStyles();

  return (
    <div className={baseCls}>
      <header className={styles.header}>
        <Title3>My Add-in</Title3>
      </header>
      <main className={styles.content}>
        {children}
      </main>
      <footer className={styles.footer}>
        {/* Action buttons */}
      </footer>
    </div>
  );
}
```

### Scrollable Content Area

Always constrain the content area to prevent the task pane from overflowing:

```tsx
const useStyles = makeStyles({
  scrollContainer: {
    overflowY: 'auto',
    overflowX: 'hidden',
    flexGrow: 1,
    // Smooth scrolling
    scrollBehavior: 'smooth',
    // Custom scrollbar styling
    '::-webkit-scrollbar': { width: '6px' },
    '::-webkit-scrollbar-thumb': {
      backgroundColor: tokens.colorNeutralStroke1,
      borderRadius: tokens.borderRadiusCircular,
    },
  },
});
```

## Common Component Patterns

### Form in Task Pane

```tsx
import {
  Button,
  Input,
  Label,
  Textarea,
  Select,
  Field,
  makeStyles,
  tokens,
} from '@fluentui/react-components';

const useStyles = makeStyles({
  form: {
    display: 'flex',
    flexDirection: 'column',
    rowGap: tokens.spacingVerticalM,
  },
  actions: {
    display: 'flex',
    justifyContent: 'flex-end',
    columnGap: tokens.spacingHorizontalS,
    marginTop: tokens.spacingVerticalL,
  },
});

function InsertContentForm() {
  const styles = useStyles();

  const handleSubmit = async () => {
    await Word.run(async (context) => {
      const range = context.document.getSelection();
      range.insertText('Hello', Word.InsertLocation.replace);
      await context.sync();
    });
  };

  return (
    <form className={styles.form}>
      <Field label="Title" required>
        <Input placeholder="Enter title..." />
      </Field>
      <Field label="Description">
        <Textarea placeholder="Enter description..." resize="vertical" />
      </Field>
      <Field label="Style">
        <Select>
          <option>Normal</option>
          <option>Heading 1</option>
          <option>Heading 2</option>
        </Select>
      </Field>
      <div className={styles.actions}>
        <Button appearance="secondary">Cancel</Button>
        <Button appearance="primary" onClick={handleSubmit}>Insert</Button>
      </div>
    </form>
  );
}
```

### List with Actions

```tsx
import {
  Card,
  CardHeader,
  Body1,
  Caption1,
  Button,
  Menu,
  MenuTrigger,
  MenuPopover,
  MenuList,
  MenuItem,
  makeStyles,
  tokens,
} from '@fluentui/react-components';
import { MoreHorizontalRegular } from '@fluentui/react-icons';

const useStyles = makeStyles({
  list: {
    display: 'flex',
    flexDirection: 'column',
    rowGap: tokens.spacingVerticalS,
  },
});

function DocumentList({ items }) {
  const styles = useStyles();

  return (
    <div className={styles.list} role="list">
      {items.map((item) => (
        <Card key={item.id} size="small" role="listitem">
          <CardHeader
            header={<Body1>{item.title}</Body1>}
            description={<Caption1>{item.date}</Caption1>}
            action={
              <Menu>
                <MenuTrigger disableButtonEnhancement>
                  <Button
                    appearance="transparent"
                    icon={<MoreHorizontalRegular />}
                    aria-label="More actions"
                  />
                </MenuTrigger>
                <MenuPopover>
                  <MenuList>
                    <MenuItem>Edit</MenuItem>
                    <MenuItem>Delete</MenuItem>
                  </MenuList>
                </MenuPopover>
              </Menu>
            }
          />
        </Card>
      ))}
    </div>
  );
}
```

### Settings Panel

```tsx
import {
  Switch,
  Dropdown,
  Option,
  SpinButton,
  Divider,
  makeStyles,
  tokens,
} from '@fluentui/react-components';

const useStyles = makeStyles({
  section: {
    display: 'flex',
    flexDirection: 'column',
    rowGap: tokens.spacingVerticalS,
  },
  settingRow: {
    display: 'flex',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
});

function SettingsPanel() {
  const styles = useStyles();

  return (
    <div className={styles.section}>
      <div className={styles.settingRow}>
        <Label>Auto-save</Label>
        <Switch />
      </div>
      <Divider />
      <Field label="Font size">
        <SpinButton min={8} max={72} defaultValue={12} />
      </Field>
      <Field label="Language">
        <Dropdown placeholder="Select language">
          <Option>English</Option>
          <Option>Spanish</Option>
          <Option>French</Option>
        </Dropdown>
      </Field>
    </div>
  );
}
```

### Progress/Loading States

```tsx
import {
  Spinner,
  ProgressBar,
  makeStyles,
  tokens,
} from '@fluentui/react-components';

const useStyles = makeStyles({
  centered: {
    display: 'flex',
    justifyContent: 'center',
    alignItems: 'center',
    flexGrow: 1,
    flexDirection: 'column',
    rowGap: tokens.spacingVerticalM,
  },
});

function LoadingState() {
  const styles = useStyles();
  return (
    <div className={styles.centered}>
      <Spinner size="medium" label="Loading documents..." />
    </div>
  );
}

// For determinate progress
function UploadProgress({ value }) {
  return <ProgressBar value={value} max={1} />;
}
```

## Responsive Design for Task Panes

Task panes can be resized. Design for 300-600px width:

```tsx
const useStyles = makeStyles({
  grid: {
    display: 'grid',
    gridTemplateColumns: '1fr',
    rowGap: tokens.spacingVerticalS,
    // Stack on narrow, side-by-side on wider
    '@media (min-width: 480px)': {
      gridTemplateColumns: '1fr 1fr',
      columnGap: tokens.spacingHorizontalM,
    },
  },
});
```

### Responsive Typography

Use Fluent typography tokens that scale appropriately:

```tsx
import { Title3, Subtitle2, Body1, Caption1 } from '@fluentui/react-components';

// These components use the correct token-based sizing
<Title3>Page Title</Title3>      // For section headers
<Subtitle2>Subsection</Subtitle2> // For subsections
<Body1>Body content</Body1>       // For regular text
<Caption1>Helper text</Caption1>  // For secondary info
```

### Overflow Handling

For content that may exceed the narrow task pane:

```tsx
const useStyles = makeStyles({
  truncated: {
    whiteSpace: 'nowrap',
    overflowX: 'hidden',
    textOverflow: 'ellipsis',
  },
  wrapping: {
    wordBreak: 'break-word',
    overflowWrap: 'anywhere',
  },
});
```

## Theme Integration

### Choosing a Theme Strategy

| Strategy | When to Use |
|----------|-------------|
| Built-in theme (`webLightTheme`/`webDarkTheme`) | Add-in should look like standard Office; no custom branding |
| Custom brand theme (`createLightTheme`) | Add-in needs branded colors (company logo color, product identity) |
| Office host theme detection | Add-in must match the user's current Office theme (light/dark/high contrast) |

### Using Built-in Themes

```tsx
import { FluentProvider, webLightTheme, webDarkTheme } from '@fluentui/react-components';

// Simplest approach — pick light or dark
<FluentProvider theme={webLightTheme}>
  <App />
</FluentProvider>
```

### Creating a Custom Branded Theme

Use the [Fluent UI Theme Designer](https://react.fluentui.dev/?path=/docs/themedesigner--docs) to generate a brand ramp from your primary color, then:

```tsx
import {
  createLightTheme,
  createDarkTheme,
  BrandVariants,
  Theme,
  FluentProvider,
} from '@fluentui/react-components';

// Brand ramp generated from your primary color (e.g., #0F6CBD)
const myBrand: BrandVariants = {
  10: '#020305', 20: '#111723', 30: '#16253D', 40: '#193253',
  50: '#1B3F6A', 60: '#1B4C82', 70: '#18599B', 80: '#1267B4',
  90: '#3174C2', 100: '#4F82C8', 110: '#6790CF', 120: '#7D9ED5',
  130: '#92ACDC', 140: '#A6BBE2', 150: '#BAC9E9', 160: '#CDD8EF',
};

const brandLightTheme: Theme = createLightTheme(myBrand);
const brandDarkTheme: Theme = createDarkTheme(myBrand);

// Important: override foreground tokens for dark theme readability
brandDarkTheme.colorBrandForeground1 = myBrand[110];
brandDarkTheme.colorBrandForeground2 = myBrand[120];
```

### Detecting and Matching the Office Host Theme

The Office JS API exposes `Office.context.officeTheme` with the host's background/foreground colors. Map these to Fluent themes:

```tsx
import {
  FluentProvider,
  webLightTheme,
  webDarkTheme,
  teamsHighContrastTheme,
  Theme,
} from '@fluentui/react-components';

// Determine the Fluent theme from Office's reported theme colors
function resolveOfficeTheme(): Theme {
  const officeTheme = Office.context?.officeTheme;
  if (!officeTheme) return webLightTheme;

  const bg = officeTheme.bodyBackgroundColor?.toLowerCase();

  // High contrast: very dark or very light with high foreground contrast
  if (officeTheme.isHighContrast) return teamsHighContrastTheme;

  // Dark themes have dark body backgrounds
  if (bg && (bg === '#1e1e1e' || bg === '#000000' || bg === '#2b2b2b' || bg.startsWith('#1') || bg.startsWith('#2'))) {
    return webDarkTheme;
  }

  return webLightTheme;
}

function App() {
  const [theme, setTheme] = React.useState<Theme>(webLightTheme);

  React.useEffect(() => {
    Office.onReady(() => {
      setTheme(resolveOfficeTheme());

      // Listen for runtime theme changes (user switches Office theme)
      if (Office.context.officeTheme) {
        Office.context.document?.addHandlerAsync(
          Office.EventType.OfficeThemeChanged,
          () => setTheme(resolveOfficeTheme()),
        );
      }
    });
  }, []);

  return (
    <FluentProvider theme={theme}>
      <TaskPane />
    </FluentProvider>
  );
}
```

### Combining Brand Theme + Office Theme Detection

For branded add-ins that still respect light/dark mode:

```tsx
function App() {
  const [isDark, setIsDark] = React.useState(false);

  React.useEffect(() => {
    Office.onReady(() => {
      const bg = Office.context?.officeTheme?.bodyBackgroundColor?.toLowerCase();
      setIsDark(bg === '#1e1e1e' || bg === '#000000');
    });
  }, []);

  return (
    <FluentProvider theme={isDark ? brandDarkTheme : brandLightTheme}>
      <TaskPane />
    </FluentProvider>
  );
}
```

### Theme-Aware Styles

Always use `tokens` so styles automatically adapt when the theme changes — no conditional logic needed:

```tsx
const useStyles = makeStyles({
  card: {
    backgroundColor: tokens.colorNeutralBackground2,
    ...shorthands.border(tokens.strokeWidthThin, 'solid', tokens.colorNeutralStroke1),
    boxShadow: tokens.shadow2,
    // These values automatically change for light/dark/high-contrast
    // because they resolve to CSS variables set by FluentProvider
  },
  brandAccent: {
    color: tokens.colorBrandForeground1,
    backgroundColor: tokens.colorBrandBackground2,
    // Uses your brand ramp if using a custom theme, or default blue otherwise
  },
  statusMessage: {
    color: tokens.colorStatusSuccessForeground1,
    backgroundColor: tokens.colorStatusSuccessBackground1,
    // Status tokens ensure consistent semantic meaning across themes
  },
});
```

### Partial Overrides for UI Sections

Override specific tokens for a sub-section without creating a full theme:

```tsx
import { webLightTheme } from '@fluentui/react-components';

const headerOverrides = {
  ...webLightTheme,
  colorNeutralBackground1: tokens.colorBrandBackground,
  colorNeutralForeground1: tokens.colorNeutralForegroundOnBrand,
};

<FluentProvider theme={webLightTheme}>
  <FluentProvider theme={headerOverrides}>
    <Header /> {/* Branded header */}
  </FluentProvider>
  <Content /> {/* Standard light theme */}
</FluentProvider>
```

## Accessibility in Add-ins

### Keyboard Navigation

- Ensure all interactive elements are keyboard reachable via Tab
- Use `TabList` for tab-based navigation within the task pane
- Implement arrow key navigation for lists using `useFocusableGroup`
- Add `Escape` key handling to close panels/dialogs

### ARIA for Add-in Context

```tsx
// Mark the task pane as an application landmark
<div role="main" aria-label="Document Editor Add-in">

// Use live regions for status updates after Word operations
<div role="status" aria-live="polite">
  {statusMessage}
</div>

// Label action buttons clearly
<Button
  appearance="primary"
  aria-label="Insert formatted text into document"
  onClick={handleInsert}
>
  Insert
</Button>
```

### Focus Management

After async operations (like inserting content into Word), return focus to the task pane:

```tsx
const buttonRef = React.useRef<HTMLButtonElement>(null);

const handleInsert = async () => {
  await Word.run(async (context) => {
    // ... insert content
    await context.sync();
  });
  // Return focus to the trigger button
  buttonRef.current?.focus();
};
```

## Performance Considerations

### Minimize Re-renders

```tsx
// Define styles outside component (module scope)
const useStyles = makeStyles({
  root: { display: 'flex' },
});

// Memoize callbacks that interact with Word API
const handleClick = React.useCallback(async () => {
  await Word.run(async (context) => {
    // ...
    await context.sync();
  });
}, []);
```

### Lazy Load Heavy Components

```tsx
const HeavyEditor = React.lazy(() => import('./HeavyEditor'));

function TaskPaneContent({ view }) {
  return (
    <React.Suspense fallback={<Spinner label="Loading..." />}>
      {view === 'editor' && <HeavyEditor />}
    </React.Suspense>
  );
}
```

### Bundle Size

- Import components individually: `import { Button } from '@fluentui/react-components'`
- Griffel supports ahead-of-time compilation to reduce runtime CSS processing
- Use tree-shaking-friendly imports for icons: `import { AddRegular } from '@fluentui/react-icons'`

## Common UI Patterns

### Command Bar / Toolbar

```tsx
import { Toolbar, ToolbarButton, ToolbarDivider, Tooltip } from '@fluentui/react-components';
import { TextBoldRegular, TextItalicRegular, TextUnderlineRegular } from '@fluentui/react-icons';

function FormattingToolbar() {
  const applyFormat = (format: string) => {
    Word.run(async (context) => {
      const range = context.document.getSelection();
      range.font[format] = true;
      await context.sync();
    });
  };

  return (
    <Toolbar size="small">
      <Tooltip content="Bold" relationship="label">
        <ToolbarButton
          icon={<TextBoldRegular />}
          onClick={() => applyFormat('bold')}
        />
      </Tooltip>
      <Tooltip content="Italic" relationship="label">
        <ToolbarButton
          icon={<TextItalicRegular />}
          onClick={() => applyFormat('italic')}
        />
      </Tooltip>
      <ToolbarDivider />
      <Tooltip content="Underline" relationship="label">
        <ToolbarButton
          icon={<TextUnderlineRegular />}
          onClick={() => applyFormat('underline')}
        />
      </Tooltip>
    </Toolbar>
  );
}
```

### Tab Navigation

```tsx
import { TabList, Tab, makeStyles, tokens } from '@fluentui/react-components';

function TaskPaneWithTabs() {
  const [selectedTab, setSelectedTab] = React.useState('edit');

  return (
    <>
      <TabList
        selectedValue={selectedTab}
        onTabSelect={(_, data) => setSelectedTab(data.value as string)}
        size="small"
      >
        <Tab value="edit">Edit</Tab>
        <Tab value="review">Review</Tab>
        <Tab value="settings">Settings</Tab>
      </TabList>
      {selectedTab === 'edit' && <EditPanel />}
      {selectedTab === 'review' && <ReviewPanel />}
      {selectedTab === 'settings' && <SettingsPanel />}
    </>
  );
}
```

### Dialog for Confirmations

```tsx
import {
  Dialog,
  DialogTrigger,
  DialogSurface,
  DialogTitle,
  DialogBody,
  DialogActions,
  DialogContent,
  Button,
} from '@fluentui/react-components';

function DeleteConfirmation({ onConfirm }) {
  return (
    <Dialog>
      <DialogTrigger disableButtonEnhancement>
        <Button appearance="subtle">Delete</Button>
      </DialogTrigger>
      <DialogSurface>
        <DialogBody>
          <DialogTitle>Delete content?</DialogTitle>
          <DialogContent>
            This will remove the selected content from the document. This action cannot be undone.
          </DialogContent>
          <DialogActions>
            <DialogTrigger disableButtonEnhancement>
              <Button appearance="secondary">Cancel</Button>
            </DialogTrigger>
            <Button appearance="primary" onClick={onConfirm}>Delete</Button>
          </DialogActions>
        </DialogBody>
      </DialogSurface>
    </Dialog>
  );
}
```

### Message Bar for Notifications

```tsx
import {
  MessageBar,
  MessageBarTitle,
  MessageBarBody,
  MessageBarActions,
  Button,
} from '@fluentui/react-components';

function StatusMessages() {
  return (
    <>
      <MessageBar intent="success">
        <MessageBarBody>
          <MessageBarTitle>Success</MessageBarTitle>
          Content inserted into document.
        </MessageBarBody>
      </MessageBar>

      <MessageBar intent="error">
        <MessageBarBody>
          <MessageBarTitle>Error</MessageBarTitle>
          Failed to connect to the document.
        </MessageBarBody>
        <MessageBarActions>
          <Button size="small">Retry</Button>
        </MessageBarActions>
      </MessageBar>
    </>
  );
}
```
