# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

Three smells, each in a different part of the module. For each one, fill in all five parts.

### Smell 1

**The smell.** Duplication over reuse: the pricing rules are implemented twice.

**Classic or agent-specific.** Agent-specific, caused by missing context. The report author
rebuilt pricing instead of reusing it: same numbers and rounding steps, but every constant
renamed (`PREMIUM_MULTIPLIER` → `PREMIUM_RATE_MULTIPLIER`, `LONG_BOOKING_MINUTES` →
`LONG_BOOKING_CUTOFF`, `EVENING_START_MINUTE` → `EVENING_CUTOFF`). The same thing happened to
interval overlap, which is written three ways (`hasConflict`, `isSlotFree`, `overlapsWindow`).

**Where in the code.** `src/reportGenerator.ts`, `ReportGenerator.priceOf`, which copies
`src/reservationManager.ts`, `ReservationManager.calculatePrice` + `applyDiscounts`.

**The principle it violates.** One home per domain rule (single source of truth / DRY). With
two copies, any pricing change turns into shotgun surgery.

**What it makes expensive.** Any pricing change, e.g. moving the evening discount to 18:00.
Edit one copy and forget the other, and the revenue report stops matching what customers were
charged. The suite would not notice: the revenue tests only use the standard room with
2-hour bookings, so a premium or long-booking change made in one copy stays green.

### Smell 2

**The smell.** God class / divergent change.

**Classic or agent-specific.** Classic.

**Where in the code.** `src/reservationManager.ts`, `ReservationManager`: `formatReceipt`,
`formatDailySummary`, `formatClock`, `formatMoney`, `dispatchNotification`, `calculatePrice`.

**The principle it violates.** Cohesion (one reason to change). Its own docstring needs a list
to say what it is for: it holds rooms, books, prices, notifies, and formats. Coordinating is
its legitimate job (GRASP Controller). The smell is that it *implements* pricing and
presentation itself, while validation and availability were already pulled out into their own
modules.

**What it makes expensive.** Presentation changes. `formatReceipt` is both the printed receipt
and the email body (`dispatchNotification`), so a cosmetic receipt tweak silently changes
confirmation emails. `formatClock` also builds the `BookingError` message. The formatters are
private, so `ReportGenerator` cannot reuse them, and the next agent that needs them will
duplicate them (see Smell 1).

### Smell 3

**The smell.** Speculative over-abstraction: a plugin registry with exactly one plugin. It
also creates a hidden dependency.

**Classic or agent-specific.** Agent-specific, caused by an underspecified request. Nothing
said how many channels were needed, so the agent built a global registry (`builders`,
`registerChannel`, `registeredChannels`, `NotifierConfig`) for a `ChannelName` whose only
option is `'email'`.

**Where in the code.** `src/notifications/notifierFactory.ts`, consumed in the
`ReservationManager` constructor, which calls
`createNotificationChannel(DEFAULT_NOTIFIER_CONFIG)`.

**The principle it violates.** YAGNI: decoupling for a change that never came. The
abstraction also sits at the wrong seam. It is extensible globally but cannot be injected, so
the constructor's signature (only `StorageProvider`) hides a dependency on global state. That
violates explicit coupling and hurts controllability.

**What it makes expensive.** (1) Testing notifications. No test checks them today, and the
only way to substitute a fake channel is `registerChannel('email', fake)`, which mutates
process-wide state that leaks across tests. (2) The very change it was built for. Adding SMS
still means editing `ChannelName`, the factory, *and* the manager constructor, so the
generality does not save that edit.

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** Smell 1, the duplicated pricing. Pricing is the rule most likely
to change in this module, and it is the smell whose failure the suite cannot see today.

**What changed.**
- New `src/pricing.ts` owns the five pricing constants and `priceFor(room, start, end)`. Its
  body is `ReservationManager.calculatePrice` + `applyDiscounts` moved verbatim (same
  rounding steps, same order).
- `ReservationManager.calculatePrice` keeps its public signature and now delegates to
  `priceFor`. Its private `applyDiscounts` and constants are gone.
- `ReportGenerator.revenue` calls `priceFor(room, booking.start, booking.end)`. `priceOf`,
  `durationOf` (only used by `priceOf`, and `noUnusedLocals` would reject it), and the renamed
  constants are gone.

Every price is the same as before. The difference is that each pricing rule now exists once.

**What you deliberately did not touch.** Scope line: pricing rules get one home; nothing
changes *what* is computed.
- `revenue` still re-prices from the room instead of reading `booking.priceCents`. Switching
  would change what the report means (e.g. after a rate change), which is a behavior decision,
  not a refactor.
