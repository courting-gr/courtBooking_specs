# Implementation Plan: Admin Web API Integration

## Overview

Replace all hardcoded mock data in `court-booking-admin-web/` with real API calls. Build services layer → query keys → QueryStateWrapper → TanStack Query hooks → rewire every page → WebSocket cache sync → notification bell. All code targets the `court-booking-admin-web/` workspace only. Every task produces real working code — no stubs, no mock data.

## Tasks

- [x] 1. Services layer — Platform Service
  - [x] 1.1 Create `src/services/dashboardService.ts` with `getDashboard()` calling `GET /api/dashboard` via `platformApi`
    - Define `DashboardSummary` interface matching design
    - _Requirements: 1.5_

  - [x] 1.2 Create `src/services/courtService.ts` with all court CRUD, availability, pricing, and cancellation functions via `platformApi`
    - `getOwnerCourts`, `getCourt`, `createCourt`, `updateCourt`, `deleteCourt`, `uploadCourtImages`, `bulkToggleVisibility`, `getAvailabilityWindows`, `updateAvailabilityWindows`, `getAvailabilityOverrides`, `createAvailabilityOverride`, `deleteAvailabilityOverride`, `getPricingRules`, `updatePricingRules`, `getCancellationTiers`, `updateCancellationTiers`
    - Define all TypeScript interfaces (`CourtSummary`, `CourtDetail`, `CourtListParams`, `AvailabilityWindow`, `AvailabilityOverride`, `PricingRule`, `CancellationTier`, `BulkVisibilityResult`)
    - _Requirements: 1.6_

  - [x] 1.3 Create `src/services/analyticsService.ts` with `getRevenue`, `getUsage`, `getHeatmap`, `exportAnalytics` via `platformApi`
    - Define `AnalyticsParams`, `RevenueAnalytics`, `UsageAnalytics`, `HeatmapData` interfaces
    - `exportAnalytics` must use `responseType: 'blob'`
    - _Requirements: 1.7_

  - [x] 1.4 Create `src/services/supportService.ts` with `getTickets`, `getTicket`, `createTicket`, `addMessage`, `updateTicketStatus`, `assignTicket` via `platformApi`
    - Define `TicketListParams`, `TicketSummary`, `TicketDetail`, `CreateTicketRequest` interfaces
    - `addMessage` must handle file attachments via `FormData`
    - _Requirements: 1.8_

  - [x] 1.5 Create `src/services/adminService.ts` with user management, verification queue, feature flags, disputes, platform analytics, and support metrics functions via `platformApi`
    - Define `UserListParams`, `UserSummary`, `UserDetail`, `FeatureFlag`, `DisputeSummary`, `DisputeDetail`, `VerificationQueueItem`, `SupportMetrics`, `PlatformAnalytics` interfaces
    - _Requirements: 1.10_

  - [x] 1.6 Create `src/services/settingsService.ts` with profile, notification preferences, reminder rules, court defaults, sessions, GDPR export, and profile photo functions via `platformApi`
    - Define `NotificationPreference`, `ReminderRule`, `CourtDefaults`, `SessionInfo` interfaces
    - _Requirements: 1.9_

  - [x] 1.7 Create `src/services/searchService.ts` with `globalSearch` and `adminSearch` via `platformApi`
    - Define `SearchResult` interface
    - _Requirements: 1.11_

  - [x] 1.8 Create `src/services/authService.ts` with `fetchCsrfToken` via `platformApi`
    - _Requirements: 1.12_

