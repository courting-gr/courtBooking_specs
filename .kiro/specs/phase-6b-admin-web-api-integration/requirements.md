# Requirements Document — Admin Web API Integration

## Introduction

This feature replaces all hardcoded mock data in the `court-booking-admin-web` React application with real API calls to the Platform Service and Transaction Service backends. The admin portal was scaffolded in Phase 6 with proper infrastructure (Axios instances with JWT/CSRF interceptors, TanStack Query key factories, Zustand auth store, WebSocket client), but every page component currently renders static `mockXxx` constants. The `src/services/index.ts` barrel is empty, no TanStack Query hooks exist, and form submissions log to console.

This integration phase creates the services layer, TanStack Query hooks, and rewires all page components to use real data with proper loading states, error handling, empty states, cache invalidation, optimistic updates, and real-time WebSocket cache synchronization.

## Glossary

- **Admin_Web_Portal**: The React SPA (`court-booking-admin-web/`) using TypeScript, Ant Design 5, React Router v6, TanStack Query v5, and Zustand.
- **Services_Layer**: TypeScript modules in `src/services/` that encapsulate Axios calls to backend endpoints using the `platformApi` and `transactionApi` instances from `src/lib/api.ts`.
- **Query_Hook**: A custom React hook wrapping TanStack Query's `useQuery` or `useMutation` with the query key factories from `src/lib/queryKeys.ts`.
- **Platform_Service**: The backend at `VITE_PLATFORM_API_URL` serving courts, dashboard, analytics, support, admin, settings, and auth endpoints.
- **Transaction_Service**: The backend at `VITE_TRANSACTION_API_URL` serving bookings, payments, and Stripe Connect endpoints.
- **Loading_Skeleton**: An Ant Design `Skeleton` component displayed while a query is in `isLoading` state, matching the layout shape of the target content.
- **Error_State**: A UI component displayed when a query enters `isError` state, showing the error message and a retry button.
- **Empty_State**: An Ant Design `Empty` component displayed when a query succeeds but returns zero results.
- **Optimistic_Update**: A TanStack Query mutation pattern that immediately updates the cache before the server responds, rolling back on error.
- **Cache_Invalidation**: Calling `queryClient.invalidateQueries()` with the appropriate query key after a mutation succeeds, triggering a background refetch.
- **WebSocket_Cache_Sync**: Subscribing to STOMP topics via the WebSocket client and updating TanStack Query caches when real-time events arrive.
- **Mock_Data**: Hardcoded `const mockXxx` arrays and objects currently used as `dataSource` in Ant Design tables and as values in Statistic components.

## Requirements

### Requirement 1: Services Layer — Platform Service API Functions

**User Story:** As a developer, I want a typed services layer that encapsulates all Platform Service API calls, so that page components and hooks have a clean interface to the backend without directly importing Axios instances.

#### Acceptance Criteria

