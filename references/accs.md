# Accounts API Reference

## Overview

The `accs` application manages your Google accounts: list and inspect them, bring in existing ones, register new ones, edit, repair and delete them, look at the anti-detect browser, and open an account's browser profile for your own automation. `gang` is the team-level twin of four of these functions.

Base URL: `https://ggl.farm/api/accs/{func}.json?id={apitoken}` (team: `https://ggl.farm/api/gang/{func}.json?id={apitoken}`)

Authentication, the `status`/`error` envelope, HTTP statuses, pagination and limits are described in [overview.md](overview.md). Read functions are GET, write functions are POST; all parameters may be sent in the query string, a form body or a JSON body (keep `id` in the URL: `authid` works in a form body only, not in a JSON body). Examples assume `ID` holds your token (`<user id>-<API key>`).

| Function | Purpose | Team twin |
|---|---|---|
| `accs/list` | search accounts, or only their IDs | `gang/list` |
| `accs/get` | one account with proxy, farm settings, cookies | `gang/get` |
| `accs/make` | register new accounts | - |
| `accs/push` | add an account that already has a browser profile | - |
| `accs/add` | connect an existing account by login and password | - |
| `accs/edit` | change account data, proxy, farm settings | - |
| `accs/fix` | bulk repair | `gang/fix` |
| `accs/del` | bulk delete | `gang/del` |
| `accs/status` | is the browser integration ready | - |
| `accs/folder` | folders of your anti-detect browser | - |
| `accs/active` | running profiles | - |
| `accs/all` | all profiles | - |
| `accs/start` | open the profile for your own automation | - |
| `accs/ping` | prolong that session | - |
| `accs/stop` | close that session | - |

The load of your plan is read with `browser/quota` ([browser.md](browser.md)); `accs/quota` no longer exists and answers `func`.

The farm needs three things set up first, in the GGL interface or through the [browser application](browser.md): the anti-detect browser, an SMS service (registration and login) and your own proxies ([proxy.md](proxy.md)).

---

## Vocabularies

**Account `status`** (lifecycle):

| Value | Name | Meaning |
|---|---|---|
| 1 | Waiting | created or connected; within about a minute GGL creates the browser profile |
| 2 | Creating | the profile is created, the registration or login task is queued or running |
| 3 | Farming | working account (also the state of an account added with `accs/push`) |

**`active`**: `true` for a live account, `false` for an inactive ("dead") one that is out of service. Error 200 and 201 switch it off by themselves; `fix` with `kill` / `ressurect` toggles it.

**`task`** (what the account is busy with right now; the same numbers as the task types in [task.md](task.md)):

| Value | Meaning |
|---|---|
| 0 | free |
| 1 | Registration |
| 2 | Farm |
| 3 | Statistics |
| 4 | 2FA |
| 5 | Login |

An account held by your own manual browser session (`accs/start`) still reads `task: 0`.

**Account `error`**:

| Code | Meaning | What to do |
|---|---|---|
| 0 | No errors | |
| 101 | Fatal error registering the account | delete the account and its profile, register a new one |
| 102 | No phone from the SMS service | top up the SMS balance or change `smsgeo`, then `fix` with `restart` |
| 200 | Dead account (also sets `active` to `false`) | delete it |
| 201 | Browser profile not found (also sets `active` to `false`) | it was removed from the browser; delete the account or add it again |
| 300 | Advertising cabinet failure found while collecting statistics | |
| 901 | Proxy not set | `fix` with `fixproxy` or `newproxy` |
| 902 | Proxy error | gone by itself within hours if the proxy is alive, otherwise `fix` with `fixproxy` |
| 903 | Error launching the browser profile | update the anti-detect browser, check the server; retried automatically |
| 904 | Profile already in use at launch | GGL stopped it and retries in a few minutes |
| 905 | Error creating the browser profile | check free profiles in the browser; retried automatically |

Codes 901-905 are temporary: they clear by themselves once the account works again. `fix` with `fixerror` hides an error without repairing it. Treat a code that is not listed as an unknown failure.

**Farm period `farmnext`** (seconds between farming runs):

| Value | Period |
|---|---|
| 85000 | once a day (default) |
| 42000 | twice a day |
| 27000 | three times a day |
| 170000 | once in two days |
| 259000 | once in three days |
| 604000 | once a week |

**`os`** (platform of a created profile): `1` Windows (default), `2` MacOS, `3` Linux. Send the number; words such as `mac` are not read and give Windows.

**Proxy block** (`accs/get`): `status` 0 waiting, 1 active, 2 error; `type` 0 http, 1 https, 2 socks5, 3 socks5h.

**Task statuses** (named by `fix` flags): 0 In queue, 1 Waiting, 2 Working, 3 Problem, 4 Error, 5 Success.