- [x] 2. Services layer — Transaction Service
  - [x] 2.1 Create `src/services/bookingService.ts` with all booking CRUD, actions, bulk, manual, recurring, and calendar export functions via `transactionApi`
    - `getBookings`, `getBooking`, `getPendingBookings`, `confirmBooking`, `rejectBooking`, `cancelBooking`, `markPaid`, `flagNoShow`, `modifyBooking`, `bulkAction`, `createManualBooking`, `createRecurringBooking`, `exportCalendar`
    - Define `BookingListParams`, `BookingListItem`, `BookingDetail`, `PaginatedResponse<T>`, `CreateManualBookingRequest`, `CreateRecurringBookingRequest`, `BulkActionRequest`, `BulkActionResult` interfaces
    - _Requirements: 2.2_

  - [x] 2.2 Create `src/services/stripeService.ts` with `startOnboarding` and `getPayouts` via `transactionApi`
    - Define `PayoutInfo` interface
    - _Requirements: 2.3_

  - [x] 2.3 Create `src/services/notificationService.ts` with `getNotifications`, `getUnreadCount`, `markAsRead`, `markAllAsRead` via `transactionApi`
    - Define `NotificationItem` interface
    - _Requirements: 2.5_

  - [x] 2.4 Create `src/services/verificationService.ts` with `getVerificationStatus` and `submitVerification` via `platformApi`
    - _Requirements: 2.4_

  - [x] 2.5 Update `src/services/index.ts` barrel to re-export all 12 service modules
    - _Requirements: 1.13_

- [x] 3. Checkpoint — Services layer build verification
  - Run `npm run build` in `court-booking-admin-web/`. Ensure zero TypeScript errors. All service files must compile and the barrel export must resolve.

- [x] 4. Query key additions and QueryStateWrapper
  - [x] 4.1 Add `notificationKeys`, `auditLogKeys`, `holidayKeys`, `verificationKeys`, `stripeKeys` to `src/lib/queryKeys.ts`
    - Append to existing file, do not modify existing key factories
    - _Requirements: 3.2_

  - [x] 4.2 Create `src/components/feedback/QueryStateWrapper.tsx`
    - Generic component accepting `UseQueryResult`, rendering Skeleton for loading, Alert with retry for error, Empty for empty data, children render prop for data
    - Props: `query`, `skeletonType` (optional), `emptyMessage`, `emptyAction` (optional), `isEmpty` (optional predicate)
    - _Requirements: 16.1, 16.2, 16.3, 16.4_

- [x] 5. TanStack Query hooks — Core domain
  - [x] 5.1 Create `src/hooks/useDashboard.ts` with `useDashboard` query hook
    - Key: `dashboardKeys.summary()`, staleTime: 60s, refetchInterval: 60s
    - _Requirements: 3.3, 5.10_

  - [x] 5.2 Create `src/hooks/useCourts.ts` with query hooks (`useOwnerCourts`, `useCourtDetail`) and mutation hooks (`useCreateCourt`, `useUpdateCourt`, `useDeleteCourt`, `useToggleCourtVisibility` with optimistic update, `useBulkToggleVisibility`)
    - `useCourtDetail` must set `enabled: !!id`
    - `useToggleCourtVisibility` must implement optimistic update with rollback per design
    - _Requirements: 3.4, 4.2, 4.3, 4.12, 17.1, 17.2_

  - [x] 5.3 Create `src/hooks/useBookings.ts` with query hooks (`useBookingList`, `usePendingBookings`, `useBookingDetail`) and mutation hooks (`useConfirmBooking`, `useRejectBooking`, `useCancelBooking`, `useMarkPaid`, `useFlagNoShow`, `useModifyBooking`, `useBulkBookingAction`, `useCreateManualBooking`, `useCreateRecurringBooking`)
    - All booking mutations invalidate `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()`
    - Per-booking mutations also invalidate `bookingKeys.detail(id)`
    - _Requirements: 3.5, 4.4, 4.5, 4.6_

  - [x] 5.4 Create `src/hooks/useAnalytics.ts` with `useRevenue`, `useUsage`, `useHeatmap` query hooks
    - Each hook sets `enabled: !!params.from && !!params.to`
    - _Requirements: 3.6_

  - [x] 5.5 Create `src/hooks/useSupport.ts` with query hooks (`useTicketList`, `useTicketDetail`) and mutation hooks (`useCreateTicket`, `useAddMessage`, `useUpdateTicketStatus`, `useAssignTicket`)
    - _Requirements: 3.7, 4.7, 4.8_

  - [x] 5.6 Create `src/hooks/useAdmin.ts` with query hooks (`useUserList`, `useUserDetail`, `useFeatureFlags`, `useDisputes`, `useDisputeDetail`, `usePlatformAnalytics`, `useSupportMetrics`) and mutation hooks (`useSuspendUser`, `useUnsuspendUser`, `useUpdateFeatureFlag` with optimistic update, `useCreateFeatureFlag`, `useDeleteFeatureFlag`, `useAddDisputeNote`)
    - `useUpdateFeatureFlag` must implement optimistic update with rollback per design
    - _Requirements: 3.9, 4.10, 4.11, 17.3, 17.4_

  - [x] 5.7 Create `src/hooks/useSettings.ts` with query hooks (`useNotificationPreferences`, `useReminderRules`, `useCourtDefaults`, `useSessions`) and mutation hooks (`useUpdateProfile`, `useUpdateNotificationPreferences`, `useCreateReminderRule`, `useUpdateReminderRule`, `useDeleteReminderRule`, `useUpdateCourtDefaults`, `useRevokeSession`)
    - _Requirements: 3.8, 4.9_

  - [x] 5.8 Create `src/hooks/useNotifications.ts` with `useUnreadCount` (refetchInterval: 30s), `useNotificationList`, `useMarkAsRead`, `useMarkAllAsRead`
    - _Requirements: 18.1, 18.2, 18.4_

  - [x] 5.9 Create `src/hooks/useSearch.ts` with `useGlobalSearch` hook using debounced query, `enabled: query.length >= 2`, and role-based routing to `globalSearch` vs `adminSearch`
    - _Requirements: 14.1, 14.2, 14.5_