1. THE Services_Layer SHALL export service modules organized by feature domain: `dashboardService`, `courtService`, `analyticsService`, `supportService`, `adminService`, `settingsService`, `authService`, and `searchService`
2. WHEN a service function is called, THE Services_Layer SHALL use the `platformApi` Axios instance from `src/lib/api.ts` for all Platform Service endpoints
3. THE Services_Layer SHALL define TypeScript request and response types for each API function that match the backend response models documented in the Phase 6 design document
4. WHEN a service function receives an error response, THE Services_Layer SHALL propagate the normalized `ApiError` object from the Axios interceptor without additional wrapping
5. THE `dashboardService` SHALL expose `getDashboard()` calling `GET /api/dashboard` and returning the enhanced dashboard summary including `todayBookings`, `pendingConfirmations`, `revenueThisMonthCents`, `nextPayoutDate`, `occupancyRateToday`, `actionRequired`, `recentNotifications`, `courtSummary`, `verificationStatus`, and `stripeConnectStatus`
6. THE `courtService` SHALL expose: `getOwnerCourts(params)` calling `GET /api/courts/owner/me`, `getCourt(id)` calling `GET /api/courts/{id}`, `createCourt(data)` calling `POST /api/courts`, `updateCourt(id, data)` calling `PUT /api/courts/{id}`, `deleteCourt(id)` calling `DELETE /api/courts/{id}`, `uploadCourtImages(id, files)` calling `POST /api/courts/{id}/images`, `bulkToggleVisibility(courtIds, visible)` calling `POST /api/courts/bulk/visibility`, `getAvailabilityWindows(courtId)` calling `GET /api/courts/{courtId}/availability/windows`, `updateAvailabilityWindows(courtId, data)` calling `PUT /api/courts/{courtId}/availability/windows`, `getAvailabilityOverrides(courtId)` calling `GET /api/courts/{courtId}/availability/overrides`, `createAvailabilityOverride(courtId, data)` calling `POST /api/courts/{courtId}/availability/overrides`, `deleteAvailabilityOverride(courtId, overrideId)` calling `DELETE /api/courts/{courtId}/availability/overrides/{overrideId}`, `getPricingRules(courtId)` calling `GET /api/courts/{courtId}/pricing-rules`, `updatePricingRules(courtId, data)` calling `PUT /api/courts/{courtId}/pricing-rules`, `getCancellationTiers(courtId)` calling `GET /api/courts/{courtId}/cancellation-tiers`, `updateCancellationTiers(courtId, data)` calling `PUT /api/courts/{courtId}/cancellation-tiers`
7. THE `analyticsService` SHALL expose: `getRevenue(params)` calling `GET /api/analytics/revenue`, `getUsage(params)` calling `GET /api/analytics/usage`, `getHeatmap(params)` calling `GET /api/analytics/heatmap`, `exportAnalytics(params)` calling `GET /api/analytics/export` with `responseType: 'blob'`
8. THE `supportService` SHALL expose: `getTickets(params)` calling `GET /api/support/tickets`, `getTicket(id)` calling `GET /api/support/tickets/{id}`, `createTicket(data)` calling `POST /api/support/tickets`, `addMessage(ticketId, data)` calling `POST /api/support/tickets/{ticketId}/messages`, `updateTicketStatus(ticketId, data)` calling `PUT /api/support/tickets/{ticketId}/status`, `assignTicket(ticketId, data)` calling `PUT /api/support/tickets/{ticketId}/assign`
9. THE `settingsService` SHALL expose: `updateProfile(data)` calling `PUT /api/users/me`, `getNotificationPreferences()` and `updateNotificationPreferences(data)` calling `GET/PUT /api/settings/notification-preferences`, `getReminderRules()`, `createReminderRule(data)`, `updateReminderRule(id, data)`, `deleteReminderRule(id)` calling the `/api/settings/reminder-rules` endpoints, `getCourtDefaults()` and `updateCourtDefaults(data)` calling `GET/PUT /api/settings/court-defaults`, `getSessions()` and `revokeSession(id)` calling `GET/DELETE /api/users/me/sessions`, `requestDataExport()` calling `POST /api/users/me/data-export`, `uploadProfilePhoto(file)` and `deleteProfilePhoto()` calling `POST/DELETE /api/users/me/profile-photo`
10. THE `adminService` SHALL expose: `getUsers(params)` calling `GET /api/admin/users`, `getUserDetail(id)` calling `GET /api/admin/users/{id}`, `suspendUser(id, data)` calling `POST /api/admin/users/{id}/suspend`, `unsuspendUser(id)` calling `POST /api/admin/users/{id}/unsuspend`, `getVerificationQueue(params)` calling `GET /api/verification/queue`, `approveVerification(id)` calling `POST /api/verification/{id}/approve`, `rejectVerification(id, data)` calling `POST /api/verification/{id}/reject`, `getFeatureFlags()` calling `GET /api/admin/feature-flags`, `createFeatureFlag(data)` calling `POST /api/admin/feature-flags`, `updateFeatureFlag(key, data)` calling `PUT /api/admin/feature-flags/{key}`, `deleteFeatureFlag(key)` calling `DELETE /api/admin/feature-flags/{key}`, `getDisputes(params)` calling `GET /api/admin/disputes`, `getDisputeDetail(id)` calling `GET /api/admin/disputes/{id}`, `addDisputeNote(id, data)` calling `POST /api/admin/disputes/{id}/notes`, `getPlatformAnalytics(params)` calling `GET /api/admin/analytics`, `getSupportMetrics(params)` calling `GET /api/admin/support/metrics`
11. THE `searchService` SHALL expose `globalSearch(params)` calling `GET /api/search` and `adminSearch(params)` calling `GET /api/admin/search`
12. THE `authService` SHALL expose `fetchCsrfToken()` calling `GET /api/auth/csrf-token`
13. THE `src/services/index.ts` barrel file SHALL re-export all service modules

### Requirement 2: Services Layer — Transaction Service API Functions

**User Story:** As a developer, I want typed service functions for all Transaction Service endpoints, so that booking and payment pages can call the real backend.

#### Acceptance Criteria

1. WHEN a service function targets a Transaction Service endpoint, THE Services_Layer SHALL use the `transactionApi` Axios instance from `src/lib/api.ts`
2. THE `bookingService` SHALL expose: `getBookings(params)` calling `GET /api/bookings`, `getBooking(id)` calling `GET /api/bookings/{id}`, `getPendingBookings(params)` calling `GET /api/bookings/pending`, `confirmBooking(id)` calling `POST /api/bookings/{id}/confirm`, `rejectBooking(id, data)` calling `POST /api/bookings/{id}/reject`, `cancelBooking(id, data)` calling `POST /api/bookings/{id}/cancel`, `markPaid(id)` calling `POST /api/bookings/{id}/mark-paid`, `flagNoShow(id)` calling `POST /api/bookings/{id}/no-show`, `modifyBooking(id, data)` calling `PUT /api/bookings/{id}`, `bulkAction(data)` calling `POST /api/bookings/bulk-action`, `createManualBooking(data)` calling `POST /api/bookings/manual`, `createRecurringBooking(data)` calling `POST /api/bookings/recurring`, `exportCalendar(params)` calling `GET /api/bookings/calendar/export`
3. THE `stripeService` SHALL expose: `startOnboarding()` calling `POST /api/payments/stripe-connect/onboard`, `getPayouts(params)` calling `GET /api/payments/stripe-connect/payouts`
4. THE `verificationService` SHALL expose: `submitVerification(data)` calling `POST /api/verification`, `getVerificationStatus()` calling `GET /api/verification`
5. THE `notificationService` SHALL expose: `getNotifications(params)` calling `GET /api/notifications`, `getUnreadCount()` calling `GET /api/notifications?unreadOnly=true` and returning the count, `markAsRead(notificationId)` calling `POST /api/notifications/{notificationId}/read`, `markAllAsRead()` calling `POST /api/notifications/read-all`, `registerDevice(data)` calling `POST /api/notifications/device`