**Farm settings** are the same parameters in `make`, `push`, `add` and `edit`:

| Name | Type | Values / default | Meaning |
|---|---|---|---|
| `farmon` | int | `0` / `1`, default 0 | turn farming on for the account |
| `farmnext` | int | the periods above, default 85000 | how often to farm |
| `noyt` | int | `0` / `1`, default 0 | do not watch YouTube while farming (saves traffic) |

In `make`, `push` and `add` an unknown `farmnext` falls back to 85000; `edit` rejects it with `bad-farmnext`.

---

## Account Object (list)

```json
{
  "id": 32,
  "user": 23,
  "active": true,
  "status": 3,
  "error": 0,
  "task": 0,
  "folder": "0f1e2d3c4b5a69788796a5b4c3d2e1f0",
  "profile": "a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "create": "1751397362",
  "name": "Jane",
  "last": "Doe",
  "login": "jane.doe",
  "email": "jane.doe@example.com",
  "recover": "recovery@example.com",
  "pass": "Example-Pass-1",
  "phone": "15550100123",
  "2fa": false,
  "dob": "1990-06-12",
  "gender": "f",
  "tags": ["batch7", "aged"],
  "geo": "us",
  "smsgeo": "zz",
  "farm": true,
  "farmed": 13
}
```

| Field | Type | Meaning |
|---|---|---|
| `id` | int | account ID in GGL |
| `user` | int | ID of the owning user (a team member for `gang/*`) |
| `active` | bool | live (`true`) or inactive (`false`) |
| `status` | int | lifecycle stage, see Vocabularies |
| `error` | int | error code, `0` = none |
| `task` | int | task the account is running now, `0` = free |
| `folder` | string or `false` | folder/tag ID in the anti-detect browser |
| `profile` | string or `false` | profile ID in the anti-detect browser |
| `create` | string | creation time, UNIX timestamp (a string) |
| `name`, `last` | string | first and last name |
| `login` | string | Google login (the part before `@`) |
| `email` | string | Google e-mail |
| `recover` | string or null | recovery e-mail |
| `pass` | string | password, returned in clear |
| `phone` | string or `false` | last phone number used for SMS |
| `2fa` | string or `false` | two-factor key (TOTP secret) |
| `dob` | string or `false` | date of birth, `YYYY-MM-DD` |
| `gender` | string | `m` or `f` |
| `tags` | array of string | tags of the account |
| `geo` | string | account country |
| `smsgeo` | string | country for SMS, `zz` = the account country |
| `farm` | bool | farming is on |
| `farmed` | int | days the account has been farming |

Credentials (`pass`, `2fa`, `recover`) come back in clear text: keep the responses private.

---

## accs/list - List accounts

```
GET https://ggl.farm/api/accs/list.json?id={apitoken}
GET https://ggl.farm/api/gang/list.json?id={apitoken}      (team)
```

Brief account data, no cookies or proxy (use `accs/get`). Sorted by ID ascending. All filters are optional and combined with AND. `farm`, `farmed`, `farmedf`, `farmedt`, `error`, `proxy` and `busy` count a present but empty value as `0` (`error=` means error 0, `busy=` means free accounts); an empty or zero `status` or `user` is no filter.

| Name | Type | Req | Values / default | Meaning |
|---|---|---|---|---|
| `page` | int | no | 1 | page number from 1 |
| `show` | int or `ids` | no | 100, no upper limit | rows per page; `ids` returns only the IDs of all matches |
| `user` | int | no | team only | team member ID |
| `status` | int | no | 1 Waiting, 2 Creating, 3 Farming | lifecycle stage |
| `farm` | int | no | 0 / 1 | farming off / on |
| `farmed` | int | no | days | exactly this many days farmed |
| `farmedf` | int | no | days | days farmed, from |
| `farmedt` | int | no | days | days farmed, up to |
| `error` | int | no | error code, `0` = no error | accounts with this error |
| `proxy` | int | no | proxy ID | accounts on this proxy |
| `geo` | string | no | country code | account country |
| `smsgeo` | string | no | country code | SMS country |
| `folder` | string | no | folder ID | accounts in this folder |
| `s` | string | no | text | search in first name, last name, e-mail, login, password, 2FA key and phone |
| `dead` | int | no | `0` active only (default), `1` all, `2` inactive only | filter by `active` |
| `busy` | int | no | `0` free, `-1` any busy, `1`-`5` by task type | filter by what the account is doing |
| `tag` | string | no | tags separated by spaces | accounts carrying all of the tags. A full-text match: a tag shorter than 3 characters is not found |

**Response:** `{"status":"ok","page":1,"show":100,"total":237,"data":[ /* account objects */ ]}`. `total` counts all matches, not only this page.

