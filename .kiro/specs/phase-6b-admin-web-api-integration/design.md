# Design Document — Admin Web API Integration

## Overview

This design replaces all hardcoded mock data in the `court-booking-admin-web` React application with real API calls. The admin portal was scaffolded in Phase 6 with correct Ant Design UI, but every page component renders static `const mockXxx` constants. The `src/services/index.ts` barrel is empty, no TanStack Query hooks exist, and form submissions log to console.

This integration creates three layers:

1. **Services Layer** (`src/services/`): Typed functions wrapping `platformApi` and `transactionApi` Axios instances
2. **Query/Mutation Hooks** (`src/hooks/`): TanStack Query v5 hooks using the query key factories from `src/lib/queryKeys.ts`
3. **Page Rewiring**: Replacing `mockXxx` constants with hook calls, wiring `data`/`isLoading`/`isError` to existing Ant Design components

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| One service file per domain | Matches the query key factory structure; keeps imports predictable |
| Hooks co-located by domain | `useDashboard.ts`, `useCourts.ts`, etc. — each file exports both queries and mutations for that domain |
| `QueryStateWrapper` component | Eliminates repeated loading/error/empty boilerplate across 15+ pages |
| Optimistic updates only for toggles | Court visibility and feature flag toggles are high-frequency, low-risk actions; other mutations use invalidation |
| `notificationKeys` added to queryKeys.ts | Notification bell needs its own cache entries separate from dashboard |
| WebSocket → `invalidateQueries` | Simpler than direct cache updates; ensures data consistency with server |
| Service functions return `response.data` | Hooks receive typed data directly; Axios error interceptor handles error normalization |

### Existing Infrastructure (DO NOT recreate)

- `src/lib/api.ts`: `platformApi`, `transactionApi` Axios instances with JWT/CSRF interceptors, 401 refresh, error normalization to `ApiError`
- `src/lib/queryKeys.ts`: Query key factories for `dashboardKeys`, `courtKeys`, `bookingKeys`, `analyticsKeys`, `supportKeys`, `adminKeys`, `settingsKeys`
- `src/stores/authStore.ts`: Zustand store with `user`, `accessToken`, `csrfToken`, `login`/`logout`/`refreshToken`/`fetchCsrfToken`
- `src/stores/uiStore.ts`: Zustand store with `sidebarCollapsed`, `theme`
- `src/lib/websocket.ts`: STOMP/SockJS client with `connect`/`disconnect`/`subscribe`, exponential backoff
- `src/hooks/useWebSocket.ts`: Hook wrapping websocket subscribe with connection status tracking
- `src/types/api/platform.ts` and `src/types/api/transaction.ts`: Auto-generated OpenAPI types

## Architecture

### Layer Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Page Components                               │
│  DashboardPage, CourtListPage, BookingListPage, SettingsPage, ...   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  QueryStateWrapper (loading / error / empty / children)      │    │
│  └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ uses
┌──────────────────────────────▼──────────────────────────────────────┐
│                     TanStack Query Hooks                             │
│  src/hooks/useDashboard.ts, useCourts.ts, useBookings.ts, ...      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐      │
│  │ useQuery()   │  │ useMutation()│  │ useWebSocketSync()   │      │
│  │ + queryKey   │  │ + onSuccess  │  │ + invalidateQueries  │      │
│  │ + staleTime  │  │   invalidate │  │   on WS events       │      │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘      │
└─────────┼─────────────────┼─────────────────────┼──────────────────┘
          │ calls            │ calls               │ calls
┌─────────▼─────────────────▼─────────────────────▼──────────────────┐
│                       Services Layer                                 │
│  src/services/dashboardService.ts, courtService.ts, ...             │
│  ┌──────────────────────┐  ┌──────────────────────┐                 │
│  │ platformApi.get(...)  │  │ transactionApi.get(.) │                │
│  └──────────┬───────────┘  └──────────┬───────────┘                 │
└─────────────┼──────────────────────────┼───────────────────────────┘
              │ HTTP                      │ HTTP
              ▼                           ▼
     Platform Service            Transaction Service
```

### File Structure

```
src/
├── services/
│   ├── index.ts                    # Barrel re-export
│   ├── dashboardService.ts         # GET /api/dashboard
│   ├── courtService.ts             # Courts CRUD, availability, pricing, cancellation
│   ├── bookingService.ts           # Bookings CRUD, bulk, manual, recurring (transactionApi)
│   ├── analyticsService.ts         # Revenue, usage, heatmap, export
│   ├── supportService.ts           # Tickets CRUD, messages, attachments
│   ├── adminService.ts             # Users, verifications, flags, disputes, platform analytics
│   ├── settingsService.ts          # Profile, notifications, reminders, defaults, sessions, GDPR
│   ├── searchService.ts            # Global search, admin search
│   ├── authService.ts              # CSRF token
│   ├── stripeService.ts            # Stripe Connect onboarding, payouts (transactionApi)
│   ├── notificationService.ts      # Notifications list, unread count, mark read (transactionApi)
│   └── verificationService.ts      # Submit/get verification status
├── hooks/
│   ├── useDashboard.ts             # useDashboard query
│   ├── useCourts.ts                # useOwnerCourts, useCourtDetail, mutations
│   ├── useBookings.ts              # useBookingList, usePendingBookings, useBookingDetail, mutations
│   ├── useAnalytics.ts             # useRevenue, useUsage, useHeatmap
│   ├── useSupport.ts               # useTicketList, useTicketDetail, mutations
│   ├── useAdmin.ts                 # useUserList, useFeatureFlags, useDisputes, mutations
│   ├── useSettings.ts              # useNotificationPreferences, useReminderRules, etc.
│   ├── useNotifications.ts         # useUnreadCount, useNotificationList, mutations
│   ├── useSearch.ts                # useGlobalSearch with debounce
│   └── useWebSocketSync.ts         # WebSocket → queryClient.invalidateQueries mapping
├── components/
│   └── feedback/
│       └── QueryStateWrapper.tsx    # Reusable loading/error/empty wrapper
└── lib/
    └── queryKeys.ts                 # ADD notificationKeys, auditLogKeys, holidayKeys
```

## Components and Interfaces

### Services Layer — TypeScript Interfaces

All service functions return `Promise<T>` where `T` is the response data type. Errors propagate as the normalized `ApiError` from the Axios interceptor (`{ error: string, message?: string, status: number }`).

#### Dashboard Service (`src/services/dashboardService.ts`)

```typescript
import { platformApi } from '@/lib/api';

// Response types
export interface DashboardSummary {
  todayBookings: number;
  pendingConfirmations: number;
  revenueThisMonthCents: number;
  nextPayoutDate: string | null;       // ISO8601 date
  nextPayoutAmountCents: number | null;
  occupancyRateToday: number;          // 0.0–1.0
  actionRequired: {
    pendingBookings: number;
    unpaidManualBookings: number;
    expiringPromos: number;
    reminderAlerts: number;
  };
  recentNotifications: Array<{
    id: string;
    type: string;
    message: string;
    createdAt: string;
    read: boolean;
  }>;
  courtSummary: {
    totalCourts: number;
    visibleCourts: number;
    hiddenCourts: number;
  };
  verificationStatus: string;
  stripeConnectStatus: string;
}

export async function getDashboard(): Promise<DashboardSummary> {
  const { data } = await platformApi.get<DashboardSummary>('/api/dashboard');
  return data;
}
```

#### Court Service (`src/services/courtService.ts`)

```typescript
import { platformApi } from '@/lib/api';

export interface CourtListParams {
  type?: string;
  locationType?: string;
  search?: string;
  page?: number;
  size?: number;
}

export interface CourtSummary {
  id: string;
  name: string;
  nameEl: string;
  nameEn: string;
  courtType: string;
  locationType: string;
  basePriceCents: number;
  confirmationMode: 'INSTANT' | 'MANUAL';
  visible: boolean;
  imageUrls: string[];
  address: string;
}

export interface CourtDetail extends CourtSummary {
  description: string;
  descriptionEl: string;
  descriptionEn: string;
  amenities: string[];
  latitude: number;
  longitude: number;
  maxCapacity: number;
  defaultDurationMinutes: number;
  waitlistEnabled: boolean;
  createdAt: string;
  updatedAt: string;
}

export interface CreateCourtRequest {
  nameEl: string;
  nameEn: string;
  descriptionEl?: string;
  descriptionEn?: string;
  courtType: string;
  locationType: string;
  basePriceCents: number;
  address: string;
  latitude: number;
  longitude: number;
  amenities?: string[];
  defaultDurationMinutes?: number;
  confirmationMode?: 'INSTANT' | 'MANUAL';
}

export interface UpdateCourtRequest extends Partial<CreateCourtRequest> {}

export interface AvailabilityWindow {
  dayOfWeek: string;
  startTime: string;
  endTime: string;
}

export interface AvailabilityOverride {
  id: string;
  date: string;
  startTime: string | null;
  endTime: string | null;
  reason: string;
  allDay: boolean;
}

export interface PricingRule {
  id: string;
  dayOfWeek: string;
  startTime: string;
  endTime: string;
  multiplier: number;
  label: string;
}

export interface CancellationTier {
  hoursBeforeBooking: number;
  refundPercentage: number;
}

export interface BulkVisibilityResult {
  successCount: number;
  failureCount: number;
  failures: Array<{ courtId: string; reason: string }>;
}

export async function getOwnerCourts(params?: CourtListParams): Promise<CourtSummary[]> {
  const { data } = await platformApi.get<CourtSummary[]>('/api/courts/owner/me', { params });
  return data;
}

export async function getCourt(id: string): Promise<CourtDetail> {
  const { data } = await platformApi.get<CourtDetail>(`/api/courts/${id}`);
  return data;
}

export async function createCourt(req: CreateCourtRequest): Promise<CourtDetail> {
  const { data } = await platformApi.post<CourtDetail>('/api/courts', req);
  return data;
}

export async function updateCourt(id: string, req: UpdateCourtRequest): Promise<CourtDetail> {
  const { data } = await platformApi.put<CourtDetail>(`/api/courts/${id}`, req);
  return data;
}

