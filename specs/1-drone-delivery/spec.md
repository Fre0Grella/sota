# Feature: Drone delivery (drone-delivery)

Created: 2025-11-02  
Author: SotA team

## Summary
Provide an online service that allows users to request package deliveries via drones. Users can schedule immediate or future deliveries, specify package weight, request a delivery time window, and track progress in real time (location + remaining ETA).

## Actors
- Sender (user initiating a delivery)
- Recipient (optional; may receive notifications/status)
- System operator (monitoring / manual overrides)
- Delivery drone (autonomous/managed vehicle) — treated as a system actor for scheduling/telemetry.

## Goals
- Enable users to request deliveries now or scheduled for a future date/time.
- Respect user-specified delivery windows and maximum allowed delivery duration.
- Provide real-time tracking of package location and a reliable ETA until completion.
- Ensure requests are validated against operational constraints (payload, range, weather, no-fly zones).

## User Scenarios (Primary flows)

1. Book immediate delivery
   - Sender opens the service, enters pickup and drop-off locations, package weight, and requests delivery "now".
   - System confirms availability, provides estimated cost and ETA, and authorizes drone dispatch.
   - Sender can view live tracking and ETA until delivery completes.

2. Schedule future delivery
   - Sender chooses pickup/drop-off, package details, desired delivery date/time or delivery window.
   - System verifies feasibility for the window, offers options if conflicts exist (alternate times or drone classes).
   - Sender confirms booking; system reserves resources and provides status updates until dispatch.

3. Track delivery
   - Sender (and optionally recipient) views a live map with package location, current state (queued, enroute, delivery attempt, completed), and remaining ETA.
   - System provides last-known telemetry and expected time to completion with periodic updates.

4. Failure & recovery
   - If the drone cannot complete delivery (weather, battery, blocked landing), system notifies sender/recipient with options:
     - Reschedule within user window
     - Hold for manual operator action
     - Refund/cancel (based on policy)

## Functional Requirements (testable)

FR-1 Booking
- The system MUST accept a delivery request containing: sender id, pickup location, drop-off location, package weight, requested delivery time (timestamp) or window (start/end), and contact details.
- Accept both "immediate" and scheduled requests.

FR-2 Feasibility check
- For each request, the system MUST validate payload vs available vehicle classes and compute feasible delivery time windows based on range, battery profile, and airspace restrictions.
- If infeasible, the system MUST return at least one alternative (earliest feasible time or available drone class).

FR-3 Scheduling & Reservation
- The system MUST reserve a drone (or fleet slot) for scheduled deliveries ensuring no double-booking for the same physical resource/time slot.
- Reservation confirmation MUST include an estimated dispatch time and ETA.

FR-4 Dispatch & Command
- For immediate or scheduled dispatch, the system MUST trigger a launch workflow that transitions the delivery through states: scheduled → preparing → enroute → delivering → completed/failed.

FR-5 Real-time tracking
- The system MUST provide live location (lat/lon), state, and ETA for any active delivery, with updates at least every 15 seconds under normal network conditions.
- The delivered ETA MUST be recomputed and updated when significant deviations occur (e.g., >30 seconds difference from previous ETA).

FR-6 Notifications
- The system MUST notify sender and recipient of key events: booking confirmation, drone dispatched, arrival ETA within configurable threshold (e.g., 5 min), delivery attempt, completion, failure.

FR-7 Constraints enforcement
- The system MUST enforce configured operational constraints (max payload per drone class, geofence/no-fly zones, weather windows). Requests violating constraints MUST be rejected with reasons.

FR-8 Audit & proof of delivery
- Upon completion, the system MUST store proof of delivery (timestamp, final location, operator actions, optional photo or recipient confirmation) for audit and customer support.

FR-9 Cancellation & rescheduling
- The system MUST allow cancellation or rescheduling within policy-defined windows; cancellations after dispatch MUST trigger recovery procedures and notifications.

