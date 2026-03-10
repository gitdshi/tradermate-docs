# Component Specifications

This document provides detailed specifications for key TraderMate UI components, including props, state, events, and usage guidelines.

## Table of Contents

1. [Layout Components](#layout-components)
   - [AppLayout](#applayout)
   - [Sidebar](#sidebar)
2. [Strategy Components](#strategy-components)
   - [StrategyList](#strategylist)
   - [StrategyDetail](#strategydetail)
   - [StrategyForm](#strategyform)
   - [StrategyOptimization](#strategyoptimization)
3. [Backtest Components](#backtest-components)
   - [BacktestJobList](#backtestjoblist)
   - [BacktestForm](#backtestform)
   - [BacktestResults](#backtestresults)
   - [BulkBacktestForm](#bulkbacktestform)
   - [BulkBacktestSummary](#bulkbacktestsummary)
4. [Analytics Components](#analytics-components)
   - [AnalyticsDashboard](#analyticsdashboard)
   - [RiskMetrics](#riskmetrics)
   - [PerformanceComparison](#performancecomparison)
5. [Portfolio Components](#portfolio-components)
   - [PortfolioManagement](#portfoliomanagement)
6. [Shared Components](#shared-components)
   - [ErrorBoundary](#errorboundary)
   - [StatusBadge](#statusbadge)
   - [LoadingSpinner](#loadingspinner)
   - [EmptyState](#emptystate)

---

## Layout Components

### AppLayout

The main layout wrapper used across all protected pages.

**Structure**:
- Fixed sidebar (left)
- Scrollable main content area (right)
- Responsive: collapses to hamburger menu on mobile

**Usage**:
```tsx
<AppLayout>
  <Dashboard />
</AppLayout>
```

**Implementation Notes**:
- Use full viewport height (`h-screen overflow-hidden`)
- Sidebar width: `w-64` on desktop, hidden on mobile
- Header with user menu at top-right

---

### Sidebar

Navigation sidebar with links to all pages.

**Props**: None (uses react-router for navigation)

**Links** (order):
1. Dashboard (`/`)
2. Strategies (`/strategies`)
3. Backtest (`/backtest`)
4. Market Data (`/market-data`)
5. Analytics (`/analytics`)
6. Portfolio (`/portfolio`)

**Active State**:
- Highlight current route with `bg-blue-50 border-l-4 border-blue-600`

**Icons** (Lucide React):
- Dashboard: `BarChart3`
- Strategies: `Folder`
- Backtest: `Play`
- Market Data: `TrendingUp`
- Analytics: `BarChart`
- Portfolio: `Briefcase`

---

## Strategy Components

### StrategyList

Displays list of strategies from database with version badges and action buttons.

**Props**:
```typescript
interface StrategyListProps {
  strategies: Strategy[];
  selectedId?: number;
  onSelect: (id: number) => void;
  onEdit?: (id: number) => void;
  onDelete?: (id: number) => void;
  loading?: boolean;
}
```

**Strategy Object**:
```typescript
type Strategy = {
  id: number;
  name: string;
  class_name: string;
  description?: string;
  version: number;
  created_at: string; // ISO
  updated_at: string; // ISO
};
```

**Features**:
- Shows strategy name + class name
- Version badge: `v{version}`
- Hover: subtle background (`hover:bg-gray-50`)
- Selected: blue border and background (`border-blue-500 bg-blue-50`)
- Edit/Delete buttons appear on hover

---

### StrategyDetail

Shows full details of a selected strategy, including code.

**Props**:
```typescript
interface StrategyDetailProps {
  strategy: StrategyDetail;
  isEditing: boolean;
  onEdit: () => void;
  onSave: (updates: StrategyUpdate) => Promise<void>;
  onCancel: () => void;
}
```

**StrategyDetail**:
```typescript
type StrategyDetail = Strategy & {
  code?: string;
  parameters?: Record<string, any> | string;
};
```

**Display Mode**:
- Class name (medium text)
- Description (if exists)
- Parameters section (if exists) - formatted as JSON in `<pre>`
- Code section - SyntaxHighlighter (python)

**Edit Mode**:
- Full-screen overlay (fixed inset-0)
- Split layout: left = Monaco code editor, right = metadata form
- Form fields: Name, Class Name, Description, Parameters (textarea with JSON validation)
- Save/Cancel buttons

---

### StrategyForm

Modal form for creating/editing strategy metadata (not code).

**Props**:
```typescript
interface StrategyFormProps {
  strategy?: Strategy; // if provided, edit mode
  onClose: () => void;
  onSubmit: (data: StrategyFormData) => Promise<void>;
}
```

**StrategyFormData**:
```typescript
type StrategyFormData = {
  name: string;
  class_name: string;
  description?: string;
  parameters?: Record<string, any>;
};
```

**Validation**:
- Name: required, unique check via API
- Class Name: required, matches Python class name convention (CamelCase)
- Parameters: valid JSON object (client-side JSON.parse check)

---

### StrategyOptimization

Interface for configuring and running parameter optimization.

**Props**:
None (self-contained component)

**State**:
- Selected strategy (dropdown)
- Symbol input
- Date range (start_date, end_date)
- Optimization method: 'genetic' | 'grid'
- Parameter ranges: dynamic rows (param_name, min, max)
- Job submission loading state
- Results table when job completes

**API Integration**:
- POST `/api/optimization` to submit
- Poll GET `/api/optimization/{job_id}` for status
- Display ranked parameter sets with metrics

**Optimization Parameters**:
```typescript
type ParamRange = {
  name: string; // e.g., "fast_window"
  min: number;
  max: number;
  step?: number; // for grid search
};
```

**Results Table Columns**:
- Rank
- Parameter set (JSON preview)
- Sharpe Ratio
- Total Return
- Max Drawdown
- Win Rate
- [Deploy] button (creates new strategy version)

---

## Backtest Components

### BacktestJobList

Table showing backtest jobs with status and actions.

**Props**:
```typescript
interface BacktestJobListProps {
  onViewResults: (jobId: string) => void;
  onViewBulkSummary?: (jobId: string) => void;
}
```

**Job Row Data**:
```typescript
type BacktestJob = {
  job_id: string;
  strategy_name: string;
  symbol: string;
  start_date: string;
  end_date: string;
  status: 'queued' | 'running' | 'completed' | 'failed';
  created_at: string;
  // optional fields for bulk jobs
  bulk_job_id?: string;
  child_job_ids?: string[];
};
```

**Status Badge Colors**:
- queued: gray
- running: blue (with spinner)
- completed: green
- failed: red

**Actions**:
- View Results: opens `BacktestResults` panel (only for completed)
- Cancel: optional cancel button (for running jobs)

---

### BacktestForm

Modal form to configure a single backtest.

**Props**:
```typescript
interface BacktestFormProps {
  onClose: () => void;
  onSubmitSuccess: (jobId: string) => void;
}
```

**Form Fields**:
- Strategy: dropdown (dynamic from `/api/strategies`)
- Symbol: text input (with autocomplete placeholder)
- Start Date: date picker
- End Date: date picker
- Initial Capital: number input (default 100000)
- Commission: number (decimal, e.g., 0.0003)
- Slippage: number (decimal)

**Validation**:
- All fields required
- start_date < end_date
- Symbol format validation (e.g., `000001.SZ`)

---

### BacktestResults

Detailed results display for a completed backtest.

**Props**:
```typescript
interface BacktestResultsProps {
  jobId: string;
  onClose: () => void;
}
```

**Data Fetch**:
- GET `/api/backtest/{job_id}`

**Sections**:
1. **Metrics Grid**: 2x3 or 3x3 grid of cards
   - Total Return
   - Annualized Return
   - Sharpe Ratio
   - Max Drawdown
   - Win Rate
   - Total Trades
   - Profit Factor
   - Average Win/Loss

2. **Equity Curve Chart** (Recharts LineChart)
   - X axis: date
   - Y axis: equity/capital
   - Optional: benchmark line

3. **Drawdown Chart** (area chart)

4. **Trades Table**: columns - Entry/Exit time, direction, price, P&L

**Export**: CSV download button for trade list

---

### BulkBacktestForm

Form for batch backtesting multiple symbols or parameter sets.

**Props**:
```typescript
interface BulkBacktestFormProps {
  onClose: () => void;
  onSubmitSuccess: (jobId: string) => void;
}
```

**Mode**:
- Multi-symbol: upload symbol list (CSV) or select from watchlist
- Multi-parameter: grid search over parameter ranges

**Output**:
- Creates a bulk job that spawns multiple child backtest jobs
- Returns bulk_job_id to track

---

### BulkBacktestSummary

Summary view of bulk backtest results with table comparison.

**Props**:
```typescript
interface BulkBacktestSummaryProps {
  jobId: string;
  onClose: () => void;
  onViewChildResult: (jobId: string) => void;
}
```

**Layout**:
- Summary stats (mean Sharpe, best/worst performers)
- Table rows: each child job as row with key metrics
- Click row to view individual results

---

## Analytics Components

### AnalyticsDashboard

Portfolio overview with charts and KPI cards.

**Props**: None (self-contained, uses global stores)

**Data Source**:
- Portfolio stats: GET `/api/portfolio/summary`
- Allocation: GET `/api/portfolio/allocation`
- Performance: GET `/api/analytics/performance`

**Layout**:
1. KPI cards row (4 cards)
2. Allocation pie chart (Recharts PieChart)
3. Performance area chart (cumulative returns)

**KPI Cards**:
- Market Value
- Total P&L (with % change, green/red)
- Day P&L
- Sharpe Ratio (YTD)

---

### RiskMetrics

Displays various risk indicators and charts.

**Props**: None

**Metrics** (cards):
- Volatility (annualized %)
- Max Drawdown (%)
- Value at Risk (VaR 95%)
- Beta
- Alpha
- Concentration (top N holdings %)
- Liquidity Ratio (if available)

**Charts**:
1. Drawdown underwater plot
2. Rolling volatility (30, 60, 90 day)

---

### PerformanceComparison

Compare multiple backtest strategies side-by-side.

**Props**:
None (self-contained)

**Features**:
- Multi-select dropdown to choose strategies (from completed backtests)
- Comparison table: metrics columns for each strategy
- Overlay equity curve chart (one line per strategy)
- Expandable strategy cards with full details

---

## Portfolio Components

### PortfolioManagement

Current positions and closed trades management.

**Props**: None

**Sections**:

1. **Summary Cards** (4 cards):
   - Total Market Value
   - Unrealized P&L
   - Realized P&L (today)
   - Overall ROI

2. **Open Positions Table**:
   - Columns: Symbol, Qty, Avg Price, Current Price, Market Value, P&L, P&L %, [Close]
   - Close button opens modal to confirm and specify exit quantity

3. **Closed Trades History**:
   - Table with pagination
   - Columns: Date, Symbol, Side (Buy/Sell), Qty, Avg Price, Exit Price, P&L

**Actions**:
- Close position: POST `/api/portfolio/positions/{id}/close`
- Refresh: refetch all data

---

## Shared Components

### ErrorBoundary

React error boundary wrapper.

**Props**:
```typescript
interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ComponentType<{ error: Error; reset: () => void }>;
}
```

**Usage**:
```tsx
<ErrorBoundary>
  <SomeComponent />
</ErrorBoundary>
```

**Default Fallback**:
- Error message
- "Try again" button calls `reset()`

---

### StatusBadge

Visual status indicator with color coding.

**Props**:
```typescript
interface StatusBadgeProps {
  status: string;
  label?: string;
}
```

**Status Colors** (defined in design system):
- synced: green
- data_newer: blue
- project_newer: yellow
- different: orange
- data_only: purple
- project_only: pink
- queued/running/completed/failed: as per backtest status

---

### LoadingSpinner

Animated spinner component.

**Props**:
```typescript
interface LoadingSpinnerProps {
  size?: 'sm' | 'md' | 'lg';
  className?: string;
}
```

**Sizes**:
- sm: size={16}
- md: size={24}
- lg: size={32}

**Usage**:
```tsx
<LoadingSpinner size="md" className="text-blue-600" />
```

---

### EmptyState

Placeholder when no data is available.

**Props**:
```typescript
interface EmptyStateProps {
  icon?: React.ReactNode;
  title: string;
  description?: string;
  action?: React.ReactNode; // e.g., a button
}
```

**Example**:
```tsx
<EmptyState
  icon={<Folder className="h-12 w-12" />}
  title="No strategies"
  description="Create your first trading strategy"
  action={<Button onClick={create}>Create Strategy</Button>}
/>
```

---

## API Hooks (patterns)

While existing code uses direct API calls, consider using custom hooks for reusability:

```typescript
// Example pattern
function useStrategies() {
  const [data, setData] = useState<Strategy[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const fetch = async () => {
    setLoading(true);
    try {
      const { data } = await strategiesAPI.list();
      setData(data);
    } catch (err) {
      setError((err as any).response?.data?.detail || 'Error');
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => { fetch(); }, []);

  return { data, loading, error, refetch: fetch };
}
```

---

## TypeScript Types

Centralize types in `src/types/index.ts`:

```typescript
// Strategy
export type Strategy = {
  id: number;
  name: string;
  class_name: string;
  description?: string;
  version: number;
  created_at: string;
  updated_at: string;
};

export type StrategyDetail = Strategy & {
  code?: string;
  parameters?: Record<string, any> | string;
};

// Backtest
export type BacktestJob = {
  job_id: string;
  strategy_name: string;
  symbol: string;
  start_date: string;
  end_date: string;
  status: 'queued' | 'running' | 'completed' | 'failed';
  created_at: string;
  bulk_job_id?: string;
};

export type BacktestResult = {
  job_id: string;
  metrics: {
    total_return: number;
    annual_return: number;
    sharpe_ratio: number;
    max_drawdown: number;
    win_rate: number;
    total_trades: number;
  };
  equity_curve: Array<{ date: string; equity: number }>;
  trades: TradeRecord[];
};

// Portfolio
export type Position = {
  symbol: string;
  quantity: number;
  avg_price: number;
  current_price: number;
  market_value: number;
  unrealized_pnl: number;
  unrealized_pnl_pct: number;
};

export type Trade = {
  id: number;
  date: string;
  symbol: string;
  side: 'buy' | 'sell';
  quantity: number;
  avg_price: number;
  pnl: number;
};
```

---

## Testing Guidelines

### Component Testing (Vitest)

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('StrategyList', () => {
  const mockStrategies = [
    { id: 1, name: 'Test Strategy', class_name: 'TestClass', version: 1 }
  ];

  test('renders strategy list', () => {
    render(
      <StrategyList
        strategies={mockStrategies}
        selectedId={1}
        onSelect={() => {}}
      />
    );
    expect(screen.getByText('Test Strategy')).toBeInTheDocument();
  });

  test('calls onSelect when clicked', async () => {
    const onSelect = vi.fn();
    render(
      <StrategyList
        strategies={mockStrategies}
        selectedId={undefined}
        onSelect={onSelect}
      />
    );
    await userEvent.click(screen.getByText('Test Strategy'));
    expect(onSelect).toHaveBeenCalledWith(1);
  });
});
```

---

## Accessibility Checklist

- [ ] All interactive elements have `aria-label` or visible text
- [ ] Form inputs have associated `<label>` or `aria-label`
- [ ] Tables have proper `<thead>` and scope attributes
- [ ] Color contrast meets WCAG AA standards
- [ ] Keyboard navigation works (Tab, Enter, Escape)
- [ ] Focus states are visible (`focus:ring`)
- [ ] Modal dialogs trap focus

---

## Performance Considerations

- **Virtualization**: For tables with >50 rows, consider `react-window`
- **Memoization**: Use `useMemo` and `useCallback` for expensive calculations
- **Chart Optimization**: Limit data points for Recharts (sample if >1000 points)
- **Code Editor**: Monaco Editor is heavy; lazy load only in edit mode
- **Polling**: Use WebSocket or polling with reasonable intervals for job status

---

## Version History

- 2026-03-02: Initial component specs document created