export async function deleteCourt(id: string): Promise<void> {
  await platformApi.delete(`/api/courts/${id}`);
}

export async function uploadCourtImages(id: string, files: File[]): Promise<string[]> {
  const formData = new FormData();
  files.forEach((f) => formData.append('images', f));
  const { data } = await platformApi.post<string[]>(`/api/courts/${id}/images`, formData, {
    headers: { 'Content-Type': 'multipart/form-data' },
  });
  return data;
}

export async function bulkToggleVisibility(
  courtIds: string[],
  visible: boolean,
): Promise<BulkVisibilityResult> {
  const { data } = await platformApi.post<BulkVisibilityResult>(
    '/api/courts/bulk/visibility',
    { courtIds, visible },
  );
  return data;
}

export async function getAvailabilityWindows(courtId: string): Promise<AvailabilityWindow[]> {
  const { data } = await platformApi.get<AvailabilityWindow[]>(
    `/api/courts/${courtId}/availability/windows`,
  );
  return data;
}

export async function updateAvailabilityWindows(
  courtId: string,
  windows: AvailabilityWindow[],
): Promise<AvailabilityWindow[]> {
  const { data } = await platformApi.put<AvailabilityWindow[]>(
    `/api/courts/${courtId}/availability/windows`,
    windows,
  );
  return data;
}

export async function getAvailabilityOverrides(courtId: string): Promise<AvailabilityOverride[]> {
  const { data } = await platformApi.get<AvailabilityOverride[]>(
    `/api/courts/${courtId}/availability/overrides`,
  );
  return data;
}

export async function createAvailabilityOverride(
  courtId: string,
  override: Omit<AvailabilityOverride, 'id'>,
): Promise<AvailabilityOverride> {
  const { data } = await platformApi.post<AvailabilityOverride>(
    `/api/courts/${courtId}/availability/overrides`,
    override,
  );
  return data;
}

export async function deleteAvailabilityOverride(
  courtId: string,
  overrideId: string,
): Promise<void> {
  await platformApi.delete(`/api/courts/${courtId}/availability/overrides/${overrideId}`);
}

export async function getPricingRules(courtId: string): Promise<PricingRule[]> {
  const { data } = await platformApi.get<PricingRule[]>(`/api/courts/${courtId}/pricing`);
  return data;
}

export async function updatePricingRules(
  courtId: string,
  rules: PricingRule[],
): Promise<PricingRule[]> {
  const { data } = await platformApi.put<PricingRule[]>(
    `/api/courts/${courtId}/pricing`,
    rules,
  );
  return data;
}

export async function getCancellationTiers(courtId: string): Promise<CancellationTier[]> {
  const { data } = await platformApi.get<CancellationTier[]>(
    `/api/courts/${courtId}/cancellation-policy`,
  );
  return data;
}

export async function updateCancellationTiers(
  courtId: string,
  tiers: CancellationTier[],
): Promise<CancellationTier[]> {
  const { data } = await platformApi.put<CancellationTier[]>(
    `/api/courts/${courtId}/cancellation-policy`,
    tiers,
  );
  return data;
}
```


#### Booking Service (`src/services/bookingService.ts`)

```typescript
import { transactionApi } from '@/lib/api';

export interface BookingListParams {
  page?: number;
  size?: number;
  courtId?: string;
  status?: string;
  paymentStatus?: string;
  from?: string;
  to?: string;
  search?: string;
  sort?: string;
}

export interface BookingListItem {
  id: string;
  courtId: string;
  courtName: string;
  courtType: string;
  customerId: string;
  date: string;
  startTime: string;
  endTime: string;
  durationMinutes: number;
  numberOfPeople: number;
  status: string;
  confirmationMode: string;
  amount: number;
  currency: string;
  paymentStatus: string;
  isManual: boolean;
  isRecurring: boolean;
  recurringGroupId: string | null;
  timezone: string;
  createdAt: string;
}

export interface PaginatedResponse<T> {
  content: T[];
  page: number;
  size: number;
  totalElements: number;
  totalPages: number;
}

export interface BookingDetail extends BookingListItem {
  courtAddress: string;
  courtLocationType: string;
  customerName: string;
  customerEmail: string;
  customerPhone: string;
  platformFeeCents: number;
  courtOwnerNetCents: number;
  stripePaymentIntentId: string | null;
  paymentMethod: string | null;
  noShow: boolean;
  cancellationReason: string | null;
  rejectionReason: string | null;
  auditTrail: Array<{
    action: string;
    actor: string;
    actorRole: string;
    timestamp: string;
  }>;
  paymentTimeline: Array<{
    action: string;
    timestamp: string;
    status: string;
  }>;
}

export interface CreateManualBookingRequest {
  courtId: string;
  date: string;
  startTime: string;
  durationMinutes: number;
  numberOfPeople?: number;
  customerName?: string;
  customerPhone?: string;
  customerEmail?: string;
  notes?: string;
}

export interface CreateRecurringBookingRequest {
  courtId: string;
  date: string;
  startTime: string;
  durationMinutes: number;
  numberOfPeople?: number;
  paymentMethodId: string;
  weeks: number;
}

export interface BulkActionRequest {
  bookingIds: string[];
  action: 'CONFIRM' | 'REJECT';
  reason?: string;
}

export interface BulkActionResult {
  successCount: number;
  failureCount: number;
  results: Array<{ bookingId: string; success: boolean; error?: string }>;
}

export async function getBookings(
  params?: BookingListParams,
): Promise<PaginatedResponse<BookingListItem>> {
  const { data } = await transactionApi.get<PaginatedResponse<BookingListItem>>(
    '/api/bookings',
    { params },
  );
  return data;
}

export async function getBooking(id: string): Promise<BookingDetail> {
  const { data } = await transactionApi.get<BookingDetail>(`/api/bookings/${id}`);
  return data;
}

export async function getPendingBookings(
  params?: { page?: number; size?: number },
): Promise<PaginatedResponse<BookingListItem>> {
  const { data } = await transactionApi.get<PaginatedResponse<BookingListItem>>(
    '/api/bookings/pending',
    { params },
  );
  return data;
}

export async function confirmBooking(id: string): Promise<void> {
  await transactionApi.post(`/api/bookings/${id}/confirm`);
}

export async function rejectBooking(
  id: string,
  reason: string,
): Promise<void> {
  await transactionApi.post(`/api/bookings/${id}/reject`, { reason });
}

export async function cancelBooking(
  id: string,
  reason: string,
): Promise<void> {
  await transactionApi.post(`/api/bookings/${id}/cancel`, { reason });
}

export async function markPaid(id: string): Promise<void> {
  await transactionApi.post(`/api/bookings/${id}/mark-paid`);
}

export async function flagNoShow(id: string): Promise<void> {
  await transactionApi.post(`/api/bookings/${id}/no-show`);
}

export async function modifyBooking(
  id: string,
  data: { date: string; startTime: string; durationMinutes: number },
): Promise<BookingDetail> {
  const { data: result } = await transactionApi.put<BookingDetail>(
    `/api/bookings/${id}/modify`,
    data,
  );
  return result;
}

export async function bulkAction(req: BulkActionRequest): Promise<BulkActionResult> {
  const { data } = await transactionApi.post<BulkActionResult>(
    '/api/bookings/bulk-action',
    req,
  );
  return data;
}

export async function createManualBooking(
  req: CreateManualBookingRequest,
): Promise<BookingDetail> {
  const { data } = await transactionApi.post<BookingDetail>('/api/bookings/manual', req);
  return data;
}

export async function createRecurringBooking(
  req: CreateRecurringBookingRequest,
): Promise<{ created: BookingListItem[]; conflicts: string[] }> {
  const { data } = await transactionApi.post<{
    created: BookingListItem[];
    conflicts: string[];
  }>('/api/bookings/recurring', req);
  return data;
}

export async function exportCalendar(params?: {
  courtId?: string;
  from?: string;
  to?: string;
}): Promise<Blob> {
  const { data } = await transactionApi.get<Blob>('/api/bookings/calendar/ical', {
    params,
    responseType: 'blob',
  });
  return data;
}
```

#### Analytics Service (`src/services/analyticsService.ts`)

```typescript
import { platformApi } from '@/lib/api';

export interface AnalyticsParams {
  from: string;
  to: string;
  courtId?: string;
  courtType?: string;
}

export interface RevenueAnalytics {
  period: { from: string; to: string };
  totalBookings: number;
  confirmedBookings: number;
  cancelledBookings: number;
  noShows: number;
  revenueGrossCents: number;
  platformFeesCents: number;
  revenueNetCents: number;
  courtBreakdown: Array<{
    courtId: string;
    courtName: string;
    bookings: number;
    revenueCents: number;
    occupancyRate: number;
  }>;
  previousPeriodComparison: {
    bookingsChange: number;
    revenueChange: number;
  };
}

export interface UsageAnalytics {
  period: { from: string; to: string };
  occupancyRate: number;
  peakHours: Array<{
    dayOfWeek: string;
    hour: number;
    bookingCount: number;
  }>;
  courtBreakdown: Array<{
    courtId: string;
    courtName: string;
    bookings: number;
    occupancyRate: number;
  }>;
}

export interface HeatmapData {
  period: { from: string; to: string };
  heatmap: Array<{
    dayOfWeek: string;
    hours: Array<{
      hour: number;
      bookingCount: number;
      occupancyRate: number;
    }>;
  }>;
}

export async function getRevenue(params: AnalyticsParams): Promise<RevenueAnalytics> {
  const { data } = await platformApi.get<RevenueAnalytics>('/api/analytics/revenue', { params });
  return data;
}

export async function getUsage(params: AnalyticsParams): Promise<UsageAnalytics> {
  const { data } = await platformApi.get<UsageAnalytics>('/api/analytics/usage', { params });
  return data;
}

export async function getHeatmap(params: AnalyticsParams): Promise<HeatmapData> {
  const { data } = await platformApi.get<HeatmapData>('/api/analytics/heatmap', { params });
  return data;
}