- [x] 6. Checkpoint — Hooks and infrastructure build verification
  - Run `npm run build` in `court-booking-admin-web/`. Ensure all hooks compile. Verify no unused imports or circular dependencies.

- [x] 7. Page rewiring — Dashboard
  - [x] 7.1 Rewire `src/features/dashboard/pages/DashboardPage.tsx`
    - Remove: `mockStats`, `mockPayout`, `mockNeedsAttention`, `mockNotifications`, `mockCourtSummary`, `mockUserState`
    - Import and call `useDashboard` hook
    - Wrap content in `QueryStateWrapper` or use `isLoading`/`isError` destructured values
    - Map all stat cards, payout card, needs attention, notifications list, court summary, verification/stripe banners to `dashboard.data.*`
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 5.8, 5.9_

- [x] 8. Page rewiring — Courts
  - [x] 8.1 Rewire `src/features/courts/pages/CourtListPage.tsx`
    - Remove: `mockCourts`, `useState(mockCourts)`, local `handleVisibilityToggle`, local `handleDelete`, client-side filtering
    - Import `useOwnerCourts`, `useToggleCourtVisibility`, `useDeleteCourt`, `useBulkToggleVisibility`
    - Wire table `dataSource` to `courts ?? []`, visibility Switch to `toggleVisibility.mutate`, delete Popconfirm to `deleteCourt.mutate`, bulk action to `useBulkToggleVisibility`
    - Pass filter bar values as params to `useOwnerCourts`
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6, 6.12_

  - [x] 8.2 Rewire `src/features/courts/pages/CourtFormPage.tsx`
    - Remove any mock data used for form population
    - In edit mode: call `useCourtDetail(id)` to populate form
    - On submit: call `useCreateCourt` (create mode) or `useUpdateCourt` (edit mode)
    - Wire image upload to `courtService.uploadCourtImages`
    - Navigate to court list on success
    - _Requirements: 6.7, 6.8, 6.9_

  - [x] 8.3 Rewire `src/features/courts/pages/CourtDetailPage.tsx`
    - Remove any mock court data
    - Call `useCourtDetail(id)` for main court data
    - Fetch availability windows, pricing rules, cancellation tiers via service calls or additional hooks
    - Show "Court not found" with back link on 404
    - _Requirements: 6.10, 6.11_

- [x] 9. Checkpoint — Dashboard and Courts build verification
  - Run `npm run build` in `court-booking-admin-web/`. Verify DashboardPage, CourtListPage, CourtFormPage, CourtDetailPage compile without errors.

