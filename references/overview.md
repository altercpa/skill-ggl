# GGL.farm API Overview

GGL.farm keeps a farm of Google accounts: it stores your accounts and proxies, runs a task queue and works through **your own anti-detect browser**. The API repeats what the web interface does, so a farm of hundreds of accounts can be driven from a script.

Base URL: `https://ggl.farm/api/{app}/{func}.json?id={apitoken}`

This file holds the rules shared by every application. Function details live in the per-application references.

---

## Applications

| App | Level | Functions | Reference |
|---|---|---|---|
| `accs` | user | `list` `get` `make` `push` `add` `edit` `fix` `del` `status` `folder` `active` `all` `start` `ping` `stop` | [accs.md](accs.md) |
| `gang` | team | `list` `get` `fix` `del` (twins of the `accs` functions with the same names) | [accs.md](accs.md) |
| `task` | user | `list` `add` `del` `restart` | [task.md](task.md) |
| `proxy` | user | `list` `get` `add` `load` `edit` `check` `del` | [proxy.md](proxy.md) |
| `browser` | user | `info` `save` `services` `sms` `captcha` `log` `quota` | [browser.md](browser.md) |

Only the account functions have a team level. Everything else works on your own data.

---

## Authentication

Every function requires a token: `{apitoken}` = `<user id>-<API key>`, for example `123-0123456789abcdef`. The API key is shown on your profile page after you sign in at https://ggl.farm. Treat it like a password.

| Request | Where the token goes |
|---|---|
| Any GET, and any POST | `?id={apitoken}` in the URL |
| Form POST (`application/x-www-form-urlencoded`) | `authid={apitoken}` form field (the URL form works too) |
| JSON POST (`application/json`) | `?id={apitoken}` in the URL **only**; the JSON body carries parameters only, `authid` is not supported in a JSON body |
| Signed (optional) | `{user id}-{hmac}` instead of the raw key. `hmac` is the hex HMAC-SHA1, keyed with your API key, of the POST fields after quotes and backslashes in the values are backslash-escaped and the fields are re-encoded as a URL query string (`a=1&b=x+y`, in the order sent), not of the raw body. Query parameters are not signed, so for a GET it is the HMAC of an empty string |

Auth errors: `key` (token missing or wrong, HTTP 401), `ban` (user banned or inactive, HTTP 403).

---

## Requests

| Rule | Detail |
|---|---|
| Methods | GET and POST are interchangeable. Parameters from the query string and the body are merged; on a clash the body wins. Use POST for anything holding passwords, 2FA keys or proxy credentials |
| Body formats | `application/x-www-form-urlencoded` or `application/json` (one JSON object) |
| Lists | `ids=1,2,3` (comma separated) or an array (`ids[]=1&ids[]=2` in forms, a JSON array in JSON) |
| Flags | `0` / `1` |
| Countries | lowercase ISO 3166-1 alpha-2 codes (`us`, `gb`, `de`). Input is case-insensitive. `zz` means "same as the account country" for `smsgeo` and "unknown" for a proxy |
| Time | UNIX timestamps, seconds |
| Format suffix | always `.json` |

---

## Response Envelope

Success is HTTP 200:

```json
{ "status": "ok", "data": { } }
```

Error:

```json
{ "status": "error", "error": "no-proxy" }
```