export async function exportAnalytics(
  params: AnalyticsParams & { format: 'csv' | 'pdf' },
): Promise<Blob> {
  const { data } = await platformApi.get<Blob>('/api/analytics/export', {
    params,
    responseType: 'blob',
  });
  return data;
}
```

#### Support Service (`src/services/supportService.ts`)

```typescript
import { platformApi } from '@/lib/api';
import type { PaginatedResponse } from './bookingService';

export interface TicketListParams {
  page?: number;
  size?: number;
  status?: string;
  sort?: string;
}

export interface TicketSummary {
  ticketId: string;
  category: string;
  subject: string;
  status: string;
  priority: string;
  assignedTo: { name: string } | null;
  createdAt: string;
  updatedAt: string;
  messageCount: number;
}

export interface TicketDetail {
  ticketId: string;
  category: string;
  subject: string;
  status: string;
  priority: string;
  assignedTo: { id: string; name: string } | null;
  relatedBookingId: string | null;
  relatedCourtId: string | null;
  diagnosticData: Record<string, unknown> | null;
  messages: Array<{
    messageId: string;
    senderId: string;
    senderName: string;
    senderRole: string;
    body: string;
    attachments: Array<{
      attachmentId: string;
      fileName: string;
      fileSize: number;
      mimeType: string;
      url: string;
    }>;
    createdAt: string;
  }>;
  createdAt: string;
  updatedAt: string;
  resolvedAt: string | null;
}

export interface CreateTicketRequest {
  category: string;
  subject: string;
  body: string;
  relatedBookingId?: string;
  relatedCourtId?: string;
  diagnosticData?: Record<string, unknown>;
}

export async function getTickets(
  params?: TicketListParams,
): Promise<PaginatedResponse<TicketSummary>> {
  const { data } = await platformApi.get<PaginatedResponse<TicketSummary>>(
    '/api/support/tickets',
    { params },
  );
  return data;
}

export async function getTicket(id: string): Promise<TicketDetail> {
  const { data } = await platformApi.get<TicketDetail>(`/api/support/tickets/${id}`);
  return data;
}

export async function createTicket(
  req: CreateTicketRequest,
): Promise<{ ticketId: string; status: string; createdAt: string }> {
  const { data } = await platformApi.post('/api/support/tickets', req);
  return data;
}

export async function addMessage(
  ticketId: string,
  body: string,
  attachments?: File[],
): Promise<void> {
  if (attachments && attachments.length > 0) {
    const formData = new FormData();
    formData.append('body', body);
    attachments.forEach((f) => formData.append('attachments', f));
    await platformApi.post(`/api/support/tickets/${ticketId}/messages`, formData, {
      headers: { 'Content-Type': 'multipart/form-data' },
    });
  } else {
    await platformApi.post(`/api/support/tickets/${ticketId}/messages`, { body });
  }
}

export async function updateTicketStatus(
  ticketId: string,
  status: string,
  resolutionNotes?: string,
): Promise<void> {
  await platformApi.put(`/api/support/tickets/${ticketId}/status`, {
    status,
    resolutionNotes,
  });
}

export async function assignTicket(
  ticketId: string,
  assignedTo: string,
): Promise<void> {
  await platformApi.put(`/api/support/tickets/${ticketId}/assign`, { assignedTo });
}
```

#### Admin Service (`src/services/adminService.ts`)

```typescript
import { platformApi } from '@/lib/api';
import type { PaginatedResponse } from './bookingService';

export interface UserListParams {
  page?: number;
  size?: number;
  role?: string;
  status?: string;
  search?: string;
}

export interface UserSummary {
  id: string;
  name: string;
  email: string;
  role: string;
  status: string;
  createdAt: string;
}

export interface UserDetail extends UserSummary {
  phone: string;
  language: string;
  verified: boolean;
  stripeConnectStatus: string;
  courtCount: number;
  bookingCount: number;
  lastLoginAt: string | null;
}

export interface FeatureFlag {
  id: string;
  flagKey: string;
  description: string;
  enabled: boolean;
  updatedBy: string;
  updatedAt: string;
}

export interface DisputeSummary {
  id: string;
  bookingId: string;
  courtName: string;
  customerName: string;
  amount: number;
  status: string;
  createdAt: string;
}

export interface DisputeDetail extends DisputeSummary {
  reason: string;
  evidence: string | null;
  notes: Array<{ note: string; author: string; createdAt: string; visibleToParties: boolean }>;
  stripeDisputeId: string;
}

export interface VerificationQueueItem {
  id: string;
  userId: string;
  userName: string;
  businessName: string;
  submittedAt: string;
  documents: Array<{ url: string; type: string }>;
}

export interface SupportMetrics {
  totalTickets: number;
  openTickets: number;
  avgResponseTimeMinutes: number;
  avgResolutionTimeMinutes: number;
  ticketsByCategory: Array<{ category: string; count: number }>;
  ticketsByStatus: Array<{ status: string; count: number }>;
}

export interface PlatformAnalytics {
  period: { from: string; to: string };
  totalUsers: number;
  newUsersInPeriod: number;
  totalCourtOwners: number;
  totalCourts: number;
  totalBookings: number;
  totalRevenueCents: number;
  totalPlatformFeesCents: number;
  activeUsers: number;
  bookingsByCourtType: Array<{ courtType: string; count: number }>;
  topCourts: Array<{ courtId: string; courtName: string; bookings: number; revenueCents: number }>;
  userGrowth: Array<{ date: string; newUsers: number; cumulativeUsers: number }>;
}

// User management
export async function getUsers(params?: UserListParams): Promise<PaginatedResponse<UserSummary>> {
  const { data } = await platformApi.get('/api/admin/users', { params });
  return data;
}

export async function getUserDetail(id: string): Promise<UserDetail> {
  const { data } = await platformApi.get(`/api/admin/users/${id}`);
  return data;
}

export async function suspendUser(id: string, reason: string): Promise<void> {
  await platformApi.post(`/api/admin/users/${id}/suspend`, { reason });
}

export async function unsuspendUser(id: string): Promise<void> {
  await platformApi.post(`/api/admin/users/${id}/unsuspend`);
}

// Verification queue
export async function getVerificationQueue(
  params?: { page?: number; size?: number },
): Promise<PaginatedResponse<VerificationQueueItem>> {
  const { data } = await platformApi.get('/api/verification/queue', { params });
  return data;
}

export async function approveVerification(id: string): Promise<void> {
  await platformApi.post(`/api/verification/${id}/approve`);
}

export async function rejectVerification(id: string, reason: string): Promise<void> {
  await platformApi.post(`/api/verification/${id}/reject`, { reason });
}

// Feature flags
export async function getFeatureFlags(): Promise<FeatureFlag[]> {
  const { data } = await platformApi.get<FeatureFlag[]>('/api/admin/feature-flags');
  return data;
}

export async function createFeatureFlag(
  req: { flagKey: string; description: string; enabled: boolean },
): Promise<FeatureFlag> {
  const { data } = await platformApi.post<FeatureFlag>('/api/admin/feature-flags', req);
  return data;
}

export async function updateFeatureFlag(
  key: string,
  req: { enabled: boolean },
): Promise<FeatureFlag> {
  const { data } = await platformApi.put<FeatureFlag>(`/api/admin/feature-flags/${key}`, req);
  return data;
}

export async function deleteFeatureFlag(key: string): Promise<void> {
  await platformApi.delete(`/api/admin/feature-flags/${key}`);
}

// Disputes
export async function getDisputes(
  params?: { page?: number; size?: number; status?: string },
): Promise<PaginatedResponse<DisputeSummary>> {
  const { data } = await platformApi.get('/api/admin/disputes', { params });
  return data;
}

export async function getDisputeDetail(id: string): Promise<DisputeDetail> {
  const { data } = await platformApi.get(`/api/admin/disputes/${id}`);
  return data;
}

export async function addDisputeNote(
  id: string,
  note: string,
  visibleToParties: boolean,
): Promise<void> {
  await platformApi.post(`/api/admin/disputes/${id}/notes`, { note, visibleToParties });
}

// Platform analytics
export async function getPlatformAnalytics(
  params: { from: string; to: string },
): Promise<PlatformAnalytics> {
  const { data } = await platformApi.get('/api/admin/analytics', { params });
  return data;
}

// Support metrics
export async function getSupportMetrics(
  params: { from: string; to: string },
): Promise<SupportMetrics> {
  const { data } = await platformApi.get('/api/admin/support/metrics', { params });
  return data;
}
```

#### Settings Service (`src/services/settingsService.ts`)

```typescript
import { platformApi } from '@/lib/api';

export interface NotificationPreference {
  eventType: string;
  email: boolean;
  push: boolean;
  inApp: boolean;
}

export interface ReminderRule {
  id: string;
  ruleType: string;
  description: string;
  enabled: boolean;
  courtIds: string[] | null;  // null = all courts
  config: Record<string, unknown>;
  createdAt: string;
}

export interface CourtDefaults {
  defaultDurationMinutes: number;
  confirmationMode: 'INSTANT' | 'MANUAL';
  defaultCancellationTiers: Array<{ hoursBeforeBooking: number; refundPercentage: number }>;
  defaultAmenities: string[];
}

export interface SessionInfo {
  sessionId: string;
  deviceType: string;
  browser: string;
  ipAddress: string;
  lastActivity: string;
  current: boolean;
}

// Profile
export async function updateProfile(
  data: { name?: string; phone?: string; language?: string },
): Promise<void> {
  await platformApi.put('/api/users/me', data);
}

export async function uploadProfilePhoto(file: File): Promise<{ profileImageUrl: string }> {
  const formData = new FormData();
  formData.append('photo', file);
  const { data } = await platformApi.post('/api/users/me/profile-photo', formData, {
    headers: { 'Content-Type': 'multipart/form-data' },
  });
  return data;
}

export async function deleteProfilePhoto(): Promise<void> {
  await platformApi.delete('/api/users/me/profile-photo');
}

// Notification preferences
export async function getNotificationPreferences(): Promise<NotificationPreference[]> {
  const { data } = await platformApi.get<NotificationPreference[]>(
    '/api/settings/notification-preferences',
  );
  return data;
}

export async function updateNotificationPreferences(
  prefs: NotificationPreference[],
): Promise<void> {
  await platformApi.put('/api/settings/notification-preferences', prefs);
}

