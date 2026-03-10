# TraderMate Wireframes & Page Layouts

This document provides wireframe descriptions and layout specifications for all TraderMate pages. Actual implementations should follow the UI Design System.

## 1. Login Page

**Route**: `/login`

**Layout**: Centered card, max-width 400px

```
┌─────────────────────────────────────────────┐
│                                             │
│            ┌─────────────────┐              │
│            │   TraderMate    │              │
│            │    Logo         │              │
│            └─────────────────┘              │
│                                             │
│  Username: [________________]              │
│  Password: [________________]              │
│                                             │
│  [ ] Remember me          [Forgot?]        │
│                                             │
│  ┌─────────────────────────────┐           │
│  │        Sign In              │           │
│  └─────────────────────────────┘           │
│                                             │
│  Don't have an account? [Sign up]         │
│                                             │
└─────────────────────────────────────────────┘
```

**Components**:
- LoginForm with validation
- Link to Register page
- Optional "Forgot Password" placeholder

**API Calls**:
- POST `/api/auth/login`
- On success: store tokens, redirect to `/dashboard`

---

## 2. Register Page

**Route**: `/register`

**Layout**: Centered card, similar to Login

```
┌─────────────────────────────────────────────┐
│  Create Account                             │
│                                             │
│  Username: [________________]              │
│  Email: [________________]                 │
│  Password: [________________]              │
│  Confirm Password: [________________]      │
│                                             │
│  ┌─────────────────────────────┐           │
│  │        Sign Up              │           │
│  └─────────────────────────────┘           │
│                                             │
│  Already have an account? [Sign in]       │
└─────────────────────────────────────────────┘
```

---

## 3. Dashboard

**Route**: `/`

**Layout**: Full-width content area with navigation sidebar

```
┌──────────────────────────────────────────────────┐
│  Sidebar │ Header (User menu)                   │
├──────────┼──────────────────────────────────────┤
│          │                                      │
│  📊      │  Welcome back, {username}           │
│  Dashboard│                                      │
│  📈      │  ┌────────┐ ┌────────┐ ┌────────┐  │
│  Strategies │ │  Queue │ │  Jobs  │ │ Active │  │
│  📉      │ │   Stats│ │ Running│ │  Users │  │
│  Backtest  │ └────────┘ └────────┘ └────────┘  │
│  📰      │                                      │
│  Market Data│  Recent Activity (table/list)    │
│  📊      │                                      │
│  Analytics │                                      │
│  💼      │                                      │
│  Portfolio │                                      │
│          │                                      │
└──────────┴──────────────────────────────────────┘
```

**Key Components**:
- Stats cards (queue size, running jobs, success rate)
- Recent backtest jobs table
- Quick action buttons

---

## 4. Strategies Page

**Route**: `/strategies`

**Layout**: Two-panel split (list + details)

```
┌────────────────────────────────────────────────────────────┐
│ Sidebar │ Header                                            │
├─────────┼──────────────────────────────────────────────────┤
│         │  [New Strategy] [Refresh]                         │
│  🔍     │  ┌────────────────────────────────────────────┐ │
│  Search │  │  Strategies (12)               ⚙️          │ │
│         │  ├────────────────────────────────────────────┤ │
│         │  │ List View | Compare                        │ │
│  📋 All │  ├────────────────────────────────────────────┤ │
│         │  │  ┌─────────────────────────────────────┐  │ │
│         │  │  │ Strategy A                v1    ✏️ 🗑️│  │ │
│  💾 My  │  │  │ Triple MA Strategy              ✏️ 🗑️│  │ │
│         │  │  │ RSI Divergence                  ✏️ 🗑️│  │ │
│  ⭐ Fav │  │  └─────────────────────────────────────┘  │ │
│         │  │                                          │ │
│         │  │  (if selected) Strategy Details Panel   │ │
│         │  │  ┌─────────────────────────────────────┐│ │
│         │  │  │ Class: TripleMAStrategy            ││ │
│         │  │  │ Parameters: {"fast": 10, "slow":20}││ │
│         │  │  │                                  ││ │
│         │  │  │ [Code Monaco Editor - full height]││ │
│         │  │  └─────────────────────────────────────┘│ │
│         │  └────────────────────────────────────────────┘ │
│         │                                                    │
└─────────┴────────────────────────────────────────────────────┘
```

**Tabs**:
- `Strategies` (files): List and manage strategy files
- `Optimize`: Strategy optimization interface (see separate section)

**Features**:
- Left: strategy list with version badge, edit/delete buttons
- Right: detail panel with class name, parameters, Monaco code editor
- Full-screen editor mode when editing
- Version history with diff view modal

