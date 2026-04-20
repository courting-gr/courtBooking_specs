# Phase 6 — Admin Portal Figma Design Reference

## Figma Source

**Figma Make Project:** https://www.figma.com/make/t47mNNk9yx1aFLaqG1w3uO/Court-Booking-Admin-Portal

This Figma Make project contains the complete UI design for the court-booking-admin-web React application, generated from the Phase 6 requirements. It covers all 14 design prompts (dashboard, court management, bookings, analytics, support, settings, platform admin, login/onboarding, court detail sub-pages, booking detail, audit log, states/modals, disputes/metrics).

## How to Use This Reference

1. The Figma project serves as the visual specification for the admin portal
2. During implementation, share specific Figma frame URLs with Kiro to generate React + Ant Design code
3. The Figma Make exported code is available in `court-booking-admin-web/figma-reference/src/` as a structural reference — do NOT use it directly as production code (it uses generic CSS, not Ant Design)
4. Production code goes in `court-booking-admin-web/src/` and must use Ant Design 5.x components

## Figma Reference Code Location

The exported source from Figma Make is stored at:
```
court-booking-admin-web/figma-reference/src/
```

This code is for reference only — it shows the component structure and layout intent but needs to be re-implemented using Ant Design components, React Query for data fetching, and the actual backend API endpoints.

## Design-to-Implementation Mapping

When implementing a page, the workflow is:
1. Open the Figma Make project and find the relevant page/component
2. Reference the `figma-reference/src/` code for component structure and data flow
3. Implement in `src/` using Ant Design components, connecting to real API endpoints
4. Validate the implementation matches the Figma visual design
