# Production lessons

Real incidents from taking this system live, and what each one changed. No
secrets, hostnames, account identifiers or deployment identifiers appear here.

## A credential that only failed in production

**What happened.** The production site answered "Dates cannot be shown at the
moment" on the booking page. The same database answered perfectly from a
laptop, using the same code.

**Diagnosis.** The hosted deployment had been given the older of the two key
formats the database platform issues. That format is a signed token carrying
timestamps, and the API gateway was rejecting it because its issued-at claim
was in the future relative to the gateway's clock. The newer key format carries
no timestamp and cannot fail that way, which is exactly why the fault was
invisible everywhere it was convenient to look: local development used the
newer key, so the failure existed only in the one environment that mattered.

**Lessons.**

- A platform that publishes two credential formats will eventually have one of
  them used somewhere it should not be. Pick one, write down which, and make
  the choice checkable.
- Clock-dependent credentials fail in ways that look like nothing else. "Works
  locally, fails in production, same code, same database" is a strong signal to
  compare the credential rather than the code.
- The more important lesson was about the failure mode, not the credential. The
  availability path fails soft: it shows the customer a polite sentence and the
  operator nothing, and the page still returns 200. That is good for the
  customer and terrible for detection. The deployment documentation now names
  this explicitly as a path where a green status code means nothing.

## A previous vendor's webhook was still subscribed to our events

**What happened.** During the pre-launch audit of the payment account, an old
webhook destination was found, left over from a site built on a different
platform before this system existed. It pointed at that platform's serverless
function, and it listened to the exact event the new system depends on:
payment intent succeeded.

**Why it mattered.** The first real customer payment would have been delivered
to both destinations. The old function was still answering when it was checked,
so this was a live hazard rather than tidying up.

**Resolution.** The old destination was disabled rather than deleted, so the
history stays and the change is one click to undo. The old integration's API
key was left alive but marked for revocation alongside decommissioning the old
project, because revoking a key is irreversible and was not required to open
payments.

**Lessons.**

- Payment provider accounts accumulate state that no repository knows about:
  destinations, keys, products, prices, domain registrations. Audit the account
  itself before enabling payments, and write down the result.
- Migrating to a new implementation is not finished when the new one works. It
  is finished when the old one can no longer receive anything.
- Prefer the reversible action when the irreversible one buys nothing today.

## Email authentication, and the record that was nearly deleted

**What happened.** The DNS documentation listed two records among the leftovers
from the previous vendor, to be removed once that plan was cancelled. They were
not leftovers. They were the studio's own signing records for the current email
provider, and the published value matched what the provider showed character
for character. Deleting them, as the document invited, would have stopped the
studio's mail.

**What was genuinely stale** was removed with the old values written down first
so the removal is reversible by typing: two verification records, and a live
nameserver delegation to the former vendor. The delegation was the one that
mattered, because a delegation to somebody else's nameservers is a standing
permission to publish mail authentication inside a namespace that reads as the
studio's.

**Two decisions that were deliberately not taken.** The sender policy record
was left as it is: it does not authorise the bulk mail provider the email
service sends through, and it does not need to, because the envelope sender is
a subdomain with its own record and the signature aligns. Adding a blanket
include for that provider would let every one of its customers send as this
domain in exchange for nothing. And the reporting policy was left at monitoring
rather than tightened, because tightening a policy before anybody has read a
report is how a business starts losing its own mail quietly.

**Lessons.**

- Ownership of a DNS record is a fact to verify against the provider, not a
  guess from its name. A cleanup list is a hypothesis until each line is
  checked.
- Write down the current value before deleting anything in DNS. Reversibility
  is cheap in advance and expensive afterwards.
- The dangerous leftovers are the ones that delegate authority, not the ones
  that look untidy.

## Verifying write-only environment variables

**The problem.** Sensitive environment variables in the hosting platform are
write-only by design: no dashboard, CLI or API can read them back. That is the
right setting, and it means the only honest way to know whether the four email
addresses in production were the intended ones was to make the deployed
application use them and see what arrived.

**What was built.** One temporary POST route. It read no request body at all.
The recipient was not a parameter: it was resolved from the environment at send
time. The subject and body were constants. Somebody holding the secret could
cause the studio to receive an email whose text they already knew, and nothing
else. It failed closed twice over: with no secret configured it answered 404
before reading a header, and a secret below a minimum length counted as no
secret, so a wrong key and a disabled route gave identical answers.