**Response with `show=ids`:** `{"status":"ok","data":[32,58,59,64]}`. The complete matching set, no paging.

**Errors:** `unpaid`, `bad-param`; team level also `no-gang`.

```bash
# first page of live US accounts with farming on
curl -s "https://ggl.farm/api/accs/list.json?id=$ID&geo=us&farm=1&show=50&page=1"
# IDs of everything with error 902, inactive accounts included
curl -s "https://ggl.farm/api/accs/list.json?id=$ID&show=ids&error=902&dead=1"
```

---

## accs/get - One account

```
GET https://ggl.farm/api/accs/get.json?id={apitoken}&acc={id}
GET https://ggl.farm/api/gang/get.json?id={apitoken}&acc={id}      (team)
```

| Name | Type | Req | Values / default | Meaning |
|---|---|---|---|---|
| `acc` | int | yes | account ID | the account to read |

**Response:**

```json
{
  "status": "ok",
  "data": {
    "id": 666, "user": 1, "active": true, "status": 3, "error": 0, "task": 0,
    "folder": "0f1e2d3c4b5a69788796a5b4c3d2e1f0",
    "profile": "a1b2c3d4e5f60718293a4b5c6d7e8f90",
    "create": "1751811704",
    "tags": [],
    "person": {
      "name": "Jane", "last": "Doe", "login": "jane.doe",
      "email": "jane.doe@example.com", "recover": "recovery@example.com",
      "pass": "Example-Pass-1", "phone": "15550100123", "2fa": false,
      "dob": "1990-06-12", "gender": "f", "geo": "gb", "smsgeo": "zz"
    },
    "farm": { "active": true, "last": 1754598800, "next": 1754751556, "days": 22,
              "config": { "next": 85000, "noyt": false } },
    "proxy": {
      "status": 1, "type": 2, "host": "203.0.113.10", "port": 1080,
      "login": "proxyuser", "pass": "proxypass", "url": "",
      "connect": "socks5://proxyuser:proxypass@203.0.113.10:1080",
      "ip": "203.0.113.10", "geo": "gb"
    },
    "meta": [ { "id": "bill", "value": "12.40", "name": "Current bill" } ],
    "cookie": []
  }
}
```

| Field | Type | Meaning |
|---|---|---|
| `id` `user` `active` `status` `error` `task` `folder` `profile` `create` `tags` | | as in the account object |
| `person` | object | `name`, `last`, `login`, `email`, `recover`, `pass`, `phone`, `2fa`, `dob`, `gender`, `geo`, `smsgeo` as in the account object |
| `farm.active` | bool | farming is on |
| `farm.last` / `farm.next` | int | last run / next scheduled run, UNIX timestamp, `0` = never / not scheduled |
| `farm.days` | int | days farmed |
| `farm.config` | object or `[]` | the farm settings: `next` (period in seconds, see `farmnext`) and `noyt` |
| `proxy` | object or `false` | the assigned proxy; `false` when the account has none |
| `proxy.status` / `proxy.type` | int | see Vocabularies |
| `proxy.host` / `port` / `login` / `pass` | | proxy address and credentials |
| `proxy.url` | string | link that resets the proxy IP, empty if none |
| `proxy.connect` | string | ready connection link; both `socks5` and `socks5h` proxies are shown with the `socks5://` scheme |
| `proxy.ip` | string or `false` | external IP of the proxy |
| `proxy.geo` | string | proxy country, `zz` = unknown |
| `meta` | array | extra account data (advertising cabinet figures): items `{id, value, name}`; `id` is one of `convid`, `convlbl`, `bill`, `billed`, `campaign`, `problem`, `name` is the label in your interface language; `[]` when there is none |
| `cookie` | array | cookies in the standard browser format, `[]` when none |

**Errors:** `no-id` (`acc` missing), `bad-id` (no such account among yours), `unpaid`; team level also `no-gang`.

---

## accs/make - Register new accounts

```
POST https://ggl.farm/api/accs/make.json?id={apitoken}
```

Creates placeholder accounts in the list and their registration tasks (login tasks in login mode, see `login`). Needs your own free proxies, a configured SMS service and the browser integration. The reply carries **no account IDs**: tag the batch and find it with `accs/list&tag=...`.

