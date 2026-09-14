# API Testing — Postman Portfolio

A hand-built Postman collection against three public APIs, then **audited against a coverage
checklist and rebuilt** — the audit is the point of this repo, not the green run.

**23 requests · 129 assertions · 3 APIs · happy path and negative path**

| File | What it is |
|---|---|
| `public-api-test-collection.postman_collection.json` | The collection (Collection v2.1) |
| `public-api-environment.postman_environment.json` | Environment template — base URLs filled in, API key left blank |

## Why these three APIs

Each one was chosen to cover something the others can't:

| API | Covers |
|---|---|
| [JSONPlaceholder](https://jsonplaceholder.typicode.com) | Full CRUD — GET / POST / PUT / PATCH / DELETE, plus malformed bodies and bad path params |
| [Rick and Morty API](https://rickandmortyapi.com) | Pagination boundaries — first page, last page, page 0, page beyond last |
| [OpenWeatherMap](https://openweathermap.org/api) | Real authentication — missing key, invalid key, and business-logic 404 with a valid key |

## Running it

1. Import both JSON files into Postman.
2. Open the **Public APIs (template)** environment and paste your own OpenWeatherMap key into
   `openweather_key`. Everything else is prefilled.
3. Select that environment and run the collection.

No URL or credential is hardcoded anywhere in the collection — all four values come from the
environment. The two OpenWeatherMap requests that need a real key will fail without one; the other
21 requests run with no setup at all.

Command line, if you prefer:

```
newman run public-api-test-collection.postman_collection.json \
  -e public-api-environment.postman_environment.json \
  --env-var openweather_key=YOUR_KEY
```

## The audit

The first version of this collection was 11 requests and ~57 assertions, and it ran green. I then
audited it against a 10-question coverage checklist. Eight of the ten questions came back PARTIAL or
NO. The gaps that mattered:

- **A hardcoded API key.** The OpenWeatherMap key sat in plain text in the request URL while
  `{{openweather_key}}` went unused. Pushing that export to a public repo would have leaked a live
  key. It also meant the mutation check below couldn't work — corrupting the key variable changed
  nothing, because nothing read it.
- **No failure path on any write endpoint.** POST, PUT, PATCH and DELETE were only ever tested
  succeeding. Write endpoints are where real APIs actually break.
- **No malformed input anywhere.** No non-numeric IDs, no truncated JSON bodies, no unicode.
- **Error tests that asserted only absence.** A 404 test that checks "no `results` property" passes
  against a 404 HTML page from a completely wrong base URL. Proven below.
- **Padding assertions.** A "Status is not 200" test sitting next to an exact-status test, and two
  DELETE tests whose failure branch was `pm.expect(true).to.be.true` — they could not fail.

All of it is fixed in what's here: the key moved to a secret environment variable, negative paths
added for every endpoint, malformed-input and unicode requests added, pagination boundaries covered
at both ends, and every error request now asserts the actual error payload rather than the absence
of a success payload.

## Verifying the tests can actually fail

A suite that only ever runs green hasn't been tested — it's been observed. Two deliberate breaks:

| Mutation | Result |
|---|---|
| `openweather_key` set to a bogus value | 7 assertions red, all in the two requests that need a real key. Before the fix: 0 red. |
| `rickandmorty_url` pointed at `/apix` | All 8 Rick and Morty requests fail, 31 of 44 assertions red. |

The second one surfaced something worth keeping: on the broken base URL, the three requests that
*expect* a 404 still passed their status assertion — the wrong URL returns 404 too. Only the
error-payload and Content-Type assertions caught it. That's the concrete argument for asserting the
body of an error response instead of just its status code.

## Defects found in the APIs under test

Three requests document behavior that looks wrong rather than behavior that's expected. Their tests
assert the observed status, so if an API ever fixes one of these, the test goes red and tells me.

| Endpoint | Observed | Expected |
|---|---|---|
| `PUT /posts/9999` (JSONPlaceholder) | 500, plain-text `TypeError` | 404 |
| `POST /posts` with a truncated JSON body | 500, `SyntaxError` in `text/html` | 400 |
| `GET /character/abc` (Rick and Morty) | 500, `{"error":"Hey! you must provide an id"}` | 400 |

All three are client-side input errors answered with a server error status.

## Known limitations

- JSONPlaceholder returns `200 {}` for a PATCH or DELETE against a nonexistent id, so those two
  endpoints have no meaningful not-found case. Recorded as an API limitation rather than forced
  into a test that would pass regardless.
- An expired-credential case isn't testable without waiting for a key to actually expire.
- The Rick and Morty last-page test asserts `info.pages === 42`. That's deliberate — if the dataset
  grows, the test fails and tells me the URL needs updating, instead of quietly testing the wrong page.
- The 1000 ms threshold on `GET all posts` is the assertion most likely to go flaky in CI.
