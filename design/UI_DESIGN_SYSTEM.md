# TraderMate UI Design System

## Overview

TraderMate Portal uses a comprehensive design system built on **TailwindCSS** with custom design tokens. This document defines the visual language, component patterns, and interaction guidelines.

## Design Principles

1. **Clarity First** - Complex trading data must be presented clearly
2. **Consistency** - Uniform patterns across all pages
3. **Efficiency** - Streamlined workflows for power users
4. **Accessibility** - WCAG 2.1 AA compliant
5. **Performance** - Fast rendering even with large datasets

## Color Palette

### Primary Colors

```css
/* Main Brand Colors */
--color-primary-50: #eff6ff;
--color-primary-100: #dbeafe;
--color-primary-200: #bfdbfe;
--color-primary-300: #93c5fd;
--color-primary-400: #60a5fa;
--color-primary-500: #3b82f6;  /* Primary brand */
--color-primary-600: #2563eb;  /* Primary hover */
--color-primary-700: #1d4ed8;
--color-primary-800: #1e40af;
--color-primary-900: #1e3a8a;

/* Neutral/Gray Scale */
--color-gray-50: #f9fafb;
--color-gray-100: #f3f4f6;
--color-gray-200: #e5e7eb;
--color-gray-300: #d1d5db;
--color-gray-400: #9ca3af;
--color-gray-500: #6b7280;
--color-gray-600: #4b5563;
--color-gray-700: #374151;
--color-gray-800: #1f2937;
--color-gray-900: #111827;

/* Semantic Colors */
--color-success: #10b981;   /* Positive P&L, success states */
--color-warning: #f59e0b;   /* Caution, warnings */
--color-error: #ef4444;     /* Losses, errors, destructive actions */
--color-info: #3b82f6;      /* Informational messages */
```

### Status Colors

```css
/* Strategy/Job Status */
--color-status-synced: #10b981;      /* Green */
--color-status-data-newer: #3b82f6;  /* Blue */
--color-status-project-newer: #f59e0b; /* Yellow */
--color-status-different: #f97316;   /* Orange */
--color-status-data-only: #8b5cf6;   /* Purple */
--color-status-project-only: #ec4899; /* Pink */
```

## Typography

### Font Family

```css
--font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
--font-mono: 'JetBrains Mono', 'Fira Code', Consolas, Monaco, monospace;
```

### Font Sizes & Line Heights

| Token | Font Size | Line Height | Usage |
|-------|-----------|-------------|-------|
| `text-xs` | 12px | 16px | Captions, metadata |
| `text-sm` | 14px | 20px | Secondary text, labels |
| `text-base` | 16px | 24px | Body text, form inputs |
| `text-lg` | 18px | 28px | Subheadings |
| `text-xl` | 20px | 28px | Card titles |
| `text-2xl` | 24px | 32px | Section headers |
| `text-3xl` | 30px | 36px | Page titles |

### Font Weights

- `font-normal` (400) - Body text, regular content
- `font-medium` (500) - Emphasized text, labels
- `font-semibold` (600) - Subheadings, button text
- `font-bold` (700) - Page titles, important headings

## Spacing

Based on 4px grid system:

```css
--space-1: 4px;
--space-2: 8px;
--space-3: 12px;
--space-4: 16px;
--space-5: 20px;
--space-6: 24px;
--space-8: 32px;
--space-10: 40px;
--space-12: 48px;
--space-16: 64px;
```

## Border Radius

```css
--radius-sm: 4px;   /* Small elements, chips */
--radius-md: 6px;   /* Default cards, inputs */
--radius-lg: 8px;   /* Larger cards, modals */
--radius-xl: 12px;  /* dialogs, popovers */
```

## Shadows

```css
--shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
--shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.1);
--shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1), 0 4px 6px -4px rgb(0 0 0 / 0.1);
--shadow-xl: 0 20px 25px -5px rgb(0 0 0 / 0.1), 0 8px 10px -6px rgb(0 0 0 / 0.1);
```

