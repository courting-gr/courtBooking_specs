# Figma AI to Code — Guidelines for the Court Booking Admin Portal

This document describes how to use Figma AI to generate UI mockups for the court-booking-admin-web React application, and how to translate those mockups into production code during Phase 6 implementation.

## Why Figma AI for This Project

The admin portal is the first client-side application in the project. Previous phases (1-5) focused on backend services. Phase 6 introduces the React admin web app alongside backend APIs. Using Figma AI to rapidly generate mockups gives us:

- Visual alignment before writing code — catch layout and flow issues early
- A shared reference between design intent and implementation
- Faster iteration than coding UI blind from requirements text

## Tech Stack Context

The Phase 6 admin portal uses:
- React 18 with TypeScript
- Ant Design 5.x as the component library
- React Router v6 for navigation
- React Query (TanStack Query) for server state
- react-i18next for internationalization
- Recharts for analytics charts
- WebSocket (STOMP) for real-time notifications

Figma mockups should target Ant Design's visual language since that's what gets implemented.

---

## Step 1: Structuring Your Figma Project

### Recommended Page Structure

Create one Figma page per major admin portal section:

```
📄 00 — Design System & Tokens
📄 01 — Login & Onboarding Flow
📄 02 — Dashboard
📄 03 — Court Management
📄 04 — Booking Management
📄 05 — Analytics & Revenue
📄 06 — Support Tickets
📄 07 — Settings (Profile, Notifications, Reminders, Defaults)
📄 08 — Platform Admin Pages
📄 09 — Shared Components & Patterns
```

### Frame Sizes

- Desktop primary: 1440 × 900 (standard admin viewport)
- Desktop wide: 1920 × 1080 (for data-heavy pages like analytics)
- Tablet: 1024 × 768 (responsive breakpoint reference)

No mobile frames needed — the admin portal is desktop-first (mobile admin is Phase 8 Flutter app).

---

## Step 2: Using Figma AI Effectively

### Prompting Strategy

When using Figma AI's "Make designs" or text-to-design features, be specific about:

1. The component library: "Using Ant Design components..."
2. The data shape: reference the actual API response fields
3. The layout pattern: sidebar nav, content area, header
4. The state: loading, empty, error, populated

### Example Prompts by Page

**Dashboard:**
```
Design an admin dashboard using Ant Design components with:
- Left sidebar navigation (collapsed by default, expandable)
- Top header with user avatar, notification bell with badge count, and language switcher
- Main content area with:
  - Row of 4 stat cards: Today's Bookings, Pending Confirmations, Revenue This Month (€), Occupancy Rate (%)
  - "Needs Attention" alert list showing booking reminders with action buttons
  - Recent notifications list (5 items)
  - Court summary card showing total/visible/hidden counts
- Color scheme: professional blue/white, green for confirmed, yellow for pending, red for cancelled
```

**Court Management List:**
```
Design a court management page using Ant Design Table with:
- Page header "My Courts" with "Add Court" primary button
- Filter bar: court type dropdown, location type toggle (Indoor/Outdoor), visibility toggle
- Table columns: Image thumbnail, Name, Type (with colored tag), Location Type (badge), Base Price (€), Confirmation Mode, Visibility (switch), Actions (edit/delete)
- Empty state: illustration with "No courts yet — add your first court" message
- Bulk action bar appears when rows are selected: "Toggle Visibility" button
```

**Booking Calendar:**
```
Design a booking calendar view using Ant Design Calendar or a weekly grid with:
- Court selector dropdown at top
- Week view showing time slots 08:00-22:00 on Y axis, Mon-Sun on X axis
- Color-coded booking blocks: green (confirmed), yellow (pending), red (cancelled), blue (manual), orange (unpaid)
- Click on empty slot opens "Create Manual Booking" drawer
- Click on booking block opens booking detail drawer
- Export button (iCal) in header
```

### Tips for Better AI Output

