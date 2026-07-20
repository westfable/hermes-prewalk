# Prewalk Task Description Template

## How to Describe a Prewalk Task

A good prewalk task description gives the frontier model enough to plan without over-constraining the approach. Use this template as a starting point.

## Template

```
Build [FEATURE] for [APP/SECTION].

Requirements:
- [Specific functional requirement 1]
- [Specific functional requirement 2]
- [Specific functional requirement 3]

Design:
- Use [existing component library / design system]
- [Layout description: e.g., "tabbed sections", "sidebar + main content", "card grid"]
- [Styling notes: e.g., "match existing settings page styling"]

Technical:
- [Framework: e.g., "Next.js App Router", "React + Vite"]
- [State management approach, if relevant]
- [API/data requirements, if relevant]

Verification:
- [How to verify: e.g., "page renders at /settings", "npm run build passes", "form submits successfully"]
```

## Examples

### Settings Page

```
Build a settings page with tabbed sections for profile, notifications, and billing.

Requirements:
- Three tabs: Profile (name, email, avatar), Notifications (email/push toggles), Billing (plan info, payment method)
- Each tab has its own form with validation
- Save button per tab, shows success toast on save

Design:
- Use existing Button, Input, Card, and Tabs components
- Tab navigation at top, content below
- Match existing dashboard styling

Technical:
- Next.js App Router, src/app/settings/page.tsx
- Client components for form interactivity
- Existing useAuth hook for current user data

Verification:
- Page renders at /settings without errors
- Tab switching works
- Form validation blocks invalid input
- npm run build passes
```

### Dashboard Widget

```
Build a revenue analytics widget for the main dashboard.

Requirements:
- Line chart showing monthly revenue for the last 12 months
- Toggle between revenue and profit metrics
- Summary stats: total, MoM change, best month
- Export to CSV button

Design:
- Use existing ChartContainer and StatCard components
- Responsive: full width on desktop, stacked on mobile
- Dark mode compatible (use existing color tokens)

Technical:
- React component in src/components/dashboard/
- Fetch from /api/analytics/revenue endpoint
- Use existing useTheme for dark mode

Verification:
- Widget renders in the dashboard layout
- Chart displays with mock data
- Toggle switches metrics
- npm run build passes
```