### Requirement 3: TanStack Query Hooks — Queries

**User Story:** As a developer, I want custom TanStack Query hooks for every data-fetching operation, so that page components get automatic caching, background refetching, and consistent loading/error state management.

#### Acceptance Criteria

1. THE Admin_Web_Portal SHALL create custom query hooks in `src/hooks/` organized by feature domain (e.g., `useDashboard.ts`, `useCourts.ts`, `useBookings.ts`, `useAnalytics.ts`, `useSupport.ts`, `useSettings.ts`, `useAdmin.ts`)
2. WHEN a query hook is created, THE hook SHALL use the query key factories from `src/lib/queryKeys.ts` to ensure consistent cache key management
3. THE `useDashboard` hook SHALL call `dashboardService.getDashboard()` with key `dashboardKeys.summary()` and a `staleTime` of 60 seconds matching the backend Redis cache TTL
4. THE `useCourts` hook SHALL expose `useOwnerCourts(filters)` calling `courtService.getOwnerCourts(filters)` with key `courtKeys.list(filters)`, and `useCourtDetail(id)` calling `courtService.getCourt(id)` with key `courtKeys.detail(id)`
5. THE `useBookings` hook SHALL expose `useBookingList(filters)` calling `bookingService.getBookings(filters)` with key `bookingKeys.list(filters)`, `usePendingBookings()` calling `bookingService.getPendingBookings()` with key `bookingKeys.pending()`, and `useBookingDetail(id)` calling `bookingService.getBooking(id)` with key `bookingKeys.detail(id)`
6. THE `useAnalytics` hook SHALL expose `useRevenue(params)` with key `analyticsKeys.revenue(params)`, `useUsage(params)` with key `analyticsKeys.usage(params)`, and `useHeatmap(params)` with key `analyticsKeys.heatmap(params)`
7. THE `useSupport` hook SHALL expose `useTicketList(filters)` with key `supportKeys.list(filters)` and `useTicketDetail(id)` with key `supportKeys.detail(id)`
8. THE `useSettings` hook SHALL expose `useNotificationPreferences()` with key `settingsKeys.notificationPrefs()`, `useReminderRules()` with key `settingsKeys.reminderRules()`, `useCourtDefaults()` with key `settingsKeys.courtDefaults()`, and `useSessions()` with key `settingsKeys.sessions()`
9. THE `useAdmin` hook SHALL expose `useUserList(params)`, `useUserDetail(id)`, `useFeatureFlags()`, `useDisputes(params)`, `useDisputeDetail(id)`, `usePlatformAnalytics(params)`, and `useSupportMetrics(params)` using the corresponding `adminKeys` factories
10. WHEN a query hook receives filter parameters, THE hook SHALL pass those parameters into the query key factory so that different filter combinations produce distinct cache entries
11. WHEN a query hook's `enabled` option is relevant (e.g., `useCourtDetail(id)` when `id` is undefined), THE hook SHALL set `enabled: !!id` to prevent unnecessary API calls

### Requirement 4: TanStack Query Hooks — Mutations with Cache Invalidation

**User Story:** As a developer, I want mutation hooks for all write operations that automatically invalidate relevant caches on success, so that the UI stays in sync with the server after user actions.

#### Acceptance Criteria

