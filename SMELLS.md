# reservation-service: Smells and One Fix

---

## Milestone 1: Three smells

### Smell 1 — pricing rules implemented twice

**The smell.** Duplicated code leading to shotgun surgery. All four pricing rules (hourly base,
premium surcharge, >=3h discount, evening discount) and all five of their constants exist in two
files, under different names.

**Classic or agent-specific.** Agent-specific. The values and the order of operations are identical,
but every constant was renamed (`PREMIUM_MULTIPLIER` -> `PREMIUM_RATE_MULTIPLIER`,
`LONG_BOOKING_MINUTES` -> `LONG_BOOKING_CUTOFF`). The second file re-derived the rules from the spec
instead of importing them — what you get when each file is written in its own pass with no
cross-file reuse.

**Where in the code.** `ReservationManager.calculatePrice` / `applyDiscounts`
(`src/reservationManager.ts:11-15, 140-158`) vs `ReportGenerator.priceOf`
(`src/reportGenerator.ts:4-8, 104-117`). `ReportGenerator.revenue` recomputes a price that the
`Booking` already carries in `priceCents`.

**The principle it violates.** Single source of truth: a pricing rule is one decision and belongs in
one place. Secondarily, a report should read recorded facts, not re-derive them.

**What it makes expensive.** Already wrong today. `revenue` prices bookings from the room's *current*
`hourlyRateCents`, so raising a rate rewrites history: I booked at $120.00, raised the rate, and the
revenue report for that same booking read $180.00 while its receipt still said $120.00. The suite
cannot catch it — `tests/reporting.test.ts:36` asserts revenue equals `priceCents` in a run where
nothing changes, pinning the coincidence that the two copies agree rather than the rule itself.

### Smell 2 — a cache with a read path and no write path

**The smell.** Dead code and speculative generality: a whole caching layer wired in one direction.

**Classic or agent-specific.** Agent-specific. The component and its call site were built in separate
passes and the loop never closed, leaving plausible-looking infrastructure for a requirement nobody
stated.

**Where in the code.** `ReservationManager.listBookingsForRoom` (`src/reservationManager.ts:117-124`)
calls `this.cache.get()`; `this.cache.set()` appears nowhere in `src/`. `QueryCache.set`,
`invalidate`, `size` and `cacheConfig.withTtl` / `disabled` have no production caller.

**The principle it violates.** YAGNI, and code should not lie: the method advertises a caching
strategy that provably never caches, so every `get` misses and the key construction is pure overhead.

**What it makes expensive.** The obvious fix is a trap. Adding the missing `set()` ships a stale-read
bug immediately, because nothing invalidates on `createBooking` or `cancelBooking`. First to break is
`formatDailySummary`, which reads through `listBookingsForRoom` and would print a cancelled booking
and a stale `Confirmed total`, while `findAvailableSlots` and `ReportGenerator.occupancy` read
storage directly and would then disagree with it.

### Smell 3 — presentation living inside the domain class

**The smell.** Mixed levels of abstraction. Clock and currency formatting sit in the class that owns
the booking lifecycle, and non-display code depends on them.

**Classic or agent-specific.** Classic. This is the ordinary drift of a class that starts as the
entry point and accretes whatever callers found convenient.

**Where in the code.** `formatReceipt`, `formatDailySummary`, `formatClock`, `formatMoney` in
`src/reservationManager.ts:176-228`; plus `createBooking` building its conflict message with
`formatClock` (`src/reservationManager.ts:74`) and `dispatchNotification` sending `formatReceipt`
output (`src/reservationManager.ts:216`).

**The principle it violates.** Separation of concerns, argued by reasons to change: "how a receipt
reads" and "when a booking is legal" are separate decisions, owned by different people and changing
on different schedules. The problem is the coupling, not the size.

**What it makes expensive.** Moving to a 12-hour clock or a non-USD currency means editing the
booking engine, and it silently rewrites two things that are not display: the text of `BookingError`
and the body of every notification. First to break is `tests/booking.test.ts:128`, which asserts the
literal `Confirmed total: $234.00` — a pure formatting change fails a test that is nominally about
booking.

---

## Milestone 2: One small fix

**Which smell you attacked.** Smell 1, the duplicated pricing rules. It is the only one of the three
that is producing wrong output today rather than just making a future change expensive, and the
smallest honest fix is a deletion rather than a new abstraction.

**What changed.** One file, `src/reportGenerator.ts`: deleted `priceOf`, `durationOf` and all five
duplicated pricing constants, and `revenue` now reads `booking.priceCents` — the price the organizer
was actually quoted and charged — instead of re-deriving it from the room's current rate. The pricing
rules now exist in exactly one place, `ReservationManager.calculatePrice`. Net 4 insertions, 29
deletions.

