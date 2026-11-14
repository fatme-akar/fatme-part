# Post and Driving review audit

Reviewed on 8 November 2026. Scope: the complete folder at
`/Users/alisaralmusa/Downloads/topic-01-campus-ride-board`, with runtime review focused
on surfaces 05 (Post/Edit) and 06 (Driving), their shared state, mock service, and the
owner workspace used for request management.

Result: three confirmed runtime defects, one required submission/documentation
omission, and one optional resilience improvement. An independent review agent
checked the findings against the source and project specification. No application
source or existing test files were edited during the initial audit.

Follow-up: the user approved finding **2 only**, and it has now been fixed. Departed
rides show a label and a View ride link; request review closes automatically when
departure time passes, including an open decline dialog. Existing stored statuses,
requests, seat calculations, and service contracts are preserved. No fixes for
findings 1 or 3, the assignment-table omission, or optional storage hardening were applied.

Follow-up validation: production build passed, all 9 logic tests passed, and 17
desktop/mobile browser tests passed with 1 intentional desktop skip for mobile-only
navigation. Three new scenarios run at both widths cover already-departed rides,
an open decline dialog crossing departure, and a dashboard left open through departure.
They also check that closing review does not change stored requests.

## Validation completed

| Check | Result |
| --- | --- |
| Root and frontend READMEs, HTML specification, all React source and existing tests | Reviewed |
| Installed dependency consistency (`npm ls --depth=0`) | Passed |
| Logic/service suite (`npm test`) | 9 passed |
| Production build (`npm run build`) | Passed |
| Existing live Google Chrome suite (`npm run test:e2e`) | 11 passed, 1 intentional desktop skip for mobile-only navigation |
| Additional live Chrome audit, desktop 1280px and mobile 390px | 25 passed, 1 intentional desktop skip for a mobile-only overflow reproduction |
| Independent second review | Confirmed the three runtime defects; downgraded damaged-storage handling to optional hardening |

Additional scenarios covered real two-tab create/edit interactions; save failure and
retry without duplicates; capacity below accepted seats; rider conflicts below and
at exactly three hours; departure edit conflicts and recovery; valid and malformed
storage; natural departure expiry using the browser clock; long text; signed-out
provider integration; and missing, other-driver, and cancelled edit routes.

Tests named `REPRO` assert the observed defect so it can be reproduced consistently.
Their passing means the defect was reproduced, not that the affected behavior is
correct. The signed-out check injected `viewer: null` into the documented provider
entry point at browser runtime; it did not test real authentication.

The first audit run also checked long-origin overflow at desktop width; that
assertion did not reproduce. The final finding and reproduction are explicitly
limited to mobile. Normal long text containing spaces passed at both widths.

## 1. High — Cross-tab updates silently discard unsaved forms

**Reproduce:**

1. Open `/post` in tab A and enter a pickup location and notes without submitting.
2. Open the same app in tab B and successfully post a ride.
3. Return to tab A. Its fields are empty; no discard dialog appeared.

The same issue occurs when tab A has unsaved edits at
`/rides/midweek-trip/edit` and tab B accepts a request on the morning trip.
Tab A resets to the saved values, even though tab B changed a different ride.
Both cases reproduced at desktop and mobile widths.

**Cause:** [RidesContext.jsx](src/state/RidesContext.jsx), lines 24–36, handles a
storage event by setting global status to `loading`.
[PostRidePage.jsx](src/pages/PostRidePage.jsx), line 13, replaces the form with a
loading component. This destroys the local draft state in
[RideForm.jsx](src/components/RideForm.jsx), lines 20–21. The navigation guard
cannot intercept a component being unmounted by a state update.

**Proposed fix:** Refresh existing data in the background while keeping the form
mounted and retaining its draft. Continue validating against the latest accepted
seats and service state. Handle externally cancelled/unavailable rides explicitly
without silently discarding draft text. Add regressions for both reproduced cases.

## 2. Fixed — Departed rides offer request actions that cannot complete

The reproduction and cause below describe the initial audit. The approved fix is
implemented in `DrivingRideRow`, `RequestReview`, and `OwnerRidePage`, using
`src/hooks/useHasDeparted.js` to update the interface at departure time. The existing
service continues to enforce its original review restrictions.

**Reproduce:** Load fresh sample rides, allow the first trip's departure to pass,
then choose its `Review 2 requests` action. Accept is unavailable, but Decline
remains offered. Confirming Decline always reports `This ride has already departed.`
The request remains pending and the dashboard continues advertising it for review.

This was reproduced at both widths by advancing the browser clock after loading
normal sample data. No stored ride data was altered for the final reproduction.