1. THE Admin_Web_Portal SHALL create mutation hooks alongside query hooks in the same feature hook files
2. WHEN a court is created or updated via `useCreateCourt` or `useUpdateCourt`, THE mutation's `onSuccess` callback SHALL invalidate `courtKeys.lists()` to refresh the court list, and for updates SHALL also invalidate `courtKeys.detail(id)` and `dashboardKeys.summary()`
3. WHEN a court is deleted via `useDeleteCourt`, THE mutation's `onSuccess` callback SHALL invalidate `courtKeys.lists()` and `dashboardKeys.summary()`
4. WHEN a booking action is performed (confirm, reject, cancel, mark-paid, no-show) via the corresponding mutation hook, THE mutation's `onSuccess` callback SHALL invalidate `bookingKeys.detail(id)`, `bookingKeys.lists()`, `bookingKeys.pending()`, and `dashboardKeys.summary()`
5. WHEN a bulk booking action is performed via `useBulkBookingAction`, THE mutation's `onSuccess` callback SHALL invalidate `bookingKeys.lists()`, `bookingKeys.pending()`, and `dashboardKeys.summary()`
6. WHEN a manual or recurring booking is created, THE mutation's `onSuccess` callback SHALL invalidate `bookingKeys.lists()` and `dashboardKeys.summary()`
7. WHEN a support ticket is created or its status is updated, THE mutation's `onSuccess` callback SHALL invalidate `supportKeys.lists()` and `supportKeys.detail(id)`
8. WHEN a support message is added, THE mutation's `onSuccess` callback SHALL invalidate `supportKeys.detail(ticketId)` to refresh the message thread
9. WHEN settings are updated (profile, notification preferences, reminder rules, court defaults), THE mutation's `onSuccess` callback SHALL invalidate the corresponding `settingsKeys` entry
10. WHEN a feature flag is created, updated, or deleted, THE mutation's `onSuccess` callback SHALL invalidate `adminKeys.featureFlags()`
11. WHEN a user is suspended or unsuspended, THE mutation's `onSuccess` callback SHALL invalidate `adminKeys.users()` and `adminKeys.userDetail(id)`
12. WHEN bulk court visibility is toggled via `useBulkToggleVisibility`, THE mutation's `onSuccess` callback SHALL invalidate `courtKeys.lists()` and `dashboardKeys.summary()`
13. WHEN a mutation hook encounters an error, THE hook SHALL NOT invalidate any caches and SHALL propagate the error to the calling component for display

### Requirement 5: Dashboard Page — Real API Integration

**User Story:** As a court owner, I want the dashboard page to display real data from the backend instead of hardcoded mock values, so that I see accurate business metrics and actionable alerts.

#### Acceptance Criteria

1. WHEN the DashboardPage component mounts, THE Admin_Web_Portal SHALL call the `useDashboard` query hook to fetch data from `GET /api/dashboard`, replacing the `mockStats`, `mockPayout`, `mockNeedsAttention`, `mockNotifications`, and `mockCourtSummary` constants
2. WHILE the dashboard query is loading, THE DashboardPage SHALL display Ant Design Skeleton components matching the layout of the stat cards, payout card, needs attention list, notifications list, and court summary card
3. IF the dashboard query fails, THEN THE DashboardPage SHALL display an error alert with the error message and a "Retry" button that calls `refetch()`
4. WHEN the dashboard data is received, THE DashboardPage SHALL map `todayBookings`, `pendingConfirmations`, `revenueThisMonthCents`, and `occupancyRateToday` to the Statistic components, replacing the `mockStats` object
5. WHEN the dashboard data includes `nextPayoutDate` and `nextPayoutAmountCents`, THE DashboardPage SHALL display the payout card with real values. WHEN `nextPayoutDate` is null, THE DashboardPage SHALL display "No upcoming payout"
6. WHEN the dashboard data includes `actionRequired` counts, THE DashboardPage SHALL display the "Needs Attention" widget with real alert items grouped by trigger type (unpaid bookings, pending confirmations, reminder alerts)
7. WHEN the dashboard data includes `recentNotifications`, THE DashboardPage SHALL render the notifications list with real notification data, preserving the unread badge styling
8. WHEN the dashboard data includes `courtSummary`, THE DashboardPage SHALL display `totalCourts`, `visibleCourts`, and `hiddenCourts` in the court summary card
9. WHEN the dashboard data includes `verificationStatus` and `stripeConnectStatus`, THE DashboardPage SHALL conditionally render the Stripe onboarding banner and verification pending banner based on real status values, replacing the `mockUserState` constant
10. THE DashboardPage SHALL set the `useDashboard` hook's `refetchInterval` to 60000 milliseconds (1 minute) to keep dashboard data fresh

### Requirement 6: Courts Pages — Real API Integration

**User Story:** As a court owner, I want the courts list, detail, and form pages to use real API data, so that I can manage my actual courts instead of seeing placeholder data.

#### Acceptance Criteria