| Name | Type | Req | Values / default | Meaning |
|---|---|---|---|---|
| `amount` | int | yes | at least 1, at most your free proxy slots | how many accounts |
| `geo` | string | yes | a supported country | account country |
| `folder` | string | yes | ID from `accs/folder` | browser folder to create the profiles in |
| `smsgeo` | string | no | country code, default = `geo` | country for SMS numbers |
| `proxygeo` | string | no | country code | proxy country. When set, only proxies of that country are used and an account without one is skipped; when omitted a proxy of the account country is preferred and any free proxy is used if there is none |
| `os` | int | no | 1, 2, 3, default 1 | profile platform |
| `delay` | int | no | 0-30 days | postpone the registration |
| `tags` | string | no | letters, digits, `-` `_` `.` (stored lowercase), 3+ characters | **one** tag for the batch. Several tags are glued into one: add more with `accs/edit`. A tag shorter than 3 characters cannot be found with the `tag` filter |
| `email` | string | no | valid e-mail | e-mail of the created accounts; meant for the single-account `accs/add`, leave it out in a batch. An invalid value answers `bad-email` |
| `name`, `last` | string | no | | first and last name |
| `pass` | string | no | | password |
| `recover` | string | no | e-mail | recovery e-mail |
| `2fa` | string | no | TOTP secret | two-factor key |
| `login` | string | no | any non-empty value | switches the batch to login mode: every account gets a login task instead of a registration (`accs/add` sends it for you). Only whether it is set matters, the login itself comes from `email` |
| `farmon`, `farmnext`, `noyt` | | no | see Vocabularies | farm settings, farming is off unless `farmon=1` |

In registration mode (no `login`) GGL generates the person data itself and replaces `name`, `last`, `pass` and `email`; send them for login mode.

**Response:** `{"status":"ok"}`. It confirms the request was accepted, not that every account was created (with a strict `proxygeo` some may be skipped); the accounts appear in the list with status 1 (Waiting).

**Errors, in the order checked:** `unpaid`, `sms-service` (SMS integration not set up), `bad-param`, `bad-amount` (amount below 1), `no-proxy` (not enough free proxy slots for `amount`), `antidetect-browser` (browser integration not set up), `wrong-folder` (folder not in the browser), `bad-geo` (country not supported, the list is in `supported-geo`), `bad-email` (invalid `email`), `make` (no account was created, for example no free proxy of a strict `proxygeo`).

```bash
curl -s -X POST "https://ggl.farm/api/accs/make.json" \
  -d "authid=$ID" -d "amount=5" -d "geo=us" -d "folder=0f1e2d3c4b5a69788796a5b4c3d2e1f0" \
  -d "proxygeo=us" -d "os=1" -d "tags=batch7" -d "farmon=1" -d "farmnext=85000"
```

---

## accs/push - Add an account that already has a browser profile

```
POST https://ggl.farm/api/accs/push.json?id={apitoken}
```

For an account that is already registered and lives in your anti-detect browser: find the folder with `accs/folder` and the profile with `accs/all`. One account per call. The account is stored with status 3 (Farming); GGL does not create a profile and queues no login. Parameters that are not in the list below are ignored.

| Name | Type | Req | Values / default | Meaning |
|---|---|---|---|---|
| `login` | string | yes | login or e-mail | with an e-mail the login is its local part; with a bare login the e-mail becomes `<login>@gmail.com` |
| `pass` | string | yes | | password |
| `geo` | string | yes | a supported country | account country |
| `folder` | string | yes, unless `account` | ID from `accs/folder` | folder in the browser; checked against the folder list and stored |
| `profile` | string | yes, unless `account` | profile ID | the browser profile |
| `account` | string | no | `folder/profile` | folder and profile in one value, split at the first `/`. It wins: when sent, `folder` and `profile` are ignored, and its folder part is the one checked and stored. Without a `/` part the profile is empty (`no-profile`) |
| `smsgeo` | string | no | country code, default = `geo` | country for SMS |
| `2fa` | string | no | TOTP secret | two-factor key |
| `recover` | string | no | e-mail | recovery e-mail |
| `name`, `last` | string | no | | first and last name |
| `tags` | string | no | lowercase, comma or space separated | tags |
| `proxy` | string | no | `proto://login:pass@host:port`, proto `http` `https` `socks5` `socks5h` | proxy to assign. If it cannot be parsed, or belongs to another user, the account is added without a proxy and the browser profile keeps its own |
| `phone` | string | no | digits | rented phone number |
| `phones` | string | no | | rental ID at the SMS service |
| `farmon`, `farmnext`, `noyt` | | no | see Vocabularies | farm settings, farming is off unless `farmon=1` |

**Response:** `{"status":"ok","id":123}` - the new account ID.

**Errors, in the order checked:** `unpaid`, `antidetect-browser`, `bad-param`, `wrong-folder`, `no-login`, `no-pass`, `no-profile` (no profile given), `already-exists` (this profile is already attached to one of your accounts), `bad-geo` (list in `supported-geo`), `push` (could not add the account).

