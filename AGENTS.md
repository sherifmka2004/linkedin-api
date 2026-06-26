# AGENTS.md

Machine-oriented guide to this repository. Optimized for an LLM/agent that needs
to understand, use, or extend the code quickly and correctly. Humans welcome too.

---

## 1. What this is

`linkedin_api` is an **unofficial Python wrapper around LinkedIn's internal
"Voyager" API** — the same private JSON API that powers `www.linkedin.com`.

- It does **not** use the official LinkedIn REST/Marketing API and needs **no API
  key or OAuth app**. You authenticate with a real LinkedIn **email + password**,
  exactly as a browser would.
- All calls hit `https://www.linkedin.com/voyager/api/...` using a normal
  authenticated session (cookies + CSRF token).
- Because it impersonates the web client, it is subject to LinkedIn rate limits
  and Terms of Service. Treat it as account-risky automation (see §9).

Package metadata: `linkedin_api`, version `1.1.0`, MIT licensed. Single runtime
dependency: `requests`.

---

## 2. Repository map

```
linkedin-api/
├── linkedin_api/
│   ├── __init__.py        # exports `Linkedin`; holds __version__
│   ├── linkedin.py        # MAIN public API — the `Linkedin` class & all endpoints
│   ├── client.py          # `Client` — auth, session, cookies, headers, base URL
│   ├── settings.py        # COOKIE_FILE_PATH (.cookie.jr next to the package)
│   └── utils/
│       ├── __init__.py
│       └── helpers.py     # get_id_from_urn()
├── examples/
│   └── basic.py           # minimal usage example (reads credentials.json)
├── tests/
│   ├── test_linkedin_api.py     # integration tests against the Linkedin class
│   └── test_linkedin_client.py  # auth test against Client
├── test_acct.py           # ad-hoc script (⚠ see §9 — contains hardcoded creds)
├── DOCS.md                # human-facing endpoint docs
├── README.md             # human-facing overview
├── setup.py / Pipfile     # packaging & dependency definitions
├── .env.example           # env vars used by the test suite
└── AGENTS.md              # this file
```

**The two files that matter for almost any task are
`linkedin_api/linkedin.py` (endpoints) and `linkedin_api/client.py` (auth).**

---

## 3. Architecture & request lifecycle

```
Linkedin(username, password)            # linkedin.py
   └─ creates Client(...)               # client.py
         └─ authenticate(user, pass)    # logs in, stores cookies + csrf-token
   └─ self.client.session               # an authenticated requests.Session

linkedin.get_profile("foo")             # any public method
   └─ self._fetch("/identity/...")      # GET  -> default_evade() sleeps 2–5s
        or self._post("/messaging/...") # POST -> default_evade() sleeps 2–5s
         └─ self.client.session.get/post(API_BASE_URL + uri)
              └─ returns requests.Response; method calls .json() and reshapes it
```

Key pieces:

- **`Client` (`client.py`)**
  - `API_BASE_URL = "https://www.linkedin.com/voyager/api"`
  - `AUTH_BASE_URL = "https://www.linkedin.com"`
  - Two header sets: `REQUEST_HEADERS` (browser-like, for API calls) and
    `AUTH_REQUEST_HEADERS` (LinkedIn iOS app-like, for the login handshake).
  - `authenticate()` flow:
    1. `_request_session_cookies()` — load cached cookies from `.cookie.jr` if
       present and `refresh_cookies` is False, else GET `/uas/authenticate` for a
       fresh `JSESSIONID` cookie.
    2. POST `/uas/authenticate` with `session_key` (username), `session_password`,
       and `JSESSIONID`.
    3. On `login_result != "PASS"` → raise `ChallengeException` (e.g. CAPTCHA /
       2FA / verification). On 401 → `UnauthorizedException`. Other non-200 →
       generic `Exception`.
    4. `_set_session_cookies()` stores cookies, sets the `csrf-token` header from
       `JSESSIONID`, and pickles cookies to `.cookie.jr` for reuse.
  - `session.verify = not debug` — **debug mode disables TLS verification** and
    silences urllib3 warnings (so you can run through a debugging proxy like
    mitmproxy/Charles). Don't enable debug in production.