1. WHEN the CourtListPage mounts, THE Admin_Web_Portal SHALL call `useOwnerCourts(filters)` to fetch courts from `GET /api/courts/owner/me`, replacing the `mockCourts` array and the `useState(mockCourts)` pattern
2. WHILE the court list query is loading, THE CourtListPage SHALL display a Skeleton table with the same column structure
3. WHEN the court list is empty, THE CourtListPage SHALL display the existing Empty component with the "Add Court" call-to-action
4. WHEN a court owner toggles visibility via the Switch component, THE CourtListPage SHALL call the `useUpdateCourt` mutation (or a dedicated visibility mutation) instead of the local `setCourts` state update
5. WHEN a court owner deletes a court via the Popconfirm, THE CourtListPage SHALL call the `useDeleteCourt` mutation instead of the local `setCourts` filter
6. WHEN the court owner selects multiple courts and clicks "Toggle Visibility", THE CourtListPage SHALL call `useBulkToggleVisibility` with the selected court IDs
7. WHEN the CourtFormPage is in create mode, THE form submission SHALL call `useCreateCourt` and navigate to the court list on success
8. WHEN the CourtFormPage is in edit mode, THE component SHALL call `useCourtDetail(id)` to populate the form, and the form submission SHALL call `useUpdateCourt`
9. WHEN the CourtFormPage includes image uploads, THE form SHALL call `courtService.uploadCourtImages(id, files)` after court creation or update
10. WHEN the CourtDetailPage mounts, THE Admin_Web_Portal SHALL call `useCourtDetail(id)` to fetch court data from `GET /api/courts/{id}`, including availability windows, pricing rules, and cancellation tiers from the sub-API endpoints
11. IF a court detail query returns 404, THEN THE CourtDetailPage SHALL display a "Court not found" message with a link back to the court list
12. WHEN the court list filter bar values change (type, location, search), THE CourtListPage SHALL pass the updated filters to `useOwnerCourts(filters)` to trigger a server-side filtered query

### Requirement 7: Bookings Pages — Real API Integration

**User Story:** As a court owner, I want the bookings list, calendar, pending queue, manual booking form, and booking detail pages to use real API data, so that I can manage actual bookings.

#### Acceptance Criteria

1. WHEN the BookingListPage mounts, THE Admin_Web_Portal SHALL call `useBookingList(filters)` to fetch bookings from `GET /api/bookings` (Transaction Service), replacing the `mockBookings` array
2. WHILE the booking list query is loading, THE BookingListPage SHALL display a Skeleton table
3. WHEN the booking list filter bar values change (date range, court, status, payment status, search), THE BookingListPage SHALL pass updated filters to `useBookingList(filters)` for server-side filtering and pagination
4. WHEN the BookingCalendarPage mounts, THE Admin_Web_Portal SHALL call `useBookingList` with calendar-specific date range parameters to populate the calendar view
5. WHEN the PendingBookingsPage mounts, THE Admin_Web_Portal SHALL call `usePendingBookings()` to fetch pending bookings from `GET /api/bookings/pending`
6. WHEN a court owner confirms a pending booking, THE PendingBookingsPage SHALL call the `useConfirmBooking` mutation, which invalidates `bookingKeys.pending()`, `bookingKeys.lists()`, and `dashboardKeys.summary()`
7. WHEN a court owner rejects a pending booking, THE PendingBookingsPage SHALL call the `useRejectBooking` mutation with a rejection reason
8. WHEN a court owner performs a bulk action on pending bookings, THE PendingBookingsPage SHALL call `useBulkBookingAction` with the selected booking IDs and action type
9. WHEN the ManualBookingPage form is submitted, THE Admin_Web_Portal SHALL call `useCreateManualBooking` for single bookings or `useCreateRecurringBooking` for recurring bookings, replacing the console.log pattern
10. WHEN the BookingDetailPage mounts, THE Admin_Web_Portal SHALL call `useBookingDetail(id)` to fetch booking data from `GET /api/bookings/{id}`
11. WHEN a court owner performs an action on a booking detail (cancel, mark-paid, no-show, modify), THE BookingDetailPage SHALL call the corresponding mutation hook
12. IF a booking detail query returns 404, THEN THE BookingDetailPage SHALL display a "Booking not found" message with a link back to the booking list

### Requirement 8: Analytics Pages — Real API Integration

**User Story:** As a court owner, I want the analytics pages to display real revenue reports, usage trends, and occupancy heatmaps from the backend, so that I can make data-driven decisions.

#### Acceptance Criteria

1. WHEN the RevenueAnalyticsPage mounts, THE Admin_Web_Portal SHALL call `useRevenue(params)` to fetch data from `GET /api/analytics/revenue`, replacing any mock analytics data
2. WHILE the revenue query is loading, THE RevenueAnalyticsPage SHALL display Skeleton components matching the chart and summary card layout
3. WHEN the date range or court filter changes, THE RevenueAnalyticsPage SHALL pass updated parameters to `useRevenue(params)` to refetch with the new filters
4. WHEN the court owner clicks "Export CSV" or "Export PDF", THE RevenueAnalyticsPage SHALL call `analyticsService.exportAnalytics(params)` with the appropriate format parameter and trigger a browser file download from the blob response
5. IF the export returns a 429 status, THEN THE RevenueAnalyticsPage SHALL display the rate limit message with the `retryAfterSeconds` countdown
6. WHEN the UsageAnalyticsPage mounts, THE Admin_Web_Portal SHALL call `useUsage(params)` to fetch data from `GET /api/analytics/usage`
7. WHEN the HeatmapPage mounts, THE Admin_Web_Portal SHALL call `useHeatmap(params)` to fetch data from `GET /api/analytics/heatmap` and render the day-of-week × hour-of-day matrix
8. IF an analytics query returns a 400 error for invalid date range, THEN THE analytics page SHALL display the validation error message from the response


### Requirement 9: Support Pages — Real API Integration