```bash
curl -s -X POST "https://ggl.farm/api/accs/push.json" \
  -d "authid=$ID" -d "geo=us" \
  -d "account=0f1e2d3c4b5a69788796a5b4c3d2e1f0/a1b2c3d4e5f60718293a4b5c6d7e8f90" \
  -d "login=jane.doe@example.com" -d "pass=Example-Pass-1" \
  -d "2fa=JBSWY3DPEHPK3PXP" -d "proxy=socks5://proxyuser:proxypass@203.0.113.10:1080" -d "farmon=1"
```

---

## accs/add - Connect an account by login and password

```
POST https://ggl.farm/api/accs/add.json?id={apitoken}
```

The API form of "Connect accounts" in single-account mode. GGL adds the account, assigns a free proxy, creates the browser profile by itself and queues the login task. One account per call. Status starts at 1 (Waiting); within a minute the profile is created and the login queued. The reply has no ID: find the account with `accs/list` (filter by `tag` or `s=<email>`).

| Name | Type | Req | Values / default | Meaning |
|---|---|---|---|---|
| `email` | string | yes | valid e-mail | Google e-mail |
| `pass` | string | yes | | password |
| `geo` | string | yes | a supported country | account country |
| `folder` | string | yes | ID from `accs/folder` | folder to create the profile in |
| `os` | int | no | 1, 2, 3, default 1 | profile platform |
| `proxygeo` | string | no | country code | proxy country (same rule as in `make`) |
| `smsgeo` | string | no | country code, default = `geo` | country for SMS |
| `2fa` | string | no | TOTP secret | two-factor key |
| `recover` | string | no | e-mail | recovery e-mail |
| `name`, `last` | string | no | | first and last name for the profile |
| `tags` | string | no | letters, digits, `-` `_` `.` (stored lowercase), 3+ characters | **one** tag; several are glued into one (add more with `accs/edit`). A tag shorter than 3 characters cannot be found with the `tag` filter |
| `delay` | int | no | 0-30 days | postpone the login task only; the profile is still created within about a minute |
| `farmon`, `farmnext`, `noyt` | | no | see Vocabularies | farm settings, farming is off unless `farmon=1` |

**Response:** `{"status":"ok"}`.

**Errors, in the order checked:** `unpaid`, `antidetect-browser`, `bad-param`, `wrong-folder`, `no-email`, `bad-email`, `no-pass`, `bad-geo` (list in `supported-geo`), `no-proxy` (no free proxy), `add` (nothing was created, for example no free proxy of a strict `proxygeo`).

```bash
curl -s -X POST "https://ggl.farm/api/accs/add.json" \
  -d "authid=$ID" -d "email=jane.doe@example.com" -d "pass=Example-Pass-1" \
  -d "geo=us" -d "folder=0f1e2d3c4b5a69788796a5b4c3d2e1f0" -d "tags=imported" -d "farmon=1"
```

---

## accs/edit - Edit an account

```
POST https://ggl.farm/api/accs/edit.json?id={apitoken}&acc={id}
```

Changes one account: the card fields, an exact proxy and the farm settings in one call. Send only what changes; the rest stays. **Own accounts only:** at team level call it with the key of the account owner.

| Name | Type | Req | Values / default | Meaning |
|---|---|---|---|---|
| `acc` | int | yes | account ID | the account |
| `geo` | string | no | a supported country | account country (changing it after creation is discouraged) |
| `smsgeo` | string | no | country code, `zz` = account country | country for SMS |
| `tags` | string | no | lowercase, comma or space separated | **replaces** all tags; empty clears them |
| `2fa` | string | no | TOTP secret | two-factor key; `c2fa` is a synonym |
| `name`, `last` | string | no | | first and last name |
| `email` | string | no | valid e-mail, empty clears | Google e-mail |
| `recover` | string | no | | recovery e-mail |
| `pass` | string | no | | password |
| `dob` | string | no | `YYYY-MM-DD` | date of birth; an unreadable value clears it |
| `phone` | string | no | digits | phone number for SMS |
| `phones` | string | no | | rental ID at the SMS service |
| `folder`, `profile` | string | no | IDs in the browser | not checked against the browser |
| `comment` | string | no | | free-text note |
| `proxy` | string | no | `proto://login:pass@host:port` | exact proxy to assign. The previous proxy is not released: its slot stays taken until the proxy counters are recalculated: `proxy/load` does it on every call, `proxy/add` only when it really added a proxy (a duplicate does not) |
| `farmon`, `farmnext`, `noyt` | | no | see Vocabularies | farm settings; any you leave out keep their current value |

**Response:** `{"status":"ok"}`. Sending the same values again is also `ok`.

