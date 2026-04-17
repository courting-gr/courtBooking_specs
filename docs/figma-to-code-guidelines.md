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


---

## Appendix: Figma AI Prompts for Phase 6 Admin Portal

These prompts are designed to be used sequentially in Figma AI (Make). Each generates one major section of the admin portal. Generate them as separate Figma Make projects, then consolidate the best elements into a single Figma design file for the Phase 6 design spec.

The prompts reference actual API response shapes from the backend so the UI matches the data contract.

---

### Prompt 1: App Shell — Sidebar Navigation + Header + Layout

```
Build a court booking admin portal with a professional dark sidebar layout (1440x900 desktop).

LEFT SIDEBAR (dark navy #001529, 240px wide, collapsible to 80px icons):
Navigation menu items with icons:
- Dashboard (home icon) — default active
- Courts (building icon)
- Bookings (calendar icon)
- Analytics (chart icon)
- Support (message icon)
- Settings (gear icon)
Divider line, then:
- Admin section (only visible for PLATFORM_ADMIN role):
  - Users (people icon)
  - Verifications (shield icon)
  - Feature Flags (toggle icon)
  - Platform Analytics (globe icon)

Bottom of sidebar: user avatar, name "Kostas E.", role badge "Court Owner", collapse toggle button.

TOP HEADER (white, 64px height):
- Left: Breadcrumb showing current page path (e.g., "Dashboard")
- Center: Global search bar with placeholder "Search courts, bookings..." (280px wide)
- Right section: Language switcher (EL/EN toggle), notification bell icon with red badge showing "3", user dropdown with avatar

MAIN CONTENT AREA: Light gray background #F5F5F5, 24px padding. Show placeholder text "Page content loads here" centered.

Include a second frame showing the sidebar in collapsed state (80px, icons only).
```

---

### Prompt 2: Court Owner Dashboard

```
Build a court owner dashboard page for a sports court booking platform (1440x900, white cards on #F5F5F5 background). This is the main landing page after login.

TOP BANNER (conditional, yellow/amber):
Show a "Complete Payment Setup" banner with text "Connect your Stripe account to accept customer bookings" and a "Setup Payments" blue button. This banner appears when Stripe Connect is not active.

STAT CARDS ROW (4 cards, equal width):
1. "Today's Bookings" — large number "12", subtitle "across 4 courts", blue accent
2. "Pending Confirmations" — large number "3", subtitle "awaiting your review", amber/warning accent, clickable
3. "Revenue This Month" — "€2,450.00", subtitle "+12% vs last month", green accent
4. "Occupancy Today" — "68%", circular progress indicator, subtitle "above average"

SECOND ROW (2 columns, 60/40 split):

LEFT COLUMN — "Needs Attention" card:
A list of 4 alert items, each with:
- Alert icon (warning triangle for unpaid, clock for pending)
- Title: "Court A — Saturday 10:00, unpaid booking, 3 days away"
- Subtitle: "Customer: K***@gmail.com"
- Action buttons row: "View Details" (link), "Contact" (secondary), "Cancel Booking" (danger text), "Dismiss" (ghost)
Group alerts by type: "Unpaid Bookings (2)" and "Pending Confirmations (1)"

RIGHT COLUMN — two stacked cards:
Card 1 — "Recent Notifications" (5 items):
Each item: icon, message text, timestamp ("2 min ago"), unread dot indicator
Example items: "New booking confirmed — Court B, Tomorrow 14:00", "Payment received — €45.00", "Booking cancelled by customer"
"View All" link at bottom

Card 2 — "Court Summary":
Three rows: "Total Courts: 6", "Visible: 4" (green dot), "Hidden: 2" (gray dot)
Verification status: green checkmark "Verified"
Stripe status: green checkmark "Connected"

BOTTOM ROW — "Quick Actions" card:
Row of 3 buttons: "Create Manual Booking" (primary), "Confirm Pending Bookings (3)" (with badge), "View Calendar"
```

---

### Prompt 3: Court Management — List + Add/Edit Form