- **`Linkedin` (`linkedin.py`)** — the public surface. Each method maps to one or
  more Voyager endpoints, calls `.json()`, and "massages" the deeply-nested
  LinkedIn response into a flatter dict/list. Pagination is handled with
  **recursion** (`search`, `get_company_updates`, `get_profile_updates`).

- **Anti-suspension evasion**: every `_fetch`/`_post` calls `default_evade()`
  which `sleep(random.randint(2, 5))`. So **every API call blocks 2–5 seconds**.
  Paginated calls multiply this. This is intentional, not a bug.

---

## 4. Identifiers you must understand

LinkedIn objects are referenced two ways. Most profile methods accept **either**
via keyword args `public_id=` or `urn_id=` (they fall through as
`public_id or urn_id`):

| Term            | Example                                  | What it is |
|-----------------|------------------------------------------|------------|
| `public_id`     | `tom-quirk-1928345`                      | The human-readable slug in a profile URL (`linkedin.com/in/<public_id>`). |
| `urn_id`        | `ACoAABQ11fIBQLGQbB1V1XPBZJsRwfK5r1U2Rzw`| The opaque member URN id. Stable; safe for storage. |
| URN (full)      | `urn:li:fs_miniProfile:<id>`             | Fully-qualified entity URN. |
| `conversation_urn_id` | `6419123050114375168`              | Numeric id of a message thread. |

`utils/helpers.get_id_from_urn(urn)` extracts the id by splitting on `:` and
taking index `3` — i.e. it expects the form `urn:li:<type>:<id>`. If you pass a
URN with a different shape, this will silently return the wrong segment or raise
`IndexError`.

---

## 5. Public API reference (`Linkedin` class)

Constructor:

```python
Linkedin(username, password, *, refresh_cookies=False, debug=True, proxies={})
```
- `refresh_cookies=True` ignores the cached `.cookie.jr` and logs in fresh.
- `debug=True` (the current default!) enables DEBUG logging **and disables TLS
  verification**. Pass `debug=False` for real use.
- `proxies` is forwarded to the `requests.Session`.

### Profiles
| Method | Returns | Notes |
|---|---|---|
| `get_profile(public_id=None, urn_id=None)` | dict | Full profile: experience, education, skills (calls `get_profile_skills`), reshaped picture/logos. Returns `{}` on non-200 status payloads. |
| `get_profile_contact_info(public_id=None, urn_id=None)` | dict | email, websites, twitter, birthdate, ims, phone_numbers. |
| `get_profile_skills(public_id=None, urn_id=None)` | list | Up to 100 skills; strips `entityUrn`. |
| `get_profile_connections(urn_id)` | list | Convenience for `search_people(connection_of=urn_id, network_depth="F")` (1st-degree). |
| `get_profile_privacy_settings(public_profile_id)` | dict | `{}` on non-200. |
| `get_profile_member_badges(public_profile_id)` | dict | `{}` on non-200. |
| `get_profile_network_info(public_profile_id)` | dict | distance, follower counts, etc. `{}` on non-200. |
| `get_current_profile_views()` | int | "Who viewed your profile" count for the logged-in user. |
| `get_user_profile()` | dict | The logged-in user's own `/me` payload. |

### Search
| Method | Returns | Notes |
|---|---|---|
| `search(params, limit=None, results=[])` | list | Low-level blended search. Recursive pagination, page size `_MAX_SEARCH_COUNT=49`. |
| `search_people(keywords=None, connection_of=None, network_depth=None, current_company=None, past_companies=None, nonprofit_interests=None, profile_languages=None, regions=None, industries=None, schools=None, include_private_profiles=False, limit=None)` | list of `{urn_id, distance, public_id}` | Builds a `filters=List(...)` query. List-valued filters are `\|`-joined. `network_depth` ∈ `F`/`S`/`O` (1st/2nd/3rd+). Skips profiles without `publicIdentifier`. |
| `stub_people_search(query, count=10, start=0)` | dict (raw) | Older `/search/hits` guided-search endpoint. Returns raw JSON, `{}` on non-200. Used by `test_acct.py`. |