**Errors, in the order checked:** `unpaid`, `no-id`, `bad-id`, `bad-param`, `no-fields` (nothing editable was sent; a lone empty `proxy=` is not an error, it returns `ok` and changes nothing), `unpaid` again when you change farm settings without a plan that includes farming, `bad-geo` (list in `supported-geo`), `bad-email`, `bad-farmnext` (allowed values in `supported-farmnext`), `bad-proxy` (cannot be parsed, or the proxy belongs to another user), `edit` (account data not saved), `farm` (farm settings not saved).

```bash
curl -s -X POST "https://ggl.farm/api/accs/edit.json?id=$ID&acc=666" \
  -d "tags=aged warm" -d "farmon=1" -d "farmnext=42000" -d "noyt=1"
```

---

## accs/fix - Repair accounts in bulk

```
POST https://ggl.farm/api/accs/fix.json?id={apitoken}&ids={ids}
POST https://ggl.farm/api/gang/fix.json?id={apitoken}&ids={ids}      (team)
```

The "Repair accounts" form. Send only the flags you need; they are independent except where a row says otherwise. IDs that are not yours (team: not in your team) are skipped silently.

| Name | Type | Req | Values / default | Meaning |
|---|---|---|---|---|
| `ids` | list | yes | account IDs, comma separated or an array | accounts to repair |
| `fixproxy` | int | no | `1` | replace broken proxies (the proxy does not answer) and give one to an account that has none |
| `newproxy` | int | no | `1` | replace all proxies. Ignored when `fixproxy` is also sent |
| `anyproxy` | int | no | `1` | allow proxies from other countries when your country has too few |
| `unproxy` | int | no | `1` | detach the proxy from the account and free its slot; the browser profile then manages its own. Does nothing when `fixproxy` or `newproxy` is also sent (precedence: `fixproxy`, then `newproxy`, then `unproxy`) |
| `fixerror` | int | no | `1` | hide errors (they are not repaired) |
| `kill` | int | no | `1` | make accounts inactive (`active: false`, out of service); their Waiting and Working tasks become Problem |
| `ressurect` | int | no | `1` | make accounts active again; stopped tasks do not restart by themselves, use `restart`. Ignored when `kill` is also sent |
| `restart` | int | no | `1` | restart a task in Problem or Error status (one task per account per call) |
| `noproblem` | int | no | `1` | delete a Problem task (one per account per call) |
| `noerrors` | int | no | `1` | delete an Error task (one per account per call) |
| `requeue` | int | no | `1` | fix a stuck queue: moves the first task to Waiting, but only when all of the account's open tasks are In queue (nothing changes if any is Waiting, Working, Problem or Error). Ignored when `unqueue` is also sent |
| `unqueue` | int | no | `1` | delete tasks In queue and Waiting; registration tasks stay |
| `farmon` | int | no | `1` | turn farming on, settings unchanged |
| `farmoff` | int | no | `1` | turn farming off, settings unchanged. Does nothing when `farmon` is also sent |
| `farmnow` | int | no | `1` | run a farming visit out of turn (also turns farming on) |
| `statson` | int | no | `1` | turn advertising-cabinet statistics on |
| `statsoff` | int | no | `1` | turn it off. Does nothing when `statson` is also sent |
| `update` | int | no | `1` | run a statistics update out of turn (also turns statistics on) |
| `sync` | int | no | `1` | update the profile description and settings in the browser |
| `cookie` | int | no | `1` | refresh the account's cookies from its browser profile (they show up in `accs/get` within a few minutes). Works with Vision and AdsPower only; when the browser returns no cookies the request is dropped and the old ones stay |
| `forcestop` | int | no | `1` | stop the browser profile even if a task or a manual session is running, and free the account; a running task goes back to Waiting |
| `existing` | int | no | `1` | check that the profile still exists in the browser; if not, the account becomes inactive with error 201. Accounts without a profile are skipped, and so is everything when the browser returned no profile list |

**Response:** `{"status":"ok"}` - no per-account result.

**Errors:** `unpaid`, `bad-param`, `no-ids` (list missing, or none of the IDs is yours); team level also `no-gang`.

```bash
# replace broken proxies (other countries allowed), hide stuck errors, restart stopped tasks
curl -s -X POST "https://ggl.farm/api/accs/fix.json" \
  -d "authid=$ID" -d "ids=101,102,145" -d "fixproxy=1" -d "anyproxy=1" -d "fixerror=1" -d "restart=1"
```

---

## accs/del - Delete accounts

```
POST https://ggl.farm/api/accs/del.json?id={apitoken}&ids={ids}
POST https://ggl.farm/api/gang/del.json?id={apitoken}&ids={ids}      (team)
```

Deletes the accounts and all their tasks. Irreversible.