```
Build a court management page for a sports booking admin portal (1440x900).

PAGE HEADER:
Title "My Courts" with subtitle "6 courts"
Right side: "Add Court" primary blue button

FILTER BAR (below header):
- Court type dropdown: All, Tennis, Padel, Basketball, Football 5x5
- Location type toggle buttons: All | Indoor | Outdoor
- Search input: "Search by name..."
- Visibility filter: "All" / "Visible" / "Hidden"

COURTS TABLE:
Columns: Thumbnail (60x40 image), Name, Type (colored tag — green for Tennis, blue for Padel, orange for Basketball, red for Football), Location (Indoor/Outdoor badge), Base Price (€/session), Confirmation Mode (Instant/Manual tag), Visibility (toggle switch), Actions (Edit pencil icon, Delete trash icon)

Show 5 sample rows with realistic Greek court names like "Γήπεδο Τένις Γλυφάδας", "Padel Arena Κηφισιά"

Table footer: pagination showing "1-5 of 6 courts"

When rows are selected (checkbox column), show a floating action bar at bottom: "2 courts selected" with "Toggle Visibility" and "Bulk Pricing" buttons

SECOND FRAME — "Add Court" drawer (slides from right, 640px wide):
Form sections:
1. Basic Info: Name (Greek) input, Name (English) input, Description (Greek) textarea, Description (English) textarea
2. Court Configuration: Type dropdown, Location Type radio (Indoor/Outdoor), Duration (number input, minutes), Max Capacity (number input), Base Price (number input with € prefix)
3. Booking Settings: Confirmation Mode radio (Instant/Manual), Confirmation Timeout (number, hours, shown only when Manual selected), Waitlist Enabled toggle
4. Address: Address text input, Map pin selector placeholder (gray box with pin icon)
5. Images: Drag-and-drop upload zone "Drop images here or click to upload. JPEG, PNG, WebP. Max 10MB each, up to 20 images"
6. Amenities: Checkbox grid — Parking, Showers, Equipment Rental, Lighting, Changing Rooms, WiFi, Cafe, Pro Shop

Footer: "Cancel" ghost button, "Create Court" primary button
```

---

### Prompt 4: Booking Management — Calendar + Pending Queue

```
Build a booking management page with two tabs for a sports court admin portal (1440x900).

TAB BAR at top: "Calendar View" (active) | "Pending Bookings" | "Manual Booking"

TAB 1 — CALENDAR VIEW:
Top controls row:
- Court selector dropdown: "All Courts" / individual court names
- Date navigation: left arrow, "This Week (Apr 14-20, 2026)", right arrow
- View toggle: Day | Week | Month
- "Export iCal" button (secondary)

Weekly calendar grid:
- Y-axis: time slots from 08:00 to 22:00 (1-hour increments)
- X-axis: Mon through Sun with dates
- Color-coded booking blocks filling time slots:
  - Green blocks: "Confirmed" — show court name + customer initial
  - Yellow blocks: "Pending" — show court name + "Awaiting confirmation"
  - Blue blocks: "Manual" — show court name + "Walk-in"
  - Red strikethrough blocks: "Cancelled"
- Empty slots are clickable (subtle hover effect with "+" icon)
- Legend bar at bottom showing color meanings

TAB 2 — PENDING BOOKINGS:
List of pending booking cards, each showing:
- Customer name (partially masked: "K*** E."), court name, date/time
- Amount held: "€45.00 authorized"
- Countdown timer: "Auto-cancels in 18h 32m"
- Two action buttons: "Confirm" (green), "Reject" (red outline)
- Optional message input that expands on click

Top right: "Bulk Actions" dropdown — "Confirm All (3)", "Reject All"

TAB 3 — MANUAL BOOKING form:
- Court selector dropdown
- Date picker
- Time slot selector (grid of available slots for selected date)
- Duration (auto-filled from court default)
- Customer info (optional): Name, Phone, Email, Notes
- Recurring toggle: "Repeat weekly" with "For X weeks" number input (1-12)
- "Create Booking" primary button
```

---

### Prompt 5: Analytics & Revenue

```
Build an analytics page for a sports court booking admin portal (1440x900).

TAB BAR: "Revenue" (active) | "Usage" | "Heatmap"

CONTROLS BAR:
- Date range picker: "Apr 1, 2026" to "Apr 17, 2026" with preset buttons (7d, 30d, 90d, 1y)
- Court filter dropdown: "All Courts"
- Court type filter: "All Types"
- "Export" button with dropdown: CSV, PDF

TAB 1 — REVENUE:
Top stat cards row (4 cards):
- Total Bookings: "156" with "+8% vs previous period" in green
- Revenue (Gross): "€7,240.00"
- Platform Fees: "€724.00"
- Revenue (Net): "€6,516.00"

Line chart: "Revenue Trend" showing daily revenue over the selected period, with a dashed line for the comparison period

Table below: "Revenue by Court"
Columns: Court Name, Type (tag), Bookings, Gross Revenue, Fees, Net Revenue, Occupancy Rate (progress bar)
5 sample rows, sortable columns
Total row at bottom in bold

TAB 2 — USAGE:
- Overall occupancy rate: large "72%" with circular gauge
- Bar chart: "Peak Hours" showing booking count by hour of day (8am-10pm)
- Bar chart: "Busiest Days" showing booking count by day of week

TAB 3 — HEATMAP:
7x15 grid (days of week x hours 8-22)
Color intensity from white (0 bookings) through light blue to dark blue (high bookings)
Hover tooltip: "Monday 18:00 — 12 bookings, 85% occupancy"
```

