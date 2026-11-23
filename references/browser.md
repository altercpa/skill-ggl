# Integration API Reference

## Overview

The `browser` application is everything on the **Integration** page of the cabinet: the link to your antidetect browser, the SMS service that supplies phone numbers, the CapMonster captcha key and the journal of requests GGL sends to the browser. You can read and save every setting without the interface, for example when you move the farm to another server. The same application also shows the thread load of your plan and the length of the work queues (`quota`).

Base URL: `https://ggl.farm/api/browser/{func}.json?id={apitoken}`

Authentication, the `status`/`error` envelope and the HTTP status of errors are described in [overview.md](overview.md). GET and POST are interchangeable there; the headings below show the usual method: GET to read (`info`, `services`, `log`, `quota`), POST to write (`save`, `sms`, `captcha`).

**User level only.** The settings belong to your own user; there is no team (`gang`) twin and no `scope` parameter. The one exception is `quota`: for a team member it shows the figures of the whole team.

**Secrets.** `info` and `log` return your keys in clear: the browser API key, the SMS and CapMonster keys, the management key and, in the journal, request headers and bodies. Do not log these responses, paste them into tickets or show them to others. In the examples below every key is a `<placeholder>`.

Examples assume `TOKEN` holds your API token (`<user id>-<API key>`).

**First-time setup order:** `save` (browser) -> `services` (pick an SMS service ID) -> `sms` -> `captcha` -> `info` to verify. To test that the browser answers, use `accs/status` (see [accs.md](accs.md)).

---

## Integration Vocabulary

**Browser `type`:**

| Value | Name | Note |
|---|---|---|
| 1 | Vision | `Vision (beta)` |
| 2 | OctoBrowser | `OctoBrowser (stable)` |
| 3 | AdsPower | `AdsPower (alpha)` |
| 0 | not configured | Returned by `info` only; `save` rejects it |

**SMS services** - the `smstype` you can choose. `services` returns the live list with capabilities; this is the list at the time of writing. IDs are stable, names come in the language of your account interface.

| ID | Name | Rent | Send | Price cap | Rental terms (days) | Renew | Default term | Default price | Needs username |
|---|---|---|---|---|---|---|---|---|---|
| 17 | GGL (rent and send) | yes | yes | - | - | yes | - | - | - |
| 19 | SMS4G (rental) | yes | - | yes | - | - | - | - | - |
| 2 | textverified.com (rental) | yes | - | - | 1, 3, 7, 14, 30, 90, 365 | yes | 30 | - | yes |
| 18 | textverified.com (rent and send) | yes | yes | - | 1, 3, 7, 14, 30, 90, 365 | yes | 30 | - | yes |
| 8 | hero-sms.com (simple) | - | - | yes | - | - | - | 2 | - |
| 4 | hero-sms.com (rental) | yes | - | - | 1, 3, 7, 30 | - | 3 | - | - |
| 1 | 5sim.net | - | - | - | - | - | - | - | - |
| 3 | grizzlysms.com | - | - | - | - | - | - | - | - |
| 5 | onlinesim.io | - | - | yes | - | - | - | - | - |
| 7 | vak-sms.com (simple) | - | - | - | - | - | - | - | - |
| 16 | vak-sms.com (rental) | yes | - | - | - | - | - | - | - |
| 9 | smspool.net | - | - | yes | - | - | - | - | - |
| 10 | @GreedySMSbot | - | - | - | - | - | - | - | - |
| 11 | simsms.org | - | - | yes | - | - | - | - | - |
| 14 | vibe-sms.net (simple) | - | - | - | - | - | - | - | - |
| 15 | vibe-sms.net (rental) | yes | - | - | - | - | - | - | - |

**Capability fields** (in `caps` of `info` and in every item of `services`):

| Field | Type | Meaning |
|---|---|---|
| `rent` | bool | The service rents phone numbers |
| `send` | bool | The service can send SMS |
| `price` | bool | The service supports a maximum number price (`smsmax`) |
| `term` | string | Rental terms: `""` - no term; `"*"` - any term from 1 to 365 days; a comma-separated list such as `"1,3,7,30"` - only those days |
| `renew` | bool | The service supports automatic rental renewal (`smsrenew`) |
| `termdef` | int | Default rental term in days, `0` when the service has none |
| `pricedef` | int | Default maximum price, used when you leave `smsmax` at 0 |
| `login` | bool | The service needs a separate username next to the API key (`smslogin`) |

A setting the chosen service does not support is ignored by `sms`.

---

## browser/info - Read all settings

```
GET https://ggl.farm/api/browser/info.json?id={apitoken}
```