| Name | Type | Req | Values / default | Meaning |
|---|---|---|---|---|
| `ids` | list | yes | account IDs, comma separated or an array | accounts to delete |
| `proxy` | int | no | `0` / `1`, default 0 | delete the proxy too. Without it the proxy stays in your pool and its slot is freed. Other accounts that share the proxy are left without one |
| `profile` | int | no | `0` / `1`, default 0 | delete the browser profile |
| `phone` | int | no | `0` / `1`, default 0 | cancel the phone rental at the SMS service |

**Response:** `{"status":"ok"}`.

**Errors:** `unpaid`, `bad-param`, `no-ids`; team level also `no-gang`.

Many anti-detect browsers limit API requests, so deleting a large batch with `profile=1` may leave some profiles in the browser; delete in smaller batches. To take accounts out of service without deleting them use `accs/fix` with `kill`.

```bash
curl -s -X POST "https://ggl.farm/api/accs/del.json" \
  -d "authid=$ID" -d "ids=101,102" -d "profile=1" -d "phone=1"
```

---

## accs/status - Is the browser integration ready

```
GET https://ggl.farm/api/accs/status.json?id={apitoken}
```

No parameters. Cached for about a minute.

**Response:** `{"status":"ok","data":{"api":true,"ip":true,"folder":true,"active":true}}`

| Field | `true` when |
|---|---|
| `api` | the browser API key is saved in the settings |
| `ip` | the server address is saved in the settings |
| `folder` | the key works: the folder list was received |
| `active` | the server answers: the list of running profiles was received |

**Errors:** `unpaid`.

---

## accs/folder - Folders of your browser

```
GET https://ggl.farm/api/accs/folder.json?id={apitoken}
```

No parameters. The folder (or tag) ID is what `make`, `push` and `add` need. Cached for about a minute.

**Response:** `{"status":"ok","data":{"0f1e2d3c4b5a69788796a5b4c3d2e1f0":"Google","7a6b5c4d3e2f10918273645546372819":"Spare"}}` - a map of folder ID to name. `data` is `false` when the integration is not set up or the browser did not answer.

**Errors:** `unpaid`.

---

## accs/active - Running profiles

```
GET https://ggl.farm/api/accs/active.json?id={apitoken}
```

No parameters. **Response:** `{"status":"ok","data":["a1b2c3d4e5f60718293a4b5c6d7e8f90","9f8e7d6c5b4a39281706f5e4d3c2b1a0"]}` - profile IDs currently running on your server, without folders. `data` is `false` when the integration is not set up or the server did not answer. With AdsPower the list is always `[]`: it does not report running profiles. Cached for about a minute. **Errors:** `unpaid`.

---

## accs/all - All profiles

```
GET https://ggl.farm/api/accs/all.json?id={apitoken}
```

No parameters. Every profile of your browser, running or stopped; use it to find the folder and profile for `accs/push`. Cached for about a minute.

**Response:** `{"status":"ok","data":[{"profile":"a1b2c3d4e5f60718293a4b5c6d7e8f90","name":"Jane Doe","folder":"0f1e2d3c4b5a69788796a5b4c3d2e1f0","dir":"Google"}]}`

| Field | Meaning |
|---|---|
| `profile` | profile ID in the browser |
| `name` | profile name |
| `folder` | folder ID |
| `dir` | folder name |

`data` is `false` when the integration is not set up or the browser did not answer. **Errors:** `unpaid`.

---

## Manual browser session: accs/start, accs/ping, accs/stop

Build your own automation (Playwright, Puppeteer, Selenium) on top of an account. GGL opens the account's profile in **your** anti-detect browser and returns a CDP (Chrome DevTools Protocol) link; your script attaches to it. While the session lasts, the robot leaves that account alone: farming, statistics, registration, login and 2FA do not start, and the accounts that share its proxy wait too.

Own accounts only (no team twin). Dead accounts can be started. A session lasts 30 minutes from `start` or from the last `ping`; GGL closes a lapsed session by itself within about a minute, but call `stop` as soon as you are done.

```
GET https://ggl.farm/api/accs/start.json?id={apitoken}&acc={id}
GET https://ggl.farm/api/accs/ping.json?id={apitoken}&acc={id}
GET https://ggl.farm/api/accs/stop.json?id={apitoken}&acc={id}
```

| Name | Type | Req | Values / default | Meaning |
|---|---|---|---|---|
| `acc` | int | yes | account ID | the account (all three functions) |

**`accs/start` response:** `{"status":"ok","connect":"ws://203.0.113.20:9222/devtools/browser/d3cd4c0f","expire":1754857600}`

| Field | Meaning |
|---|---|
| `connect` | WebSocket (CDP) link to the running profile. It gives full control of the browser: keep it secret |
| `expire` | end of the session, UNIX timestamp |

The launch can take up to a minute and a half: use a request timeout of 120 s or more.

**`accs/ping` response:** `{"status":"ok","expire":1754859400}` - the new end of the session (30 minutes from now). Call it at least every half hour.