- [ ] 10. Page rewiring — Bookings
  - [x] 10.1 Rewire `src/features/bookings/pages/BookingListPage.tsx`
    - Remove: `mockBookings`
    - Import `useBookingList`, wire table to `data?.content ?? []`
    - Wire server-side pagination via `data?.page`, `data?.totalElements`
    - Wire filter bar (date range, court, status, payment status, search) to `filters` state passed to `useBookingList`
    - _Requirements: 7.1, 7.2, 7.3_

  - [x] 10.2 Rewire `src/features/bookings/pages/BookingCalendarPage.tsx`
    - Remove any mock calendar data
    - Call `useBookingList` with calendar date range params
    - _Requirements: 7.4_

  - [x] 10.3 Rewire `src/features/bookings/pages/PendingBookingsPage.tsx`
    - Remove any mock pending bookings
    - Call `usePendingBookings`, wire confirm/reject buttons to `useConfirmBooking`/`useRejectBooking` mutations
    - Wire bulk action to `useBulkBookingAction`
    - _Requirements: 7.5, 7.6, 7.7, 7.8_

  - [x] 10.4 Rewire `src/features/bookings/pages/ManualBookingPage.tsx`
    - Remove any console.log submission patterns
    - Wire form submit to `useCreateManualBooking` for single bookings, `useCreateRecurringBooking` for recurring
    - Show success notification and navigate on completion
    - _Requirements: 7.9_

  - [x] 10.5 Rewire `src/features/bookings/pages/BookingDetailPage.tsx`
    - Remove: `mockBooking`
    - Call `useBookingDetail(bookingId)` from URL params
    - Wire cancel, mark-paid, no-show, modify actions to corresponding mutation hooks
    - Show "Booking not found" with back link on 404
    - _Requirements: 7.10, 7.11, 7.12_

  - [x] 10.6 Check `src/features/bookings/pages/BookingsPage.tsx` (tab container)
    - This is a tab wrapper that imports BookingListPage, BookingCalendarPage, PendingBookingsPage, ManualBookingPage — verify it has no mock data of its own. If it does, remove it
    - _Requirements: 7.1_

- [x] 11. Checkpoint — Bookings build verification
  - Run `npm run build` in `court-booking-admin-web/`. Verify all booking pages compile without errors.

- [x] 12. Page rewiring — Analytics
  - [x] 12.1 Rewire `src/features/analytics/pages/RevenueAnalyticsPage.tsx`
    - Remove any mock analytics data
    - Call `useRevenue(params)`, wire date range picker and court filter to `params` state
    - Wire export button to `analyticsService.exportAnalytics` with blob download
    - Handle 429 rate limit with countdown alert
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

  - [x] 12.2 Rewire `src/features/analytics/pages/UsageAnalyticsPage.tsx`
    - Remove any mock usage data
    - Call `useUsage(params)`, wire filters
    - _Requirements: 8.6_

  - [x] 12.3 Rewire `src/features/analytics/pages/HeatmapPage.tsx`
    - Remove any mock heatmap data
    - Call `useHeatmap(params)`, render day×hour matrix from real data
    - _Requirements: 8.7, 8.8_

- [x] 13. Page rewiring — Support
  - [x] 13.1 Rewire `src/features/support/pages/SupportTicketListPage.tsx`
    - Remove: `mockTickets`, `useState(mockTickets[0].id)`
    - Call `useTicketList(filters)`, wire filter bar to `filters` state
    - Wire ticket selection to `useTicketDetail(selectedId)` for right panel
    - _Requirements: 9.1, 9.2, 9.3_

  - [x] 13.2 Rewire `src/features/support/pages/SupportTicketDetailPage.tsx`
    - Remove any mock ticket detail data
    - Call `useTicketDetail(id)` for full message thread
    - Wire "Send Reply" to `useAddMessage` mutation
    - Wire status update to `useUpdateTicketStatus`, assign to `useAssignTicket`
    - Wire new ticket creation to `useCreateTicket` with navigation on success
    - Show "Ticket not found" on 404
    - _Requirements: 9.4, 9.5, 9.6, 9.7, 9.8, 9.9_

- [x] 14. Page rewiring — Settings
  - [x] 14.1 Rewire `src/features/settings/pages/SettingsPage.tsx`
    - Remove all hardcoded form values, mock sessions, mock payout data
    - Profile tab: populate from `useAuthStore`, save via `useUpdateProfile` mutation, wire profile photo upload/delete to `settingsService.uploadProfilePhoto`/`deleteProfilePhoto`
    - Notifications tab: `useNotificationPreferences()` query, `useUpdateNotificationPreferences` mutation on toggle
    - Reminders tab: `useReminderRules()` query, `useCreateReminderRule`/`useUpdateReminderRule`/`useDeleteReminderRule` mutations
    - Court Defaults tab: `useCourtDefaults()` query, `useUpdateCourtDefaults` mutation
    - Security tab: `useSessions()` query, `useRevokeSession` mutation, `settingsService.requestDataExport()` for GDPR
    - Stripe tab: `stripeService.getPayouts()` for payout data
    - Show Skeleton per tab while loading, error toast on mutation failure
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7, 10.8, 10.9_