- No `PricingPolicy` Strategy. There is one policy, so it would be speculative
  over-abstraction (Smell 3's smell).
- The triplicated overlap check (`hasConflict` / `isSlotFree` / `overlapsWindow`) and the
  formatters are left for their own changes. Mixing them in would make this diff harder to
  check.

**How you know behavior is preserved.** 39/39 tests pass, `npm run typecheck` is clean, and
`git status tests/` shows no edits. The pricing tests in `booking.test.ts` pin all four rules
through the manager (12000 / 16200 / 18400 / 11400 cents). The revenue tests check the
report's prices against `priceCents`, but only for the base and evening rules. The suite
would *not* catch a change in rounding order when several rules combine (e.g. premium + long
+ evening), because no test combines them. That is covered by moving the code verbatim
rather than rewriting it. Check: before the fix, changing the premium multiplier or
long-booking cutoff in the report's copy left the suite at 39/39. After it, the same change
in `pricing.ts` fails a test, because there is no second copy left to drift.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A (not coded)

**The problem.** Smell 2, God class / divergent change. `ReservationManager` implements
presentation and notification alongside the booking lifecycle, so unrelated changes (receipt
layout, email wording, clock format) all land in one class.

**The decomposition.** Three pieces, plus `pricing.ts` from Milestone 2.
- `ReservationManager` stays the controller. It owns the room registry and the booking
  lifecycle (create, cancel, queries) and coordinates validation, availability, pricing,
  storage, and notification. It builds no text.
- `bookingFormat.ts` holds pure functions (`formatClock`, `formatMoney`, `formatReceipt`,
  `formatDailySummary`). All presentation rules live here, so `ReportGenerator` and the
  `BookingError` message can reuse them instead of copying them.
- `BookingNotifier` owns turning a booking event into a message, sending it through the
  channel, and the notification log. The email body now has its own home, so it can differ
  from the printed receipt without touching the receipt.

Rules: booking rules in `validation.ts` / `availability.ts`, price rules in `pricing.ts`,
presentation in `bookingFormat.ts`, delivery in `BookingNotifier` + channel.

**One cost.** The public API. `formatReceipt`, `formatDailySummary`, and `recentNotifications`
are public methods on the manager today. Moving them breaks callers. Keeping forwarding
methods avoids that, but then the manager's interface still changes every time a formatter's
signature does, so the divergent change moves out of the implementation but not out of the
interface.

### Proposal B (not coded)

**The problem.** Smell 3, speculative over-abstraction. There is a global plugin registry for
one channel, and the manager cannot be handed a channel, so its notification dependency is
hidden.

**The decomposition.** Keep the real seam, the `NotificationChannel` interface and
`EmailChannel`. Delete `notifierFactory.ts` entirely (`builders`, `registerChannel`,
`registeredChannels`, `ChannelName`, `NotifierConfig`, `DEFAULT_NOTIFIER_CONFIG`). The
`ReservationManager` constructor takes
`notifier: NotificationChannel = new EmailChannel()` next to `storage`, the same pattern it
already uses for storage. Which channel to use is decided by the code that builds the manager.
How to deliver is decided by the channel. A test passes a fake per instance, with no global
state touched. Adding SMS becomes one new `SmsChannel` class, with no edits to the manager or
a union type. As a side effect, the from-address default stops being defined twice
(`emailChannel.ts` and `notifierFactory.ts`).

**One cost.** Choosing a channel by name from configuration goes away. If deployment later
needs to pick email vs. SMS from a config file or environment variable, whoever builds the
manager has to write that `switch` itself. That brings back a small factory, but only in one
place and only once a second channel actually exists.

### The thing that looks smelly but is fine

**What it is.** `src/validation.ts`, `validateReservationRequest`. It looks like a long
method: about 45 lines of `if`s split by section comments, mixing request-shape checks with
building rules.

**Why it is fine.**
- It is a pure function of `(request, room)`: no loops, no shared state, no I/O, no hidden
  dependencies. It is the easiest thing in the module to test.
- Every block is an independent guard clause that returns one reason. It has a single job:
  report the first rule a request breaks.
- The order *is* behavior, and the tests pin it. `{start: 1380, end: 1500}` breaks both the
  single-day rule and opening hours, and `tests/validation.test.ts:23` expects "single day".
  Splitting the checks into separate rule objects would hide that ordering.
- The section comments already work as a table of contents. Extracting each 2-line block into
  its own function would add names without removing a single reason to change. Each rule
  change is still a local 1–2 line edit.

**What would flip your verdict.** Building rules that vary by building or room, such as a
second building with different hours or the cleaning gap from lecture. Opening hours,
15-minute boundaries, and the premium rule are module constants sitting next to universal
shape checks. The first per-building variation would mean `if (building === …)` branches
inside this function. At that point, split universal request-shape checks from a per-building
policy that is passed in. A second flip: callers needing *all* failures at once (e.g. a form),
which the first-failure design cannot give them.