---

### Prompt 6: Support Tickets

```
Build a support ticket management page for an admin portal (1440x900).

SPLIT LAYOUT: Left panel (400px) ticket list, Right panel ticket detail.

LEFT PANEL — Ticket List:
- Header: "Support Tickets" with "New Ticket" button
- Filter tabs: All (12) | Open (5) | In Progress (3) | Resolved (4)
- Search input: "Search tickets..."
- List of ticket cards, each showing:
  - Subject line (bold, truncated)
  - Category tag (colored: Booking=blue, Payment=green, Technical=red, Payout Issues=purple)
  - Priority badge (Normal=gray, High=orange, Urgent=red)
  - Status badge
  - "Updated 2h ago" timestamp
  - Assigned to avatar or "Unassigned"
- Selected ticket highlighted with blue left border

RIGHT PANEL — Ticket Detail:
- Header: Subject, Status dropdown (changeable), Priority, Category
- Info bar: Created date, Ticket ID, Related Booking link (clickable), Assigned to dropdown
- Message thread (chat-style):
  - Customer messages: left-aligned, light gray bubble, name + timestamp
  - Agent messages: right-aligned, light blue bubble, name + "Support Agent" badge + timestamp
  - Attachment chips: file icon + filename + size, clickable
- Reply box at bottom: textarea with formatting toolbar, "Attach Files" button (paperclip icon), "Send Reply" primary button
- Right sidebar (collapsible): Customer info card (masked email/phone), booking context card, ticket metadata
```

---

### Prompt 7: Settings Page — Profile, Notifications, Reminders, Court Defaults

```
Build a settings page with vertical tabs for a court owner admin portal (1440x900).

LEFT VERTICAL TABS (200px):
- Profile & Business (active)
- Notification Preferences
- Reminder Rules
- Court Defaults
- Security & Privacy
- Active Sessions

TAB 1 — PROFILE & BUSINESS:
Two-column form:
Left column: Profile photo upload circle with "Change Photo" link, Display Name input, Email (read-only with edit icon), Phone input, Language selector (Greek/English radio)
Right column (Court Owner fields): Business Name, Tax ID (AFM), Business Type dropdown (Sole Proprietor/Company/Association), Business Address, Contact Phone
"Save Changes" button at bottom

TAB 2 — NOTIFICATION PREFERENCES:
Table with columns: Event Type, Email (toggle), Push (toggle), In-App (toggle)
Rows: New Booking, Booking Cancelled, Pending Confirmation, Payment Received, Payout Completed, Payment Dispute, Reminder Alerts
Below table: "Do Not Disturb" section with Enable toggle, Start Time picker (default 23:00), End Time picker (default 07:00)
"Save Preferences" button

TAB 3 — REMINDER RULES:
Header: "Smart Reminder Rules" with "Add Rule" button, subtitle "Get notified about bookings that need attention. Max 10 rules."
List of rule cards, each showing:
- Rule type tag (Unpaid Booking, Pending Confirmation, Low Occupancy, No Contact Manual)
- Description: "Alert me X days before event if booking is unpaid, Y hours before threshold"
- Applies to: "All Courts" or specific court names
- Channels: email/push/in-app icons (highlighted if active)
- Enable/disable toggle on the right
- Edit (pencil) and Delete (trash) icons

TAB 4 — COURT DEFAULTS:
Form: Default Duration (number, minutes), Default Confirmation Mode (Instant/Manual radio), Default Confirmation Timeout (hours), Default Cancellation Policy section with tier rows (threshold hours + refund percentage), "Add Tier" button
Default Amenities: checkbox grid
"Save Defaults" button

TAB 5 — SECURITY & PRIVACY:
Active Sessions list (device icon, browser name, IP partially masked, last active timestamp, "Revoke" red link)
Data Export: "Request Data Export" button with description "Download all your personal data (GDPR)"
Account Deletion: "Delete Account" danger button with warning text
```

---

### Prompt 8: Platform Admin Pages (PLATFORM_ADMIN role only)

