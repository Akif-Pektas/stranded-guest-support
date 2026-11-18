# Stranded Guest Support

A Python project for stranded guest support. Requires Python 3.10 or newer.

## Local setup

```sh
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e .
```

Package code lives in `src/stranded_guest_support/`. Add tests in `tests/` and run them with `python -m unittest discover -s tests`.

## Lisbon incident ticket handling

| Ticket type | Specialist | Authorized action |
| --- | --- | --- |
| Rebooking | Rebooking Agent | `hold_booking` for a comparable, available home |
| Refund | Refund Agent | `issue_refund` for an eligible amount |
| Safety | Safety Agent | `handoff_to_safety_agent` only |

Read the guest message for any of five AirCover safety conditions: injury or medical need, threat or violence, nowhere safe to wait, an unaccompanied minor, or stated fear of a person. Any one condition sends the ticket to a live Trust and Safety agent, even when the guest asks only for money, a call, or a new flat. No agent may call `issue_refund` or `hold_booking` before safety handoff completes. The live safety agent then decides any remedy. Angry words or a callback request alone stay with the Rebooking Agent.

A replacement home must cover the whole remaining stay, fit the party, have enough bedrooms, be available to hold, and cost at most 120% of the original `nightly_price`. An eligible guest who declines rebooking or has no comparable home is due a full refund. Refunds never exceed `total_paid` in total, and the same nights receive one remedy.

See the [ticket-handling specification](specs/001-ticket-handling-rules/spec.md) for the full policy and acceptance scenarios.
