# Storybook Integration — 100xSystems

## Overview

Storybook provides an isolated environment for developing and documenting UI components. Every atomic component in the design system MUST have a corresponding Storybook story.

---

## 1. Setup

### Installation

```bash
# Install Storybook with Next.js framework support
npx storybook@latest init --builder webpack5

# Or use the framework-specific approach:
npx @storybook/nextjs@latest init
```

### Configuration

Create `.storybook/main.ts`:

```typescript
import type { StorybookConfig } from '@storybook/nextjs';

const config: StorybookConfig = {
  stories: [
    '../src/presentation/_storybook/**/*.stories.@(ts|tsx)',
    '../src/presentation/_storybook/**/*.mdx',
  ],
  addons: [
    '@storybook/addon-links',
    '@storybook/addon-essentials',
    '@storybook/addon-interactions',
    '@storybook/addon-a11y',          // Accessibility testing
    '@storybook/addon-viewport',      // Responsive testing
    '@chromatic-com/storybook',       // Visual regression testing
  ],
  framework: {
    name: '@storybook/nextjs',
    options: {},
  },
  staticDirs: ['../public'],
};

export default config;
```

Create `.storybook/preview.ts`:

```typescript
import type { Preview } from '@storybook/react';
import '../src/pages/globals.css';     // Import global styles

const preview: Preview = {
  parameters: {
    actions: { argTypesRegex: '^on[A-Z].*' },
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/i,
      },
    },
    viewport: {
      viewports: {
        mobile: { name: 'Mobile', styles: { width: '375px', height: '667px' } },
        tablet: { name: 'Tablet', styles: { width: '768px', height: '1024px' } },
        desktop: { name: 'Desktop', styles: { width: '1280px', height: '800px' } },
      },
    },
    backgrounds: {
      default: 'dark',
      values: [
        { name: 'dark', value: '#0a0a0a' },
        { name: 'light', value: '#ffffff' },
      ],
    },
  },
};

export default preview;
```

---

## 2. Story File Conventions

### Location

All stories live in:
```
src/presentation/_storybook/
├── atoms/        # Stories for atomic components
├── molecules/    # Stories for composite components
├── organisms/    # Stories for complex components
└── pages/        # Full page stories (optional)
```

### File Naming

```
[component-name].stories.tsx
```

### Story Template

```tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from '../../__components/atoms/Button';

const meta = {
  title: 'Atoms/Button',
  component: Button,
  parameters: {
    layout: 'centered',
  },
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['primary', 'secondary', 'ghost', 'destructive'],
    },
    size: {
      control: 'select',
      options: ['sm', 'default', 'lg'],
    },
    disabled: { control: 'boolean' },
    loading: { control: 'boolean' },
  },
} satisfies Meta<typeof Button>;

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = {
  args: {
    variant: 'primary',
    children: 'Click Me',
  },
};

export const Secondary: Story = {
  args: {
    variant: 'secondary',
    children: 'Cancel',
  },
};

export const Loading: Story = {
  args: {
    loading: true,
    children: 'Saving...',
  },
};

export const Disabled: Story = {
  args: {
    disabled: true,
    children: 'Disabled',
  },
};
```

---

## 3. MDX Documentation

For comprehensive component documentation, use MDX:

```mdx
import { Meta, Canvas, Controls, Story } from '@storybook/blocks';
import * as ButtonStories from './Button.stories';

<Meta of={ButtonStories} />

# Button Component

Primary action button with variant and size support.

## Usage

```tsx
import { Button } from '@/__components/atoms/Button';

<Button variant="primary" size="default">
  Click Me
</Button>
```

## Variants

<Canvas>
  <Story of={ButtonStories.Primary} />
  <Story of={ButtonStories.Secondary} />
</Canvas>

## Props

<Controls of={ButtonStories.Primary} />
```

---

## 4. Design System Documentation

The `_storybook/` folder should also contain:

```
_storybook/
├── .docs/                    # Design system documentation
│   ├── INTRODUCTION.mdx       # Welcome page for Storybook
│   ├── DESIGN_TOKENS.mdx      # Colors, typography, spacing
│   └── CONTRIBUTING.mdx       # How to add components
├── atoms/                     # Atomic component stories
│   ├── Button.stories.tsx
│   ├── Badge.stories.tsx
│   ├── Input.stories.tsx
│   ├── Card.stories.tsx
│   ├── Modal.stories.tsx
│   └── Spinner.stories.tsx
├── molecules/                 # Molecular component stories
│   ├── SearchInput.stories.tsx
│   ├── TabBar.stories.tsx
│   ├── FilterBar.stories.tsx
│   └── DataGrid.stories.tsx
└── organisms/                 # Organism component stories
    ├── Navbar.stories.tsx
    ├── Footer.stories.tsx
    └── DashboardLayout.stories.tsx
```