**Components**:
- StrategyList
- StrategyDetail
- StrategyForm (modal)
- MonacoEditor wrapper

---

## 5. Backtest Page

**Route**: `/backtest`

**Layout**: Job management + results display

```
┌────────────────────────────────────────────────────────────┐
│ Sidebar │ Header                                            │
├─────────┼──────────────────────────────────────────────────┤
│         │  [New Backtest] [Bulk Test]                       │
│         │  ┌────────────────────────────────────────────┐  │
│         │  │ Backtest Jobs                          🔍   │  │
│         │  ├────────────────────────────────────────────┤  │
│         │  │ Status │ Strategy │ Symbol │ Date │ ...   │  │
│         │  │ ───────┼──────────┼────────┼──────┼───── │  │
│         │  │ ⏳     │ TripleMA │ 000001 │ 2024 │ ...  │  │
│         │  │ ✅     │ RSI      │ 600000 │ 2024 │ ...  │  │
│         │  └────────────────────────────────────────────┘  │
│         │                                                    │
│         │  (if job selected) Results Panel                  │
│         │  ┌─────────────────────────────────────────────┐ │
│         │  │ 📈 Equity Curve                 [Export]   │ │
│         │  │  [Line chart: equity over time]            │ │
│         │  │                                              │ │
│         │  │  Metrics Grid:                              │ │
│         │  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐         │ │
│         │  │  │Total│ │Sharpe│ │MaxDD │ │Win% │         │ │
│         │  │  │ +15%│ │ 1.2 │ │ -8%  │ │ 58% │         │ │
│         │  │  └─────┘ └─────┘ └─────┘ └─────┘         │ │
│         │  └─────────────────────────────────────────────┘ │
│         │                                                    │
└─────────┴────────────────────────────────────────────────────┘
```

**Components**:
- BacktestJobList (table with filters)
- BacktestForm (modal with strategy/symbol/date/capital inputs)
- BacktestResults (equity curve + metrics + trades table)

---

## 6. Market Data Page

**Route**: `/market-data`

**Layout**: Data exploration with charts

```
┌────────────────────────────────────────────────────────────┐
│ Sidebar │ Header: Market Data                              │
├─────────┼──────────────────────────────────────────────────┤
│         │  [Search Symbol] [Date Range] [Indicator]        │
│         │                                                  │
│         │  ┌──────────────────────────────────────────┐   │
│         │  │ candlestick chart with volume           │   │
│         │  │ [TradingView or custom Recharts impl]   │   │
│         │  └──────────────────────────────────────────┘   │
│         │                                                  │
│         │  Symbol Information Table                       │
│         │  ┌─────────────────────────────────────┐       │
│         │  │ Name | Code | Exchange | Industry  │       │
│         │  │ ...  | ...  | ...      | ...       │       │
│         │  └─────────────────────────────────────┘       │
│         │                                                  │
│         │  Fundamentals Panel (tabs: overview, ratios)   │
│         │                                                  │
└─────────┴────────────────────────────────────────────────────┘
```

---

## 7. Analytics Page

**Route**: `/analytics`

**Tabs**:
1. **Portfolio Analytics** (Dashboard)
2. **Risk Metrics**
3. **Performance Comparison**

### 7.1 Portfolio Analytics Tab

```
┌────────────────────────────────────────────────────────────┐
│ Sidebar │ Analytics / Portfolio Analytics                 │
├─────────┼──────────────────────────────────────────────────┤
│         │  Summary Cards:                                  │
│         │  ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│         │  │Total P&L │ │Win Rate  │ │Sharpe    │       │
│         │  │ +$12,345 │ │ 58%      │ │ 1.42     │       │
│         │  └──────────┘ └──────────┘ └──────────┘       │
│         │                                                  │
│         │  ┌──────────────────────────────────────────┐   │
│         │  │ Portfolio Allocation Pie Chart           │   │
│         │  │ [Sector/Strategy breakdown]              │   │
│         │  └──────────────────────────────────────────┘   │
│         │                                                  │
│         │  ┌──────────────────────────────────────────┐   │
│         │  │ Performance Over Time [Area chart]      │   │
│         │  │ [Cumulative returns vs benchmark]       │   │
│         │  └──────────────────────────────────────────┘   │
│         │                                                  │
└─────────┴────────────────────────────────────────────────────┘
```

**Components**:
- AnalyticsDashboard: summary cards + allocation chart + performance chart

### 7.2 Risk Metrics Tab