```
Build platform admin pages for a court booking admin portal (1440x900). These pages are only visible to platform administrators.

PAGE 1 — USER MANAGEMENT:
Header: "User Management" with search bar and role filter dropdown (All/Customer/Court Owner/Support Agent/Admin)
Table columns: Avatar, Name, Email (masked: k***@gmail.com), Role (colored tag), Status (Active/Suspended badge), Verified (checkmark/x for court owners), Stripe (connected/not for court owners), Registered date, Actions (View/Suspend/Unsuspend)
Show 6 sample rows mixing roles
Clicking a row opens a detail drawer showing full user info, linked OAuth providers, session count, and action buttons

PAGE 2 — VERIFICATION QUEUE:
Header: "Verification Requests" with count badge "(4 pending)"
Filter tabs: Pending (4) | Approved | Rejected
Card list for pending requests, each showing:
- Business name, Tax ID, Business Type
- Submitted date, SLA countdown ("36h remaining" or "SLA exceeded" in red)
- Document preview thumbnail (PDF icon)
- "View Documents" link
- Two action buttons: "Approve" (green) with optional notes field, "Reject" (red) with required reason textarea
Previous submission history shown if this is a re-submission

PAGE 3 — FEATURE FLAGS:
Header: "Feature Flags" with "Add Flag" button
Table: Flag Key (monospace font), Description, Status (toggle switch: Enabled green / Disabled gray), Last Modified date, Modified By
Sample flags: "open_matches_enabled", "waitlist_enabled", "advertisements_enabled", "split_payments_enabled"
Toggle switches are interactive

PAGE 4 — GLOBAL SEARCH RESULTS:
Search bar at top (full width, prominent)
Results grouped by type with tabs: All | Courts (3) | Bookings (5) | Users (2)
Each result card shows: entity type icon, primary text (court name / booking ID / user name), secondary text (address / date+time / email), status badge, "View" link
```

---

### Tips for Using These Prompts

1. Generate each prompt as a separate Figma Make project
2. Review and refine each one — adjust spacing, colors, data to look realistic
3. Once satisfied, consolidate the best frames into a single Figma design file
4. Use that consolidated file as the design reference in the Phase 6 design spec
5. During implementation, share specific Figma frame URLs with Kiro to generate React + Ant Design code

The prompts are ordered by priority — Dashboard and Court Management are the most critical to get right first since they're the court owner's primary workflow.


---

### Additional Prompts — Missing Flows and Corner Cases

The following prompts cover flows and pages that were missing from the initial 8 prompts. These are needed for complete Phase 6 coverage.

---

### Prompt 9: Login Page + Stripe Connect Onboarding + Verification Flow

```
Build the authentication and onboarding screens for a court booking admin portal (1440x900).

FRAME 1 — LOGIN PAGE (centered card on gradient background):
- App logo "CourtBooking" at top
- "Welcome back" heading, "Sign in to your admin portal" subtitle
- Three OAuth buttons stacked vertically:
  - "Continue with Google" (white button with Google icon)
  - "Continue with Facebook" (blue button with Facebook icon)
  - "Continue with Apple" (black button with Apple icon)
- Divider line
- "Don't have an account? Register as Court Owner" link
- Footer: language toggle (EL/EN), "Terms & Privacy" link

FRAME 2 — STRIPE CONNECT ONBOARDING STATUS PAGE:
Full-page card showing Stripe Connect setup progress:
- Step indicator (3 steps): 1. Account Created (green check), 2. Identity Verification (in progress, spinner), 3. Bank Account (gray, pending)
- Current status: "PENDING" with amber badge
- "Your Stripe account is being verified. This usually takes 1-2 business days."
- Action buttons: "Resume Onboarding" (primary, redirects to Stripe hosted page), "Check Status" (secondary)
- Info box: "Until setup is complete, you can create manual bookings but cannot accept customer payments."
- Payout details section (disabled/grayed out): Payout schedule, balance, payout history — all showing "Available after setup"

FRAME 3 — VERIFICATION SUBMISSION FORM (court owner):
Card with form:
- Header: "Verify Your Business" with progress indicator
- Business Name input
- Tax ID (AFM) input with Greek format hint
- Business Type dropdown: Sole Proprietor, Company, Association
- Business Address input
- Document upload zone: "Upload proof of court ownership or lease (PDF, JPEG, PNG, max 10MB)"
- Uploaded file preview with filename, size, remove button
- "Submit for Verification" primary button
- Status states shown as separate frames:
  - PENDING_REVIEW: amber banner "Your verification is under review. You can set up courts while waiting."
  - APPROVED: green banner with checkmark "Your business is verified. Courts are now publicly visible."
  - REJECTED: red banner "Verification rejected: [reason text]. Please re-submit with corrections." with "Re-submit" button
```

---

### Prompt 10: Court Detail Sub-Pages — Availability, Holidays, Pricing