// Reminder rules
export async function getReminderRules(): Promise<ReminderRule[]> {
  const { data } = await platformApi.get<ReminderRule[]>('/api/settings/reminder-rules');
  return data;
}

export async function createReminderRule(
  rule: Omit<ReminderRule, 'id' | 'createdAt'>,
): Promise<ReminderRule> {
  const { data } = await platformApi.post<ReminderRule>('/api/settings/reminder-rules', rule);
  return data;
}

export async function updateReminderRule(
  id: string,
  rule: Partial<Omit<ReminderRule, 'id' | 'createdAt'>>,
): Promise<ReminderRule> {
  const { data } = await platformApi.put<ReminderRule>(
    `/api/settings/reminder-rules/${id}`,
    rule,
  );
  return data;
}

export async function deleteReminderRule(id: string): Promise<void> {
  await platformApi.delete(`/api/settings/reminder-rules/${id}`);
}

// Court defaults
export async function getCourtDefaults(): Promise<CourtDefaults> {
  const { data } = await platformApi.get<CourtDefaults>('/api/settings/court-defaults');
  return data;
}

export async function updateCourtDefaults(defaults: CourtDefaults): Promise<void> {
  await platformApi.put('/api/settings/court-defaults', defaults);
}

// Sessions
export async function getSessions(): Promise<SessionInfo[]> {
  const { data } = await platformApi.get<SessionInfo[]>('/api/users/me/sessions');
  return data;
}

export async function revokeSession(sessionId: string): Promise<void> {
  await platformApi.delete(`/api/users/me/sessions/${sessionId}`);
}

// GDPR export
export async function requestDataExport(): Promise<{ exportId: string; status: string }> {
  const { data } = await platformApi.post('/api/users/me/data-export');
  return data;
}
```

#### Remaining Services

```typescript
// src/services/searchService.ts
import { platformApi } from '@/lib/api';

export interface SearchResult {
  results: Array<{
    type: 'COURT' | 'BOOKING' | 'PROMO_CODE';
    id: string;
    title: string;
    subtitle: string;
  }>;
  totalElements: number;
}

export async function globalSearch(params: {
  q: string;
  type?: string;
  page?: number;
  size?: number;
}): Promise<SearchResult> {
  const { data } = await platformApi.get<SearchResult>('/api/search', { params });
  return data;
}

export async function adminSearch(params: {
  q: string;
  type?: string;
  page?: number;
  size?: number;
}): Promise<SearchResult> {
  const { data } = await platformApi.get<SearchResult>('/api/admin/search', { params });
  return data;
}
```

```typescript
// src/services/authService.ts
import { platformApi } from '@/lib/api';

export async function fetchCsrfToken(): Promise<{ csrfToken: string }> {
  const { data } = await platformApi.get<{ csrfToken: string }>('/api/auth/csrf-token');
  return data;
}
```

```typescript
// src/services/stripeService.ts
import { transactionApi } from '@/lib/api';

export interface PayoutInfo {
  balance: { availableCents: number; pendingCents: number };
  schedule: { interval: string; weeklyAnchor?: string; monthlyAnchor?: number };
  payouts: Array<{
    id: string;
    amountCents: number;
    status: string;
    arrivalDate: string;
    createdAt: string;
  }>;
}

export async function startOnboarding(): Promise<{ onboardingUrl: string }> {
  const { data } = await transactionApi.post('/api/payments/stripe-connect/onboard');
  return data;
}

export async function getPayouts(params?: {
  page?: number;
  size?: number;
}): Promise<PayoutInfo> {
  const { data } = await transactionApi.get<PayoutInfo>(
    '/api/payments/stripe-connect/payouts',
    { params },
  );
  return data;
}
```

```typescript
// src/services/notificationService.ts
import { transactionApi } from '@/lib/api';

export interface NotificationItem {
  id: string;
  type: string;
  title: string;
  body: string;
  createdAt: string;
  read: boolean;
  data: Record<string, string> | null;
}

export async function getNotifications(params?: {
  page?: number;
  size?: number;
  unreadOnly?: boolean;
}): Promise<{ content: NotificationItem[]; totalElements: number }> {
  const { data } = await transactionApi.get('/api/notifications', { params });
  return data;
}

export async function getUnreadCount(): Promise<number> {
  const { data } = await transactionApi.get('/api/notifications', {
    params: { unreadOnly: true, size: 0 },
  });
  return data.totalElements;
}

export async function markAsRead(notificationId: string): Promise<void> {
  await transactionApi.post(`/api/notifications/${notificationId}/read`);
}

export async function markAllAsRead(): Promise<void> {
  await transactionApi.post('/api/notifications/read-all');
}
```

```typescript
// src/services/verificationService.ts
import { platformApi } from '@/lib/api';

export async function getVerificationStatus(): Promise<{
  status: string;
  submittedAt: string | null;
  reviewedAt: string | null;
  rejectionReason: string | null;
}> {
  const { data } = await platformApi.get('/api/verification');
  return data;
}

export async function submitVerification(formData: FormData): Promise<void> {
  await platformApi.post('/api/verification', formData, {
    headers: { 'Content-Type': 'multipart/form-data' },
  });
}
```


### TanStack Query Hooks — Detailed Signatures

#### Query Key Additions (`src/lib/queryKeys.ts`)

Add these to the existing file:

```typescript
export const notificationKeys = {
  all: ['notifications'] as const,
  unreadCount: () => [...notificationKeys.all, 'unread-count'] as const,
  list: (params: Record<string, unknown>) =>
    [...notificationKeys.all, 'list', params] as const,
};

export const auditLogKeys = {
  all: ['audit-logs'] as const,
  list: (filters: Record<string, unknown>) =>
    [...auditLogKeys.all, 'list', filters] as const,
};

export const holidayKeys = {
  national: () => ['holidays', 'national'] as const,
  custom: () => ['holidays', 'custom'] as const,
  calendar: (params: Record<string, unknown>) =>
    ['holidays', 'calendar', params] as const,
};

export const verificationKeys = {
  status: () => ['verification', 'status'] as const,
  queue: (params: Record<string, unknown>) =>
    ['verification', 'queue', params] as const,
};

export const stripeKeys = {
  payouts: (params: Record<string, unknown>) =>
    ['stripe', 'payouts', params] as const,
};
```

#### Hook: `useDashboard.ts`

```typescript
import { useQuery } from '@tanstack/react-query';
import { dashboardKeys } from '@/lib/queryKeys';
import * as dashboardService from '@/services/dashboardService';

export function useDashboard() {
  return useQuery({
    queryKey: dashboardKeys.summary(),
    queryFn: dashboardService.getDashboard,
    staleTime: 60_000,       // 1 min — matches backend Redis TTL
    refetchInterval: 60_000, // Auto-refresh every minute
  });
}
```

#### Hook: `useCourts.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { courtKeys, dashboardKeys } from '@/lib/queryKeys';
import * as courtService from '@/services/courtService';
import type { CourtListParams, CourtSummary } from '@/services/courtService';

export function useOwnerCourts(filters?: CourtListParams) {
  return useQuery({
    queryKey: courtKeys.list(filters ?? {}),
    queryFn: () => courtService.getOwnerCourts(filters),
  });
}

export function useCourtDetail(id: string | undefined) {
  return useQuery({
    queryKey: courtKeys.detail(id!),
    queryFn: () => courtService.getCourt(id!),
    enabled: !!id,
  });
}

export function useCreateCourt() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: courtService.createCourt,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: courtKeys.lists() });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}

export function useUpdateCourt() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: courtService.UpdateCourtRequest }) =>
      courtService.updateCourt(id, data),
    onSuccess: (_, { id }) => {
      qc.invalidateQueries({ queryKey: courtKeys.lists() });
      qc.invalidateQueries({ queryKey: courtKeys.detail(id) });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}

export function useDeleteCourt() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: courtService.deleteCourt,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: courtKeys.lists() });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}

// Optimistic update for visibility toggle
export function useToggleCourtVisibility() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, visible }: { id: string; visible: boolean }) =>
      courtService.updateCourt(id, { visible } as courtService.UpdateCourtRequest),
    onMutate: async ({ id, visible }) => {
      await qc.cancelQueries({ queryKey: courtKeys.lists() });
      const previousLists = qc.getQueriesData({ queryKey: courtKeys.lists() });
      // Optimistically update all court list caches
      qc.setQueriesData({ queryKey: courtKeys.lists() }, (old: CourtSummary[] | undefined) =>
        old?.map((c) => (c.id === id ? { ...c, visible } : c)),
      );
      return { previousLists };
    },
    onError: (_err, _vars, context) => {
      // Roll back all court list caches
      context?.previousLists.forEach(([key, data]) => {
        qc.setQueryData(key, data);
      });
    },
    onSettled: () => {
      qc.invalidateQueries({ queryKey: courtKeys.lists() });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}

export function useBulkToggleVisibility() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ courtIds, visible }: { courtIds: string[]; visible: boolean }) =>
      courtService.bulkToggleVisibility(courtIds, visible),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: courtKeys.lists() });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}
```

#### Hook: `useBookings.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { bookingKeys, dashboardKeys } from '@/lib/queryKeys';
import * as bookingService from '@/services/bookingService';

export function useBookingList(filters?: bookingService.BookingListParams) {
  return useQuery({
    queryKey: bookingKeys.list(filters ?? {}),
    queryFn: () => bookingService.getBookings(filters),
  });
}

export function usePendingBookings(params?: { page?: number; size?: number }) {
  return useQuery({
    queryKey: bookingKeys.pending(),
    queryFn: () => bookingService.getPendingBookings(params),
  });
}

export function useBookingDetail(id: string | undefined) {
  return useQuery({
    queryKey: bookingKeys.detail(id!),
    queryFn: () => bookingService.getBooking(id!),
    enabled: !!id,
  });
}

// Shared invalidation for booking actions
function useBookingActionMutation<TArgs>(
  mutationFn: (args: TArgs) => Promise<unknown>,
) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: bookingKeys.lists() });
      qc.invalidateQueries({ queryKey: bookingKeys.pending() });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}

