# Threats by STRIDE category

Each row names a threat, what it would cost, the control that exists today, and how the control is verified. "Tests" means automated tests in the application's suite; "database tests" means pgTAP tests that run as an anonymous user, an outsider and each role.

## S: Spoofing

| # | Threat | Impact | Control | Verified by |
| --- | --- | --- | --- | --- |
| S1 | Someone guesses or reuses a team member's password | Full read access to personal data | Passwords of 16 to 72 characters with a letter, a number and a symbol, checked against known breached passwords (only a 5-character hash prefix leaves the server). Passwords expire after 60 days; once expired, every page leads to Account until the password is changed. Self sign-up and anonymous sign-in are off. | Tests of the password rules and the redirect |
| S2 | A stolen session cookie is replayed | Same as S1, without the password | Session cookies are HttpOnly, SameSite=Lax and Secure. After 15 minutes with no activity the account is signed out everywhere; the idle clock lives in the database, so it holds across devices. Closing the app tab signs that browser out. Refresh tokens rotate. | Tests of the request gate's idle and sign-out behaviour |
| S3 | A forged webhook pretends to be the telephony provider, the scheduling service, the survey service or the website | Fake calls, bookings or contacts written into records | Each sender has its own secret. The body's HMAC-SHA256 is compared in constant time. Where the sender stamps a time, anything more than five minutes away is refused. Events are de-duplicated by their id. A missing secret refuses everything (503) instead of accepting everything. | Signature tests for each scheme |
| S4 | A forged call to a scheduled-job endpoint | Jobs run on an attacker's timing, spending money or fetching pages | A bearer secret compared in constant time; missing configuration refuses the call | Route tests |
| S5 | Sign-in redirect sends the user to a look-alike site | Credential phishing | After sign-in the app redirects only to paths on its own site | Redirect tests |

## T: Tampering

| # | Threat | Impact | Control | Verified by |
| --- | --- | --- | --- | --- |
| T1 | A signed-in user edits another team's or another person's rows | Altered claims, filings or contact data | Row-level security on all 193 tables. Policies call one privileged helper that looks up team membership for the caller's user id. Admin-only tables and actions use a second helper. | 91 database test files |
| T2 | Two people (or two tabs) save over each other | Silent loss of work on a legal document | Saves are conditional on the version the user opened ("updated_at" guard). A stale save is refused and the current version is shown. | Action tests |
| T3 | Bad input reaches the database | Corrupt records, or a crash that hides a bigger fault | Server actions validate input with zod schemas. The database adds about 1,500 check constraints of its own. | Schema tests; constraint tests |
| T4 | Someone alters a do-not-call record or a call attempt after the fact | Evidence of compliance is lost | These records are append-only: triggers refuse changes and deletes, and they are kept for five years | Database tests |
| T5 | A crafted document injects script into a page | Session theft through cross-site scripting | Every place that renders HTML is fed escaped text; paper documents are rebuilt from an allow-list of elements. File downloads are served with `nosniff`, and uploaded legal files with a sandboxing Content-Security-Policy. | Rendering tests |

## R: Repudiation

| # | Threat | Impact | Control | Verified by |
| --- | --- | --- | --- | --- |
| R1 | "I didn't change that claim" | Disputes inside the team or with a claimant | Every claim change is written to an event history. The author defaults to the caller and the time is set by the server. Users can only add notes as themselves, and events can't be edited or deleted. | Database tests |
| R2 | "We never called that number" / "We called within hours" | Telemarketing complaints | Every dial attempt is recorded with the check the database made before allowing it. An admin override of the daily cap requires a written reason. | Database tests |
| R3 | Who loaded or changed a script version or a list | Unclear accountability | 58 "who" columns are stamped by the database from the signed-in user, not taken from the request | Database tests |

## I: Information disclosure

| # | Threat | Impact | Control | Verified by |
| --- | --- | --- | --- | --- |
| I1 | An outsider reads personal data | Fraud against claimants | Row-level security, as in T1. The browser receives only the database address and a publishable key; it never queries tables directly. | Database tests |
| I2 | A secret reaches the browser | Total compromise | Server code is marked server-only (231 modules), so the build fails if the browser imports it. Only two public environment variables exist, neither secret. The stored email-account token is sealed with AES-256-GCM. | Build checks |
| I3 | Files are opened by someone who shouldn't see them | Disclosure of IDs and documents | Every storage bucket is private, with a size and type limit. Downloads go through short-lived signed links; uploads use signed upload links that can't overwrite. | Code review of the storage policies |
| I4 | Personal data ends up in logs | Disclosure to anyone with log access | Logs record outcomes (status, timing, cost), never document text or webhook payloads | Tests that check what a log line contains |
| I5 | Personal data is sent to the AI provider | Wider exposure than needed | See [ai-threats.md](ai-threats.md), A2 | Privacy tests |
| I6 | The server is tricked into fetching internal addresses (SSRF) | Access to cloud metadata or internal services | See T-SSRF below and [ssrf-case-study.md](ssrf-case-study.md) | 18 address tests, fetch tests |

## D: Denial of service

| # | Threat | Impact | Control | Verified by |
| --- | --- | --- | --- | --- |
| D1 | Huge webhook bodies or uploads | Exhausted memory or storage | Size limits on every webhook body, every bucket, and server-action bodies | Code review; webhook route tests |
| D2 | A slow or endless remote page holds the server | Jobs time out; costs rise | Fetches have timeouts (10 to 15 seconds), size caps (2 to 5 MB), and at most five redirects | Fetch tests |
| D3 | Repeated presses of an expensive button | Spend on paid APIs | Per-user and per-feature limits per minute and per day; only one AI call at a time per user; a monthly spending cap on scheduled research | Limit tests |

## E: Elevation of privilege

| # | Threat | Impact | Control | Verified by |
| --- | --- | --- | --- | --- |
| E1 | A team member reaches admin-only actions (sending email, connecting an email account, AI reading of forms) | Actions the business hasn't approved for that person | Each admin action checks the role on the server, and the database's admin helper backs it | Action tests; database tests |
| E2 | A privileged database function is hijacked through the search path | Running code with the function owner's rights | Every privileged function sets an empty search path, and execute rights are revoked from anonymous users | Database tests |
| E3 | The database's own service key is used outside signed jobs | Bypass of row-level security | The service key is used only in server-only modules for scheduled jobs, signed webhooks and one password-date write | Code review |

## T-SSRF: fetching addresses supplied from outside

The app fetches public county pages and documents. Those addresses come from research data and from links found on county pages, so they're treated as untrusted.

- Only `http` and `https` on ports 80 and 443, with no user name or password in the address.
- Internal names are refused: `localhost`, `.local`, `.internal`, `.home.arpa`, and names with no dot.
- Every address the name resolves to must be public. That excludes private, loopback, link-local, carrier-grade NAT, multicast, reserved and cloud-metadata ranges, including IPv4 written as IPv6, NAT64 and 6to4.
- The connection is pinned to the addresses that were checked, which defeats DNS rebinding.
- Every redirect is checked again, with at most five.
- A file the browser asks the server to fetch is accepted only if it matches a link the server itself just found on the county's official page.