```
Build court detail management sub-pages for a sports court admin portal (1440x900). These are accessed by clicking "Edit" on a court from the court list, shown as tabs within a court detail view.

TAB BAR: "Details" | "Availability" (active) | "Holidays" | "Pricing" | "Cancellation Policy"

TAB — AVAILABILITY:
Section 1 — "Recurring Weekly Schedule":
A 7-row grid (Monday through Sunday), each row showing:
- Day name on the left
- Time range bars: visual bars showing open hours (e.g., 08:00-22:00 as a colored bar)
- Edit icon to modify each day's windows
- "Add Window" button per day for split schedules (e.g., 08:00-12:00 and 14:00-22:00)
"Save Schedule" button at bottom

Section 2 — "Date Overrides" (below):
- Calendar month view with blocked dates highlighted in red
- List of overrides next to calendar: date, reason, time range (or "Full day"), source (Manual/Holiday), delete button
- "Add Override" button opens a form: date picker, optional start/end time, reason text input

TAB — HOLIDAYS:
Two sections side by side:
Left: "National Holidays" — list of 13 Greek holidays with checkboxes to subscribe this court. Each shows name (Greek), calculated date for current year. "Apply Selected" button.
Right: "Custom Holidays" — list of court-owner-created holidays with name, date pattern, recurrence indicator. "Add Custom Holiday" button. Each has edit/delete icons.
Bottom: "Applied Holidays" calendar view showing all subscribed holidays as colored blocks on a year calendar.

TAB — PRICING:
Section 1 — "Base Price": Large display "€25.00 / session" with edit icon
Section 2 — "Pricing Rules" table:
Columns: Day(s), Time Range, Multiplier, Effective Price
Sample rows: "Mon-Fri 08:00-16:00, 1.0x, €25.00", "Mon-Fri 16:00-22:00, 1.5x, €37.50 (Peak)", "Sat-Sun All Day, 1.3x, €32.50"
"Add Rule" button, each row has edit/delete icons

TAB — CANCELLATION POLICY:
Tier list showing refund rules:
- "24+ hours before: 100% refund" (green bar)
- "12-24 hours before: 50% refund" (yellow bar)
- "Less than 12 hours: 0% refund" (red bar)
Visual timeline bar showing the tiers
"Add Tier" button, each tier has edit/delete
Note: "Platform fee (10%) is always non-refundable" info text
"Use Default Policy" toggle at top (applies court defaults)
```

---

### Prompt 11: Booking Detail Drawer + Audit Trail

```
Build a booking detail view as a right-side drawer (640px wide) for a court booking admin portal.

HEADER:
- Booking ID: "#BK-2026-0412" with copy icon
- Status badge: large colored badge (CONFIRMED green / PENDING yellow / CANCELLED red)
- Close (X) button top right

SECTION 1 — BOOKING INFO:
Two-column layout:
- Court: "Γήπεδο Τένις Γλυφάδας" (link to court detail)
- Date: "Saturday, April 18, 2026"
- Time: "10:00 - 11:30 (90 min)"
- Type: "Customer Booking" blue tag (or "Manual" blue tag)
- People: "4 / 4"
- Confirmation Mode: "Instant" tag

SECTION 2 — CUSTOMER INFO (with data masking):
- Name: "Kostas E."
- Email: "k***@gmail.com" with "Show full" link (triggers audit log entry)
- Phone: "+30 69** *** *45" with "Show full" link
- Booking created: "Apr 15, 2026 14:32"

SECTION 3 — PAYMENT DETAILS:
- Amount: "€37.50"
- Platform Fee: "€3.75 (10%)"
- Court Owner Net: "€33.75"
- Payment Status: "CAPTURED" green badge
- Stripe Payment ID: "pi_3Ox..." with link to Stripe dashboard
- Payment Method: "Visa ending 4242"
Payment timeline (vertical):
  - "Apr 15 14:32 — Payment authorized (€37.50)"
  - "Apr 15 14:32 — Payment captured"
  - "Apr 16 — Transfer to court owner initiated"

SECTION 4 — AUDIT TRAIL (scrollable):
Timeline list of all status changes:
- "Apr 15 14:32 — Created by Customer (Kostas E.)"
- "Apr 15 14:32 — Confirmed (Instant mode)"
- "Apr 15 14:33 — Notification sent to court owner"
Each entry shows: timestamp, action, actor name, actor role badge

SECTION 5 — ACTIONS (bottom, sticky):
Conditional buttons based on status:
- If CONFIRMED: "Flag No-Show" (orange), "Cancel Booking" (red outline)
- If PENDING: "Confirm" (green), "Reject" (red outline) with message input
- If CANCELLED: "View Refund Details" (link)
- "Mark as Paid Externally" button (for manual bookings with NOT_REQUIRED status)
```

---

### Prompt 12: Audit Log Viewer

