*Marketplace Deal Finder*


A command-line tool that automates a search workflow a major resale marketplace
doesn't support natively — built by reverse-engineering an undocumented internal
API, with a focus on correctness, resilience, and responsible request behavior.



Personal/demo project. It interacts with an undocumented internal API, so it's
designed for light personal use, not production. The point of this repository is to
document the engineering process and the decisions behind it.


The problem

I wanted to find sellers on a large second-hand marketplace who offered a specific
type of discount on multi-item purchases, within a defined price range. The platform's
own search has no filter for this: the discount is a property of the seller, not of
the listing, and the only way to check it manually is to open each seller's profile
one by one. Across hundreds of search results, that's not practical.

So the real problem wasn't "scrape a website" — it was: how do you reliably surface a
seller-level attribute that the search layer doesn't expose, without hammering the
platform?


Approach

The interesting part of this project is the sequence of obstacles and the decisions
each one forced. I've kept them in the order they actually happened.

1. Finding the right data source

The public/official surface didn't expose the data I needed. I identified the internal
REST API the site's own front-end uses (/api/v2/...) and confirmed which endpoints
held the relevant fields: a catalog/search endpoint to find listings, and a user
endpoint that carried the seller-level discount object.

2. Debugging a silent failure

My first implementation, built on an existing wrapper library, returned HTTP 200 but
failed to parse the response as JSON. Instead of guessing, I traced it: the request was
succeeding, but the server was returning an HTML anti-bot challenge page rather than
JSON. The fix was to drive the request directly — establishing a session against the
home page first to obtain the necessary cookies, then sending realistic browser headers
(User-Agent, Accept, Accept-Language, Referer). That turned the HTML challenge
into clean JSON responses.

Skill shown: methodical debugging — reading status codes, content types, and response
bodies to find the actual cause instead of trial-and-error.

3. Discovering the data shape instead of assuming it

The discount field wasn't documented anywhere. Rather than hard-coding a guess, I wrote
an inspection step that prints the full raw profile object and recursively flags any key
related to the target attribute. This revealed the real structure — an enabled flag
plus a list of tiered discounts (minimal_item_count + fraction) — which I then
parsed and normalized (fractions like "0.05" into a clean "max %" per seller).

Skill shown: treating an unknown API empirically — verify, then build on what's real.

4. Designing for resilience and good behavior

A naive loop would have been slow, wasteful, and likely to get rate-limited. Design
decisions, each with a reason:


Deliberate pacing. Fixed delays between requests, so the traffic pattern stays
modest and respectful of the platform rather than bursting.
Persistent cache. Every seller checked is stored to disk and never re-queried.
Subsequent runs skip known sellers entirely, so request volume drops over time
while coverage grows.
Pagination + de-duplication. Walks multiple result pages, collecting unique
sellers and stopping cleanly when results are exhausted.
Sort rotation. Because no single ordering exposes the whole market, the tool
supports switching sort order between runs; combined with the cache, each new run
surfaces mostly new sellers instead of re-scanning the same ones.
Server-side filtering. Price bounds are pushed to the API's own query parameters,
so out-of-range (and many low-effort scam) listings are never even downloaded.


5. Making it usable

Wrapped the script in a small interactive launcher (a guided .bat) with sensible
defaults and inline guidance, so the tool is runnable without memorizing flags, while
the underlying CLI stays fully scriptable.



A note on responsible use

This tool talks to an internal, undocumented API, which sits outside a platform's
intended usage and can change or break at any time. It's intentionally paced to keep
request volume low and is meant for light personal use only — not for scale, resale, or
production. I built it to solve a personal problem and to practice the engineering
involved; the repository exists to document that process.