Returns what the Integration page shows, keys included. It also asks the SMS service and CapMonster for their balance, so a reply can take a few seconds, and a balance may be a few minutes old. Do not call it in a tight loop.

**Parameters:** none.

**Response:**

```json
{
  "status": "ok",
  "data": {
    "browser": {
      "type": 2,
      "name": "OctoBrowser (stable)",
      "api": "<browser-api-key>",
      "ip": "203.0.113.10",
      "port": 0,
      "head": true,
      "wrap": true,
      "code": "<management-key>",
      "limits": { "total": 10, "reg": 3, "farm": 7 }
    },
    "sms": {
      "type": 17,
      "name": "GGL (rent and send)",
      "api": "<sms-api-key>",
      "login": "",
      "max": 0,
      "term": 0,
      "renew": true,
      "balance": 148.5,
      "caps": { "rent": true, "send": true, "price": false, "term": "", "renew": true,
                "termdef": 0, "pricedef": 0, "login": false }
    },
    "capmonster": { "api": "<capmonster-key>", "balance": 12.34 }
  }
}
```

**`browser` block:**

| Field | Type | Meaning |
|---|---|---|
| `type` | int | Browser type, `0` = not configured |
| `name` | string / false | Browser type name, `false` when not configured |
| `api` | string | Browser API key |
| `ip` | string | IP address of the server running the browser, empty if not set |
| `port` | int | Port of the browser local daemon, `0` when not set |
| `head` | bool | Browser runs in headless mode. `true` until you save the form for the first time |
| `wrap` | bool | The browser management server (the GGL connector) is installed |
| `code` | string | Management key of that server. It is part of the browser link, so treat it as a secret |
| `limits` | object | Thread limits: `total` overall, `reg` registration, `farm` farming. `0` = no limit |

**`sms` block:**

| Field | Type | Meaning |
|---|---|---|
| `type` | int | Service ID, `0` when not configured |
| `name` | string / false | Service name, `false` when not configured or the service is no longer offered |
| `api` | string | Service API key (for a two-part key, only the key part) |
| `login` | string | Service username, empty for services without one |
| `max` | number | Maximum number price you set, `0` = the service default |
| `term` | int | Rental term in days as it will be applied (`0` for services without terms) |
| `renew` | bool | Automatic rental renewal |
| `balance` | number / false | Balance at the service: `false` when no key is saved, `0` when the call to the service fails |
| `caps` | object | Capabilities of the selected service, see above |

**`capmonster` block:** `api` - the key (empty if none); `balance` - the CapMonster balance: `false` when no key is saved, `0` when the call to CapMonster fails. After you change the key, the balance can lag by about a minute.

**Errors:** `unpaid`

---

## browser/save - Configure the antidetect browser

```
POST https://ggl.farm/api/browser/save.json?id={apitoken}
```

Saves the browser form of the Integration page. **The form is rewritten as a whole:** a parameter that is missing from the request clears its field or resets it to zero. To change one value, read `info` and send everything back. In particular, omitting `head` turns headless mode off (headless is the recommended mode: send `head=1`).

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `type` | int | yes | Browser type: `1` Vision, `2` OctoBrowser, `3` AdsPower |
| `api` | string | yes | Browser API key |
| `ip` | string | no | IP address of the server the browser runs on. A value that is not a valid IP address is saved as empty |
| `port` | int | no | Port of the browser local daemon, `0`-`65535`; a value out of range is brought to the nearest bound. Leave it empty unless you were told to set it |
| `head` | int | no | `1` headless mode, `0` visible browser. Default 0; any non-empty value other than `0` counts as `1` |
| `wrap` | int | no | `1` if the browser management server is installed, `0` if not. Default 0; same reading as `head` |
| `lim` | int | no | Total thread limit on the server, not less than 0. `0` = no limit |
| `lim1` | int | no | Thread limit for registration, not less than 0. `0` = no limit |
| `lim2` | int | no | Thread limit for farming, not less than 0. `0` = no limit |

A task that would exceed a limit is postponed and tried again a few minutes later.

**Response:** `{ "status": "ok" }`

**Errors:** `bad-type` (unknown browser type), `no-api` (empty `api`), `bad-param` (a request value is an array; the reply also carries `param`, the name of that parameter), `unpaid`

```bash
curl -s -X POST "https://ggl.farm/api/browser/save.json?id=$TOKEN" \
  -d "type=2" -d "api=<browser-api-key>" -d "ip=203.0.113.10" \
  -d "head=1" -d "wrap=1" -d "lim=10" -d "lim1=3" -d "lim2=7"
```

---

## browser/services - List SMS services

```
GET https://ggl.farm/api/browser/services.json?id={apitoken}
```