## Component Patterns

### Layout Structure

All pages follow this structure:

```tsx
<div className="flex flex-col h-screen overflow-hidden">
  {/* Header/Navigation (fixed) */}
  <header>...</header>
  
  {/* Main Content Area */}
  <div className="flex-1 overflow-auto p-6">
    {/* Page content */}
  </div>
</div>
```

### Card Component Pattern

```tsx
<div className="bg-white rounded-lg shadow border border-gray-200 p-6">
  <h3 className="text-lg font-semibold mb-4">Card Title</h3>
  {/* Card content */}
</div>
```

### Form Element Styling

```tsx
// Input fields
<input
  className="w-full px-3 py-2 border border-gray-300 rounded-md 
             focus:outline-none focus:ring-2 focus:ring-blue-500 
             focus:border-transparent"
  {...props}
/>

// Buttons
<button className="px-4 py-2 bg-blue-600 text-white rounded-md 
                   hover:bg-blue-700 transition-colors disabled:opacity-50">
  Action
</button>

// Secondary button
<button className="px-4 py-2 bg-gray-100 text-gray-700 rounded-md 
                   hover:bg-gray-200 transition-colors">
  Cancel
</button>
```

### Table Styling

```tsx
<table className="min-w-full divide-y divide-gray-200">
  <thead className="bg-gray-50">
    <tr>
      <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
        Column Header
      </th>
    </tr>
  </thead>
  <tbody className="bg-white divide-y divide-gray-200">
    {/* Rows */}
  </tbody>
</table>
```

## Page-Specific Patterns

### Dashboard

- Grid layout using CSS Grid or Flexbox
- Stat cards: 4-column grid on desktop, stacked on mobile
- Chart containers: fixed height (300-400px)

### Strategies Page

- Two-column layout: list (left, 4 columns) + details (right, 8 columns)
- Code editor: Monaco Editor with Python syntax highlighting
- Diff view: Git-style diff with green (added) / red (removed)

### Backtest Page

- Job list table with status badges
- Results: metrics cards + equity curve chart + drawdown chart

### Analytics Pages

- Tabs for navigation between dashboard, risk metrics, comparison
- Charts: Recharts library with responsive containers
- KPI cards with trend indicators

### Portfolio Page

- Summary cards at top (market value, P&L, etc.)
- Positions table with expandable rows
- Closed trades history in separate section

## Icon Usage

Icons from **Lucide React**. Consistent sizing:

- Small icons (in buttons): `size={16}`
- Normal icons: `size={20}`
- Large icons (featured): `size={24}` or `size={32}`

## Interactions

### Loading States

```tsx
// Button loading
<button disabled={loading} className="...">
  {loading ? (
    <RefreshCw size={16} className="animate-spin" />
  ) : (
    'Submit'
  )}
</button>

// Table loading skeleton
{loading ? (
  <tr>
    <td colSpan={N} className="text-center py-8">Loading...</td>
  </tr>
) : (...)}
```

### Empty States

- Centered message with optional icon
- Secondary button to create/add new items if applicable

### Error States

- Red background border/background for error messages
- Inline validation: red text below form field
- Toast/alert component for API errors

## Accessibility

- All interactive elements keyboard-navigable
- Focus rings: `focus:ring-2 focus:ring-blue-500`
- ARIA labels on icon-only buttons
- Color contrast ratio > 4.5:1 for normal text

## Responsive Breakpoints

Tailwind breakpoints:
- `sm`: 640px
- `md`: 768px
- `lg`: 1024px
- `xl`: 1280px

Use responsive classes like `grid-cols-1 lg:grid-cols-12` for adaptive layouts.

## Implementation Notes

- All styles use Tailwind utility classes, avoid custom CSS files
- For complex reusable patterns, create components in `src/components/`
- Use TypeScript for all components with proper prop typing
- Follow existing patterns in the codebase for consistency