**Cause:** [DrivingPage.jsx](src/pages/DrivingPage.jsx), line 11, includes scheduled
rides regardless of departure. [DrivingRideRow.jsx](src/components/DrivingRideRow.jsx),
lines 18–20, offers review whenever pending requests exist.
[RequestReview.jsx](src/components/RequestReview.jsx), line 41, only disables
Decline while busy. [rideService.js](src/data/rideService.js), line 72, rejects both
review decisions after departure.

**Proposed fix:** Show a departed state and remove or disable request actions that
the service will reject. Align dashboard review links with that state. A full
history/archive feature is not required to resolve the inconsistency.

## 3. Low — Accepted long origins break the required 390px layout

**Reproduce:** Post an otherwise valid ride with an origin consisting of 100
consecutive `A` characters. The form permits and saves it. On mobile, Driving's
document width becomes **1028px for a 390px viewport**. The owner workspace also
overflows when opening the ride.

**Cause:** The origin input allows 100 characters
([RideForm.jsx](src/components/RideForm.jsx), line 79), but the displayed origin
has no wrapping for unbroken text in
[DrivingRideRow.jsx](src/components/DrivingRideRow.jsx), line 10, and
[OwnerRidePage.jsx](src/pages/OwnerRidePage.jsx), line 33. The root README explicitly
requires 390px support.

**Proposed fix:** Allow long origin text to wrap within its container on the
dashboard and owner workspace, and verify the accepted maximum-length value at
390px. Desktop overflow was not reproduced.

## Required documentation omission

The [root README](../README.md), lines 20–21, requires a named surface-assignment
table for the individual examination. Lines 82–83 still use `Current assignee`
and `Other group members` placeholders. Record the actual owner for Post/Driving
and the team's agreed names before submission. This is a submission gap, not a
runtime defect; the actual assignment names need to come from the team.

## Optional — Validate damaged persisted data before rendering

With valid JSON containing an invalid `departureAt`, the storage adapter accepts
the record and the dashboard displays React Router's `Unexpected Application Error!`
with `Invalid time value`, instead of the existing recoverable load-error panel.
This reproduced at both widths.

[rideService.js](src/data/rideService.js), line 18, checks only the outer array,
ride ID, and requests array. Additional field validation could route damaged data
through the existing error/retry interface. Existing invalid-JSON handling works.

The current form/service does not generate the damaged record used here. This is
optional corruption resilience, not an ordinary-flow defect or a security finding.

## Behaviors verified as working

- Create, reload persistence, and edit, including prices and accepted-seat preservation.
- Required fields, capacity overflow, fractional seats, past departures, price rules,
  and Beirut date/time helpers through the existing logic tests.
- Multi-seat acceptance, full-car prevention, and accepted-seat totals computed from requests.
- The three-hour rider rule, including editing existing accepted rides and the exact boundary.
- Discard confirmation, Keep editing, Escape, and browser Back navigation.
- Request-decline and ride-cancellation confirmations, cancellation persistence,
  storage-write failure, and successful retry without duplicate creation.
- Loading, populated, empty, and recoverable error flows, mobile navigation,
  signed-out Post/Driving prompts, and owner checks.

## Scope and limitations

Browse, Riding, Sign In, Home, driver profile, and the complete Ride Detail surface
belong to the team's remaining work. The frontend README explicitly documents the
available placeholder routes and the owner-workspace integration boundary. Their
incompleteness is not counted as a bug in these two assigned screens. The mock
service, demo viewer, and absence of a backend match the documented Phase 1 setup.

Live tests used installed Google Chrome at desktop and mobile viewport sizes;
mobile viewport testing does not establish compatibility with Safari or physical
devices. Tests used isolated browser contexts and temporary data. Running the
build and original browser suite regenerated their normal output folders. This
folder has no Git metadata, so there is no Git diff/history to compare.

## Approval scope

Recommended implementation: fix findings 1–3 with the relevant regressions.
The README assignment table needs the team's actual names. Persisted-data
validation can be approved separately as optional hardening.

User decision: finding 2 approved and implemented; the other findings remain unchanged.

The temporary reproduction suite and screenshots are in
`/private/tmp/darb-review-20261108/`. The original departed-ride reproduction asserts
the old bug and is now superseded by the regression tests in `tests/browser/rides.spec.js`.
Use `npm run test:e2e` for the current behavior. The historical suite can be run
while that directory exists, but its departed-ride reproduction is expected to fail
after the fix:

```sh
npm run test:e2e -- --config /private/tmp/darb-review-20261108/playwright.config.mjs
```
