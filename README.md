# Zippopotam API — test automation exercise

Automated checks for `GET https://api.zippopotam.us/{country}/{postal-code}/`, written in
Java 17 with REST Assured, TestNG and AssertJ.

## Running

```bash
mvn clean test                                  # whole suite
mvn test -Dgroups=smoke                         # quick post-deploy check
mvn test -Dgroups=contract                      # schema / shape only
mvn test -Dexcludedgroups=non-functional        # e.g. on a noisy shared runner
mvn test -Dbase.uri=http://localhost:8080        # point at a mock or staging host
mvn test -Dmax.response.time.ms=5000
```

Every setting in `src/test/resources/config.properties` can be overridden with `-D`.
No IDE-specific files, no hardcoded URLs, nothing to edit before the first run.

## How the framework is put together

```
client/      ZippopotamClient  – the only class that knows the URL shape. No assertions.
model/       ZipLookupResponse, Place – records mapping the JSON contract.
support/     BaseTest (REST Assured setup), ApiAssertions (reusable expectations).
config/      Config – system property -> properties file -> fail fast.
data/        LookupCase + LookupData – all test data, separate from test logic.
tests/       One class per concern: happy path, contract, negative, data integrity,
             non-functional.
resources/   config.properties, schemas/zip-lookup.json
```

Four decisions worth calling out:

**The client never asserts.** Validation lives in the tests and in `ApiAssertions`. A
client that throws on a 404 makes negative testing impossible, which is where most of
the interesting behaviour is.

**Responses are deserialised into records.** `body.places().get(0).placeName()` reads
better than a JSON path string, and a renamed field breaks in one mapping file instead
of silently matching nothing in fifteen assertions. Coordinates stay `String` in the
model — the API returns them as strings, and quietly coercing them would hide a real
contract change. Typed accessors handle the conversion where tests need numbers.

**Expected values are data, not code.** Adding a country is a row in `LookupData`, not a
new test method. TestNG reports show the `LookupCase` description, so a failure names
the case without anyone opening the data provider.

**Assertions are soft where a test has several independent expectations.** If the place
name, state and country abbreviation are all wrong, that should be one report showing
three failures, not three runs of the suite.

## Test cases

### Happy path — `ZipLookupHappyPathTest`
| # | Case | Why |
|---|------|-----|
| 1 | Known US ZIP returns 200 and the expected place, state and country abbreviation | Core behaviour |
| 2 | Leading-zero ZIP (`01001`) | Classic bug: numeric parsing strips the zero |
| 3 | Response echoes the requested postal code | Prevents "always returns 90210" style caching bugs |
| 4 | Every supported country returns a well-formed, non-empty result | Guards against a country dropping out of the dataset |
| 5 | Country code is case-insensitive (`us` vs `US`) | Documented URL shows lowercase; consumers will send both |
| 6 | Trailing slash returns the same resource | The documented URL has one; the common usage does not |
| 7 | Repeated identical requests return identical data | GET must be idempotent |

### Contract — `ZipLookupContractTest`
| # | Case | Why |
|---|------|-----|
| 8 | Body matches the JSON schema, for every country | Catches renamed fields, type changes, missing keys |
| 9 | `additionalProperties: false` in the schema | An unannounced new field should be a conscious decision, not a surprise |
| 10 | `Content-Type` is `application/json` | Clients parse on this header |
| 11 | POST is not accepted | The endpoint is read-only |

### Negative — `ZipLookupNegativeTest`
| # | Case | Why |
|---|------|-----|
| 12 | Unassigned ZIP (`us/99999`) → 404 | Must not be 200-with-empty-places |
| 13 | Unknown country (`zz/90210`) → 404 | |
| 14 | Right ZIP, wrong country (`de/90210`) → 404 | Country must actually be part of the lookup key |
| 15 | Wrong format for the country (letters, 4 digits, 6 digits, ZIP+4, negative) → 404 | |
| 16 | Missing postal code (`/us/`, `/us`) | |
| 17 | Missing country (`//90210`, `/90210`) | |
| 18 | Extra path segment (`/us/90210/extra`) | Should not be silently truncated to a valid lookup |
| 19 | Unknown query parameters do not change the response | |
| 20 | SQL/script injection, path traversal, 2 KB input, Unicode digits, whitespace | No 5xx, no stack trace, no HTML error page leaked |

404 bodies are asserted loosely — status code, no location data, no leaked internals.
The API only promises "not found"; pinning the exact empty-body representation would be
brittle for no extra coverage.

### Data integrity — `ZipLookupDataIntegrityTest`
| # | Case | Why |
|---|------|-----|
| 21 | Latitude in [-90, 90], longitude in [-180, 180] | A schema proves it is a numeric string, not that it is a coordinate |
| 22 | Known ZIP resolves to the right region (Beverly Hills box) | Catches swapped lat/long and sign errors |
| 23 | Multi-locality postal code returns no duplicate places | Duplicates double-count for consumers |
| 24 | Text fields are trimmed, non-blank, and free of `"null"` / `"N/A"` placeholders | Data quality, not shape |

### Non-functional — `ZipLookupNonFunctionalTest`
| # | Case | Why |
|---|------|-----|
| 25 | Single lookup within a configurable budget | Tripwire, not a benchmark — a single-threaded test says nothing about throughput |
| 26 | Served over HTTPS | |
| 27 | No `X-Powered-By` fingerprinting header | |

## What I would add next, given more time

- **A mocked layer** (WireMock) so the negative and contract tests run without the
  internet and can force conditions the real API will not produce on demand: 500s,
  timeouts, truncated JSON, a field changing type. Right now the suite is an integration
  suite and inherits upstream availability as a flakiness source.
- **Reporting** — Allure or ExtentReports, with request/response attached to failures.
- **CI** — a GitHub Actions workflow running `smoke` on every push and the full suite
  nightly, publishing the TestNG report.
- **Rate limiting and caching checks** — behaviour under burst load, `Cache-Control` and
  `ETag`/`If-None-Match` handling, `HEAD` support. Left out here because the API
  publishes no contract for them and the tests would be guesses.
- **A wider country matrix** driven from a CSV rather than a Java data provider, so
  non-developers on the team can extend the coverage.

## Assumptions

- The suite runs against live production data, so expected values are restricted to
  reference points that are stable (well-known US ZIP codes) or to contract-level
  assertions (`countryAbbreviation` matches the request). Asserting that a particular
  Dutch village name is returned would produce failures that say nothing about the code
  under test.
- Tests run in parallel by method. Nothing in the framework holds mutable shared state,
  which is what makes that safe.