export function useConfirmBooking() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: bookingService.confirmBooking,
    onSuccess: (_data, id) => {
      qc.invalidateQueries({ queryKey: bookingKeys.detail(id) });
      qc.invalidateQueries({ queryKey: bookingKeys.lists() });
      qc.invalidateQueries({ queryKey: bookingKeys.pending() });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}

export function useRejectBooking() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, reason }: { id: string; reason: string }) =>
      bookingService.rejectBooking(id, reason),
    onSuccess: (_data, { id }) => {
      qc.invalidateQueries({ queryKey: bookingKeys.detail(id) });
      qc.invalidateQueries({ queryKey: bookingKeys.lists() });
      qc.invalidateQueries({ queryKey: bookingKeys.pending() });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}

export function useCancelBooking() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, reason }: { id: string; reason: string }) =>
      bookingService.cancelBooking(id, reason),
    onSuccess: (_data, { id }) => {
      qc.invalidateQueries({ queryKey: bookingKeys.detail(id) });
      qc.invalidateQueries({ queryKey: bookingKeys.lists() });
      qc.invalidateQueries({ queryKey: bookingKeys.pending() });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}

export function useMarkPaid() {
  return useBookingActionMutation(bookingService.markPaid);
}

export function useFlagNoShow() {
  return useBookingActionMutation(bookingService.flagNoShow);
}

export function useModifyBooking() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Parameters<typeof bookingService.modifyBooking>[1] }) =>
      bookingService.modifyBooking(id, data),
    onSuccess: (_data, { id }) => {
      qc.invalidateQueries({ queryKey: bookingKeys.detail(id) });
      qc.invalidateQueries({ queryKey: bookingKeys.lists() });
      qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
    },
  });
}

export function useBulkBookingAction() {
  return useBookingActionMutation(bookingService.bulkAction);
}

export function useCreateManualBooking() {
  return useBookingActionMutation(bookingService.createManualBooking);
}

export function useCreateRecurringBooking() {
  return useBookingActionMutation(bookingService.createRecurringBooking);
}
```

#### Hook: `useAnalytics.ts`

```typescript
import { useQuery } from '@tanstack/react-query';
import { analyticsKeys } from '@/lib/queryKeys';
import * as analyticsService from '@/services/analyticsService';
import type { AnalyticsParams } from '@/services/analyticsService';

export function useRevenue(params: AnalyticsParams) {
  return useQuery({
    queryKey: analyticsKeys.revenue(params),
    queryFn: () => analyticsService.getRevenue(params),
    enabled: !!params.from && !!params.to,
  });
}

export function useUsage(params: AnalyticsParams) {
  return useQuery({
    queryKey: analyticsKeys.usage(params),
    queryFn: () => analyticsService.getUsage(params),
    enabled: !!params.from && !!params.to,
  });
}

export function useHeatmap(params: AnalyticsParams) {
  return useQuery({
    queryKey: analyticsKeys.heatmap(params),
    queryFn: () => analyticsService.getHeatmap(params),
    enabled: !!params.from && !!params.to,
  });
}
```

#### Hook: `useAdmin.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { adminKeys } from '@/lib/queryKeys';
import * as adminService from '@/services/adminService';
import type { FeatureFlag } from '@/services/adminService';

export function useUserList(params?: adminService.UserListParams) {
  return useQuery({
    queryKey: adminKeys.users(),
    queryFn: () => adminService.getUsers(params),
  });
}

export function useUserDetail(id: string | undefined) {
  return useQuery({
    queryKey: adminKeys.userDetail(id!),
    queryFn: () => adminService.getUserDetail(id!),
    enabled: !!id,
  });
}

export function useSuspendUser() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, reason }: { id: string; reason: string }) =>
      adminService.suspendUser(id, reason),
    onSuccess: (_data, { id }) => {
      qc.invalidateQueries({ queryKey: adminKeys.users() });
      qc.invalidateQueries({ queryKey: adminKeys.userDetail(id) });
    },
  });
}

export function useUnsuspendUser() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => adminService.unsuspendUser(id),
    onSuccess: (_data, id) => {
      qc.invalidateQueries({ queryKey: adminKeys.users() });
      qc.invalidateQueries({ queryKey: adminKeys.userDetail(id) });
    },
  });
}

export function useFeatureFlags() {
  return useQuery({
    queryKey: adminKeys.featureFlags(),
    queryFn: adminService.getFeatureFlags,
  });
}

// Optimistic update for feature flag toggle
export function useUpdateFeatureFlag() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ key, enabled }: { key: string; enabled: boolean }) =>
      adminService.updateFeatureFlag(key, { enabled }),
    onMutate: async ({ key, enabled }) => {
      await qc.cancelQueries({ queryKey: adminKeys.featureFlags() });
      const previous = qc.getQueryData<FeatureFlag[]>(adminKeys.featureFlags());
      qc.setQueryData<FeatureFlag[]>(adminKeys.featureFlags(), (old) =>
        old?.map((f) => (f.flagKey === key ? { ...f, enabled } : f)),
      );
      return { previous };
    },
    onError: (_err, _vars, context) => {
      qc.setQueryData(adminKeys.featureFlags(), context?.previous);
    },
    onSettled: () => {
      qc.invalidateQueries({ queryKey: adminKeys.featureFlags() });
    },
  });
}

export function useCreateFeatureFlag() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: adminService.createFeatureFlag,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: adminKeys.featureFlags() });
    },
  });
}

export function useDeleteFeatureFlag() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: adminService.deleteFeatureFlag,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: adminKeys.featureFlags() });
    },
  });
}

export function useDisputes(params?: { page?: number; size?: number; status?: string }) {
  return useQuery({
    queryKey: adminKeys.disputes(),
    queryFn: () => adminService.getDisputes(params),
  });
}

export function useDisputeDetail(id: string | undefined) {
  return useQuery({
    queryKey: adminKeys.disputeDetail(id!),
    queryFn: () => adminService.getDisputeDetail(id!),
    enabled: !!id,
  });
}

export function useAddDisputeNote() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, note, visibleToParties }: { id: string; note: string; visibleToParties: boolean }) =>
      adminService.addDisputeNote(id, note, visibleToParties),
    onSuccess: (_data, { id }) => {
      qc.invalidateQueries({ queryKey: adminKeys.disputeDetail(id) });
    },
  });
}

export function usePlatformAnalytics(params: { from: string; to: string }) {
  return useQuery({
    queryKey: adminKeys.platformAnalytics(params),
    queryFn: () => adminService.getPlatformAnalytics(params),
    enabled: !!params.from && !!params.to,
  });
}

export function useSupportMetrics(params: { from: string; to: string }) {
  return useQuery({
    queryKey: [...supportKeys.all, 'metrics', params] as const,
    queryFn: () => adminService.getSupportMetrics(params),
    enabled: !!params.from && !!params.to,
  });
}
```

#### Hook: `useSupport.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { supportKeys } from '@/lib/queryKeys';
import * as supportService from '@/services/supportService';

export function useTicketList(filters?: supportService.TicketListParams) {
  return useQuery({
    queryKey: supportKeys.list(filters ?? {}),
    queryFn: () => supportService.getTickets(filters),
  });
}

export function useTicketDetail(id: string | undefined) {
  return useQuery({
    queryKey: supportKeys.detail(id!),
    queryFn: () => supportService.getTicket(id!),
    enabled: !!id,
  });
}

export function useCreateTicket() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: supportService.createTicket,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: supportKeys.lists() });
    },
  });
}

export function useAddMessage() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ ticketId, body, attachments }: { ticketId: string; body: string; attachments?: File[] }) =>
      supportService.addMessage(ticketId, body, attachments),
    onSuccess: (_data, { ticketId }) => {
      qc.invalidateQueries({ queryKey: supportKeys.detail(ticketId) });
    },
  });
}

export function useUpdateTicketStatus() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ ticketId, status, resolutionNotes }: { ticketId: string; status: string; resolutionNotes?: string }) =>
      supportService.updateTicketStatus(ticketId, status, resolutionNotes),
    onSuccess: (_data, { ticketId }) => {
      qc.invalidateQueries({ queryKey: supportKeys.lists() });
      qc.invalidateQueries({ queryKey: supportKeys.detail(ticketId) });
    },
  });
}

export function useAssignTicket() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ ticketId, assignedTo }: { ticketId: string; assignedTo: string }) =>
      supportService.assignTicket(ticketId, assignedTo),
    onSuccess: (_data, { ticketId }) => {
      qc.invalidateQueries({ queryKey: supportKeys.lists() });
      qc.invalidateQueries({ queryKey: supportKeys.detail(ticketId) });
    },
  });
}
```

#### Hook: `useSettings.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { settingsKeys } from '@/lib/queryKeys';
import * as settingsService from '@/services/settingsService';

export function useNotificationPreferences() {
  return useQuery({
    queryKey: settingsKeys.notificationPrefs(),
    queryFn: settingsService.getNotificationPreferences,
  });
}

export function useUpdateNotificationPreferences() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: settingsService.updateNotificationPreferences,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: settingsKeys.notificationPrefs() });
    },
  });
}

export function useReminderRules() {
  return useQuery({
    queryKey: settingsKeys.reminderRules(),
    queryFn: settingsService.getReminderRules,
  });
}

export function useCreateReminderRule() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: settingsService.createReminderRule,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: settingsKeys.reminderRules() });
    },
  });
}

export function useUpdateReminderRule() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Parameters<typeof settingsService.updateReminderRule>[1] }) =>
      settingsService.updateReminderRule(id, data),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: settingsKeys.reminderRules() });
    },
  });
}

export function useDeleteReminderRule() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: settingsService.deleteReminderRule,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: settingsKeys.reminderRules() });
    },
  });
}

export function useCourtDefaults() {
  return useQuery({
    queryKey: settingsKeys.courtDefaults(),
    queryFn: settingsService.getCourtDefaults,
  });
}

export function useUpdateCourtDefaults() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: settingsService.updateCourtDefaults,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: settingsKeys.courtDefaults() });
    },
  });
}

