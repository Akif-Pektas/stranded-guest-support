# Feature Specification — Stranded Guest Ticket Handling

**Feature Branch** `001-ticket-handling-rules`

**Created** 2026-11-18

**Status** Ready for planning

**Input** Ticket-handling rules for the Lisbon check-in incident of 16 November 2026, based on the supplied AirCover policy and ticket export.

## User Scenarios & Testing

### User Story 1 — Reach live Safety (Priority P1)

As a stranded guest reporting a safety condition, I need a live Trust and Safety agent to take over even if I am asking only for money, a call, or a replacement home.

**Why this priority** Safety takes precedence over accommodation and payment.

**Independent test** Submit messages describing each safety condition individually, vary the requested remedy, and review the classification and action order.

1. **Given** a guest requests a refund and says they are hurt, **when** the message is triaged, **then** the Safety Agent calls `handoff_to_safety_agent` and no agent holds a home or issues a refund before handoff.
2. **Given** a guest asks for a callback and says they are under 18 and travelling alone, **when** the message is triaged, **then** the ticket goes to a live Trust and Safety agent.
3. **Given** a guest asks for a new flat and describes having nowhere safe to wait, **when** the message is triaged, **then** the ticket goes to a live Trust and Safety agent before any booking action.
4. **Given** either a report of threats or violence or an expression of fear of a person, **when** that is the sole safety condition, **then** it independently triggers the same handoff.

### User Story 2 — Find a comparable home (Priority P2)

As an eligible stranded guest seeking accommodation or help deciding what to do, I need a replacement that covers my party and full remaining stay within the AirCover price limit.

**Why this priority** A valid replacement gives the guest somewhere to stay for all remaining nights.

**Independent test** Submit a callback-only ticket and evaluate candidate homes on both sides of the price and comparability boundaries.

1. **Given** a guest expresses anger or requests a callback alone, **when** their message describes none of the five safety conditions, **then** the Rebooking Agent handles the ticket.
2. **Given** an original nightly rate of EUR 110.00, **when** otherwise comparable homes cost EUR 132.00 and EUR 132.01, **then** the first passes the price check and the second fails.
3. **Given** a guest rejects one comparable offer, **when** another comparable home is available, **then** the guest remains eligible for another offer.
4. **Given** a candidate fails the date, party size, bedroom, or availability check, **when** the Rebooking Agent evaluates it, **then** it is ineligible for a hold.

### User Story 3 — Receive a policy-compliant refund (Priority P2)

As an eligible guest who declines rebooking or cannot be offered a comparable home, I need a refund up to what I paid for the original stay.

**Why this priority** Eligible guests need a payment remedy when rebooking cannot resolve the stay.

**Independent test** Evaluate a guest who declines rebooking, a completed search with no comparable home, and refund attempts at and above the cap.

1. **Given** a guest without a safety condition declines rebooking or a completed search finds no comparable home, **when** the Refund Agent verifies eligibility, **then** the guest is due the full `total_paid`.
2. **Given** prior permitted partial refunds, **when** a new refund would bring the cumulative amount above `total_paid`, **then** the Refund Agent blocks the excess.
3. **Given** a guest was rebooked for the same nights, **when** a refund is requested for those nights, **then** the Refund Agent blocks the overlapping remedy.

### Edge Cases

- Guest anger and a guest's report that someone shouted at them are distinct. The latter is assessed as a report of threatening or violent conduct under the supplied policy.
- A callback request, a request label, and the absence of the word “safety” do not override what the guest describes.
- If handoff fails or is pending, retain the safety classification and keep booking and refund actions blocked. After handoff, the live safety agent decides any remedy.
- A listing that becomes unavailable before the hold fails comparability. A failed or incomplete inventory search does not establish that there are no comparable homes.

## Requirements

### Functional Requirements

- **FR-001** Assign each ticket a primary type of rebooking, refund, or safety. Evaluate `guest_message` for safety before selecting a remedy. Without a safety condition, send explicit requests to abandon rebooking and obtain a refund to the Refund Agent for eligibility review. Send replacement-home requests, callback-only requests, expressions of anger alone, and undecided guests to the Rebooking Agent. When a complete search finds no comparable home for an eligible guest, transfer the ticket to the Refund Agent.
- **FR-002** Each of these conditions independently triggers live safety handoff, based on what the guest describes.

  | AirCover condition | What the guest may describe |
  | --- | --- |
  | `injury_or_medical_need` | Someone is hurt, unwell, pregnant, needs medication, or needs a doctor. |
  | `threat_or_violence` | Someone threatened, followed, shouted at, or harmed the guest. |
  | `nowhere_safe_to_wait` | The guest has nowhere safe to wait, such as being alone outside at night with a dying phone. |
  | `unaccompanied_minor` | A guest under 18 is travelling or waiting without an adult. |
  | `stated_fear_of_person` | The guest says they fear the host, a neighbour, a stranger, or someone in their party. |