```
Build an audit log viewer page for a court booking admin portal (1440x900).

PAGE HEADER: "Activity Log" with subtitle "Immutable record of all your actions"

FILTER BAR:
- Date range picker (from/to)
- Action type dropdown: All, Court Created, Court Updated, Court Deleted, Pricing Updated, Availability Updated, Cancellation Policy Updated, Settings Updated, Data Exported, Verification Submitted
- Court filter dropdown: All Courts / specific court names
- Search input: "Search by details..."

AUDIT LOG TABLE:
Columns: Timestamp, Action (colored tag), Court/Entity, Details (truncated JSON preview), expand arrow
Sample rows:
- "Apr 17, 10:32" | "COURT_UPDATED" blue tag | "Padel Arena Κηφισιά" | "basePriceCents: 2500 → 3000" | expand
- "Apr 16, 15:10" | "AVAILABILITY_UPDATED" green tag | "Γήπεδο Τένις" | "Added window: Mon 08:00-22:00" | expand
- "Apr 16, 09:00" | "DATA_EXPORTED" gray tag | "—" | "CSV export, Apr 1-16, 3 courts" | expand

Expanded row shows full before/after JSON diff in a code block with syntax highlighting, green for additions, red for removals.

Pagination at bottom: "Showing 1-20 of 156 entries"

Note at bottom: "Audit logs are retained for 2 years and cannot be modified or deleted."
```

---

### Prompt 13: Empty States, Loading Skeletons, Error States, Confirmation Modals

```
Build a collection of UI states and modals for a court booking admin portal (1440x900). Show each as a separate frame.

FRAME 1 — EMPTY STATES (4 variants):
a) Empty courts: Illustration of an empty court, "No courts yet", "Add your first court to get started", "Add Court" primary button
b) Empty bookings: Calendar illustration, "No bookings this week", "Bookings will appear here when customers book your courts"
c) Empty analytics: Chart illustration, "Not enough data yet", "Analytics will be available after your first bookings"
d) Empty support tickets: Message illustration, "No support tickets", "All clear! No open tickets"

FRAME 2 — LOADING SKELETONS (3 variants):
a) Dashboard loading: Gray shimmer rectangles matching the stat cards layout, shimmer bars for the needs-attention list
b) Table loading: Gray shimmer rows matching the courts table layout (6 rows of shimmer bars)
c) Detail drawer loading: Shimmer blocks matching the booking detail drawer sections

FRAME 3 — ERROR STATES (3 variants):
a) API error: Red alert banner at top of page "Something went wrong loading your data" with "Retry" button and "Contact Support" link
b) 404 page: Large "404" text, "Page not found", "The page you're looking for doesn't exist or has been moved", "Back to Dashboard" button
c) Network error: Offline icon, "No internet connection", "Check your connection and try again", "Retry" button

FRAME 4 — CONFIRMATION MODALS (4 variants):
a) Delete court: "Delete Court?" title, "This will permanently delete 'Padel Arena Κηφισιά' and all its availability settings. This cannot be undone.", "Cancel" ghost button, "Delete Court" red button
b) Reject booking: "Reject Booking?" title, "The customer will receive a full refund of €37.50.", Reason textarea (required), "Cancel" ghost button, "Reject Booking" red button
c) Suspend user: "Suspend User?" title, "This will prevent k***@gmail.com from logging in.", Reason textarea (required), "Cancel" ghost button, "Suspend User" red button
d) Bulk confirm: "Confirm 3 Bookings?" title, list of 3 booking summaries, "This will capture €112.50 in payments.", "Cancel" ghost button, "Confirm All" green button

FRAME 5 — NOTIFICATION DROPDOWN (from header bell icon):
Dropdown panel (360px wide, max 400px tall):
- Header: "Notifications" with "Mark all as read" link and unread count badge
- List of 5 notification items, each with: icon (booking/payment/alert), message text, relative timestamp ("2 min ago"), unread blue dot
- Unread items have light blue background
- "View All Notifications" link at bottom
- Empty state: "No new notifications"

FRAME 6 — SESSION TIMEOUT WARNING:
Modal overlay: "Session Expiring", "Your session will expire in 5 minutes due to inactivity.", "Stay Logged In" primary button, "Log Out" secondary button, countdown timer "4:32 remaining"
```

---

### Prompt 14: Dispute Escalation + Support Metrics (Platform Admin)