- [x] 15. Checkpoint — Analytics, Support, Settings build verification
  - Run `npm run build` in `court-booking-admin-web/`. Verify all analytics, support, and settings pages compile.

- [x] 16. Page rewiring — Admin pages
  - [x] 16.1 Rewire `src/features/admin/pages/UserManagementPage.tsx`
    - Remove any mock user data
    - Call `useUserList(params)`, wire suspend/unsuspend to `useSuspendUser`/`useUnsuspendUser` mutations
    - _Requirements: 11.1, 11.2, 11.3_

  - [x] 16.2 Rewire `src/features/admin/pages/VerificationQueuePage.tsx`
    - Remove any mock verification data
    - Call query hook for `GET /api/verification/queue`
    - Wire approve/reject to corresponding mutation hooks
    - _Requirements: 11.4, 11.5_

  - [x] 16.3 Rewire `src/features/admin/pages/FeatureFlagsPage.tsx`
    - Remove: `initialFlags`, `useState(initialFlags)`, local `toggleFlag`, local `handleAddFlag`
    - Call `useFeatureFlags()`, wire toggle Switch to `useUpdateFeatureFlag` (optimistic), create to `useCreateFeatureFlag`, delete to `useDeleteFeatureFlag`
    - _Requirements: 11.6, 11.7, 11.8_

  - [x] 16.4 Rewire `src/features/admin/pages/PlatformAnalyticsPage.tsx`
    - Remove any mock platform analytics data
    - Call `usePlatformAnalytics(params)`, wire date range filter
    - _Requirements: 11.9_

  - [x] 16.5 Rewire `src/features/admin/pages/SupportMetricsPage.tsx`
    - Remove any mock support metrics data
    - Call `useSupportMetrics(params)`, wire date range filter
    - _Requirements: 11.10_

  - [x] 16.6 Rewire `src/features/admin/pages/DisputesPage.tsx`
    - Remove any mock dispute data
    - Call `useDisputes(params)` for list, `useDisputeDetail(id)` for detail view
    - Wire "Add Note" to `useAddDisputeNote` mutation
    - _Requirements: 11.11, 11.12, 11.13_

- [x] 17. Page rewiring — Audit log, Holidays, Auth, Search
  - [x] 17.1 Rewire `src/features/settings/pages/AuditLogPage.tsx`
    - Remove: `mockEntries`
    - Use inline `useQuery` with `auditLogKeys.list(filters)` and `platformApi.get('/api/audit-logs', { params: filters })`
    - Wire filter bar and server-side pagination
    - _Requirements: 12.1, 12.2, 12.3_

  - [x] 17.2 Rewire `src/features/settings/pages/HolidayCalendarPage.tsx`
    - Remove: `mockNationalHolidays`, `mockCustomHolidays`, `mockCourts`
    - Use `useQuery` with `holidayKeys.national()` and `holidayKeys.custom()` for holiday data
    - Use `useOwnerCourts()` for court multi-select dropdown
    - Wire create/update/delete custom holiday and apply holidays mutations
    - _Requirements: 12.4, 12.5, 12.6_

  - [x] 17.3 Rewire `src/features/auth/pages/StripeOnboardingPage.tsx`
    - Remove any mock Stripe data
    - Call `stripeService.startOnboarding()` for onboarding flow
    - Call `stripeService.getPayouts()` for payout display
    - _Requirements: 13.2, 13.3_

  - [x] 17.4 Rewire `src/features/auth/pages/VerificationPage.tsx`
    - Remove any mock verification status
    - Call `verificationService.getVerificationStatus()` on mount
    - Wire form submit to `verificationService.submitVerification(data)`
    - _Requirements: 13.4, 13.5_

  - [x] 17.5 Rewire `src/features/auth/pages/LoginPage.tsx`
    - Verify OAuth login calls `useAuthStore.login()` (already wired) and `fetchCsrfToken()` after success
    - Ensure no mock user data or hardcoded tokens — login must go through the real OAuth flow
    - _Requirements: 13.1_

  - [x] 17.6 Rewire global search in `src/components/layout/AppLayout.tsx` header search bar
    - The `Input.Search` in AppLayout header currently does nothing
    - Use `useGlobalSearch` hook from `src/hooks/useSearch.ts`
    - Wire search input with 300ms debounce, min 2 char threshold
    - Display results in a dropdown/popover grouped by type, navigate on click
    - _Requirements: 14.1, 14.2, 14.3, 14.4, 14.5_

