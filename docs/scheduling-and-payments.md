# Scheduling and payments

## The problem with check-then-insert

The obvious way to book an appointment is to ask whether the slot is free and
then write the row. Under concurrency that is wrong, and it is wrong in a way
that testing on a laptop never shows: two requests both read a free calendar,
both pass the check, and both insert. Nothing in the application is between
them. Wrapping the two statements in a transaction does not help by itself,
because the read sees a state the write then invalidates.

The usual escalations each have a cost. Serializable isolation makes every
booking a retry loop. A table lock makes the calendar single threaded. An
advisory lock on the day works, but it is a convention: it holds only as long
as every future code path remembers to take it.

## What this system does instead

Appointments, checkout holds and manual blocks are all rows in one schedule
table, each carrying two ranges: the appointment itself, and the appointment
plus the reset time the practitioner needs afterwards. Two GiST exclusion
constraints sit on that table:

- **Overlap.** No active entry may overlap another active entry. Bookings,
  live holds and manual blocks all participate.
- **Separation.** The reset between two different clients. This one applies to
  bookings only, so a manual block can sit flush against an appointment while a
  second client cannot.

Both constraints are partial: they consider only entries that are still held or
confirmed. A hold therefore stops occupying the calendar at the instant its
expiry passes, with no cleanup job needing to be punctual for that to be true.
Both ranges are half open, so an appointment ending at 11:30 and one starting
at 11:30 do not collide.

The result is that a double booking is not a bug that has to be prevented. It
is a write PostgreSQL will not accept. The two concurrent requests both reach
the insert, one succeeds and the other receives a constraint violation, which
the application turns into "that time was taken while you were deciding".

The availability query that renders the calendar still exists. Its job is to
give the customer a useful page, not to keep the invariant.

## The rules that sit above the constraints

Duration, price and reset time belong to the service, not to a global setting,
so a shorter top-up can start where a full treatment cannot. Operating days,
the start of the customer facing booking window, the slot interval, the minimum
notice, the first bookable date, the hold duration and the deposit percentage
all live in a single settings row that the practitioner can change.

One subtlety the constraints deliberately do not know about: the studio's
working day. An appointment must finish inside it, but its trailing reset need
not, because the reset is owed to the next client and the last appointment of
the day has none. That check is applied to the appointment's own end, not to
its reset.

Times are stored in UTC and converted at the boundary from Helsinki wall clock,
so "Saturday 09:00" survives the daylight saving transitions rather than
drifting by an hour twice a year.

## Payments

### The browser cannot name a price

There is no amount field anywhere on the path from the browser to Stripe. The
booking flow sends a service, a day, a time and contact details. The server
writes the hold, and in that same transaction the database snapshots the price,
the deposit split and the currency onto the booking. The PaymentIntent's amount
is then read back from that snapshot.

The system uses no Stripe Products and no Prices. The live catalogue was
archived deliberately so that there is exactly one place a price can come from.

### Idempotency

Creating a hold is idempotent on a checkout key, so a retry does not produce a
second hold. The PaymentIntent is created with an idempotency key derived from
the booking's own identifier, so a customer who reloads the payment step
receives the same intent rather than a second one. Refunds carry a
deterministic key derived from the payment intent and the reason, so a
redelivered webhook asks Stripe for the same refund and Stripe answers with the
first one.

### The webhook

The route subscribes to three events: succeeded, failed and canceled. Anything
else is acknowledged and ignored, because a webhook that returns 500 on an
event it does not care about is a webhook Stripe eventually disables.

Order matters in the route:

1. The raw request body is read as text and handed to signature verification
   before any parsing. Parsing first would mean trusting an unauthenticated
   payload, and re-encoding it would break the signature anyway.
2. A 200 response means "do not send this again". A handled event, an ignored
   event and a deliberate refusal all answer 200. Only a genuine transient
   fault answers 500, so Stripe retries exactly the cases where retrying helps.
3. Nothing that could identify a person is logged. Event types, references and
   outcomes only.

### Money in hand and no appointment to give

This is the case that decides whether a payment integration is serious. A
customer can be mid payment while somebody else takes the slot. When the
webhook arrives, confirmation is attempted atomically and the same exclusion
constraint refuses it.

The system does not override the constraint. It records a payment exception
first, then refunds. First, deliberately: a crash between the two leaves a
record that somebody paid and has nothing, which is recoverable, rather than a
refund nobody can explain.

The same path handles a payment whose amount or currency does not match the
booking's snapshot, a payment carrying no booking reference, and a payment for
a booking that was cancelled. All of them refund. None of them confirm.

Both the customer and the studio are emailed when this happens, because a
silent refund is a customer who thinks she has an appointment.

### Deposit and balance

Fifty percent is taken online, the balance is settled at the studio, and each
booking snapshots whichever percentage was in force when it was made. The
studio records the balance in admin as cash, card or other, so the appointment
carries its own settled state rather than the practitioner keeping it in her
head.

### The kill switch

A single environment variable decides whether a deployment may take money, and
it is opt-in: only the exact string `true` opens payments. Unset, empty,
mistyped or forgotten during an environment migration all mean closed, because
a deployment that silently charges cards is the worse failure. Three separate
server-side paths ask it: before a hold is written, before a PaymentIntent is
created, and in the webhook before it will even look at a signature.