**User Story:** As a user, I want the support ticket list and detail pages to use real API data, so that I can create, view, and manage actual support tickets.

#### Acceptance Criteria

1. WHEN the SupportTicketListPage mounts, THE Admin_Web_Portal SHALL call `useTicketList(filters)` to fetch tickets from `GET /api/support/tickets`, replacing any mock ticket data
2. WHILE the ticket list query is loading, THE SupportTicketListPage SHALL display a Skeleton table
3. WHEN the ticket list filter bar values change (status, sort), THE SupportTicketListPage SHALL pass updated filters to `useTicketList(filters)` for server-side filtering
4. WHEN the SupportTicketDetailPage mounts, THE Admin_Web_Portal SHALL call `useTicketDetail(id)` to fetch the ticket with its full message thread from `GET /api/support/tickets/{id}`
5. WHEN a user submits a new message in the ticket detail view, THE SupportTicketDetailPage SHALL call the `useAddMessage` mutation, which invalidates `supportKeys.detail(ticketId)` to refresh the thread
6. WHEN a PLATFORM_ADMIN or SUPPORT_AGENT updates ticket status, THE SupportTicketDetailPage SHALL call the `useUpdateTicketStatus` mutation
7. WHEN a PLATFORM_ADMIN assigns a ticket, THE SupportTicketDetailPage SHALL call the `useAssignTicket` mutation
8. IF a ticket detail query returns 404, THEN THE SupportTicketDetailPage SHALL display a "Ticket not found" message
9. WHEN a user creates a new support ticket via a creation form, THE Admin_Web_Portal SHALL call the `useCreateTicket` mutation and navigate to the new ticket's detail page on success

### Requirement 10: Settings Page — Real API Integration

**User Story:** As a court owner, I want the settings page tabs to load and save real data, so that my profile, notification preferences, reminder rules, court defaults, and security settings are persisted.

#### Acceptance Criteria

1. WHEN the Settings Profile tab loads, THE Admin_Web_Portal SHALL populate the form with the current user data from the Zustand auth store and call `useUpdateProfile` mutation on form submission via `PUT /api/users/me`
2. WHEN the Settings Notification Preferences tab loads, THE Admin_Web_Portal SHALL call `useNotificationPreferences()` to fetch current preferences and call `useUpdateNotificationPreferences` mutation on toggle changes
3. WHEN the Settings Reminder Rules tab loads, THE Admin_Web_Portal SHALL call `useReminderRules()` to fetch the list and provide `useCreateReminderRule`, `useUpdateReminderRule`, and `useDeleteReminderRule` mutations for CRUD operations
4. WHEN the Settings Court Defaults tab loads, THE Admin_Web_Portal SHALL call `useCourtDefaults()` to fetch current defaults and call `useUpdateCourtDefaults` mutation on form submission
5. WHEN the Settings Security tab loads, THE Admin_Web_Portal SHALL call `useSessions()` to fetch active sessions and provide a `useRevokeSession` mutation for the "Revoke" button
6. WHEN the court owner clicks "Request Data Export", THE Admin_Web_Portal SHALL call `settingsService.requestDataExport()` and display a success toast with the expected delivery information
7. WHEN the court owner uploads or deletes a profile photo, THE Admin_Web_Portal SHALL call `settingsService.uploadProfilePhoto(file)` or `settingsService.deleteProfilePhoto()` and update the auth store user data on success
8. WHILE any settings tab query is loading, THE Admin_Web_Portal SHALL display Skeleton components matching the form layout
9. IF a settings mutation fails, THEN THE Admin_Web_Portal SHALL display an error toast with the error message and preserve the form state for retry

### Requirement 11: Admin Pages — Real API Integration

**User Story:** As a platform admin, I want the admin-only pages (user management, verification queue, feature flags, platform analytics, support metrics, disputes) to use real API data, so that I can perform actual platform administration.

#### Acceptance Criteria

1. WHEN the UserManagementPage mounts, THE Admin_Web_Portal SHALL call `useUserList(params)` to fetch users from `GET /api/admin/users`, replacing any mock user data
2. WHEN a PLATFORM_ADMIN suspends a user, THE UserManagementPage SHALL call the `useSuspendUser` mutation with the reason, and display a success toast on completion
3. WHEN a PLATFORM_ADMIN unsuspends a user, THE UserManagementPage SHALL call the `useUnsuspendUser` mutation
4. WHEN the VerificationQueuePage mounts, THE Admin_Web_Portal SHALL call a query hook fetching from `GET /api/verification/queue`
5. WHEN a PLATFORM_ADMIN approves or rejects a verification, THE VerificationQueuePage SHALL call the corresponding mutation hook and invalidate the verification queue cache
6. WHEN the FeatureFlagsPage mounts, THE Admin_Web_Portal SHALL call `useFeatureFlags()` to fetch flags from `GET /api/admin/feature-flags`
7. WHEN a PLATFORM_ADMIN toggles a feature flag, THE FeatureFlagsPage SHALL call the `useUpdateFeatureFlag` mutation with optimistic update — immediately toggling the switch in the UI and rolling back on error
8. WHEN a PLATFORM_ADMIN creates or deletes a feature flag, THE FeatureFlagsPage SHALL call the corresponding mutation hook
9. WHEN the PlatformAnalyticsPage mounts, THE Admin_Web_Portal SHALL call `usePlatformAnalytics(params)` to fetch data from `GET /api/admin/analytics`
10. WHEN the SupportMetricsPage mounts, THE Admin_Web_Portal SHALL call `useSupportMetrics(params)` to fetch data from `GET /api/admin/support/metrics`
11. WHEN the DisputesPage mounts, THE Admin_Web_Portal SHALL call `useDisputes(params)` to fetch disputes from `GET /api/admin/disputes`
12. WHEN a PLATFORM_ADMIN views a dispute detail, THE DisputesPage SHALL call `useDisputeDetail(id)` and display the full dispute context
13. WHEN a PLATFORM_ADMIN adds a note to a dispute, THE DisputesPage SHALL call the `useAddDisputeNote` mutation