export function useSessions() {
  return useQuery({
    queryKey: settingsKeys.sessions(),
    queryFn: settingsService.getSessions,
  });
}

export function useRevokeSession() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: settingsService.revokeSession,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: settingsKeys.sessions() });
    },
  });
}

export function useUpdateProfile() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: settingsService.updateProfile,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: settingsKeys.profile() });
    },
  });
}
```

#### Hook: `useNotifications.ts`

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { notificationKeys } from '@/lib/queryKeys';
import * as notificationService from '@/services/notificationService';

export function useUnreadCount() {
  return useQuery({
    queryKey: notificationKeys.unreadCount(),
    queryFn: notificationService.getUnreadCount,
    refetchInterval: 30_000, // Poll every 30s as fallback to WebSocket
  });
}

export function useNotificationList(params?: { page?: number; size?: number }) {
  return useQuery({
    queryKey: notificationKeys.list(params ?? {}),
    queryFn: () => notificationService.getNotifications(params),
  });
}

export function useMarkAsRead() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: notificationService.markAsRead,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: notificationKeys.all });
    },
  });
}

export function useMarkAllAsRead() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: notificationService.markAllAsRead,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: notificationKeys.all });
    },
  });
}
```


### WebSocket → Cache Synchronization (`src/hooks/useWebSocketSync.ts`)

This hook subscribes to STOMP topics and invalidates TanStack Query caches when real-time events arrive. It should be mounted once in `AppLayout`.

```typescript
import { useEffect } from 'react';
import { useQueryClient } from '@tanstack/react-query';
import { useWebSocket } from '@/hooks/useWebSocket';
import { useAuthStore } from '@/stores/authStore';
import {
  bookingKeys,
  dashboardKeys,
  supportKeys,
  notificationKeys,
  courtKeys,
} from '@/lib/queryKeys';

interface WebSocketEvent {
  eventType: string;
  payload: Record<string, string>;
}

export function useWebSocketSync() {
  const qc = useQueryClient();
  const user = useAuthStore((s) => s.user);

  // Subscribe to personal notification channel
  const { isConnected } = useWebSocket(
    '/user/queue/notifications',
    (message) => {
      const event: WebSocketEvent = JSON.parse(message.body);

      switch (event.eventType) {
        case 'BOOKING_CREATED':
        case 'BOOKING_UPDATED':
          qc.invalidateQueries({ queryKey: bookingKeys.lists() });
          qc.invalidateQueries({ queryKey: bookingKeys.pending() });
          qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
          break;

        case 'BOOKING_CONFIRMED':
        case 'BOOKING_CANCELLED':
          qc.invalidateQueries({ queryKey: bookingKeys.lists() });
          qc.invalidateQueries({ queryKey: bookingKeys.pending() });
          qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
          if (event.payload.bookingId) {
            qc.invalidateQueries({
              queryKey: bookingKeys.detail(event.payload.bookingId),
            });
          }
          break;

        case 'NOTIFICATION_RECEIVED':
          qc.invalidateQueries({ queryKey: notificationKeys.all });
          qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
          break;

        case 'SUPPORT_TICKET_UPDATED':
          qc.invalidateQueries({ queryKey: supportKeys.lists() });
          if (event.payload.ticketId) {
            qc.invalidateQueries({
              queryKey: supportKeys.detail(event.payload.ticketId),
            });
          }
          break;

        case 'COURT_UPDATED':
          qc.invalidateQueries({ queryKey: courtKeys.lists() });
          if (event.payload.courtId) {
            qc.invalidateQueries({
              queryKey: courtKeys.detail(event.payload.courtId),
            });
          }
          qc.invalidateQueries({ queryKey: dashboardKeys.summary() });
          break;
      }
    },
  );

  // On reconnect after disconnection, invalidate everything to catch up
  useEffect(() => {
    if (isConnected) {
      // This fires on initial connect AND reconnect
      // On reconnect, stale data may exist — invalidate all active queries
      qc.invalidateQueries();
    }
  }, [isConnected, qc]);

  return { isConnected };
}
```

**Event Type → Cache Invalidation Map:**

| WebSocket Event | Query Keys Invalidated |
|----------------|----------------------|
| `BOOKING_CREATED` | `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `BOOKING_UPDATED` | `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `BOOKING_CONFIRMED` | `bookingKeys.detail(id)`, `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `BOOKING_CANCELLED` | `bookingKeys.detail(id)`, `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `NOTIFICATION_RECEIVED` | `notificationKeys.all`, `dashboardKeys.summary()` |
| `SUPPORT_TICKET_UPDATED` | `supportKeys.lists()`, `supportKeys.detail(id)` |
| `COURT_UPDATED` | `courtKeys.lists()`, `courtKeys.detail(id)`, `dashboardKeys.summary()` |
| Reconnect (any) | All active queries (full invalidation) |

### QueryStateWrapper Component (`src/components/feedback/QueryStateWrapper.tsx`)

```typescript
import { ReactNode } from 'react';
import { Alert, Button, Empty, Skeleton } from 'antd';
import type { UseQueryResult } from '@tanstack/react-query';

interface QueryStateWrapperProps<T> {
  query: UseQueryResult<T>;
  skeletonType?: 'table' | 'card' | 'form';
  skeletonRows?: number;
  emptyMessage?: string;
  emptyAction?: ReactNode;
  isEmpty?: (data: T) => boolean;
  children: (data: T) => ReactNode;
}

export function QueryStateWrapper<T>({
  query,
  skeletonType = 'card',
  skeletonRows = 5,
  emptyMessage = 'No data found',
  emptyAction,
  isEmpty,
  children,
}: QueryStateWrapperProps<T>) {
  if (query.isLoading) {
    return (
      <Skeleton
        active
        paragraph={{ rows: skeletonRows }}
        title={skeletonType === 'table'}
      />
    );
  }

  if (query.isError) {
    const error = query.error as { message?: string } | undefined;
    return (
      <Alert
        type="error"
        showIcon
        message="Something went wrong"
        description={error?.message ?? 'An unexpected error occurred'}
        action={
          <Button size="small" onClick={() => query.refetch()}>
            Retry
          </Button>
        }
      />
    );
  }

  const data = query.data;
  if (data == null) return null;

  const dataIsEmpty = isEmpty
    ? isEmpty(data)
    : Array.isArray(data)
      ? data.length === 0
      : typeof data === 'object' && 'content' in (data as Record<string, unknown>)
        ? (data as { content: unknown[] }).content.length === 0
        : false;

  if (dataIsEmpty) {
    return <Empty description={emptyMessage}>{emptyAction}</Empty>;
  }

  return <>{children(data)}</>;
}
```

### Cache Invalidation Map (Complete)

| Mutation | Invalidated Query Keys |
|----------|----------------------|
| `useCreateCourt` | `courtKeys.lists()`, `dashboardKeys.summary()` |
| `useUpdateCourt` | `courtKeys.lists()`, `courtKeys.detail(id)`, `dashboardKeys.summary()` |
| `useDeleteCourt` | `courtKeys.lists()`, `dashboardKeys.summary()` |
| `useToggleCourtVisibility` | `courtKeys.lists()` (optimistic), `dashboardKeys.summary()` |
| `useBulkToggleVisibility` | `courtKeys.lists()`, `dashboardKeys.summary()` |
| `useConfirmBooking` | `bookingKeys.detail(id)`, `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `useRejectBooking` | `bookingKeys.detail(id)`, `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `useCancelBooking` | `bookingKeys.detail(id)`, `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `useMarkPaid` | `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `useFlagNoShow` | `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `useModifyBooking` | `bookingKeys.detail(id)`, `bookingKeys.lists()`, `dashboardKeys.summary()` |
| `useBulkBookingAction` | `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `useCreateManualBooking` | `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `useCreateRecurringBooking` | `bookingKeys.lists()`, `bookingKeys.pending()`, `dashboardKeys.summary()` |
| `useCreateTicket` | `supportKeys.lists()` |
| `useAddMessage` | `supportKeys.detail(ticketId)` |
| `useUpdateTicketStatus` | `supportKeys.lists()`, `supportKeys.detail(ticketId)` |
| `useAssignTicket` | `supportKeys.lists()`, `supportKeys.detail(ticketId)` |
| `useUpdateProfile` | `settingsKeys.profile()` |
| `useUpdateNotificationPreferences` | `settingsKeys.notificationPrefs()` |
| `useCreateReminderRule` | `settingsKeys.reminderRules()` |
| `useUpdateReminderRule` | `settingsKeys.reminderRules()` |
| `useDeleteReminderRule` | `settingsKeys.reminderRules()` |
| `useUpdateCourtDefaults` | `settingsKeys.courtDefaults()` |
| `useRevokeSession` | `settingsKeys.sessions()` |
| `useUpdateFeatureFlag` | `adminKeys.featureFlags()` (optimistic) |
| `useCreateFeatureFlag` | `adminKeys.featureFlags()` |
| `useDeleteFeatureFlag` | `adminKeys.featureFlags()` |
| `useSuspendUser` | `adminKeys.users()`, `adminKeys.userDetail(id)` |
| `useUnsuspendUser` | `adminKeys.users()`, `adminKeys.userDetail(id)` |
| `useMarkAsRead` | `notificationKeys.all` |
| `useMarkAllAsRead` | `notificationKeys.all` |

### Page Rewiring Instructions

For each page, the pattern is:

1. Remove `const mockXxx` constants and `useState(mockXxx)` patterns
2. Import the appropriate hook
3. Call the hook, destructure `{ data, isLoading, isError, error, refetch }`
4. Wrap content in `QueryStateWrapper` or use the destructured values directly
5. Replace local `setCourts`/`setFlags` state mutations with mutation hook calls
6. Add `notification.success()` / `notification.error()` on mutation results

#### DashboardPage

**Remove:** `mockStats`, `mockPayout`, `mockNeedsAttention`, `mockNotifications`, `mockCourtSummary`, `mockUserState`

**Add:**
```typescript
import { useDashboard } from '@/hooks/useDashboard';
import { QueryStateWrapper } from '@/components/feedback/QueryStateWrapper';

