# Architecture

## Shape

Defined PMU is one Next.js App Router application deployed on Vercel, talking
to a PostgreSQL database on Supabase, Stripe for money and Resend for email.
There is no separate API service and no client-side data layer.

Three properties define the shape more than anything else:

1. **The database is the authority.** Slot conflicts, payment confirmation,
   referral qualification and credit balances are all decided in PostgreSQL,
   not in application code. Application checks exist so the customer sees a
   useful message, not so the invariant holds.

2. **The browser holds no privilege.** One module knows Supabase exists. It is
   marked `server-only`. Business tables live in a schema no role has
   privileges on, and the entire API is a small set of `SECURITY DEFINER`
   functions callable by the service role alone. The publishable Supabase key
   in the client bundle can read nothing.

3. **Private surfaces are capabilities, not accounts.** Customers never create
   an account. A questionnaire link and a Client Card link each carry 256 bits
   of randomness in the URL path, stored server-side only as a SHA-256 digest,
   scoped and revocable.

```mermaid
flowchart TB
  subgraph Browser
    Public[Public site and booking flow]
    Card[Client Card at a capability URL]
    Form[Health questionnaire at a capability URL]
    AdminUI[Practitioner admin]
  end

  subgraph Vercel["Next.js on Vercel"]
    RSC[Server Components and Server Actions]
    Hook[Stripe webhook route]
    Cron[Lifecycle worker route on a daily cron]
  end

  subgraph Data["PostgreSQL on Supabase"]
    Fns[SECURITY DEFINER functions]
    Tables[(Private schema: schedule, bookings, payments, questionnaire, lifecycle jobs, outbox, credit ledger)]
  end

  Stripe[Stripe]
  Resend[Resend]

  Public --> RSC
  Card --> RSC
  Form --> RSC
  AdminUI --> RSC
  RSC --> Fns
  Hook --> Fns
  Cron --> Fns
  Fns --> Tables
  RSC -->|PaymentIntent| Stripe
  Stripe -->|signed event| Hook
  Cron --> Resend
  RSC --> Resend
```

## Booking and payment flow

The important detail is what does not confirm a booking. Selecting a time
confirms nothing. Paying in the browser confirms nothing. The signed webhook
is the only thing that turns a hold into an appointment.

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant S as Next.js server
  participant P as PostgreSQL
  participant St as Stripe
  participant R as Resend

  B->>S: chosen service, day, time, contact details
  S->>P: create hold in one transaction
  P-->>S: hold accepted, or refused by the exclusion constraint
  Note over P: the constraint is the arbiter, not the availability query
  S->>P: read amount due online from the booking snapshot
  S->>St: create PaymentIntent with a booking derived idempotency key
  St-->>S: client secret
  S-->>B: client secret and the amounts the server computed
  B->>St: card, Apple Pay or Google Pay
  St->>S: payment_intent.succeeded, signed
  S->>S: verify signature against the raw body
  S->>P: compare amount and currency, then confirm atomically
  alt confirmed
    P-->>S: appointment exists
    S->>P: claim the confirmation email in the outbox
    S->>R: send
    S->>P: schedule the lifecycle jobs
  else slot taken, cancelled, or amount mismatch
    S->>P: record a payment exception with a deterministic refund key
    S->>St: refund
  end
```

An abandoned checkout stops occupying the calendar the moment its hold expiry
passes, whether or not any cleanup job has run, because the exclusion
constraints only consider entries that are still held or confirmed. A separate
sweep cancels the Stripe PaymentIntents whose holds lapsed, so an abandoned tab
cannot be paid a week later.

## Lifecycle flow

```mermaid
flowchart LR
  State[Authoritative appointment state: confirmed_at, starts_at, completed_at]
  Derive[Lifecycle derivation]
  Jobs[(Durable lifecycle jobs, one per event per booking)]
  Worker[Worker on a daily cron]
  Recheck{Still owed?}
  Outbox[(Email outbox, unique dedupe key)]
  Resend[Resend]
  CardView[Client Card lifecycle state]
  AdminView[Admin lifecycle panel]

  State --> Derive
  Derive --> Jobs
  Jobs --> Worker
  Worker --> Recheck
  Recheck -->|yes| Outbox
  Recheck -->|no| Cancelled[Job cancelled with a reason]
  Outbox --> Resend
  State --> CardView
  Jobs --> AdminView
```

The same derivation feeds three surfaces: the emails, the Client Card and the
admin views. There is one function that answers "where is this client in her
healing, and when does her top-up window open", and everything reads it.

## Defined Credit flow

```mermaid
flowchart TB
  Ref[Referred client books through a referral link]
  Attr[Attribution recorded at booking time]
  Done[Practitioner marks the treatment completed]
  Trig[Database trigger qualifies the referral]
  Grant[Grant of 25 euro, expiring in 12 months]
  Res[Reservation, inside the hold transaction, behind a per customer lock]
  Paid{Payment confirmed by webhook?}
  Redeem[Redemption]
  Rel[Release back to the original lot]

  Ref --> Attr --> Done --> Trig --> Grant
  Grant --> Res
  Res --> Paid
  Paid -->|yes| Redeem
  Paid -->|no, hold lapsed or cancelled| Rel
  Rel --> Grant
```

Every line is an append-only ledger entry. There is no balance column to drift.
Released credit returns to the lot it came from with that lot's original
expiry, so booking and cancelling repeatedly cannot refresh credit that was
about to expire.

## Why these boundaries

**One module owns the database.** Every other file asks it for a typed result.
This is what makes "the browser cannot reach the database" a structural fact
rather than a convention.

**One module owns Stripe.** It reads the secret key, creates intents, verifies
events and issues refunds. Nothing else imports the Stripe SDK.

**One module owns token minting.** Both capability schemes call the same digest
helper, because two hashing implementations eventually disagree about what a
digest is.

**One module owns money formatting and the deposit split.** The split is
computed by the database and rendered in one place, so the booking page, the
confirmation email, the Client Card and the admin panel cannot show different
numbers.