FR-10 Observability & logs
- The system MUST persist telemetry and delivery state transitions for post-incident review for a configurable retention period.

## Non-functional Requirements (testable where possible)

NFR-1 Availability
- Core booking and tracking APIs MUST be available 99% over a 30-day rolling window (measurable).

NFR-2 Latency (user-facing)
- System responses for booking and status queries MUST return within 2 seconds 90% of the time under normal load.

NFR-3 Timeliness of tracking updates
- Live tracking updates MUST be available to users at least every 15 seconds in normal conditions.

NFR-4 Data retention & privacy
- Delivery records and telemetry MUST be retained according to policy; personal location data MUST be handled per privacy requirements (assumed compliance: redact/export on request).

## Success Criteria (measurable, tech-agnostic)

SC-1 Booking lead time
- 95% of immediate delivery requests that are feasible get confirmed within 2 minutes of request submission.

SC-2 On-time deliveries
- At least 90% of completed deliveries finish within the user-specified requested window or within the agreed SLA.

SC-3 Tracking fidelity
- 95% of active deliveries show location and ETA updates at least every 15 seconds during the enroute phase.

SC-4 Recovery & notification
- For delivery failures requiring customer action, 95% of affected customers receive the first notification within 3 minutes of detection.

SC-5 Auditability
- 100% of deliveries persist required audit records (state transitions + proof of delivery) within 24 hours of completion.

## Key Data Entities
- Delivery (id, sender_id, recipient_id, pickup_location, dropoff_location, weight, dimensions, requested_time/window, status, assigned_drone_id, audit_log_id)
- Drone (id, class, max_payload, range, current_state, telemetry_reference)
- Telemetry (timestamp, lat, lon, altitude, speed, heading, battery_percent)
- Reservation (id, delivery_id, drone_id, reserved_from, reserved_until)
- Notification (id, delivery_id, recipient_contact, event_type, timestamp, delivered)

## Acceptance Tests (high-level, traceable to FRs)
- AT-1 Booking valid immediate request → returns CONFIRMED and assigned drone/reservation (tests FR-1, FR-3).
- AT-2 Booking scheduled request within range → returns CONFIRMED with reservation and estimated dispatch (FR-1, FR-3).
- AT-3 Booking request beyond payload/range → returns REJECTED with actionable alternative (FR-2, FR-7).
- AT-4 Live tracking endpoint returns recent telemetry and ETA for enroute delivery (FR-5).
- AT-5 Simulated delivery failure triggers notification to customer within 3 minutes and persists audit record (FR-6, FR-8).

## Assumptions
- Reasonable defaults unless specified by policy: default max payload class = 5 kg (used only for planning/assumption; operational values configurable).
- Weather, geofence and regulatory constraints are enforced by external services or datasets the system can query.
- Recipient contact is optional; primary notifications are sent to sender unless recipient provided.
- Privacy/compliance specifics (GDPR, local laws) will be applied according to the deployment region.

## Out of Scope
- Low-level flight control algorithms and avionics firmware (handled by drone subsystem).
- Regulatory permitting and airspace approvals (operational/legal responsibilities).
- Payment/charging integration for pricing (may be a separate feature).

## Dependencies
- Maps/geocoding service for coordinates and routing estimates.
- Weather and no-fly zone data sources for feasibility checks.
- Drone telemetry ingestion pipeline.

## Risks
- Inaccurate ETAs due to dynamic urban obstacles.
- Regulatory changes impacting operational areas.
- Connectivity interruptions affecting telemetry frequency.

## Deliverables for initial implementation
- Booking API (immediate + scheduled)
- Feasibility & scheduling service
- Dispatch state machine + reservation system
- Tracking API with telemetry+ETA
- Notification hooks for events
- Audit record store for proof of delivery

## Next steps
- Create ADRs for payload classes, SLA definitions, and privacy/data-retention policies.
- Prototype UI for booking + live tracking demo.
- Define test harness to simulate telemetry and failures for acceptance tests.