| Field | Meaning |
|---|---|
| `status` | `ok` or `error` |
| `data` | the result: an object, an array, a map or `false`. Some functions return only `{"status":"ok"}`, and a few add top-level fields (`id`, `connect`, `expire`) |
| `error` | machine-readable error code, only when `status` is `error` |
| `nav` | `task/list`, `proxy/list`: the paging block, see [Pagination](#pagination) |
| `dataset` | `task/list`, `proxy/list`, `proxy/get`: code-to-name dictionaries, see [Dictionaries](#dictionaries-dataset) |
| `current` | `browser/services`: ID of the SMS service selected now (`0` when none) |
| `added`, `duplicates`, `rejected` | `proxy/load`: how many lines were added, skipped as already known, or unreadable; the IDs of the added proxies are in `data` |
| `param` | with `bad-param`: the name of the request parameter that held an array |
| `supported-geo` | with `bad-geo`: the countries you may use |
| `supported-farmnext` | with `bad-farmnext`: the allowed farming periods |
| `message` | with `no-gang`: a human hint |

`data: false` on `accs/folder`, `accs/active`, `accs/all` means the anti-detect browser integration is not set up or did not answer.

The HTTP status follows the error code, and two headers repeat the result: `X-API-Status` (`ok` / `error`) and `X-API-Error` (the code).

| HTTP | Error codes |
|---|---|
| 200 | success |
| 400 | everything not listed below |
| 401 | `key` |
| 402 | `unpaid` |
| 403 | `ban` |
| 404 | `no-id`, `bad-id`, `bad-status`, `not-found` |
| 418 | `bot` |
| 500 | `func`, `app`, `db` |

---

## Pagination

Functions that return many rows take the same two parameters:

| Parameter | Default | Meaning |
|---|---|---|
| `page` | 1 | page number, counted from 1 |
| `show` | 100 (`browser/log`: 50) | rows per page, no upper limit (`browser/log`: at most 200) |

Two reply shapes exist:

| Shape | Used by | Fields |
|---|---|---|
| flat | `accs/list`, `gang/list`, `browser/log` | `page`, `show`, `total` next to `data` (an array); `page` and `show` echo the request |
| `nav` block | `task/list`, `proxy/list` | `nav: {total, page, pages, show}` next to `data` (an object keyed by row ID) |

`show=ids` (on `accs/list`, `gang/list` and `proxy/list`) switches to ID-only mode: no paging, `data` is the array of every matching ID. Use it to feed `fix`, `del` and `check`.

---

## Dictionaries (`dataset`)

`task/list`, `proxy/list` and `proxy/get` return a `dataset` block beside `data`. It maps the numeric codes used in `data` to display names. The block is plain JSON, so its shape depends on the codes:

```json
"dataset": { "types": ["HTTP", "HTTPS", "SOCKS5", "SOCKS5H"], "statuses": ["Waiting", "Working", "Error"] }
```

| Function | `types` | `statuses` |
|---|---|---|
| `proxy/list`, `proxy/get` | plain array, the code is the index | plain array, the code is the index |
| `task/list` | object keyed by the code, `"1"`-`"5"` | plain array, the code is the index (`0`-`5`) |

Always match on the number, never on the name: names can follow the interface language of your account. The account functions (`accs/*`) carry no `dataset`; their vocabularies are listed in [accs.md](accs.md).

---

## User Level and Team Level

| | user level | team level |
|---|---|---|
| Paths | `api/accs/*` and the other apps | `api/gang/*` |
| Who may call | any user | the team administrator only |
| Data seen | your own accounts | the accounts of every team member |
| Functions | all | `list`, `get`, `fix`, `del` |
| Extra filter | none | `user={member id}` on `gang/list` |

Creating, editing and starting profiles stay user-level: they are attributed to the calling user. A team member who is not an administrator, or a user without a team, gets `{"status":"error","error":"no-gang","message":"Use user-level API"}` from `gang/*`.

There is no `scope` parameter: the level is chosen by the path.

---

## Common Errors

Any function can return these before its own errors:

| Error | Meaning |
|---|---|
| `key` | token missing or wrong |
| `ban` | the user is banned or inactive |
| `unpaid` | the subscription is not active |
| `request` | malformed API address (application, function or format missing) |
| `func` | unknown function |
| `bot` | request from a link-preview crawler |
| `no-gang` | `gang/*` called by a non-administrator |
| `bad-param` | a request value is an array (`name[]=...`, or a JSON array or object) where one value is expected; the name comes in the `param` field. The `ids` list of `fix`, `del` and `check` is exempt. Functions that read only an ID (`get`, `start`, `ping`, `stop`, `proxy/get`, `task/del`, `task/restart`) do not check |
| `no-id` / `bad-id` | the object ID is missing / not found among your objects |

Function-specific codes are listed with each function. An unknown code should be treated as a failure and logged, not retried blindly.

---

## Limits and Quotas

| Limit | What you see |
|---|---|
| Subscription | without an active plan every function answers `unpaid` (HTTP 402) |
| Threads | the plan limits how many accounts the robot works on **at the same time** (one running browser window each), not how many accounts you store. Read it with `browser/quota`. A team shares one limit |
| Proxy slots | a proxy carries a number of accounts; creating accounts needs free slots, otherwise `no-proxy` |
| Manual browser session | `accs/start` holds an account for 30 minutes; prolong it with `accs/ping` |
| Call rate | calls are not counted against a separate quota. The browser lookups (`accs/folder`, `accs/active`, `accs/all`, `accs/status`) and `browser/quota` are cached for about a minute, so polling them faster gains nothing; large page sizes are allowed |
| Browser-side limits | an anti-detect browser may throttle its own API; bulk profile deletion through `accs/del` can leave some profiles in the browser |
| Timeouts | `accs/start` can take up to a minute and a half; use a request timeout of 120 s or more |

---

## Typical Order of Calls

| Step | Call | Why |
|---|---|---|
| 1 | `accs/status` | the browser integration is set up and answering |
| 2 | `accs/folder` | pick the folder ID for new accounts |
| 3 | `accs/add` / `accs/push` / `accs/make` | bring accounts in |
| 4 | `accs/list` (filter by `tag`) | watch their state; the create calls return no IDs |
| 5 | `accs/fix`, `task/restart`, `proxy/check` | repair what stopped |
| 6 | `browser/quota` | see how much of the thread limit is used |