```
┌────────────────────────────────────────────────────────────┐
│ Sidebar │ Analytics / Risk Metrics                        │
├─────────┼──────────────────────────────────────────────────┤
│         │  ┌──────────────────────────────────────────┐   │
│         │  │ Volatility   ──►▼ 18.5%                 │   │
│         │  │ Max Drawdown ──►▼ -12.3%                │   │
│         │  │ VaR (95%)    ──►▼ -$8,765               │   │
│         │  │ Beta         ──►▼ 1.15                  │   │
│         │  │ Alpha        ──►▼ 0.82%                 │   │
│         │  │ Concentration ──► Top 3: 67%            │   │
│         │  └──────────────────────────────────────────┘   │
│         │                                                  │
│         │  Drawdown Chart:                                │
│         │  [Line chart showing peak-to-trough declines]  │
│         │                                                  │
│         │  Rolling Volatility (30/60/90 day)             │
│         │  [Multiple line chart]                         │
│         │                                                  │
└─────────┴────────────────────────────────────────────────────┘
```

**Components**:
- RiskMetrics: key metrics cards + drawdown chart + rolling volatility

### 7.3 Performance Comparison Tab

```
┌────────────────────────────────────────────────────────────┐
│ Sidebar │ Analytics / Performance Comparison             │
├─────────┼──────────────────────────────────────────────────┤
│         │  Select 2+ strategies: [Dropdowns] [Compare]   │
│         │                                                  │
│         │  Comparison Table:                              │
│         │  ┌─────────────────────────────────────────┐   │
│         │  │ Metric         │ Strat A │ Strat B │ ... │   │
│         │  │ Total Return   │  +15%   │  +12%   │     │   │
│         │  │ Sharpe Ratio   │  1.42   │  1.18   │     │   │
│         │  │ Max Drawdown   │ -8.2%   │ -10.5%  │     │   │
│         │  │ Win Rate       │  58%    │  53%    │     │   │
│         │  └─────────────────────────────────────────┘   │
│         │                                                  │
│         │  Overlay Equity Curve Chart:                    │
│         │  [Multi-line chart comparing all strategies]   │
│         │                                                  │
│         │  Individual Strategy Cards (expandable)        │
│         │                                                  │
└─────────┴────────────────────────────────────────────────────┘
```

**Components**:
- PerformanceComparison: strategy selector + comparison table + overlay chart

---

## 8. Portfolio Page

**Route**: `/portfolio`

**Layout**: Two sections - current positions + trade history

```
┌────────────────────────────────────────────────────────────┐
│ Sidebar │ Portfolio                                        │
├─────────┼──────────────────────────────────────────────────┤
│         │  Summary Cards:                                 │
│         │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐             │
│         │  │ MV  │ │ P&L │ │ Day │ │ ROI │             │
│         │  │$... │ │+... │ │+... │ │...% │             │
│         │  └─────┘ └─────┘ └─────┘ └─────┘             │
│         │                                                  │
│         │  Open Positions:                               │
│         │  ┌─────────────────────────────────────────┐   │
│         │  │ Symbol | Qty | Price | P&L | [Close]  │   │
│         │  │ ...    | ... | ...  | ... │ ...       │   │
│         │  └─────────────────────────────────────────┘   │
│         │                                                  │
│         │  Closed Trades:                                │
│         │  ┌─────────────────────────────────────────┐   │
│         │  │ Date | Symbol | Side | P&L | Notes    │   │
│         │  │ ...  | ...    | ...  | ... │ ...      │   │
│         │  └─────────────────────────────────────────┘   │
│         │                                                  │
└─────────┴────────────────────────────────────────────────────┘
```

**Components**:
- PortfolioManagement: position list + close position modal + trade history table

---

## 9. Strategy Optimization (Tab on Strategies Page)

**Route**: Part of `/strategies` (Optimize tab)

**Layout**: Configuration form + results display

```
┌────────────────────────────────────────────────────────────┐
│ Sidebar │ Strategies / Optimize                           │
├─────────┼──────────────────────────────────────────────────┤
│         │  ┌──────────────────────────────────────────┐   │
│         │  │ Configure Optimization                 │   │
│         │  │ Strategy: [Dropdown]                   │   │
│         │  │ Symbol: [000001.SZ]                    │   │
│         │  │ Period: [2020-01-01] to [2023-12-31]  │   │
│         │  │ Method: [GA │ Grid Search]             │   │
│         │  │                                      │   │
│         │  │ Parameter Ranges:                     │   │
│         │  │ fast_window: [min=5]─►▼[max=20]      │   │
│         │  │ slow_window: [min=10]─►▼[max=60]     │   │
│         │  │                                      │   │
│         │  │ [Start Optimization]                 │   │
│         │  └──────────────────────────────────────────┘   │
│         │                                                  │
│         │  Progress Bar (if job running):                 │
│         │  ████████████░░░░░░░░░░ 50% (gen 25/50)       │
│         │  Current best Sharpe: 1.85                     │
│         │                                                  │
│         │  Results Table (when complete):                │
│         │  ┌─────────────────────────────────────────┐   │
│         │  │ Sharpe │ Params │ Actions │ [Deploy] │   │
│         │  │ 1.85   │ {...}  │ View    │ Apply    │   │
│         │  │ 1.72   │ {...}  │ View    │ Apply    │   │
│         │  └─────────────────────────────────────────┘   │
│         │                                                  │
└─────────┴────────────────────────────────────────────────────┘
```

