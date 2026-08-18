# Testing and reliability

## What is actually run

Ten verification suites, plus TypeScript, ESLint and a production build. The
suites are custom runners rather than a test framework: each one prints named
sections and individual checks and exits non-zero on the first failure.

Full run on 18 August 2026, against a freshly reset local PostgreSQL stack and
the Stripe sandbox:

| Suite | Checks | Needs a database |
| --- | --- | --- |
| Scheduling rules | 112 | no |
| Booking engine | 94 | yes |
| Payments | 134 | yes, plus Stripe sandbox |
| Email | 224 | yes |
| Questionnaire | 127 | yes |
| Admin | 246 | yes |
| Lifecycle | 278 | yes |
| Client Card and referrals | 191 | yes |
| Consent | 72 | yes |
| Defined Credit | 90 | yes |
| **Total** | **1,568** | |

All passed. `tsc --noEmit` passed, ESLint reported nothing, and the Next.js
production build succeeded.

The count is checks, not test files and not assertions in a framework. Calling
them 1,568 tests would be the wrong word.

## What the suites cover

Rather than list every check, here is the kind of thing each suite proves.

### Scheduling

The studio's locked rules, with no database in the way: per service price,
duration and reset time; the 09:00 and 18:00 start boundaries; that an 18:00
treatment is a valid final appointment because the reset is owed to the next
client and there is none; collisions between two treatments, a treatment and
each length of top-up, and top-up against top-up; adjacent bookings and full
days; weekend only operation, the minimum notice and the opening date;
Helsinki wall clock to instant conversion and DST-safe date arithmetic; busy
intervals that straddle a Helsinki day boundary.

### Booking engine

The same rules again, but asserted against a real PostgreSQL, because the
guarantee is the database's. Two people booking one slot at the same moment.
Holds expiring on the clock rather than on a cron job. Releasing and cancelling
returning the time. Manual blocks. Money in integer cents. Idempotent hold
creation. Confirming a payment the way the webhook does. That availability
tells a customer only whether a time is free and nothing about who has it. And
that the publishable key cannot create a booking, read occupancy, read the
schedule, or list customers or bookings.

### Payments

This suite creates real PaymentIntents, real charges and real refunds in the
Stripe sandbox. It refuses to start against a key that is not test mode, an
account that is not the studio's sandbox, or a database that is not local.
There is no override for that refusal, because this suite charges cards.

It proves that payments can be switched off server-side and fail closed; that
the price and the split come from the database rather than from React; that the
architecture does not assume 50 percent; that a double submit produces one
PaymentIntent and not two; that the webhook is what makes an appointment; that
the same event again changes nothing; that money for a slot somebody else now
holds is refunded, and that redelivering that event refunds it once rather than
twice; that an amount which is not the one owed is refused and refunded; that a
cancelled payment returns the slot; that an abandoned checkout reopens the slot
without a cron job; that unsubscribed event types are ignored; that signature
verification behaves; and that the public hold endpoint is rate limited.

The late payment race is reproduced without waiting fifteen minutes, by writing
a hold whose expiry is already past, so the next transaction legitimately takes
the slot and the first customer's payment lands too late. That is the real
sequence, with no test-only branch in the code under test.

### Email

Every template renders completely. Customer emails carry nothing internal.
Interpolation is escaped. The confirmation says the right things and only
those. The outbox claims a send exactly once, outcomes are recorded and cannot
be rewritten, and unfinished work is findable while finished work is not.

### Questionnaire

That the removed step is gone rather than hidden. That there is no generic
start URL and the form has one address which carries a token. Server-side
validation applying the same rules to everything. That a link is issued only
for a confirmed appointment. What is in the URL and what is not. Expiry,
revocation, and idempotent submission. That consent is required by the database
and not only by the form. And that the answers cannot be reached without the
studio's own key.

### Admin

That without a session there is nobody, and that holding an auth account is not
the same as being an operator. Sessions, deactivation ending access
immediately, and that a customer capability is not an operator capability. The
appointment list, the payment snapshot rather than today's price list,
questionnaire status without the answers, studio notes, idempotent completion,
blocking the practitioner's own time, rescheduling, the client directory, the
questionnaire queue, reissuing links, manual emails, the balance collected at
the studio, refunds, cancelling without touching the money, the attention
queue, grouped payments, settings, the audit trail, and that no public key
reaches any of it.

### Lifecycle

That the timings are stated in two places and agree. That PostgreSQL and
TypeScript compute the same send instants, DST included. That a booking made
well ahead gets all three pre-appointment messages and one made for tomorrow
gets one rather than three. That scheduling is idempotent. That a reschedule
recalculates every appointment-relative message. That the questionnaire
reminder stops the moment the form arrives. That cancelling stops everything
the appointment owed. That nothing post-treatment exists until the practitioner
says so, that completing twice does not schedule twice, and that undoing a
completion withdraws the chain. That the top-up window is four to six weeks
from completion, that a top-up treatment does not itself start a new top-up
lifecycle, and that a review is asked for once per client rather than once per
treatment. That the worker sends each due message once.

### Client Card and referrals

That the card's SQL cannot reach what the card must not show. That card tokens
are digests. That following a referral link reveals nothing about the referrer.
That the private routes say they are private and nothing logs a token. That the
QR code carries the referral link and nothing else. That a card link opens
exactly one client's card and that refusals say as little as possible. That
lifecycle state is read rather than recomputed. That a referral identity is one
per client. That attribution is one booking, one referrer, first link wins.
That payment does not qualify a referral and only a treatment does, and that a
qualification which stops being true stops counting.

### Consent

That the record a device carries is what it should be, that a visitor is not
asked again once they have answered, that a raw `Cookie` header is read
correctly before anything is written, and that the invitation handover works.
That no answer weighs more than the other in the interface. That the payment
provider script waits to be asked. That the optional cookie carries only what
it should and that a referral token is not a client card token. And, against
the database, what a live invitation does before anybody has been asked and
after each possible answer.

### Defined Credit

That the browser cannot name an amount. That one place computes a balance and
the 25 euro is one number. That credit is not money and no screen says it is.
That the referred client is never named to the referrer. That a completed
treatment is what earns credit and a qualification undone takes it back. That
the summary adds up. That the online payment never moves. That an abandoned
hold gives credit back, that paying makes it a redemption exactly once, that
the studio balance is the credit adjusted one, that rescheduling keeps credit
where it is, and that cancelling before treatment returns it. That credit is
capped at what is owed in the room, that it cannot be taken by asking nicely,
that two tabs cannot spend the same credit, that expired credit stops counting,
and that the practitioner's correction path works.

## Reliability decisions that are not tests

**Fail closed by default.** Payments require an explicit opt-in string. The
lifecycle worker refuses everything if its shared secret is unset. A missing
webhook signing secret answers 500 rather than 200, so Stripe holds the event
and redelivers it once the deployment is fixed.

**Retry the right things.** The webhook returns 200 for handled, ignored and
deliberately refused events, and 500 only for genuine transient faults, because
Stripe retries with backoff for days and that behaviour is right for a briefly
unreachable database and wrong after a refund has been issued.

**Write the record before the irreversible act.** A payment exception row is
created before the refund is requested. A crash between the two leaves
something recoverable rather than a refund nobody can explain.

**Make the second attempt boring.** Holds, PaymentIntents, refunds, emails,
lifecycle jobs and credit releases all carry deterministic keys, so redelivery,
retry and double submit produce the same state as the first attempt.