### Requirement 12: Audit Log and Holiday Calendar Pages — Real API Integration

**User Story:** As a court owner, I want the audit log and holiday calendar pages to use real API data, so that I can review my activity history and manage court holidays.

#### Acceptance Criteria

1. WHEN the AuditLogPage mounts, THE Admin_Web_Portal SHALL call a query hook fetching from `GET /api/audit-logs` with pagination and filter parameters
2. WHEN the audit log filter bar values change (date range, action type, court), THE AuditLogPage SHALL pass updated filters to the query hook for server-side filtering
3. WHILE the audit log query is loading, THE AuditLogPage SHALL display a Skeleton table
4. WHEN the HolidayCalendarPage mounts, THE Admin_Web_Portal SHALL call query hooks fetching national holidays from `GET /api/holidays/national` and custom holidays from `GET /api/holidays/custom`
5. WHEN a court owner creates, updates, or deletes a custom holiday, THE HolidayCalendarPage SHALL call the corresponding mutation hook and invalidate the custom holidays cache
6. WHEN a court owner applies holidays to courts, THE HolidayCalendarPage SHALL call a mutation hook for `POST /api/holidays/apply`

### Requirement 13: Auth Pages — Real API Integration

**User Story:** As a user, I want the login, Stripe onboarding, and verification pages to use real API calls, so that authentication and onboarding flows work end-to-end.

#### Acceptance Criteria

1. WHEN the LoginPage initiates OAuth login, THE Admin_Web_Portal SHALL use the existing `useAuthStore.login()` method which already calls the real OAuth endpoint, and SHALL call `fetchCsrfToken()` after successful login
2. WHEN the StripeOnboardingPage mounts, THE Admin_Web_Portal SHALL call `stripeService.startOnboarding()` to initiate the Stripe Connect flow and redirect to the Stripe-hosted onboarding URL
3. WHEN the StripeOnboardingPage displays payout information, THE Admin_Web_Portal SHALL call `stripeService.getPayouts(params)` to fetch real payout data
4. WHEN the VerificationPage mounts, THE Admin_Web_Portal SHALL call `verificationService.getVerificationStatus()` to display the current verification state
5. WHEN the VerificationPage form is submitted, THE Admin_Web_Portal SHALL call `verificationService.submitVerification(data)` with the form data and uploaded documents

### Requirement 14: Global Search — Real API Integration

**User Story:** As a court owner, I want the global search bar to return real search results from the backend, so that I can find actual bookings, courts, and records.

#### Acceptance Criteria

1. WHEN a user types in the global search bar, THE Admin_Web_Portal SHALL debounce the input by 300 milliseconds before calling `searchService.globalSearch(params)` via `GET /api/search`
2. WHEN the search query is fewer than 2 characters, THE Admin_Web_Portal SHALL clear the results and not call the API
3. WHEN search results are returned, THE Admin_Web_Portal SHALL display them grouped by type (Court, Booking, Promo Code) with the `title`, `subtitle`, and type badge
4. WHEN a user clicks a search result, THE Admin_Web_Portal SHALL navigate to the corresponding detail page (court detail, booking detail)
5. WHEN a PLATFORM_ADMIN uses the search, THE Admin_Web_Portal SHALL call `searchService.adminSearch(params)` via `GET /api/admin/search` for platform-wide results

### Requirement 15: Real-Time WebSocket Cache Synchronization

**User Story:** As a court owner, I want the admin portal to receive real-time updates via WebSocket and automatically refresh displayed data, so that I see new bookings, status changes, and notifications without manually refreshing.

#### Acceptance Criteria