```
Build dispute escalation and support metrics pages for platform admin users (1440x900).

FRAME 1 — DISPUTE ESCALATION PAGE:
Header: "Payment Disputes" with filter tabs: All | Open | Under Review | Won | Lost
Table columns: Dispute ID, Booking ID (link), Court Owner, Customer, Amount (€), Reason (tag: fraudulent/duplicate/product_not_received), Status (badge), Created Date, Actions
Sample rows with realistic data

Dispute detail drawer (right side, 640px):
- Dispute info: Stripe dispute ID, amount, reason, evidence deadline countdown
- Booking context: court name, date, customer, payment details
- Evidence section: list of auto-attached evidence (receipt, booking confirmation, customer communication)
- Admin notes section: textarea for internal notes, "Add Note" button, chronological list of previous notes with admin name and timestamp
- Actions: "Submit Evidence to Stripe" primary button (links to Stripe dashboard), "Mark as Resolved" button

FRAME 2 — SUPPORT METRICS DASHBOARD:
Header: "Support Metrics" with date range picker
Stat cards row:
- Total Tickets: "47" with trend arrow
- Avg Response Time: "2.4 hours"
- Avg Resolution Time: "18 hours"
- Resolution Rate: "89%"
Charts:
- Bar chart: "Tickets by Category" (Booking, Payment, Court, Account, Technical, Payout Issues)
- Line chart: "Tickets Over Time" (daily count for selected period)
- Pie chart: "Status Distribution" (Open, In Progress, Waiting on User, Resolved, Closed)
Table: "Agent Performance" — columns: Agent Name, Assigned Tickets, Avg Response Time, Resolved Count, Satisfaction Score
```

---

### Coverage Checklist — Prompts vs Phase 6 Requirements

| Phase 6 Requirement | Prompt(s) |
|---------------------|-----------|
| Req 1: Dashboard Enhancement | Prompt 2 |
| Req 2: Revenue Analytics | Prompt 5 |
| Req 3: Occupancy Heatmap | Prompt 5 |
| Req 4: CSV/PDF Export | Prompt 5 |
| Req 5: Audit Log Query | Prompt 12 |
| Req 6: Support Tickets | Prompt 6 |
| Req 7: Support Metrics | Prompt 14 |
| Req 8: Feature Flags | Prompt 8 |
| Req 9: User Management | Prompt 8 |
| Req 10: Dispute Escalation | Prompt 14 |
| Req 11: Platform Analytics | Prompt 8 |
| Req 12: Global Search | Prompt 8 |
| Req 13: Bulk Visibility | Prompt 3 |
| Req 14: Admin UI Security | Prompt 13 (session timeout) |
| Req 15: Core App (scaffolding) | Prompt 1 |
| Req 16: Platform Admin Pages | Prompt 8 |
| Req 17: Analytics Events | Backend only (no UI) |
| Req 18: Reminder Evaluation | Prompt 2 (needs attention widget) |
| Req 19: Settings Integration | Prompt 7 |
| Req 20: PBT Correctness | Backend only (no UI) |
| Req 21: GDPR Data Export | Prompt 7 |
| Req 22: Active Sessions | Prompt 7 |
| Req 23: Profile Photo | Prompt 7 |
| Req 24: Project Scaffolding | Prompt 1 |
| Req 25: Court Management Pages | Prompts 3, 10 |
| Req 26: Booking Management Pages | Prompts 4, 11 |
| Req 27: Stripe + Verification Flow | Prompt 9 |
| Req 28: Real-Time Notifications | Prompt 13 (notification dropdown) |
| Req 29: Error Handling + UX | Prompt 13 |
| Req 30: Multi-Language (i18n) | Prompt 1 (language switcher) |

All 30 Phase 6 requirements now have corresponding Figma prompts.


---

### Final Review — Detail Fixes and Missing Elements

After a final cross-check against all 30 Phase 6 requirements, the Court Owner Admin Journey, and the Admin UI principles, the following details should be added to the existing prompts when refining in Figma. These are not new prompts — they are additions to include in the relevant prompt's output.

---

**Prompt 2 additions (Dashboard):**
- Add a "Payout" card showing: next payout date, next payout amount, and a "View Payouts" link (from `GET /api/payments/stripe-connect/payouts`). This is in the Req 1 AC 4 response but missing from the prompt.
- The Stripe Connect banner should show different states: NOT_STARTED (setup button), PENDING (checking status), RESTRICTED (action required, red), ACTIVE (hidden).
- Add a "Verification Pending" amber banner for unverified court owners: "Your business verification is under review. Courts are hidden until approved."

**Prompt 3 additions (Court Management):**
- The "Add Court" drawer should show a "Use Defaults" toggle at the top that pre-fills duration, capacity, confirmation mode, and cancellation tiers from `GET /api/settings/court-defaults`.
- Court images in the table should show a count badge (e.g., "5 photos") if no thumbnail is available.
- The edit form should show a version conflict warning: "This court was modified by another session. Review changes before saving." (optimistic locking from Req 2 AC 1 — 409 Conflict response).

**Prompt 4 additions (Booking Management):**
- The booking calendar should show a "No-Show" flag option when right-clicking a completed booking (from `POST /api/bookings/{id}/no-show`).
- The pending bookings queue should show the court's confirmation timeout setting: "Auto-cancels in 18h 32m (24h timeout)".
- Add a "Recurring" badge on booking blocks that are part of a recurring group, with a "View Group" link.
- The manual booking form should show a "Mark as Paid Externally" checkbox (from `POST /api/bookings/{id}/paid-externally`).