Returns the SMS services you can choose from and what each one can do; the service picker of the Integration page is built from the same list. Take the `id` from here for `browser/sms`.

**Parameters:** none.

**Response:**

```json
{
  "status": "ok",
  "current": 17,
  "data": [
    { "id": 17, "name": "GGL (rent and send)", "rent": true, "send": true, "price": false,
      "term": "", "renew": true, "termdef": 0, "pricedef": 0, "login": false },
    { "id": 19, "name": "SMS4G (rental)", "rent": true, "send": false, "price": true,
      "term": "", "renew": false, "termdef": 0, "pricedef": 0, "login": false },
    { "id": 2, "name": "textverified.com (rental)", "rent": true, "send": false, "price": false,
      "term": "1,3,7,14,30,90,365", "renew": true, "termdef": 30, "pricedef": 0, "login": true }
  ]
}
```

`current` is the ID of the service selected now, `0` when none. It can be an ID that is no longer in `data` (a service that was withdrawn); pick a new one with `sms`. `data` is a plain list in display order; its items are the capability fields from the vocabulary above plus `id` and `name`.

**Errors:** `unpaid`

---

## browser/sms - Configure the SMS service

```
POST https://ggl.farm/api/browser/sms.json?id={apitoken}
```

Saves the SMS form of the Integration page. **The form is rewritten as a whole:** a missing parameter resets its setting (an omitted `smsrenew` switches renewal off). Get the service ID and its capabilities from `services` first. Parameters the chosen service does not support are ignored.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `smstype` | int | yes | Service ID from `services` |
| `smsapi` | string | yes | Service API key. For a service with a username (`login: true`) pass the key only, and the username in `smslogin`. A combined `"username key"` value is also accepted and split on the first space |
| `smslogin` | string | for `login` services | Service username (textverified.com); ignored by other services |
| `smsmax` | number | no | Maximum phone number price, `0`-`1000000`, only for services with `price: true`. Not a number or `0` = the service default (`pricedef`) |
| `smsterm` | int | no | Rental term in days, only for services with a `term`. With `"*"` any value `0`-`365`; with a list it must be one of the listed days. `0` or an off-list value = the default (`termdef`) |
| `smsrenew` | int | no | `1` renew the rental automatically, only for services with `renew: true`. Default 0; any non-empty value other than `0` counts as `1` |

**Response:** `{ "status": "ok" }`

**Errors:** `bad-type` (unknown or withdrawn service), `no-api` (no key left after splitting), `bad-param` (a request value is an array; the reply also carries `param`), `unpaid`

```bash
# A service with a username: the key and the username go in separate fields
curl -s -X POST "https://ggl.farm/api/browser/sms.json?id=$TOKEN" \
  -d "smstype=2" -d "smslogin=<username>" -d "smsapi=<sms-api-key>" \
  -d "smsterm=30" -d "smsrenew=1"

# A service with a price cap
curl -s -X POST "https://ggl.farm/api/browser/sms.json?id=$TOKEN" \
  -d "smstype=19" -d "smsapi=<sms-api-key>" -d "smsmax=1.5"
```

---

## browser/captcha - CapMonster key

```
POST https://ggl.farm/api/browser/captcha.json?id={apitoken}
```

Saves the API key of the CapMonster captcha recognition service. The key can be replaced but **not cleared**: an empty value is rejected.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `capmonster` | string | yes | CapMonster API key |

**Response:** `{ "status": "ok" }`

**Errors:** `no-capmonster` (empty key), `bad-param` (a request value is an array; the reply also carries `param`), `unpaid`

```bash
curl -s -X POST "https://ggl.farm/api/browser/captcha.json?id=$TOKEN" -d "capmonster=<capmonster-key>"
```

---

## browser/log - Integration journal

```
GET https://ggl.farm/api/browser/log.json?id={apitoken}
```

Downloads the journal of the requests GGL sends to your antidetect browser, newest first: the address, the data sent and the browser reply. These are the records of the Integration journal page. Start here when an account does not launch.

**Parameters** (all optional, combined with AND):

| Name | Type | Description |
|---|---|---|
| `page` | int | Page number, from 1. Default 1 |
| `show` | int | Records per page. Default 50, at most 200: a bigger value is cut to 200 |
| `error` | int | `0` successful requests only, `1` failed requests only. Present but empty counts as `0` |
| `acc` | int | Account ID in GGL: only the records of this account |
| `func` | string | Browser function name as shown in the journal (for example `start`): only those records |

**Response:**