- **FR-003** Apply these specialist permissions for the named actions.

  | Specialist | Authorized action | Conditions |
  | --- | --- | --- |
  | Rebooking Agent | `hold_booking` | An eligible rebooking ticket and a currently available comparable home. |
  | Refund Agent | `issue_refund` | An eligible refund ticket and an authorized amount within the remaining cap. |
  | Safety Agent | `handoff_to_safety_agent` only | A safety ticket must go to a live Trust and Safety agent. |

- **FR-004** No agent may call `issue_refund` or `hold_booking` on a safety ticket before `handoff_to_safety_agent` completes. After handoff, the live safety agent owns the decision on any booking or refund.
- **FR-005** A guest qualifies for rebooking support if they lack working access at check-in, the host is unreachable, or the host cancelled within 30 days of check-in. When assessing host reachability, use the guest's account and whether the host replied after the guest contacted them.
- **FR-006** A comparable home must cover the entire remaining stay from `check_in_date` through `check_out_date`, fit the party, have at least as many bedrooms as the original, be available at hold time, and cost at most 1.20 times the original `nightly_price`. Prefer the same or a neighbouring Lisbon neighbourhood among qualifying homes. Use current open inventory for a replacement.
- **FR-007** Enforce the incident-wide EUR 9,000 uplift budget. Calculate uplift as any increase above the original nightly rate multiplied by the nights rebooked. The EUR 150 planning average does not set an individual limit. The 120% nightly ceiling applies even if incident funds remain.
- **FR-008** Cap cumulative refunds per stay at `total_paid`, including goodwill. An eligible guest is due a full refund when no comparable home is available or they decline rebooking. An authorized partial refund must be at least EUR 0.01 and within the remaining cap. A refund closes the stay, and the same nights cannot receive both remedies.

### Key Entities

- **Guest ticket** — Guest message, requested remedy, reservation dates, party size, bedroom count, host contact or cancellation, original nightly rate, and amount paid.
- **Open listing** — Listing identifier, availability dates and status, guest capacity, bedrooms, neighbourhood, and nightly rate.
- **Ticket resolution** — Primary type, assigned specialist, policy reason, and resulting rebooking, refund, or live handoff.
- **Incident uplift budget** — Shared amount available to cover the rate increases for rebooked nights.

## Success Criteria

### Measurable Outcomes

- **SC-001** All six safety-labeled tickets in the supplied export route to live Safety. Independently exercise all five policy conditions, including when the guest asks only for money, a callback, or accommodation. In every safety scenario, zero booking or refund actions precede completed handoff.
- **SC-002** All 34 rebooking-labeled and 20 refund-labeled tickets in the export match their expected initial routing. Callback-only and anger-only acceptance cases route to rebooking.
- **SC-003** Every accepted replacement meets all comparability checks, stays at or below 120% of the original nightly rate, and fits within the shared uplift budget.
- **SC-004** Zero refunds exceed the amount paid, and zero stays receive overlapping remedies for the same nights.
- **SC-005** The incident policy target is a bed, refund, or live safety agent for each guest before midnight Lisbon time on 16 November 2026. This is the historical service target, not a claim about actual resolutions.

## Assumptions

- Scope is the 16 November 2026 Lisbon check-in incident. This document specifies behaviour for planning and does not implement it.
- Sources are the supplied `airbnb_lisbon_stranded_guest_tickets_2026-11-16.csv` and `aircover_rebooking_refund_rules.md`, provided locally as “Lisbon Stranded Guest Tickets (16 Nov 2026).csv” and “AirCover Rebooking and Refund Rules - Lisbon Check-In Incident.md.” Expected outcomes are verification labels and do not drive live routing.
- Comparable-home acceptance tests use representative inventory. The policy refers to a separate open-listings export that was not supplied for this specification.
- Explicitly declining rebooking differs from declining a single offer. The policy allows partial amounts but supplies no formula for choosing one, so partial refunds require an authorized amount.