- Reference Ant Design component names explicitly (Table, Card, Statistic, Tag, Badge, Drawer, Modal, Form, Select, DatePicker, Calendar, Tabs, Alert, Notification)
- Specify data density — admin UIs are information-dense, not marketing pages
- Include empty states and loading states in your prompts
- Ask for both the list view and the detail/edit view for each entity
- Specify the sidebar navigation structure so it's consistent across pages

---

## Step 3: Design System Tokens

### Colors (Map to Ant Design Theme)

| Token | Hex | Usage |
|-------|-----|-------|
| Primary | `#1677FF` | Ant Design default blue — buttons, links, active states |
| Success | `#52C41A` | Confirmed bookings, active status, verified badge |
| Warning | `#FAAD14` | Pending confirmations, trial banner, attention items |
| Error | `#FF4D4F` | Cancelled, rejected, errors, overdue items |
| Info | `#1677FF` | Manual bookings, informational badges |
| Neutral BG | `#F5F5F5` | Page background, card backgrounds |
| Sidebar BG | `#001529` | Dark sidebar (Ant Design Sider default) |

### Booking Status Color Map

| Status | Color | Ant Design Tag |
|--------|-------|----------------|
| CONFIRMED | Green | `<Tag color="success">` |
| PENDING_CONFIRMATION | Gold | `<Tag color="warning">` |
| CANCELLED | Red | `<Tag color="error">` |
| REJECTED | Red | `<Tag color="error">` |
| COMPLETED | Default | `<Tag color="default">` |
| MANUAL | Blue | `<Tag color="processing">` |
| NO_SHOW | Volcano | `<Tag color="volcano">` |

### Typography

- Use Ant Design's default font stack (system fonts)
- Page titles: 24px semibold
- Section headers: 18px semibold
- Table headers: 14px medium
- Body text: 14px regular
- Small/caption: 12px regular

---

## Step 4: Page-by-Page Mockup Checklist

For each admin portal page, create mockups for these states:

### Required States Per Page

1. **Populated** — normal state with realistic data
2. **Empty** — no data yet (first-time user experience)
3. **Loading** — skeleton/shimmer placeholders
4. **Error** — API failure with retry option

### Pages to Mock (Priority Order)

**P0 — Core court owner pages (implement first):**
- [ ] Dashboard (populated + empty court owner)
- [ ] Court list + add/edit court form
- [ ] Booking calendar (week view)
- [ ] Pending bookings queue
- [ ] Settings: profile, notification preferences, reminder rules, court defaults

**P1 — Analytics and support:**
- [ ] Revenue analytics (charts + table)
- [ ] Usage analytics (heatmap)
- [ ] Export dialog
- [ ] Support ticket list + detail + create form
- [ ] Audit log viewer

**P2 — Platform admin pages:**
- [ ] User management list + detail
- [ ] Verification review queue
- [ ] Feature flag management
- [ ] Platform-wide analytics
- [ ] Dispute escalation view
- [ ] Global search results

**P3 — Onboarding flows:**
- [ ] Stripe Connect onboarding banner + status
- [ ] Verification submission form
- [ ] Verification pending/approved/rejected states

---

## Step 5: From Figma to Code

### Using Figma with Kiro

Kiro has a Figma power that can read Figma designs and help translate them to code. The workflow:

1. Design the page in Figma (using AI or manually)
2. Share the Figma file URL
3. In Kiro, paste the Figma URL in chat and ask to implement the component
4. Kiro reads the design and generates React + Ant Design code matching the layout

### Naming Convention: Figma Frame → React Component