**What you deliberately did not touch.** I did not extract a shared pricing module. The duplicate is
gone because one copy had no business existing: a revenue report should read the recorded price, not
recompute it. Extracting `pricing.ts` would have kept two callers of the rules alive and made the
change bigger while fixing less. I also kept `revenue`'s existing guard that skips bookings whose
room was not handed to the generator — swapping `find` for `some` preserves that filter exactly,
because removing it would have changed which bookings count, and that is a behavior change no test
pins down. Pricing living inside `ReservationManager` is Smell 3's territory and stays for Milestone 3.

**How you know behavior is preserved.** All 39 tests pass and `npm run typecheck` exits 0, with no
test file edited. The suite covers this fix well: `tests/reporting.test.ts` asserts `totalCents`,
`averageCents` (11700) and `byRoom` (`{ r1: 23400 }`) across a normal and an evening booking, and
checks that cancellations drop out — so any arithmetic drift would fail. What it would not catch is
the divergence itself, since it only ever compares the two implementations in a run where nothing
changes. I verified that separately in a scratch test: before the fix, a booking charged $120.00
appeared in the revenue report as $180.00 once the room rate was raised; after the fix both read
$120.00.

## Milestone 3: Two proposals and one false positive

### Proposal A (not coded) — the half-wired cache

**The problem.** Smell 2. `ReservationManager.listBookingsForRoom` reads through `QueryCache` but
nothing ever writes to it, and the layer cannot be completed where it sits: reads happen in the
manager while writes happen in `StorageProvider`, so no single class is in a position to keep the two
consistent. That is why the missing `set()` is a trap rather than a one-line fix.

**The decomposition.** Remove the cache from `ReservationManager` — it goes back to not knowing a
cache exists. If caching is actually needed later, add a `CachingStorageProvider implements
StorageProvider` that wraps another provider. It owns both halves: it caches `findByRoom` / `findAll`
and invalidates on the `save` / `update` calls that flow through the same object. Reads and their
invalidation live in one class, and the cache becomes a composition choice at construction
(`new ReservationManager(new CachingStorageProvider(new InMemoryStorageProvider()))`) rather than a
hidden field. `QueryCache` stays as the plain expiry map it already is; `cacheConfig` keeps the knobs.

**One cost.** The decorator can only key on storage-level queries, not on the higher-level results
callers actually repeat — `formatDailySummary` still recomputes its rows and totals on every call. It
also makes the defensive copying in `InMemoryStorageProvider` load-bearing in a new way: a decorator
that hands out its cached array directly reintroduces exactly the aliasing bugs the current provider
copies to avoid, so the wrapper has to copy on every hit and gives back part of what caching bought.

### Proposal B (not coded) — presentation inside the domain class

**The problem.** Smell 3. Clock and currency formatting sit in the class that owns the booking
lifecycle, and two things that are not display depend on them: the text of `BookingError` and the body
of every notification.

**The decomposition.** A `receipts.ts` module of pure functions — `formatClock`, `formatMoney`,
`formatReceipt(booking, room)`, `formatDailySummary(bookings, room)` — taking the data they render as
parameters instead of reaching into `this.rooms` and `this.storage`. `ReservationManager` keeps the
lifecycle only: register, create, cancel, price, conflict detection. Presentation rules live in
`receipts.ts`; `BookingError` stops interpolating a display clock and carries structured fields
(`roomId`, `start`, `end`) so the caller decides how to render them; `dispatchNotification` gets a
formatter passed in rather than calling back into the manager.

**One cost.** This is not behavior-preserving, so it could not have been Milestone 2's fix. Changing
`BookingError` to structured fields changes the message text, and `tests/booking.test.ts` asserts on
`toThrow('already booked')` and the literal `Confirmed total: $234.00` — the refactor requires editing
tests. It also costs callers ergonomics: the simplest use case now holds two things, a manager and a
formatter, where one object used to answer both questions.

### The thing that looks smelly but is fine

**What it is.** `validateReservationRequest` in `src/validation.ts:14-63` — one function, eleven
sequential guard clauses, about fifty lines. It reads like a textbook Long Method.

**Why it is fine.** It is a pure function of `(request, room)` with no state and no collaborators, so
there is nothing to untangle. Every branch is flat — zero nesting, each one returns immediately — so
the cyclomatic complexity is spread across independent checks rather than interacting paths. The
ordering is the one real coupling and it is deliberate and documented: shape before times before
duration before house rules, so the first failure reported is the most fundamental one, which is what
lets `tests/validation.test.ts` assert exactly one reason per bad request. Every threshold is a named
constant at the top, and the whole function has one reason to change: this building's booking rules.
Splitting it into `validateTimes` / `validateCapacity` / `validateHouseRules` would add three names and
force a policy for combining their results without removing a single decision.

**What would flip your verdict.** A rule that varies by something other than the request — premium
rooms opening later, a department granted a longer maximum duration, different hours on weekends. The
first such rule turns the flat sequence into branching on room or tenant type, and then the shape
should become a list of rule objects, `(request, room) => ValidationResult`, composed by a policy that
can differ per room class. A caller that needs *every* failure rather than the first would flip it too,
since returning on the first problem is baked into the current design.
