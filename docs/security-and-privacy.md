# Security and privacy

This document describes design decisions. It deliberately omits the details
that would help somebody attack the running system.

## Capability links instead of accounts

A permanent makeup client books once or twice a year. Making her create an
account, remember a password and reset it eleven months later is friction that
buys the studio nothing. So the two private customer surfaces are capability
URLs.

Both work the same way:

- 256 bits from a cryptographically secure random source, encoded into the URL
  path.
- The server stores only a SHA-256 digest. The token itself exists in one email
  and on one device, and is never written down anywhere on the server.
- Lookup is by digest, which means a database dump does not yield working
  links.
- Tokens are revocable and reissuable, and revoking revokes every outstanding
  token for that subject at once.
- One helper produces every digest in the system, because two hashing
  implementations eventually disagree about what a digest is.

The two differ in what they are scoped to, and the difference is a product
decision rather than a technical one. A questionnaire token opens exactly one
booking's questionnaire and nothing else. A Client Card token is scoped to the
client rather than to an appointment, because a card that expired with the
appointment would stop working at the moment it becomes most useful.

## Health data

Health answers are a special category under GDPR Article 9, and the system
treats that as an architectural constraint rather than a label.

- Answers live in their own table that exactly one database function reads.
- They are excluded from the ordinary admin appointment listings. The
  practitioner opens them deliberately, on a separate view.
- They never reach a URL, a log, an analytics event, an email body, or Stripe
  metadata.
- The operational facts the studio needs day to day (has she filled it in,
  when, which version, what did she agree to) are stored separately from the
  clinical answers, so every routine question is answerable without opening a
  medical record.
- Submission is versioned and validated server-side, and is idempotent, so a
  double tap on a phone produces one submission.
- The software flags answers for the practitioner to read. It never decides
  suitability. That is a clinical judgment and the code does not make it.

The clinical content belongs to the practitioner. When she instructed that the
emergency contact step be removed, it was removed: not hidden, not made
optional, no dormant fields left in the schema.

## Admin authentication

Passwords are handled by Supabase Auth, which hashes them, rate limits sign-ins
and knows how to change one. No password hash is stored or invented by the
application.

The session is the application's own. What the auth provider returns on a
successful sign-in is discarded immediately, and the application mints its own
random session secret, stores only its digest, and puts the secret in an
httpOnly cookie scoped to the admin path. Three reasons:

- A leaked cookie extends to one row in one table in one application, rather
  than being a token any provider endpoint would honour.
- Signing out is an update to a row. A stateless token cannot be recalled; a
  row can, including every row belonging to a deactivated operator.
- Server Components cannot set cookies, so a refresh dance has nowhere to live.

Holding an auth account is not the same as being the practitioner. Session
creation refuses any account without an active row in an operator allowlist,
and admission is granted deliberately from a command line script.

Every mutation in admin writes an audit line. There is no way to write one from
outside the function that performs the action.

## Consent

The consent system is first party and has no vendor in it.

Exactly one optional category exists today: remembering, for thirty days, that
a visitor arrived through a client's referral link. Everything else the site
stores is there because the visitor asked for the thing that stores it, so it
is not presented as a choice. A category a visitor cannot refuse is not a
choice, and printing it as one is theatre.

A visitor who chooses essential only can browse, book and pay. Nothing about
the booking path depends on the optional category.

The interesting case is a first time visitor arriving on a referral link before
the question has been asked. The cookie cannot be written, and the token cannot
be discarded either, because agreeing one second later should still credit the
client who made the introduction. So the token is carried to the landing page
in the URL fragment, which never reaches a server and never appears in a
`Referer` header, and the page removes it from the address bar on mount.

The stored record is a version, a timestamp and the list of categories granted.
There is no visitor identifier in it and no server-side row behind it.
Withdrawal deletes the cookie from the device. The category list is built so
that adding analytics or marketing later is an entry in a list rather than a
rewrite.

Stripe.js is loaded only when the customer reaches the payment step, using the
Stripe SDK's variant that has no import-time side effect, so browsing the site
loads no payment provider script.

## Headers and metadata on private surfaces

The three paths where the URL itself is the secret (the questionnaire, the
Client Card and admin) send `Referrer-Policy: no-referrer` and `Cache-Control:
no-store`. The referrer header matters because these URLs must never travel to
a third party; `no-store` matters because a shared or borrowed phone should not
be able to press Back into somebody's answers or somebody's appointment. The
pages declare the referrer policy in their metadata as well as in the header,
rather than relying on one of the two.

Private surfaces are `noindex, nofollow`, carry no canonical URL that would
point a crawler at a private token, and their social metadata is generic. A
Client Card link pasted into a chat app produces a studio-branded preview with
no client information in it.

The public site sends a strict origin referrer policy, because a booking URL
carries the treatment somebody chose and that should not travel either.

## Secrets

Live credentials live in the hosting platform's environment, never in a file in
the working copy. The deployment ignore list explicitly excludes local
environment files and local database state, because a `NEXT_PUBLIC_` value read
from a file on disk would be inlined into the production client bundle before
platform variables ever mattered.

Before payments were switched on, every JavaScript file served by the
production deployment was downloaded and searched for secret key prefixes,
webhook signing secrets, service role keys, database hostnames and internal
salts. The only key present in the bundle was the publishable one, which is
where it belongs. Source maps are not served.