**What happened next.** Production sent one message with the real credential.
It arrived, the four addresses were confirmed, and sender authentication passed
on the way in. The one-time secret was then removed from the environment, which
made the route inert, and the route itself was deleted.

**Lessons.**

- A temporary diagnostic endpoint should be designed as though it will be
  found, because it will be. No parameterised recipient, no body, no reflected
  content, indistinguishable failure modes.
- Commit it rather than smuggling it through a working copy. A backdoor that
  was never in the history is a backdoor nobody can audit afterwards. Both
  halves of the removal, the secret and the code, are separate commits with
  reasons.
- When a platform makes a value unreadable, verification has to come from
  observed behaviour rather than from inspection.

## Deployments upload a working copy, not a commit

**What happened.** Production is deployed with the platform CLI, which uploads
the working directory rather than a commit. A working copy contains the local
environment file, and any value prefixed for browser exposure is read from a
file on disk long before platform variables matter. An uploaded local
environment file would therefore have inlined the wrong publishable key into
the production client bundle.

**Resolution.** An explicit deployment ignore list, covering local environment
files and local database state, with the reason written in the file.

**Lesson.** Know which artifact your deploy command actually uploads. "It is in
`.gitignore`" is not the same as "it will not be deployed" when the deploy path
does not go through Git.

## Scanning the client bundle before opening payments

Before the payment switch was turned on, every JavaScript file that the
homepage and the booking page load was downloaded from the production
deployment and searched for secret key prefixes, webhook signing secrets,
service role keys, database hostnames, legacy token prefixes and internal
salts. None were present. The only key in the bundle was the publishable one.
Source maps are not served.

**Lesson.** The interesting question is not whether the code looks correct. It
is what the deployed bundle actually contains. That is a check to run against
the real deployment, and it takes minutes.

## Going live without a smoke payment

The payment switch was turned on without the owner making a test payment
against the live account. That was a deliberate choice, and it left exactly one
thing unproven: the live signing secret had never verified a real Stripe
signature, because doing so requires an event Stripe actually signed.

Everything that could be proven without creating money was: the database row
counts before and after, the deployed bundle containing the correct account's
publishable key, the webhook destination subscribing to exactly three events
with the API version matched to the SDK, both failure answers from the webhook
route (missing signature and invalid signature, which together prove the switch
is on and both secrets are present), a real email through the production
credential, and every public route responding.

What replaced the missing test was written down as a control rather than as
advice: a named person watching the first real customer payment, checking one
payment intent and not two, the delivery and its 2xx response, the confirmed
booking, the slot disappearing from the calendar, zero payment exceptions and
exactly one confirmation email. Plus a two-line rollback that flips the switch
back and redeploys.

The rollback procedure ends with an instruction: do not ask the customer to pay
again. The money and the event both already exist on the provider's side and
the confirmation path is idempotent, so redelivering the existing event is the
repair. A second charge is a second problem.

**Lessons.**

- When you consciously skip a test, name what is unproven in the same document
  that records everything that passed. An honest gap is manageable; a forgotten
  one is not.
- A control can replace a test if it is specific enough to execute under
  pressure. "Watch it carefully" is not; a numbered list with a rollback is.
- Design the recovery path before you need it, because the tempting wrong move
  under pressure (charge the customer again) is obvious and fast.

## Things only a browser found

Two issues came out of clicking through the real checkout, and neither could
have come from a test suite.

The payment provider's saved-details feature, enabled by default, opened a form
inside the studio's checkout asking for an email, a phone number and a full
name that had all been collected on the previous step, in order to create an
account with a third party the customer had not come to. It was disabled both
in the account and in the element options. The device wallets were kept,
because they are the device's own and ask for nothing.

And an abandoned checkout gave its slot back on the clock but left a payable
client secret sitting in a tab. Paying it a week later was already safe, since
the late payment path refunds rather than double books, but safe is not the
same as impossible. A sweep now cancels the payment intents whose holds lapsed
half an hour earlier, which is long enough that nobody is still finishing a 3-D
Secure challenge.

**Lesson.** Automated checks prove the rules you thought of. Some defects are
only visible as an interface a stranger is looking at, and the fix for the
second one closed a window rather than a bug.