**Prompt 5 additions (Analytics):**
- The export dialog should show a rate limit indicator: "8 of 10 exports used today" (from Req 4 AC 3).
- Add a "No-Shows" stat card in the revenue tab (from Req 2 AC 8).
- The comparison period should be shown as a dashed line overlay on the revenue chart with a legend.

**Prompt 6 additions (Support Tickets):**
- The ticket creation form should show auto-attached context: "Related Booking: #BK-2026-0412" and "Related Court: Padel Arena" when creating from a booking or court context.
- Add a "PAYOUT_ISSUES" category option in the category dropdown (from the Flyway migration adding this category).
- The ticket detail should show a "Waiting on User" status option with a visual indicator.

**Prompt 7 additions (Settings):**
- The reminder rule "Add Rule" form should show: Rule Type dropdown (Unpaid Booking, Payment Held Not Captured, Pending Confirmation, No Contact Manual, Low Occupancy), Days Before threshold, Hours Before to send, Court selector (All Courts or specific), Channel toggles (push/email/in-app), Active toggle. Max 10 rules enforced with a counter.
- The GDPR Data Export section should show: "Request Data Export" button, status of pending exports ("Export requested Apr 15, processing..."), download link when ready, rate limit "2 of 10 exports used today".
- The profile photo upload should show: current photo circle (or initials fallback), "Change Photo" link, accepted formats note "JPEG, PNG, WebP, max 2MB", crop/preview before upload.

**Prompt 8 additions (Platform Admin):**
- The user detail drawer should show: linked OAuth providers (Google/Facebook/Apple icons), active session count, Stripe Connect status (for court owners), subscription status, "Created by" info for SUPPORT_AGENT accounts.
- The admin user management should include a "Create Support Agent" button (since SUPPORT_AGENT is not self-registerable — created by PLATFORM_ADMIN).
- Platform-wide analytics (Req 11) should show: total platform revenue, total bookings across all courts, active court owners count, active customers count, top 10 courts by revenue.

**Prompt 10 additions (Court Detail Sub-Pages):**
- The availability windows editor should show existing bookings as non-editable blocks overlaid on the schedule, so the court owner can see conflicts before changing windows.
- The holiday bulk application should show a conflict preview: "Applying Christmas to 4 courts. 2 courts have bookings on Dec 25 — choose: Skip / Cancel Bookings / Abort" (from Phase 3 Req 10 AC 3).

**Prompt 13 additions (States and Modals):**
- Add a "Rate Limit" toast notification: "Too many requests. Please wait 30 seconds." with a countdown timer (from Req 29 AC 5).
- Add a "Concurrent Edit" conflict modal: "This court was modified by another session. Your version: 3, Current version: 4. Review the changes and try again." with a "Reload" button (from admin UI error handling principles).
- Add a "Data Export Processing" state: modal with progress bar "Generating your data export... This may take a few minutes." with a "We'll notify you when it's ready" message.

---

### Court Owner Journey Step Coverage Verification

Cross-checking the Court Owner Admin Journey (Steps 1-9) against prompts:

| Journey Step | Prompt(s) | Status |
|---|---|---|
| Step 1: Registration & Verification | Prompt 9 | Covered |
| Step 2: Stripe Connect Onboarding | Prompt 9 | Covered |
| Step 3: Trial & Subscription | N/A (Phase 10 — stubbed as ACTIVE) | Correctly deferred |
| Step 4: Dashboard | Prompt 2 | Covered (with additions above) |
| Step 5: Court Management (Add/Edit/Remove/Availability/Holidays/Bulk) | Prompts 3, 10 | Covered |
| Step 6: Booking Management (Calendar/Pending/Manual/Details/Reminders) | Prompts 4, 11 | Covered (with additions above) |
| Step 7: Pricing & Policies (Dynamic Pricing/Cancellation/Promo Codes) | Prompt 10 | Covered (promo codes deferred to Phase 10) |
| Step 8: Analytics & Revenue (Revenue/Usage/Export) | Prompt 5 | Covered (scheduled reports deferred to Phase 10) |
| Step 9: Personal Settings (Profile/Notifications/Reminders/Defaults/Security) | Prompt 7 | Covered (with additions above) |

All journey steps are covered. The only deferred items are Trial/Subscription (Phase 10), Promo Codes (Phase 10), and Scheduled Reports (Phase 10) — all correctly out of scope.

---

### Final Verdict

14 prompts total. All 30 Phase 6 requirements covered. All 9 Court Owner Journey steps covered. All Admin UI error handling patterns covered. The additions above fill in the remaining detail gaps found during this final review. You're ready to start generating in Figma.
