# Component Architecture — 100xSystems

## 1. Atomic Design System

Components follow the **Atomic Design** methodology:

```
ATOMS        → MOLECULES      → ORGANISMS      → TEMPLATES → PAGES
(Button,      (SearchInput,    (Navbar,          (Layouts)   (Full pages)
 Input,        FilterBar,       Footer,
 Badge)        DataGrid)        Dashboard)
```

## 2. Directory Structure

```
src/presentation/
├── __components/           # NEW: Clean, well-typed atomic components
│   ├── index.ts            # Barrel exports
│   ├── atoms/              # Atomic components
│   │   ├── Button.tsx
│   │   ├── Badge.tsx
│   │   ├── Input.tsx
│   │   ├── Card.tsx
│   │   ├── Modal.tsx
│   │   ├── Spinner.tsx
│   │   ├── Typography.tsx
│   │   └── Icon.tsx
│   ├── molecules/          # Composite components
│   │   ├── SearchInput.tsx
│   │   ├── TabBar.tsx
│   │   ├── FilterBar.tsx
│   │   └── DataGrid.tsx
│   └── organisms/          # Complex components
│       ├── Navbar.tsx
│       ├── Footer.tsx
│       ├── LessonCard.tsx
│       ├── ProjectCard.tsx
│       └── KnowledgeCheck.tsx
├── _components/            # OLD: Kept for backward compatibility
├── _storybook/             # NEW: Storybook stories
├── _styles/                # NEW: Consolidated styling
├── features/               # Page-level feature components (slowly migrated)
└── _tokens/                # NEW: Design tokens
    ├── colors.ts
    ├── typography.ts
    └── spacing.ts
```

## 3. Component Rules

### Rule 1: Every Component MUST

- Be a **TypeScript function component** with typed props
- Accept and forward a `className` prop (for extension)
- Accept `...rest` / `...props` for native HTML attributes
- Have a **descriptive JSDoc comment**
- Be **accessible** (proper ARIA attributes, keyboard navigation)
- Be **unit tested** (or have Storybook interaction tests)

### Rule 2: Server Components by Default

```tsx
// WITHOUT 'use client' — this is a Server Component
interface CardProps {
  title: string;
  children: React.ReactNode;
  className?: string;
}

export function Card({ title, children, className }: CardProps) {
  return (
    <div className={`rounded-lg border p-4 ${className ?? ''}`}>
      <h3 className="text-lg font-semibold">{title}</h3>
      <div>{children}</div>
    </div>
  );
}
```

Only add `'use client'` when you need:
- `useState`, `useEffect`, `useRef`
- Event handlers
- Browser-only APIs
- Context providers

### Rule 3: Props Interface Naming

```tsx
// ComponentName + Props
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'ghost' | 'destructive';
  size?: 'sm' | 'default' | 'lg';
  loading?: boolean;
  disabled?: boolean;
  children: React.ReactNode;
  className?: string;
}
```

### Rule 4: Use cn() for Class Merging

```tsx
import { cn } from '../_components/utils'; // clsx + tailwind-merge

export function Button({ className, variant = 'primary', ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        'inline-flex items-center justify-center rounded-lg px-4 py-2',
        variant === 'primary' && 'bg-blue-600 text-white hover:bg-blue-700',
        variant === 'secondary' && 'bg-gray-200 text-gray-800 hover:bg-gray-300',
        className
      )}
      {...props}
    />
  );
}
```

### Rule 5: No Default Exports

Always use **named exports**:

```tsx
// ✅ GOOD
export function Button(props: ButtonProps) { ... }
export function Card(props: CardProps) { ... }

// ❌ BAD
export default function Button(props: ButtonProps) { ... }
```

## 4. Design Tokens

Design tokens live in `src/presentation/_tokens/`:

```typescript
// colors.ts
export const colors = {
  primary: {
    50: '#f0f0ff',
    100: '#e0e0ff',
    500: '#572EFF',
    600: '#4625CC',
    700: '#3A1FA8',
  },
  neutral: {
    50: '#fafafa',
    100: '#f5f5f5',
    200: '#e5e5e5',
    600: '#76777d',
    700: '#45464d',
    900: '#0a0a0a',
  },
  semantic: {
    success: '#22c55e',
    warning: '#f59e0b',
    error: '#ef4444',
    info: '#3b82f6',
  },
} as const;

// typography.ts
export const typography = {
  fontFamily: {
    sans: 'Inter, system-ui, sans-serif',
    mono: 'JetBrains Mono, Fira Code, monospace',
  },
  fontSize: {
    xs: '0.75rem',
    sm: '0.875rem',
    base: '1rem',
    lg: '1.125rem',
    xl: '1.25rem',
    '2xl': '1.5rem',
    '3xl': '1.875rem',
    '4xl': '2.25rem',
  },
  fontWeight: {
    normal: 400,
    medium: 500,
    semibold: 600,
    bold: 700,
  },
} as const;
```

## 5. The __components/ vs _components/ Distinction

| Directory | Status | Purpose |
|-----------|--------|---------|
| `__components/` | **NEW** | Future design system. Clean, well-typed, atomic components built from scratch |
| `_components/` | **LEGACY** | Existing components. Kept for backward compatibility. Will be deprecated. |

**Migration Path**: 
1. Build new components in `__components/` 
2. Add Storybook stories in `_storybook/`
3. Create wrapper components in `_components/` that delegate to `__components/` 
4. Eventually, `_components/` is removed

## 6. Styling Strategy

**New components** use Tailwind CSS utility classes with `cn()`:
```tsx
className={cn('px-4 py-2 rounded-lg', className)}
```

**Legacy components** use CSS Modules:
```tsx
import styles from '../_styles/css/component.module.css';
<div className={styles.container} />
```

**Migration**: New components should use Tailwind. CSS Module files should be consolidated during migrations.