**Components**:
- StrategyOptimization: config form + progress indicator + results table

---

## 10. Modals & Dialogs

### Strategy Form Modal (Create/Edit)

```
┌────────────────────────────────────────────────────────────┐
│                                                           │
│  Edit Strategy                      [X]                   │
│  ────────────────────────────────────────────            │
│                                                           │
│  Name:           [Triple MA Strategy          ]         │
│  Class Name:     [TripleMAStrategy           ]         │
│  Description:    [My custom triple MA...      ]         │
│                                                           │
│  Default Parameters (JSON):                              │
│  ┌──────────────────────────────────────────────────┐   │
│  │ {                                                │   │
│  │   "fast_window": 10,                            │   │
│  │   "slow_window": 20,                            │   │
│  │   "fixed_size": 1                               │   │
│  │ }                                               │   │
│  └──────────────────────────────────────────────────┘   │
│                                                           │
│  Code: (Monaco Editor - full width, ~400px height)      │
│  ┌──────────────────────────────────────────────────┐   │
│  │ from vnpy_ctastrategy import CtaTemplate        │   │
│  │ ...                                               │   │
│  └──────────────────────────────────────────────────┘   │
│                                                           │
│                    [Cancel]  [Save]                     │
│                                                           │
└────────────────────────────────────────────────────────────┘
```

### Backtest Form Modal

```
┌────────────────────────────────────────────────────────────┐
│  New Backtest                             [X]             │
│  ────────────────────────────────────────────             │
│                                                           │
│  Strategy:        [Select strategy ──►▼]                │
│  Symbol:          [000001.SZ                   ]        │
│  Start Date:      [2024-01-01          📅]             │
│  End Date:        [2024-12-31          📅]             │
│  Initial Capital: [100000                    ]        │
│  Commission:      [0.0003 (0.03%)          ]        │
│  Slippage:        [0.0 (0%)                ]        │
│                                                           │
│                    [Cancel]  [Submit]                   │
│                                                           │
└────────────────────────────────────────────────────────────┘
```

---

## 11. Responsive Behavior

### Mobile (< 768px)

- Sidebar: hidden behind hamburger menu or bottom nav
- Two-panel layouts: stack vertically
- Tables: horizontal scroll or card view
- Modals: full-width, sheet-style

### Tablet (768px - 1024px)

- Sidebar: collapsible to icons-only
- Grid layouts: adjust column counts
- Maintain reasonable touch targets (min 44px)

---

## 12. State & Loading Patterns

### Empty State

```
┌──────────────────────────────────┐
│            ⚡                     │
│    No strategies found           │
│    Create your first strategy    │
│    [Create Strategy]             │
└──────────────────────────────────┘
```

### Loading State

- Full page: skeleton loader covering content area
- Component: spinner (RefreshCw animate-spin) + loading text
- Button: disabled + spinner

---

## 13. Navigation Flow

```
Unauthenticated:
  /login → /register (to sign up)
  /login → /dashboard (after login)

Authenticated:
  / → redirect to /dashboard
  Any /protected route → redirect to /login if not authenticated

Protected Routes:
  /dashboard
  /strategies
  /backtest
  /market-data
  /analytics
  /portfolio

Logout:
  Clear tokens, redirect to /login
```

---

## 14. Page Transitions

- No full-page reloads (SPA using React Router)
- Smooth fade-in for modal overlays (100-200ms)
- Tab switches: instant, no animation

---

## 15. Error Handling

- API errors: Toast notification or alert banner at top
- Form validation: Inline error messages below fields
- 404 pages: Show "Page not found" with link to home
- 500 errors: User-friendly message + support contact

---

## Notes for Implementation

- All dimensions use Tailwind's spacing scale (4px grid)
- Use Tailwind's responsive prefixes (sm:, md:, lg:, xl:)
- Maintain consistency with the existing component patterns
- Refer to `UI_DESIGN_SYSTEM.md` for complete style guide