const dashboard = useDashboard();
```

**Wire:**
- `mockStats.todayBookings` → `dashboard.data.todayBookings`
- `mockStats.pendingConfirmations` → `dashboard.data.pendingConfirmations`
- `mockStats.revenueThisMonth` → `dashboard.data.revenueThisMonthCents`
- `mockStats.occupancyToday` → `Math.round(dashboard.data.occupancyRateToday * 100)`
- `mockPayout.nextAmount` → `dashboard.data.nextPayoutAmountCents`
- `mockPayout.scheduledFor` → `dashboard.data.nextPayoutDate` (format with dayjs)
- `mockNeedsAttention` → derive from `dashboard.data.actionRequired` counts
- `mockNotifications` → `dashboard.data.recentNotifications`
- `mockCourtSummary` → `dashboard.data.courtSummary`
- `mockUserState.stripeStatus` → `dashboard.data.stripeConnectStatus`
- `mockUserState.isVerified` → `dashboard.data.verificationStatus === 'APPROVED'`
- Wrap stat cards row in loading skeleton when `dashboard.isLoading`
- Show error alert with retry when `dashboard.isError`

#### CourtListPage

**Remove:** `mockCourts`, `useState(mockCourts)`, local `handleVisibilityToggle`, local `handleDelete`

**Add:**
```typescript
import { useOwnerCourts, useToggleCourtVisibility, useDeleteCourt } from '@/hooks/useCourts';

const filters = { type: typeFilter, locationType: locationFilter, search };
const { data: courts, isLoading, isError, error, refetch } = useOwnerCourts(filters);
const toggleVisibility = useToggleCourtVisibility();
const deleteCourt = useDeleteCourt();
```

**Wire:**
- `dataSource={filtered}` → `dataSource={courts ?? []}`
- Remove client-side filtering (server handles it via `filters` param)
- `handleVisibilityToggle(id, checked)` → `toggleVisibility.mutate({ id, visible: checked })`
- `handleDelete(id)` → `deleteCourt.mutate(id, { onSuccess: () => message.success('Court deleted') })`
- `{courts.length} courts` → `{courts?.length ?? 0} courts`
- Bulk "Toggle Visibility" button → `useBulkToggleVisibility().mutate({ courtIds: selectedRowKeys, visible: true })`

#### BookingListPage

**Remove:** `mockBookings`

**Add:**
```typescript
import { useBookingList } from '@/hooks/useBookings';

const [filters, setFilters] = useState<BookingListParams>({});
const { data, isLoading } = useBookingList(filters);
```

**Wire:**
- `dataSource={mockBookings}` → `dataSource={data?.content ?? []}`
- `pagination={{ pageSize: 10 }}` → `pagination={{ current: (data?.page ?? 0) + 1, pageSize: data?.size ?? 10, total: data?.totalElements ?? 0, onChange: (p, s) => setFilters(prev => ({ ...prev, page: p - 1, size: s })) }}`
- Filter bar `onChange` handlers → update `filters` state which triggers refetch
- RangePicker → `setFilters(prev => ({ ...prev, from: dates[0], to: dates[1] }))`
- Court select → `setFilters(prev => ({ ...prev, courtId: value }))`
- Status select → `setFilters(prev => ({ ...prev, status: value }))`
- Search input → `setFilters(prev => ({ ...prev, search: value }))` (debounced)

#### BookingDetailPage

**Remove:** `mockBooking`

**Add:**
```typescript
import { useParams } from 'react-router-dom';
import { useBookingDetail, useCancelBooking, useFlagNoShow } from '@/hooks/useBookings';

const { bookingId } = useParams<{ bookingId: string }>();
const { data: booking, isLoading, isError } = useBookingDetail(bookingId);
const cancelBooking = useCancelBooking();
const flagNoShow = useFlagNoShow();
```

**Wire:**
- All `mockBooking.xxx` → `booking.xxx`
- `handleFlagNoShow` → `flagNoShow.mutate(bookingId, { onSuccess: () => notification.success(...) })`
- `handleCancel` → `cancelBooking.mutate({ id: bookingId, reason }, { onSuccess: () => notification.success(...) })`
- If `isError` and 404 → show "Booking not found" with link to `/bookings`

#### FeatureFlagsPage

**Remove:** `initialFlags`, `useState(initialFlags)`, local `toggleFlag`, local `handleAddFlag`

**Add:**
```typescript
import { useFeatureFlags, useUpdateFeatureFlag, useCreateFeatureFlag, useDeleteFeatureFlag } from '@/hooks/useAdmin';

const { data: flags, isLoading } = useFeatureFlags();
const updateFlag = useUpdateFeatureFlag();
const createFlag = useCreateFeatureFlag();
const deleteFlag = useDeleteFeatureFlag();
```

**Wire:**
- `dataSource={flags}` → `dataSource={flags ?? []}`
- Switch `onChange={() => toggleFlag(record.id)}` → `onChange={(checked) => updateFlag.mutate({ key: record.flagKey, enabled: checked }))` (optimistic)
- `handleAddFlag` → `createFlag.mutate(values, { onSuccess: () => { notification.success(...); setAddModalOpen(false); form.resetFields(); } })`
- `enabledCount` → `(flags ?? []).filter(f => f.enabled).length`

#### SupportTicketListPage

**Remove:** `mockTickets`, `useState(mockTickets[0].id)`

**Add:**
```typescript
import { useTicketList, useTicketDetail, useAddMessage } from '@/hooks/useSupport';

const { data: ticketData, isLoading } = useTicketList({ status: filter === 'all' ? undefined : filter });
const { data: selectedTicket } = useTicketDetail(selectedId);
const addMessage = useAddMessage();
```

**Wire:**
- Left panel list → `ticketData?.content ?? []`
- Right panel detail → `selectedTicket`
- "Send Reply" button → `addMessage.mutate({ ticketId: selectedId, body: replyText })`
- Status counts → derive from `ticketData` or use separate queries

#### SettingsPage

**Remove:** All hardcoded form values, mock sessions, mock payout data

**Add per tab:**
- Profile: Use `useAuthStore` for initial values, `useUpdateProfile` mutation for save
- Notifications: `useNotificationPreferences()` query, `useUpdateNotificationPreferences` mutation
- Reminders: `useReminderRules()` query, `useCreateReminderRule`/`useUpdateReminderRule`/`useDeleteReminderRule` mutations
- Court Defaults: `useCourtDefaults()` query, `useUpdateCourtDefaults` mutation
- Security: `useSessions()` query, `useRevokeSession` mutation, `settingsService.requestDataExport()` for GDPR
- Stripe: `stripeService.getPayouts()` for payout data

#### NotificationBell

**Remove:** `mockNotifications`, `useState(mockNotifications)`

**Add:**
```typescript
import { useUnreadCount, useNotificationList, useMarkAsRead, useMarkAllAsRead } from '@/hooks/useNotifications';

const { data: unreadCount } = useUnreadCount();
const { data: notificationData } = useNotificationList({ size: 20 });
const markAsRead = useMarkAsRead();
const markAllAsRead = useMarkAllAsRead();
```

**Wire:**
- `Badge count={unreadCount}` → `count={unreadCount ?? 0}`
- `List dataSource` → `notificationData?.content ?? []`
- `handleMarkAllRead` → `markAllAsRead.mutate()`
- `handleNotificationClick(id)` → `markAsRead.mutate(id)` + navigate to deep link

#### Analytics Pages (Revenue, Usage, Heatmap)

**Add per page:**
```typescript
const [params, setParams] = useState<AnalyticsParams>({ from: defaultFrom, to: defaultTo });
const { data, isLoading, isError } = useRevenue(params); // or useUsage / useHeatmap
```

- Date range picker → updates `params`
- Court filter → updates `params.courtId`
- Export button → `analyticsService.exportAnalytics({ ...params, format: 'csv' })` then trigger download
- 429 handling → check error status, display rate limit alert with countdown

#### Admin Pages (Users, Verifications, Disputes, Platform Analytics, Support Metrics)

Each follows the same pattern: replace mock data with the corresponding `useAdmin` hook, wire mutations for actions.

#### Audit Log Page

**Remove:** `mockEntries`

**Add:**
```typescript
import { useQuery } from '@tanstack/react-query';
import { auditLogKeys } from '@/lib/queryKeys';
import { platformApi } from '@/lib/api';

// Note: auditLogService is lightweight enough to inline here,
// but the query key factory ensures proper cache management
const [filters, setFilters] = useState<Record<string, unknown>>({});
const { data, isLoading } = useQuery({
  queryKey: auditLogKeys.list(filters),
  queryFn: async () => {
    const { data } = await platformApi.get('/api/audit-logs', { params: filters });
    return data;
  },
});
```

**Wire:**
- `dataSource={mockEntries}` → `dataSource={data?.content ?? []}`
- Filter bar onChange → update `filters` state
- Pagination → server-side via `filters.page` / `filters.size`

#### Holiday Calendar Page

**Remove:** `mockNationalHolidays`, `mockCustomHolidays`, `mockCourts`

**Add:**
```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { holidayKeys, courtKeys } from '@/lib/queryKeys';
import { platformApi } from '@/lib/api';

const { data: national } = useQuery({
  queryKey: holidayKeys.national(),
  queryFn: async () => {
    const { data } = await platformApi.get('/api/holidays/national');
    return data;
  },
});
const { data: custom } = useQuery({
  queryKey: holidayKeys.custom(),
  queryFn: async () => {
    const { data } = await platformApi.get('/api/holidays/custom');
    return data;
  },
});
// Court list for the multi-select dropdown
const { data: courts } = useOwnerCourts();

const qc = useQueryClient();
const createCustomHoliday = useMutation({
  mutationFn: (data: Record<string, unknown>) => platformApi.post('/api/holidays/custom', data),
  onSuccess: () => qc.invalidateQueries({ queryKey: holidayKeys.custom() }),
});
const applyHolidays = useMutation({
  mutationFn: (data: Record<string, unknown>) => platformApi.post('/api/holidays/apply', data),
  onSuccess: () => notification.success({ message: 'Holidays applied' }),
});
```

#### Global Search

**Add:**
```typescript
import { useQuery } from '@tanstack/react-query';
import { useDebouncedValue } from '@/hooks/useSearch';
import * as searchService from '@/services/searchService';
import { useAuthStore } from '@/stores/authStore';