| Figma Frame Name | React Component | File Path |
|-----------------|-----------------|-----------|
| `Dashboard/Main` | `DashboardPage` | `src/pages/Dashboard/DashboardPage.tsx` |
| `Dashboard/StatCards` | `DashboardStatCards` | `src/pages/Dashboard/components/StatCards.tsx` |
| `Courts/List` | `CourtListPage` | `src/pages/Courts/CourtListPage.tsx` |
| `Courts/AddEdit` | `CourtFormDrawer` | `src/pages/Courts/components/CourtFormDrawer.tsx` |
| `Bookings/Calendar` | `BookingCalendarPage` | `src/pages/Bookings/BookingCalendarPage.tsx` |
| `Bookings/PendingQueue` | `PendingBookingsPage` | `src/pages/Bookings/PendingBookingsPage.tsx` |
| `Analytics/Revenue` | `RevenueAnalyticsPage` | `src/pages/Analytics/RevenueAnalyticsPage.tsx` |
| `Support/TicketList` | `SupportTicketListPage` | `src/pages/Support/SupportTicketListPage.tsx` |
| `Settings/Profile` | `ProfileSettingsPage` | `src/pages/Settings/ProfileSettingsPage.tsx` |
| `Admin/Users` | `AdminUserManagementPage` | `src/pages/Admin/UserManagementPage.tsx` |

### What to Include in Figma Annotations

For each frame, add annotations (Figma comments or text notes) specifying:

- API endpoint the page calls (e.g., `GET /api/dashboard`)
- Key interaction behaviors (e.g., "clicking row opens detail drawer")
- Which fields are editable vs read-only
- Validation rules for form fields
- Which actions trigger confirmation modals
- Real-time update behavior (e.g., "notification count updates via WebSocket")

---

## Step 6: Responsive Considerations

The admin portal is desktop-first but should degrade gracefully:

- Sidebar collapses to icons at < 1200px
- Tables switch to card layout at < 768px (but this is low priority)
- Charts resize responsively via Recharts' `ResponsiveContainer`
- Modals and drawers use Ant Design's responsive width props

In Figma, the 1440px frame is the primary design target. A 1024px frame is nice-to-have for verifying sidebar collapse behavior.

---

## Step 7: Ant Design Component Mapping

Quick reference for which Ant Design components to use for common admin patterns:

| UI Pattern | Ant Design Component |
|-----------|---------------------|
| Data tables | `Table` with `columns`, pagination, sorting, filtering |
| Forms | `Form` + `Form.Item` with validation rules |
| Side panel editing | `Drawer` (right-side, 600px width) |
| Confirmation dialogs | `Modal.confirm()` |
| Status indicators | `Tag` (colored), `Badge` (with count) |
| Stat cards | `Card` + `Statistic` |
| Charts | Recharts (`LineChart`, `BarChart`, `PieChart`) inside Ant `Card` |
| Navigation | `Layout.Sider` + `Menu` with `Menu.Item` |
| Notifications | `notification.open()` for toasts, `Badge` for bell icon |
| Loading | `Skeleton` for page load, `Spin` for actions |
| Empty states | `Empty` with custom description and action button |
| File upload | `Upload` with `Dragger` variant for images |
| Date/time | `DatePicker`, `RangePicker`, `TimePicker` |
| Search | `Input.Search` or `AutoComplete` |
| Tabs | `Tabs` for sub-navigation within a page |
| Breadcrumbs | `Breadcrumb` for page hierarchy |

---

## Step 8: Iteration Workflow

1. **Generate** — Use Figma AI to create initial mockups from the prompts above
2. **Refine** — Adjust layouts, spacing, and data density manually in Figma
3. **Annotate** — Add API endpoint references and interaction notes
4. **Review** — Walk through the court owner journey (Steps 4-9 from master requirements) against the mockups
5. **Implement** — Use Kiro's Figma power or manually translate to React + Ant Design
6. **Validate** — Compare implemented UI against Figma mockups for fidelity

### Figma AI Iteration Tips

- Start with the dashboard — it touches the most data and sets the visual tone
- Generate the sidebar navigation as a shared component first, then reuse across pages
- For forms, generate the "add" version first, then adapt for "edit" (pre-populated fields)
- Generate both the table view and the detail drawer for each entity in the same session so they're visually consistent