- [x] 18. Checkpoint — Admin, Audit, Holidays, Auth, Search build verification
  - Run `npm run build` in `court-booking-admin-web/`. Verify all remaining pages compile.

- [x] 19. WebSocket cache sync and notification bell
  - [x] 19.1 Create `src/hooks/useWebSocketSync.ts`
    - Subscribe to `/user/queue/notifications` via existing `useWebSocket` hook
    - Map event types to cache invalidations per design: `BOOKING_CREATED`/`BOOKING_UPDATED` → booking + dashboard keys, `BOOKING_CONFIRMED`/`BOOKING_CANCELLED` → + detail key, `NOTIFICATION_RECEIVED` → notification + dashboard keys, `SUPPORT_TICKET_UPDATED` → support keys, `COURT_UPDATED` → court + dashboard keys
    - On reconnect (isConnected transitions to true): invalidate ALL active queries
    - _Requirements: 15.1, 15.2, 15.3, 15.4, 15.5, 15.7_

  - [x] 19.2 Mount `useWebSocketSync()` in `src/components/layout/AppLayout.tsx`
    - Call the hook once at the layout level so it's active for all authenticated pages
    - _Requirements: 15.6_

  - [x] 19.3 Rewire `src/components/layout/NotificationBell.tsx`
    - Remove: `mockNotifications`, `useState(mockNotifications)`
    - Call `useUnreadCount()` for badge count, `useNotificationList({ size: 20 })` for dropdown
    - Wire click to `useMarkAsRead`, "Mark all read" to `useMarkAllAsRead`
    - Navigate to relevant page on notification click
    - _Requirements: 18.1, 18.2, 18.3, 18.4, 18.5_

- [x] 20. Checkpoint — Full build verification
  - Run `npm run build` in `court-booking-admin-web/`. Ensure zero TypeScript errors across the entire project.

- [x] 21. Property-based tests
  - [x] 21.1 Write property test for query key uniqueness
    - **Property 1: Query Key Uniqueness** — different filter objects produce distinct query keys
    - **Validates: Requirements 3.10**
    - File: `src/__tests__/properties/queryKeyUniqueness.property.test.ts`

  - [x] 21.2 Write property test for mutation error isolation
    - **Property 2: Mutation Error Isolation** — failed mutations never trigger cache invalidation
    - **Validates: Requirements 4.13**
    - File: `src/__tests__/properties/mutationErrorIsolation.property.test.ts`

  - [x] 21.3 Write property test for search input gating
    - **Property 3: Search Input Gating** — queries < 2 chars never call API, queries >= 2 chars call exactly once after debounce
    - **Validates: Requirements 14.1, 14.2**
    - File: `src/__tests__/properties/searchInputGating.property.test.ts`

  - [x] 21.4 Write property test for court visibility optimistic round-trip
    - **Property 4: Court Visibility Optimistic Round-Trip** — toggle + server error → cache restored to pre-mutation state
    - **Validates: Requirements 17.1, 17.2, 17.5**
    - File: `src/__tests__/properties/courtVisibilityOptimistic.property.test.ts`

  - [x] 21.5 Write property test for feature flag optimistic round-trip
    - **Property 5: Feature Flag Optimistic Round-Trip** — toggle + server error → cache restored to pre-mutation state
    - **Validates: Requirements 17.3, 17.4, 17.5**
    - File: `src/__tests__/properties/featureFlagOptimistic.property.test.ts`

  - [x] 21.6 Write property test for WebSocket reconnect full invalidation
    - **Property 6: WebSocket Reconnect Full Invalidation** — reconnect invalidates ALL active query keys
    - **Validates: Requirements 15.7**
    - File: `src/__tests__/properties/wsReconnectInvalidation.property.test.ts`

  - [x] 21.7 Write property test for WebSocket event → cache invalidation mapping
    - **Property 7: WS Event Cache Mapping** — each event type invalidates the correct set of query keys
    - **Validates: Requirements 15.2, 15.3**
    - File: `src/__tests__/properties/wsEventCacheMapping.property.test.ts`