const [query, setQuery] = useState('');
const debouncedQuery = useDebouncedValue(query, 300);
const user = useAuthStore(s => s.user);

const { data: results } = useQuery({
  queryKey: ['search', debouncedQuery],
  queryFn: () =>
    user?.role === 'PLATFORM_ADMIN'
      ? searchService.adminSearch({ q: debouncedQuery })
      : searchService.globalSearch({ q: debouncedQuery }),
  enabled: debouncedQuery.length >= 2,
});
```

## Data Models

All TypeScript interfaces are defined in the service files as shown above. They match the backend response models from the Phase 6 design document:

- `DashboardSummary` → matches `DashboardResult` Java record
- `CourtSummary` / `CourtDetail` → matches `Court` domain entity response
- `BookingListItem` / `BookingDetail` → matches `Booking` / `BookingDetail` schemas from OpenAPI
- `RevenueAnalytics` → matches `RevenueAnalyticsResult` Java record
- `UsageAnalytics` → matches `UsageAnalyticsResult` Java record
- `HeatmapData` → matches `HeatmapResult` Java record
- `TicketSummary` / `TicketDetail` → matches `SupportTicket` / `SupportTicketDetail` response
- `FeatureFlag` → matches `FeatureFlag` Java record
- `UserSummary` / `UserDetail` → matches `UserSummary` / `UserDetail` Java records
- `PlatformAnalytics` → matches `PlatformAnalyticsResult` Java record
- `NotificationItem` → matches notification response from Transaction Service
- `SessionInfo` → matches `SessionInfo` Java record


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Query Key Uniqueness

*For any* two distinct filter objects passed to a query key factory (e.g., `courtKeys.list(filtersA)` vs `courtKeys.list(filtersB)` where `filtersA !== filtersB`), the resulting query key arrays SHALL NOT be deeply equal. Different filter combinations must produce distinct cache entries.

**Validates: Requirements 3.10**

### Property 2: Mutation Error Isolation

*For any* mutation hook that encounters an error (the mutation function rejects), the `queryClient.invalidateQueries` function SHALL NOT be called. Failed mutations must never trigger cache invalidation.

**Validates: Requirements 4.13**

### Property 3: Search Input Gating

*For any* search query string of length less than 2, the search service function SHALL NOT be called. *For any* search query string of length 2 or greater, after a 300ms debounce period with no further input, exactly one API call SHALL be made with the final query value.

**Validates: Requirements 14.1, 14.2**

### Property 4: Court Visibility Optimistic Round-Trip

*For any* court list in the cache and any court ID within that list, toggling visibility optimistically SHALL immediately update the court's `visible` field in the cache. If the mutation subsequently fails, the cache SHALL be restored to its exact pre-mutation state. The final cache state after a failed toggle SHALL be identical to the initial cache state.

**Validates: Requirements 17.1, 17.2, 17.5**

### Property 5: Feature Flag Optimistic Round-Trip

*For any* feature flag list in the cache and any flag key within that list, toggling the `enabled` field optimistically SHALL immediately update the flag in the cache. If the mutation subsequently fails, the cache SHALL be restored to its exact pre-mutation state. The final cache state after a failed toggle SHALL be identical to the initial cache state.

**Validates: Requirements 17.3, 17.4, 17.5**

### Property 6: WebSocket Reconnect Full Invalidation

*For any* set of active queries in the TanStack Query cache, when the WebSocket connection transitions from disconnected to connected (reconnect), ALL active queries SHALL be invalidated. No active query key SHALL remain un-invalidated after a reconnect event.

**Validates: Requirements 15.7**

### Property 7: WebSocket Event → Cache Invalidation Mapping

*For any* WebSocket event of type `BOOKING_CREATED`, `BOOKING_UPDATED`, `BOOKING_CONFIRMED`, or `BOOKING_CANCELLED`, the `bookingKeys.lists()` and `dashboardKeys.summary()` query keys SHALL always be invalidated. For events that include a `bookingId` in the payload and are of type `BOOKING_CONFIRMED` or `BOOKING_CANCELLED`, `bookingKeys.detail(bookingId)` SHALL additionally be invalidated.

**Validates: Requirements 15.2, 15.3**

## Error Handling

### Service Layer Error Propagation

All service functions let errors propagate from the Axios interceptor. The interceptor normalizes errors to:

```typescript
type ApiError = {
  error: string;    // e.g., 'RATE_LIMIT_EXCEEDED', 'INVALID_DATE_RANGE'
  message?: string; // Human-readable message
  status: number;   // HTTP status code
};
```

No additional wrapping or transformation occurs in the service layer.

### Hook-Level Error Handling

- **Query errors**: Exposed via `query.isError` and `query.error`. The `QueryStateWrapper` component renders an error alert with retry button.
- **Mutation errors**: Exposed via `mutation.isError` and `mutation.error`. Page components display `notification.error()` toasts.
- **Optimistic update errors**: The `onError` callback restores the cache snapshot. Then `onSettled` invalidates to ensure consistency.

### Specific Error Scenarios

| HTTP Status | Error Code | UI Handling |
|-------------|-----------|-------------|
| 400 | `INVALID_DATE_RANGE` | Inline alert on analytics pages with validation message |
| 403 | `FORBIDDEN` | Redirect to dashboard or show "Access denied" |
| 404 | Not found | "Not found" message with back link (booking detail, court detail, ticket detail) |
| 409 | `FLAG_KEY_EXISTS` | Error toast on feature flag creation form |
| 422 | `INVALID_STATUS_TRANSITION` | Error toast on ticket status update |
| 429 | `RATE_LIMIT_EXCEEDED` | Warning alert with `retryAfterSeconds` countdown, disable action button |
| 429 | `EXPORT_RATE_LIMIT_EXCEEDED` | Warning alert with countdown on export button |
| 401 | Session expired | Silent refresh via interceptor; if refresh fails, redirect to login |
| 5xx | Server error | Generic "Something went wrong" toast with retry |

### Network Error Handling

When `status === 0` (network error), the Axios interceptor returns `{ error: 'NETWORK_ERROR', message: 'Network error occurred', status: 0 }`. The `QueryStateWrapper` displays this with a retry button.

## Testing Strategy

### Property-Based Testing

Property-based tests use **fast-check** (TypeScript). Each property test runs a minimum of 100 iterations.

Tag format: `Feature: phase-6b-admin-web-api-integration, Property {number}: {title}`

| Property | Test File | Generator Strategy |
|----------|----------|-------------------|
| P1: Query Key Uniqueness | `queryKeyUniqueness.property.test.ts` | Generate pairs of random filter objects (varying keys and values). Pass both to the same key factory. Assert deep inequality when inputs differ. |
| P2: Mutation Error Isolation | `mutationErrorIsolation.property.test.ts` | Generate random error objects. For each mutation hook, simulate rejection. Assert `invalidateQueries` was never called. |
| P3: Search Input Gating | `searchInputGating.property.test.ts` | Generate random strings of length 0-100. For strings < 2 chars, assert no API call. For strings >= 2 chars, assert exactly one call after debounce. |
| P4: Court Visibility Optimistic Round-Trip | `courtVisibilityOptimistic.property.test.ts` | Generate random court arrays (1-20 courts with random visibility). Pick a random court, toggle, simulate error. Assert final cache equals initial cache. |
| P5: Feature Flag Optimistic Round-Trip | `featureFlagOptimistic.property.test.ts` | Generate random flag arrays (1-10 flags with random enabled state). Pick a random flag, toggle, simulate error. Assert final cache equals initial cache. |
| P6: WebSocket Reconnect Invalidation | `wsReconnectInvalidation.property.test.ts` | Generate random sets of active query keys (1-20 keys). Simulate reconnect. Assert all keys were passed to `invalidateQueries`. |
| P7: WS Event Cache Mapping | `wsEventCacheMapping.property.test.ts` | Generate random booking events with random event types and payload. Assert correct keys invalidated per event type. |

### Unit Tests (Example-Based)

| Area | Test Focus | Framework |
|------|-----------|-----------|
| Service functions | Each service function calls correct URL with correct method | Vitest + MSW or axios-mock-adapter |
| Service functions | platformApi vs transactionApi routing | Vitest |
| Query hooks | Correct query key, staleTime, enabled condition | Vitest + @testing-library/react-hooks |
| Mutation hooks | Correct cache invalidation on success | Vitest + @testing-library/react-hooks |
| QueryStateWrapper | Renders skeleton for loading, alert for error, empty for no data, children for data | Vitest + React Testing Library |
| DashboardPage | Renders with mock query data, shows skeleton when loading | Vitest + React Testing Library |
| CourtListPage | Renders court list from hook data, not from mockCourts | Vitest + React Testing Library |
| BookingListPage | Renders booking list from hook data, pagination works | Vitest + React Testing Library |
| FeatureFlagsPage | Toggle calls useUpdateFeatureFlag, not local setState | Vitest + React Testing Library |
| NotificationBell | Renders unread count from hook, mark-all-read calls mutation | Vitest + React Testing Library |
| WebSocket sync | Correct event type → invalidation mapping | Vitest |
| Search debounce | 300ms debounce, min 2 char threshold | Vitest + fake timers |

### Integration Tests

| Area | Test Focus | Framework |
|------|-----------|-----------|
| Full page render | Each page renders without crash with mocked API responses | Vitest + MSW + React Testing Library |
| Mutation → refetch cycle | Mutation success triggers cache invalidation and background refetch | Vitest + @tanstack/react-query test utils |
| Optimistic update cycle | Toggle → immediate UI update → server error → rollback | Vitest + React Testing Library |
| WebSocket → cache sync | Simulated WS message → correct queries invalidated | Vitest |

### Smoke Tests

| Area | Test Focus |
|------|-----------|
| Build | `npm run build` succeeds without TypeScript errors |
| Service barrel | `import * from '@/services'` resolves all modules |
| No mock data | `grep -r "const mock" src/features/` returns zero results |
| Query key completeness | All query key factories used by at least one hook |