**`accs/stop` response:** `{"status":"ok"}` - the profile is closed and the account is free for the robot again.

| Function | Errors |
|---|---|
| `start` | `unpaid`, `no-id`, `bad-id`, `no-profile` (the account has no browser profile), `antidetect-browser` (integration not set up), `locked` (your per-user launch lock, shared with the robot, is held: the robot is launching a task for you or another `start` is running; retry in a moment), `busy` |
| `ping` | `unpaid`, `no-id`, `bad-id`, `not-started` (no manual session is running), `ping` (could not prolong) |
| `stop` | `unpaid`, `no-id`, `bad-id`, `not-started`, `stop` (could not stop the profile) |

`busy` on `start` means a task or another manual session holds the account (or an account that shares its proxy), or the browser failed to launch the profile; the account is released again. When the profile was already open in the browser (for example by hand) the reply also has `connect` set to a browser code such as `-904` and a `debug` field with the browser's answer: repeat the request and the profile starts anew. A running session is never extended by a second `start`. A session started here can be ended by `accs/fix` with `forcestop`.

```bash
# open the profile (up to ~90 s), keep it alive, close it
curl -s -m 120 "https://ggl.farm/api/accs/start.json?id=$ID&acc=501"
curl -s "https://ggl.farm/api/accs/ping.json?id=$ID&acc=501"
curl -s "https://ggl.farm/api/accs/stop.json?id=$ID&acc=501"
```

With Playwright for Python: `browser = p.chromium.connect_over_cdp(started["connect"])`, then work with `browser.contexts[0]` and its first page; put `stop` in a `finally` block.

---

## gang/* - Team level

`gang/list`, `gang/get`, `gang/fix`, `gang/del` take the same parameters, return the same fields and raise the same errors as the `accs/*` function of the same name. Differences:

| | `accs/*` | `gang/*` |
|---|---|---|
| Accounts addressed | yours | every account of your team |
| Who may call | any user | the team administrator |
| Extra parameter | - | `user` on `gang/list`: only this member's accounts |
| Extra error | - | `no-gang` (HTTP 400): the caller is not a team administrator or has no team |

`gang/fix` and `gang/del` change and delete team members' accounts without asking them.

```bash
# every team account with error 200, one member only
curl -s "https://ggl.farm/api/gang/list.json?id=$ID&error=200&dead=1&user=57"
# resurrect two members' accounts
curl -s -X POST "https://ggl.farm/api/gang/fix.json" -d "authid=$ID" -d "ids=901,902" -d "ressurect=1"
```

---

## Common Errors Reference

| Error | Functions | Meaning |
|---|---|---|
| `unpaid` | all | subscription not paid (on `edit` also: farm settings without a plan that includes farming) |
| `bad-param` | `list`, `make`, `push`, `add`, `edit`, `fix`, `del` | a request value is an array (`name[]=...`, or a JSON array or object); the name comes in the `param` field. `ids` of `fix` and `del` is exempt; `get`, `start`, `ping`, `stop` read only `acc` and do not check |
| `no-gang` | `gang/*` | caller is not a team administrator |
| `no-id` / `bad-id` | `get`, `edit`, `start`, `ping`, `stop` | account ID missing / not found among yours |
| `no-ids` | `fix`, `del` | no IDs sent, or none of them is yours |
| `antidetect-browser` | `make`, `push`, `add`, `start` | browser integration not set up |
| `wrong-folder` | `make`, `push`, `add` | folder not found in the browser |
| `sms-service` | `make` | SMS integration not set up |
| `bad-amount` | `make` | amount below 1 |
| `no-proxy` | `make`, `add` | not enough free proxy slots |
| `bad-geo` | `make`, `push`, `add`, `edit` | country not supported; allowed list in `supported-geo` |
| `no-login` | `push` | login missing |
| `no-pass` | `push`, `add` | password missing |
| `no-profile` | `push`, `start` | `push`: no profile given; `start`: the account has no browser profile |
| `no-email` | `add` | e-mail missing |
| `bad-email` | `make`, `add`, `edit` | e-mail invalid |
| `already-exists` | `push` | the profile is already attached to one of your accounts |
| `no-fields` | `edit` | no editable parameter sent |
| `bad-proxy` | `edit` | proxy line unreadable or owned by another user |
| `bad-farmnext` | `edit` | period not allowed; list in `supported-farmnext` |
| `make` / `push` / `add` / `edit` / `farm` | the function of that name | the operation failed |
| `locked` / `busy` | `start` | your per-user launch lock is held, retry / account held by a task or another session |
| `not-started` | `ping`, `stop` | no manual session is running |
| `ping` / `stop` | `ping`, `stop` | the session could not be prolonged / the profile could not be stopped |
