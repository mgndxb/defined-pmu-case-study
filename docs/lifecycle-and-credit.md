# Lifecycle automation and Defined Credit

## Why the lifecycle engine exists

A permanent makeup treatment is not a single appointment. It is a health
questionnaire that has to arrive before the client sits down, preparation notes
that are useless if they arrive too early or too late, a reminder, aftercare
that matters most in the first days, a healing check, and a top-up window that
opens weeks later and closes.

Before this system the practitioner remembered all of it. The engineering
problem is that "remembering" has to become a property of data, not of a
crontab and not of somebody opening a page.

## Derivation, not scheduling by hand

Every message derives its due time from authoritative appointment state: the
confirmation timestamp, the appointment start, or the completion timestamp.
Nothing is scheduled from a guess and nothing is triggered by a page view.

Before the appointment, counting back from the day itself: the questionnaire
chase at three days, preparation notes at two, and the reminder the day before.
For a booking made at short notice these collapse to one, because a "how to
prepare" arriving four minutes after a confirmation is worse than no
preparation email at all.

After the practitioner records the treatment as completed: aftercare
immediately, a healing check at seven days, a review request at fourteen (once
per client, never again), a heads-up at twenty-one that the top-up window opens
in a week, and the window itself at twenty-eight, closing at forty-two.

Timing is expressed in the studio's own time zone. Dates are computed in
Europe/Helsinki and stored in UTC, so the transitions in and out of daylight
saving do not move an appointment reminder by an hour.

The sending hour is a property of the data, not of the crontab. The worker
sends everything already due, so a sweep at 08:00 and a sweep at 14:00 produce
the same messages.

## Durability

Jobs are rows in a table, not timers in a process. Three separate mechanisms
make the whole thing safe to run twice, and it takes all three:

1. **The claim.** The worker claims due rows with `FOR UPDATE SKIP LOCKED`, so
   two overlapping runs divide the queue rather than waiting on each other or
   sending the same message twice.
2. **The job key.** One job per event per booking, forever, enforced by a
   unique index. No amount of re-derivation produces a second aftercare.
3. **The outbox key.** Sending is a row in an email outbox with a unique
   deterministic key. Even if a job row were somehow rebuilt, the send has
   already been claimed and the second attempt reports a duplicate rather than
   sending. The same key is handed to the email provider as its own idempotency
   key.

Running the worker by hand while the cron is running sends nothing twice.

## State is re-asked at send time

A job that was scheduled a week ago is a job whose reason may no longer be
true. In the second before sending, the worker re-asks the authoritative
questions: is the appointment still confirmed, has the questionnaire already
arrived, was the completion undone, has this client already been asked for a
review. A job that fails any of them is cancelled with the reason recorded, and
the reason is what the practitioner sees in admin.

Cancelling or rescheduling an appointment recalculates its jobs. A cancelled
appointment cannot be reminded about.

Failures are per job. A bounced address fails its own row, is retried later,
and the rest of the sweep carries on. After five attempts the job is left for a
person, visible in admin rather than silently dead.

## The email outbox

Email is a row before it is a side effect. Enqueueing inserts on a unique
dedupe key and reports whether it created the row or found one, which is what
makes a redelivered Stripe webhook an hour later do nothing. Provider message
ids and failure reasons are stored, so the studio can see what was sent, what
bounced and what is being retried, instead of trusting that a function ran.

Fifteen transactional templates exist, all in the studio's own visual language:
booking confirmation, questionnaire link, questionnaire reminder, pre-care,
appointment reminder, aftercare, healing follow-up, review request, the two
top-up messages, reschedule, cancellation, Client Card access, and payment
exception messages to both the customer and the operator.

## Defined Credit

### The rule

One qualified referral earns the referring client 25 euro of Defined Credit.
Credit is stackable, expires twelve months after it is earned, has no cash
value and is not transferable. The client chooses whether to use it.

It reduces the amount due at the studio and never the deposit required online.
A 299 euro treatment with 50 euro of credit is still 299 euro: 149.50 online,
50 in credit, 99.50 in the room. That is a deliberate business rule, and it
also means credit can never turn into a Stripe refund of real money.

### Qualification

A referral is attributed when the referred client books, and qualifies only
when the practitioner marks the treatment as completed. The qualification is
written by a database trigger on the completion transition, and by a second
trigger for the case of an appointment cancelled after it was marked done.
Between them there is no route to a qualified referral that does not pass
through a treatment the practitioner recorded as having happened. No
application path can grant credit.

Self referral by the same customer record is refused, duplicate attribution is
refused, and attribution only exists at all for a visitor who consented to it.

### Why a ledger and not a balance column

A single mutable balance is the obvious design and the wrong one. It has no
history, so "why does she have 25 euro" is unanswerable. It drifts, because
every path that touches it has to get the arithmetic right. It cannot express
expiry, because 25 euro earned in March and 25 euro earned in September are not
the same 25 euro. And it makes concurrent spending a read-modify-write race.

So there is no balance column anywhere. There are lines: grants, reservations,
releases, redemptions, adjustments and reversals. Every figure shown on the
Client Card, in admin, in the booking flow and in the confirmation email is a
sum over those lines, computed by one view and nowhere else. Amounts are always
positive magnitudes and direction is carried by the kind of line, because a
signed column is a place to get a minus sign wrong.

### Spending, and two tabs

Credit is spent oldest expiry first, so a client always spends the credit that
was about to be lost rather than the credit she has had longest.

Reserving is the concurrency-sensitive part. Reading a balance and then writing
a reservation is exactly the check-then-act race that lets a client with 25
euro spend it in two browser tabs. So reserving happens behind a transaction
scoped advisory lock keyed on the customer, inside the same transaction as the
checkout hold: the second checkout waits, then reads a balance that already
includes the first one's reservation. A hold that fails takes its reservation
down with it.

A reservation becomes a redemption only when the webhook confirms the payment.

### Giving it back

Every path by which a booking stops being a live appointment (an abandoned
checkout, a lapsed hold, a cancellation by the client, a cancellation by the
studio) releases the credit. The release is idempotent by construction: it acts
only on lines nothing has superseded yet, so a second call finds none.

Credit returns to the lot it came from, with that lot's original expiry.
Minting a fresh lot with a fresh year on it would mean a client could refresh
expiring credit indefinitely by booking and cancelling, and the studio would
have written that rule by accident. The one case this handles badly, a
cancellation so late that the lot has already expired, is handled by a person:
an admin adjustment that takes a reason and writes to the audit log.

Credit restoration is independent of the Stripe refund path. Money and credit
are separate ledgers and neither is ever converted into the other.

### What the browser is allowed to say

One boolean: use my credit. There is no amount field in the request, in the
server-side input type, or in the database function beneath either. Whose
credit it is comes from the Client Card's own handover, and the database
refuses to apply any credit unless that handover's customer is the customer the
booking resolved to.