1. WHEN the Admin_Web_Portal establishes a WebSocket connection, THE Admin_Web_Portal SHALL subscribe to `/user/queue/notifications` for personal notifications and `/topic/bookings/{courtId}` for each owned court's booking updates
2. WHEN a `BOOKING_CREATED` or `BOOKING_UPDATED` event is received via WebSocket, THE Admin_Web_Portal SHALL invalidate `bookingKeys.lists()`, `bookingKeys.pending()`, and `dashboardKeys.summary()` to trigger background refetches
3. WHEN a `BOOKING_CONFIRMED` or `BOOKING_CANCELLED` event is received, THE Admin_Web_Portal SHALL invalidate `bookingKeys.detail(bookingId)` in addition to the list and dashboard keys
4. WHEN a `NOTIFICATION_RECEIVED` event is received, THE Admin_Web_Portal SHALL invalidate `dashboardKeys.summary()` to refresh the recent notifications list
5. WHEN a `SUPPORT_TICKET_UPDATED` event is received, THE Admin_Web_Portal SHALL invalidate `supportKeys.lists()` and `supportKeys.detail(ticketId)` if the ticket is currently being viewed
6. WHEN the WebSocket connection is lost, THE Admin_Web_Portal SHALL display a subtle connection status indicator and rely on the existing exponential backoff reconnection logic in `src/lib/websocket.ts`
7. WHEN the WebSocket reconnects after a disconnection, THE Admin_Web_Portal SHALL invalidate all active query keys to ensure data consistency after the gap in real-time updates

### Requirement 16: Loading, Error, and Empty State Patterns

**User Story:** As a user, I want consistent visual feedback for loading, error, and empty states across all pages, so that the application feels polished and I always know what is happening.

#### Acceptance Criteria

1. THE Admin_Web_Portal SHALL implement a reusable `QueryStateWrapper` component (or equivalent pattern) that accepts a TanStack Query result and renders: a Skeleton when `isLoading` is true, an error alert with retry button when `isError` is true, an Empty component when data is an empty array or null, and the children when data is available
2. WHEN a page displays a table with loading state, THE Admin_Web_Portal SHALL render an Ant Design Skeleton with `active` animation and row count matching the expected page size
3. WHEN a page displays an error state, THE error alert SHALL include the error message from the API response and a "Retry" button that calls the query's `refetch()` function
4. WHEN a page displays an empty state, THE Empty component SHALL include a contextual message (e.g., "No bookings found" for bookings, "No courts yet" for courts) and a primary action button where applicable (e.g., "Add Court", "Create Booking")
5. WHEN a mutation is in progress, THE Admin_Web_Portal SHALL disable the triggering button and show a loading spinner to prevent double-submission
6. WHEN a mutation succeeds, THE Admin_Web_Portal SHALL display an Ant Design success toast notification with a brief confirmation message
7. WHEN a mutation fails, THE Admin_Web_Portal SHALL display an Ant Design error toast notification with the error message from the API response
8. WHEN the Admin_Web_Portal receives a 429 rate limit response, THE Admin_Web_Portal SHALL display a warning message with the `retryAfterSeconds` value and a countdown timer, and SHALL NOT silently drop the request

### Requirement 17: Optimistic Updates for High-Frequency Actions

**User Story:** As a court owner, I want instant visual feedback when I toggle court visibility or feature flags, so that the UI feels responsive even before the server confirms the change.

#### Acceptance Criteria

1. WHEN a court owner toggles a court's visibility switch, THE Admin_Web_Portal SHALL optimistically update the court's `visible` field in the TanStack Query cache before the mutation completes
2. IF the visibility toggle mutation fails, THEN THE Admin_Web_Portal SHALL roll back the cache to the previous value and display an error toast
3. WHEN a PLATFORM_ADMIN toggles a feature flag, THE Admin_Web_Portal SHALL optimistically update the flag's `enabled` field in the TanStack Query cache
4. IF the feature flag toggle mutation fails, THEN THE Admin_Web_Portal SHALL roll back the cache to the previous value and display an error toast
5. THE optimistic update pattern SHALL use TanStack Query's `onMutate` to snapshot the previous cache value, update the cache optimistically, and use `onError` to restore the snapshot

### Requirement 18: Notification Bell and Dropdown — Real API Integration

**User Story:** As a court owner, I want the notification bell in the header to show my real unread notification count and recent notifications, so that I can stay informed about booking activity and platform events.

#### Acceptance Criteria

1. WHEN the AppLayout header mounts, THE Admin_Web_Portal SHALL call `notificationService.getUnreadCount()` to display the unread badge count on the notification bell icon
2. WHEN the user opens the notification dropdown, THE Admin_Web_Portal SHALL call `notificationService.getNotifications({ limit: 20 })` to fetch the 20 most recent notifications
3. WHEN a user clicks a notification in the dropdown, THE Admin_Web_Portal SHALL call `notificationService.markAsRead(notificationId)` and navigate to the relevant page (e.g., booking detail for booking notifications)
4. WHEN a user clicks "Mark all as read", THE Admin_Web_Portal SHALL call `notificationService.markAllAsRead()` and update the badge count to zero
5. WHEN a `NOTIFICATION_RECEIVED` WebSocket event arrives, THE Admin_Web_Portal SHALL increment the unread badge count and prepend the new notification to the dropdown list without requiring a full refetch