- [x] 22. Unit tests
  - [x] 22.1 Write unit tests for service functions (correct URL, method, platformApi vs transactionApi routing)
    - File: `src/__tests__/services/*.test.ts`
    - _Requirements: 1.2, 1.4, 2.1_

  - [x] 22.2 Write unit tests for QueryStateWrapper (loading skeleton, error alert, empty state, data rendering)
    - File: `src/__tests__/components/QueryStateWrapper.test.tsx`
    - _Requirements: 16.1, 16.2, 16.3, 16.4_

  - [x] 22.3 Write unit tests for key page components (DashboardPage, CourtListPage, BookingListPage, FeatureFlagsPage, NotificationBell) verifying they use hook data not mock data
    - File: `src/__tests__/pages/*.test.tsx`
    - _Requirements: 5.1, 6.1, 7.1, 11.6, 18.1_

- [x] 23. Final verification — No mock data, full build, local testing
  - Run `grep -r "const mock" src/features/` in `court-booking-admin-web/` — must return zero results
  - Run `grep -r "const mock" src/components/` in `court-booking-admin-web/` — must return zero results (catches NotificationBell mock data)
  - Run `npm run build` — must succeed with zero errors
  - Run `npm run test` (if tests exist) — must pass
  - Verify `.env.local` has correct values: `VITE_PLATFORM_API_URL=http://localhost:8080`, `VITE_TRANSACTION_API_URL=http://localhost:8081`
  - Verify Vite dev server proxy config (if any) allows CORS to backend services on different ports
  - Local testing checklist (manual verification by user):
    1. Start infrastructure: `docker-compose up -d` (PostgreSQL, Redis, Kafka/Redpanda)
    2. Start Platform Service: `./gradlew bootRun --args='--spring.profiles.active=local'` on `:8080`
    3. Start Transaction Service: `./gradlew bootRun --args='--spring.profiles.active=local'` on `:8081`
    4. Start admin-web: `npm run dev` on `:5173`
    5. Open `http://localhost:5173` in browser
    6. Verify login page loads and OAuth buttons are functional
    7. After login: verify dashboard loads real data (not mock), stat cards show numbers from API
    8. Navigate to Courts → verify court list loads from API (empty list is OK if no courts exist)
    9. Navigate to Bookings → verify booking list loads from API
    10. Navigate to Analytics → verify charts render with real data or show empty state
    11. Navigate to Support → verify ticket list loads from API
    12. Navigate to Settings → verify each tab loads real data
    13. Check browser DevTools Network tab → confirm all XHR requests go to `:8080` or `:8081`, no `mockXxx` in responses
    14. Check browser console → no "mock" references, no unhandled promise rejections from API calls
  - _Requirements: All_

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- All code targets `court-booking-admin-web/` only — no backend changes needed
- Every page rewiring task must remove `const mockXxx` constants and replace with real hook calls
- Checkpoints after each group ensure incremental build stability
- Property tests validate universal correctness properties from the design document
- The final task verifies zero mock data remains and the app is testable locally
- `BookingsPage.tsx` is a tab container wrapping BookingListPage, BookingCalendarPage, PendingBookingsPage, ManualBookingPage — it may not have mock data itself but must be checked
- `NotificationBell.tsx` in `src/components/layout/` has its own mock notification data that must be replaced (task 19.3)
- `AppLayout.tsx` has a search `Input.Search` in the header that currently does nothing — must be wired to `useGlobalSearch` (task 17.6)
- `LoginPage.tsx` already uses `useAuthStore.login()` but must be verified to call `fetchCsrfToken()` after login (task 17.5)
- The `grep` check in task 23 must cover both `src/features/` AND `src/components/` to catch all mock data
