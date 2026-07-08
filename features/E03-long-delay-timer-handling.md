---
id: E03
epic: E — Native notifications
title: Long-delay timer handling
size: S
requires: [E01]
novel: false
---

## What
NextAction times can be far in the future — weeks or months out. JavaScript's
`setTimeout` cannot represent delays beyond ~24.8 days (its 32-bit millisecond
ceiling), so the notifier handles long delays specially instead of firing
immediately or overflowing.

## Why it exists
Without this, any P0 notification scheduled more than ~24.8 days ahead would
misbehave — a naive `setTimeout` with an out-of-range delay fires right away.
Getting far-future alerts right is required for the notification system to be
trustworthy, which is the whole point of E01.

## Acceptance criteria (EARS)
- When a P0 NextAction is more than ~24.8 days in the future, the system shall
  not fire the notification early and shall not let the timer overflow.
- The system shall still fire the notification at the correct time once that time
  is within `setTimeout`'s representable range.
- The system shall coexist with auto-clear (E02): a far-future scheduled alert
  can still be cancelled or rescheduled before it fires.

## Build notes
- Standard technique: cap each `setTimeout` at the ~24.8-day maximum and
  re-arm/chain the timer as the target time approaches, rather than scheduling the
  full delay at once.
- Implemented in the Electron-main notifier alongside E01/E02; make sure a chained
  timer is still findable and cancellable when E02 needs to clear it.