### Companies & Schools
| Method | Returns | Notes |
|---|---|---|
| `get_company(public_id)` | dict | Full company by universal name (e.g. `linkedin`). `{}` on non-200. |
| `get_school(public_id)` | dict | Full school by universal name (e.g. `university-of-queensland`). `{}` on non-200. |
| `get_company_updates(public_id=None, urn_id=None, max_results=None, results=[])` | list | Company feed posts. Recursive pagination, page size `_MAX_UPDATE_COUNT=100`. |
| `get_profile_updates(public_id=None, urn_id=None, max_results=None, results=[])` | list | A member's feed posts. Same pagination. |

### Messaging
| Method | Returns | Notes |
|---|---|---|
| `get_conversations()` | dict | All conversations for the logged-in user. |
| `get_conversation_details(profile_urn_id)` | dict | Thread with one participant; adds flattened `id`. |
| `get_conversation(conversation_urn_id)` | dict | All events/messages in a thread. |
| `send_message(conversation_urn_id=None, recipients=[], message_body=None)` | bool | **Returns `True` on ERROR**, `False` on success. Provide either an existing `conversation_urn_id` OR a `recipients` list of profile urn ids (to start a new thread), not both. |
| `mark_conversation_as_seen(conversation_urn_id)` | bool | `True` on error, `False` on success. |

### Network / Invitations
| Method | Returns | Notes |
|---|---|---|
| `get_invitations(start=0, limit=3)` | list | Received invitations. `[]` on non-200. |
| `reply_invitation(invitation_entity_urn, invitation_shared_secret, action="accept")` | bool | `action` ∈ `"accept"`/`"ignore"`. **Returns `True` on success** (note: opposite convention to messaging methods). |
| `remove_connection(public_profile_id)` | bool | `True` on error, `False` on success. |

> ⚠ **Return-value convention is inconsistent.** Messaging/connection mutators
> return `True` on *error*; `reply_invitation` returns `True` on *success*. Always
> check the specific method's docstring/source before relying on the boolean.