```json
{
  "status": "ok",
  "page": 1,
  "show": 50,
  "total": 100500,
  "data": [
    {
      "id": 100500,
      "time": 1754857600,
      "error": 0,
      "acc": 101,
      "func": "start",
      "code": 200,
      "codeok": true,
      "timer": 0.412,
      "url": "http://203.0.113.10:58307/status",
      "get": false,
      "post": false,
      "postdata": false,
      "reply": "{\"code\":0}",
      "header": "<request headers, may contain your browser API token>",
      "cc": false
    }
  ]
}
```

The paging is flat (`page`, `show`, `total`, as in the accounts list), not a `nav` block; `page` and `show` echo your request (`page` at least 1, `show` at most 200). `data` is a plain list, `[]` when nothing matches.

**Record fields:**

| Field | Type | Meaning |
|---|---|---|
| `id` | int | Record ID |
| `time` | int | Record time, UNIX timestamp |
| `error` | int | `1` when the browser gave no reply at all, otherwise `0`. A reply with an HTTP error code is still `0`: check `code` / `codeok` too |
| `acc` | int | Account ID in GGL the request belongs to |
| `func` | string | Browser function the request performed |
| `code` | int | HTTP code of the browser reply |
| `codeok` | bool | `true` when `code` is 2xx |
| `timer` | float | Request duration, seconds |
| `url` | string / false | Address the request was sent to |
| `get` | object / false | Query parameters of `url`, for requests without a body |
| `post` | string / false | Request body as sent |
| `postdata` | object / `null` / false | The body decoded. A body that starts with `{` or `[` is read as JSON: `null` when that JSON is invalid. Any other body is read as form data, and only a form with at least two fields is decoded: a single-field body, or an empty one, gives `false` |
| `reply` | string / false | Browser reply as is |
| `header` | string / false | Request HTTP headers, one per line |
| `cc` | object / false | Additional request options (may include the proxy used) |

Nothing is masked: `url`, `post`, `header` and `cc` can contain your browser API token, the management key, proxy credentials and account credentials.

**Errors:** `bad-param` (a request value is an array; the reply also carries `param`), `unpaid`

```bash
# The last 20 failed requests of account 101
curl -s "https://ggl.farm/api/browser/log.json?id=$TOKEN&acc=101&error=1&show=20"
```

---

## browser/quota - Thread load of your plan

```
GET https://ggl.farm/api/browser/quota.json?id={apitoken}
```

Shows how many threads of your plan are busy and how long the work queues are: the same block as on the main page of the cabinet. Look at it before you add accounts or tasks. A thread is one running antidetect-browser window, and the plan limits how many run at the same time (see Limits and Quotas in [overview.md](overview.md)). The figures are cached for about a minute, so polling faster gains nothing and does not load the service. For a team member every figure, the limit included, is counted for the whole team.

**Parameters:** none.

**Response:**

```json
{
  "status": "ok",
  "data": {
    "threads": { "used": 37, "limit": 100, "percent": 37 },
    "tasks":   { "queue": 12, "percent": 12 },
    "farm":    { "queue": 25, "percent": 25 },
    "stats":   { "queue": 3,  "percent": 3 }
  }
}
```

| Field | Type | Meaning |
|---|---|---|
| `threads.used` | int | Threads busy right now: accounts whose browser profile is running a task, farming or statistics collection |
| `threads.limit` | int | The most threads your plan allows at once, at least 1. For a team member, the shared team limit |
| `threads.percent` | int | `used` as a percentage of `limit`, rounded, never above `100` |
| `tasks.queue` | int | Queued tasks (registration, login, 2FA) whose start time has come while no thread is free for them yet |
| `farm.queue` | int | Accounts waiting for their next farming run |
| `stats.queue` | int | Accounts waiting for their next statistics collection |
| `tasks.percent`, `farm.percent`, `stats.percent` | int | The queue length as a percentage of `threads.limit`, rounded. **Not capped**: it can go above `100` |

A queue is normal while all threads are busy. A queue percentage above `100` means more work is waiting than the plan can run at once: wait, or move to a bigger plan.

**Errors:** `unpaid`

```bash
curl -s "https://ggl.farm/api/browser/quota.json?id=$TOKEN"
```

---

## Errors Reference

All errors come as `{"status":"error","error":"<code>"}`. The HTTP status is the one described in [overview.md](overview.md).

| Error | HTTP | Functions | Meaning |
|---|---|---|---|
| `bad-type` | 400 | save, sms | Unknown browser type or SMS service ID |
| `no-api` | 400 | save, sms | API key not specified (for `sms`: empty after the username was split off) |
| `no-capmonster` | 400 | captcha | CapMonster key not specified |
| `bad-param` | 400 | save, sms, captcha, log | A request value is an array; the reply also carries `param` with the name of that parameter |
| `unpaid` | 402 | all | Subscription not paid |