### Not implemented / disabled
`add_connection` and `view_profile` exist only as commented-out stubs in
`linkedin.py` (they don't currently work). Don't call them — they aren't defined.

---

## 6. Quick start

```python
from linkedin_api import Linkedin

# Authenticate (use a real LinkedIn account; debug=False for normal use)
api = Linkedin("you@example.com", "password", debug=False)

# Profile by public id (URL slug) or urn id — either works
profile = api.get_profile("tom-quirk-1928345")
contact = api.get_profile_contact_info("tom-quirk-1928345")

# Search people
results = api.search_people(keywords="data scientist", limit=10)
# -> [{"urn_id": "...", "distance": 2, "public_id": "..."}, ...]

# Companies / schools
company = api.get_company("linkedin")
school = api.get_school("university-of-queensland")

# Messaging (returns True on ERROR)
errored = api.send_message(recipients=[results[0]["urn_id"]], message_body="hi")
```

See `examples/basic.py` for the credentials-file pattern.

---

## 7. Setup & dependencies

Runtime requires only `requests` (Python 3). Two supported workflows:

```bash
# pip / setuptools
pip install -e .

# or pipenv (Pipfile present)
pipenv install --dev
```

Dev tooling declared in `Pipfile`: `pytest`, `black`, `pylint`, `ipdb`.

---

## 8. Tests

The test suite is **integration-only**: it logs into a real LinkedIn account and
hits live endpoints. There are **no mocks/unit tests**, so tests are slow (2–5s
sleep per call), network-dependent, and can fail due to rate limiting, challenges,
or stale fixture ids.

Configuration comes from environment variables (see `.env.example`):

```
LINKEDIN_USERNAME       # test account email
LINKEDIN_PASSWORD       # test account password
TEST_PROFILE_ID         # a urn id to fetch
TEST_PUBLIC_PROFILE_ID  # a public id slug (some tests need this; not in .env.example)
TEST_CONVERSATION_ID    # an existing conversation urn id
```

`tests/test_linkedin_api.py` **`sys.exit()`s immediately** if any of
`LINKEDIN_USERNAME`, `LINKEDIN_PASSWORD`, `TEST_PROFILE_ID`,
`TEST_PUBLIC_PROFILE_ID`, `TEST_CONVERSATION_ID` are unset — so all five must be
exported or the suite silently does nothing.

Run:

```bash
export $(grep -v '^#' .env | xargs)   # after copying .env.example -> .env
pytest                                # or: pytest tests/test_linkedin_api.py
```

Note: `test_send_message_*` tests actually **send messages** and
`test_accept/reject_invitation` actually mutate invitations on the test account.

---

## 9. Safety, security & gotchas (read before editing)

- **⚠ `test_acct.py` contains hardcoded real-looking credentials** and a plaintext
  password on line 5. It is an ad-hoc script, not part of the package. Do not add
  more secrets to source. Consider removing it / rotating that account. Never
  commit credentials; `.env`, `.cookie.jr`, and `credentials.json` should stay out
  of git (`.env` and `.cookie.jr` are already gitignored; `credentials.json` is
  not — add it if you use the `examples/basic.py` pattern).
- **Cookie cache**: successful auth pickles cookies to
  `linkedin_api/.cookie.jr`. Subsequent `Linkedin(...)` calls reuse them unless
  `refresh_cookies=True`. Stale/invalid cached cookies can cause confusing auth
  failures — delete `.cookie.jr` or pass `refresh_cookies=True` to force a fresh
  login.
- **`debug=True` is the constructor default** and disables TLS verification. For
  any real usage pass `debug=False`.
- **Mutable default arguments**: `search`, `get_company_updates`,
  `get_profile_updates` use `results=[]` and `proxies={}` as defaults. Because
  these are evaluated once, results can **leak between calls** within a process.
  When fixing/extending, prefer `results=None` + `results = results or []`. Be
  careful not to break the existing recursive pagination contract.
- **Rate limiting**: `_MAX_REPEATED_REQUESTS = 200` is a conservative ceiling.
  Aggressive looping risks account challenge/suspension. Keep the `default_evade`
  sleeps; don't remove them to "speed things up."
- **`ChallengeException`** on login usually means LinkedIn wants a CAPTCHA / email
  / 2FA verification. There's no programmatic bypass here — log in via a browser
  first, then retry.
- **Inconsistent boolean returns** between mutator methods (see §5). Verify per
  method.
- **Brittle response parsing**: methods index deep into LinkedIn's JSON (e.g.
  `data["positionView"]["elements"]`, hard-coded `com.linkedin.voyager...` keys).
  LinkedIn changes these without notice, so endpoints break periodically; expect
  `KeyError`/`IndexError` when the upstream shape shifts.

---

## 10. How to extend (adding a new endpoint)

1. Find the Voyager path by inspecting requests in a browser/devtools or a proxy
   (run with `debug=True` to route through a TLS-intercepting proxy).
2. Add a method to the `Linkedin` class in `linkedin_api/linkedin.py`.
3. Use `self._fetch(uri, **kwargs)` for GET and `self._post(uri, **kwargs)` for
   POST. `uri` is appended to `API_BASE_URL`; do **not** include the host. Keep the
   `default_evade()` behavior (it's automatic via `_fetch`/`_post`).
4. For "normalized" responses, pass
   `headers={"accept": "application/vnd.linkedin.normalized+json+2.1"}`.
5. Reshape the JSON into a flat dict/list, mirroring existing methods. Guard with
   the non-200 pattern: `if data and "status" in data and data["status"] != 200:
   return {}`.
6. For paginated lists, follow the recursive `start`/`count` pattern used by
   `search` / `*_updates`.
7. Add an integration test in `tests/test_linkedin_api.py` and document the method
   in this file's §5 table and in `DOCS.md`.

Style: the codebase uses `black` formatting and f-strings. Match surrounding
conventions (keyword-only `public_id`/`urn_id` args, docstrings describing each
parameter).
```
