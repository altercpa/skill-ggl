---
name: altercpa-ggl-api
description: GGL.farm API and helper - a Google-account farm run through your own antidetect browser (Vision, OctoBrowser, AdsPower). Use whenever making calls to the ggl.farm API, writing scripts or integrations against it, or explaining how GGL.farm behaves. Covers accounts (list, get, make, push, add, edit, fix, del, manual browser sessions over CDP), the team-level gang functions, proxies (list, get, add, load, edit, check, del), tasks (queue login or 2FA, restart, cancel), integration settings (antidetect browser, SMS services, CapMonster, request journal, thread quota) and the customer FAQ (farming, threads, error codes 101-906, the repair form). Triggers on ggl.farm, GGL farm, Google account farm, antidetect browser profiles, proxy slots, 2FA task, SMS number rental, CapMonster key, thread quota, accs/make, accs/push, accs/add, accs/edit, accs/fix, proxy/load, task/add, browser/save, browser/quota.
---

# GGL.farm - API and helper

GGL.farm keeps a farm of Google accounts: it stores your accounts and proxies, runs a task queue and works through **your own antidetect browser** on your own server. For every account the robot opens a profile, logs in, farms (browsing visits, by default once a day) and collects advertising-cabinet statistics. The API repeats the web interface, so a script can drive hundreds of accounts. Plans, prices and thread limits: https://ggl.farm.

Read the matching reference before writing a call: exact parameters, types and error codes live there. On the site: the [function catalog](https://ggl.farm/en/api.html) and the customer [FAQ](https://ggl.farm/en/faq.html).

---

## The product in short

| Thing | What it is |
|---|---|
| **Account** | a Google account with a browser profile, proxy, tags, country (`geo`) and farm settings. `status` 1 Waiting, 2 Creating, 3 Farming; `active` false = out of service ("dead"); `error` a code, 0 none; `task` what it does now (0 free, 1 registration, 2 farm, 3 statistics, 4 2FA, 5 login) |
| **Proxy** | your own (HTTP, HTTPS, SOCKS5, SOCKS5H) with `max` slots, how many accounts share it. Status 0 Waiting, 1 Working, 2 Error; only Working proxies with a free slot go to accounts |
| **Task** | a unit of robot work, status 0 In queue, 1 Waiting, 2 Working, 3 Problem, 4 Error, 5 Success. The API queues `login` and `make2fa`; the service makes the rest. One task per account at a time, in order; a Problem or Error task **pauses that account's queue** until restarted or deleted |
| **Thread** | what the plan limits: one running browser window, one account in work, not the number of stored accounts. A team shares one limit |
| **Team level** | the team administrator also gets `gang/*`, twins of `list`, `get`, `fix`, `del` over every member's accounts (`fix` and `del` act without asking them). The level is chosen by the path, there is no `scope` parameter |

---

## Authentication and base URL

```
https://ggl.farm/api/{app}/{func}.json?id={apitoken}
```

`{apitoken}` is `<user id>-<API key>`, for example `123-0123456789abcdef`; the key is on your profile page at https://ggl.farm, keep it secret.

- **Keep `?id={apitoken}` in the URL on every call.** A form POST may send `authid` instead; a JSON body carries parameters only, `authid` is not read there.
- GET and POST are interchangeable; query and body parameters merge, the body wins. Use POST for passwords, 2FA keys and proxy credentials.
- Lists are `ids=1,2,3` or an array; flags `0` / `1`; countries lowercase ISO (`us`); time UNIX seconds; the suffix is `.json`.
- Success is HTTP 200 `{"status":"ok","data":...}`. An error is `{"status":"error","error":"<code>"}`, repeated in the headers `X-API-Status` and `X-API-Error`, with HTTP 401 `key`, 402 `unpaid` (no subscription, any function), 403 `ban`, 404 `no-id` / `bad-id` / `bad-status` / `not-found`, 418 `bot`, 500 `func` / `app` / `db`, otherwise 400. An unknown code is a failure, never retry blindly.
- Paging: `page` (from 1), `show` (default 100, no upper limit; `browser/log` default 50, at most 200). `accs/list`, `gang/list`, `browser/log` answer flat `page` / `show` / `total`; `task/list`, `proxy/list` a `nav` block. `show=ids` (accounts, proxies) returns every matching ID, to feed `fix`, `del`, `check`.

---

## Function index

### accs - 15 functions, `gang` twins marked - [references/accs.md](references/accs.md)

| Function | Purpose |
|---|---|
| `accs/list` (+`gang`) | filters `status`, `error`, `geo`, `tag`, `folder`, `proxy`, `farm`, `busy`, `dead`, `s` (text); `show=ids` for IDs only; `gang/list` adds `user` |
| `accs/get` (+`gang`) | `acc`: one account with person data, proxy, farm settings, cookies |
| `accs/make` | register new accounts: `amount`, `geo`, `folder` (needs free proxy slots, SMS service, browser) |
| `accs/push` | an account that already has a browser profile: `login`, `pass`, `geo`, `folder` + `profile` (or `account=folder/profile`); returns its `id` |
| `accs/add` | connect an account by `email`, `pass`, `geo`, `folder`; the robot creates the profile and logs in |
| `accs/edit` | `acc` plus the fields that change: card data, exact `proxy`, farm settings; own accounts only |
| `accs/fix` (+`gang`) | `ids` plus independent flags (below) |
| `accs/del` (+`gang`) | `ids`; `proxy=1`, `profile=1`, `phone=1` also delete the proxy and the profile, cancel the phone rental |
| `accs/status` | is the browser integration ready (`api`, `ip`, `folder`, `active`) |
| `accs/folder` | browser folders, the ID that `make`, `push`, `add` need |
| `accs/active` / `accs/all` | running profiles / all profiles (find the profile for `push`) |
| `accs/start` / `accs/ping` / `accs/stop` | `acc`: open the profile for your own automation over CDP, prolong the 30-minute session, close it |

`accs/fix` flags: proxies `fixproxy` `newproxy` `anyproxy` `unproxy`; errors `fixerror`; tasks `restart` `noproblem` `noerrors` `requeue` `unqueue`; farming `farmon` `farmoff` `farmnow`; statistics `statson` `statsoff` `update`; account `kill` `ressurect` (sic) `sync` `cookie` `forcestop` `existing`. The thread load is not an `accs` function: use `browser/quota`.

### task - 4 functions - [references/task.md](references/task.md)

| Function | Purpose |
|---|---|
| `task/list` | tasks of one `acc` or of all, filter by `type` and `status` |
| `task/add` | `acc`, `type=login` or `make2fa`, optional `delay` in hours (`make2fa` only) |
| `task/del` | `task`: cancel (not registration, not Success) |
| `task/restart` | `task`: put a Problem or Error task back in the queue |

### proxy - 7 functions - [references/proxy.md](references/proxy.md)

| Function | Purpose |
|---|---|
| `proxy/list` | search by `status`, `geo`, `type`, `ip6`, `host`, `port`, `user`, `pass`, `s`; `show=ids` for IDs only |
| `proxy/get` | `proxy`: one proxy in full, with the connect string |
| `proxy/add` | one proxy from fields (`host`, `port`, `type`, `user`, `pass`, `max`, `url`) |
| `proxy/load` | bulk add, `proxy` = one proxy per line in many formats, with slots and a reset link |
| `proxy/edit` | `proxy` plus **every** field again |
| `proxy/check` | `ids`: health check right now, at most 25 per call |
| `proxy/del` | `ids`: delete; their accounts are left without a proxy |

### browser - 7 functions, settings and thread load - [references/browser.md](references/browser.md)

| Function | Purpose |
|---|---|
| `browser/info` | read every setting with the keys and balances |
| `browser/save` | the antidetect browser: `type`, `api`, `ip`, `head`, `wrap`, limits `lim` `lim1` `lim2` |
| `browser/services` | the SMS services you can choose and what each can do |
| `browser/sms` | the SMS service: `smstype`, `smsapi`, `smslogin`, `smsmax`, `smsterm`, `smsrenew` |
| `browser/captcha` | the CapMonster key (`capmonster`) |
| `browser/log` | journal of the requests GGL sent to your browser, newest first (`acc`, `error`, `func`) |
| `browser/quota` | threads used against the plan limit, length of the work queues ([details](references/browser.md#browserquota---thread-load-of-your-plan)) |

---

## Typical workflow

1. **Connect the antidetect browser.** `browser/save` (`type` 1 Vision, 2 OctoBrowser, 3 AdsPower, `api`, `ip`, `head=1`, thread limits), then `browser/services` -> `browser/sms` -> `browser/captcha`; verify with `browser/info`, `accs/status` (all flags `true`).
2. **Add proxies.** `proxy/load` a list (`socks5://user:pass@host:port||25` gives 25 slots), `proxy/check` in batches of at most 25 with a long timeout, `proxy/list&status=1` shows the usable ones.
3. **Bring accounts in.** `accs/folder` for the folder ID, then `accs/add` (login and password, GGL makes the profile), `accs/push` (profile exists, found with `accs/all`) or `accs/make` (register new); `add` and `push` take one account per call. Give a batch a `tags` value: `make` and `add` return no IDs, find the accounts with `accs/list&tag=...`.
4. **Queue tasks.** `task/add` with `type=login` or `make2fa`; watch `task/list&status=3` and `status=4`.
5. **Watch quotas and logs.** `browser/quota` for threads and queues (before adding many accounts), `accs/list` by `error`, `busy`, `farm`; `browser/log&acc=...&error=1` when a profile does not launch.
6. **Repair.** Fix the cause (replace the proxy, top up the SMS balance), then `accs/fix` (`fixproxy`, `restart`, `farmon`...) or `task/restart`; `kill` / `ressurect` take accounts out of service and back.

---

## Gotchas

- **Credentials come back in clear:** `pass`, `2fa`, `recover`, proxy logins, the keys of `browser/info` and `browser/log`, the `connect` link of `accs/start`. Never log or paste them.
- **`browser/save` and `browser/sms` rewrite the whole form.** An omitted parameter is cleared (no `head` turns headless off, no `smsrenew` turns renewal off): read `browser/info` and send everything back. `proxy/edit` is the same. The CapMonster key can be replaced, not cleared.
- **Create calls answer `{"status":"ok"}` without IDs** (`make`, `add`; `push` returns `id`); `make` and `add`, not `push`, need free proxy slots (`no-proxy`) and take **one** tag, several are glued into one; `accs/edit` `tags` replaces the whole list, and a new `proxy` there keeps the old slot taken until a `proxy/load` or `proxy/add` call recounts.
- **`ok` is not "done".** `make` only accepts the request; `proxy/load` is `ok` even when every line failed, read `added`, `duplicates`, `rejected`; `accs/fix` and `proxy/del` give no per-ID result and skip foreign IDs silently.
- **A stuck task blocks its account.** `task/add` answers `duplicate` while an unfinished task of that type exists. Restart after the cause is fixed, a blind restart fails again. `delay` is hours, 0 to 8760, for `make2fa` only; `login` starts at once.
- **Arrays are rejected.** A value that is an array (`name[]=...`, a JSON array or object) answers `bad-param`, and `param` names it. Only the `ids` of `fix`, `del`, `proxy/check` and `proxy/del` may be an array.
- **Validation errors.** `bad-geo` (with `supported-geo`), `bad-email`, `bad-amount` (`make`, below 1), `no-profile` (`push` without a profile; `start` on an account without one), `bad-farmnext` (with `supported-farmnext`), `bad-proxy`, `no-fields` (`edit`).
- **Account `error`.** 101 registration failed, 102 no SMS number (top up or change `smsgeo`, then `restart`), 200 dead, 201 profile gone, 300 advertising cabinet, 901-905 temporary proxy or browser trouble: [table](references/accs.md#vocabularies).
- **Long calls.** `accs/start` takes up to 90 s (timeout 120 s), `proxy/check` up to 10 s per proxy. A session keeps the robot off the account and its proxy-mates: `ping` in time, `stop` in a `finally`.
- **Data shapes.** `task/list` and `proxy/list` return `data` as an object keyed by ID, `[]` when nothing matches. `dataset` is plain JSON (arrays for proxies and task statuses, an object keyed 1-5 for task types): match on the number, never the name. `data: false` on `accs/folder`, `active`, `all` means the integration is not set up. Browser lookups and `browser/quota` are cached a minute.
- **Deleting.** `accs/del` is irreversible; `proxy=1` also removes a proxy others share; big `profile=1` batches can leave profiles in the browser. `proxy/del` leaves accounts without a proxy, they do not run.
- **Farm period.** `farmnext` is in seconds: 85000 once a day (default), 42000, 27000, 170000, 259000, 604000.

---

## Reference files

Read the file, not this page, for parameters and errors.

| File | Read it when |
|---|---|
| [references/overview.md](references/overview.md) | auth, envelope, HTTP statuses, paging, `dataset`, limits, the order of first calls: read it first |
| [references/accs.md](references/accs.md) | accounts: vocabularies, the account object, make / push / add / edit, fix flags, sessions, `gang/*` ([site](https://ggl.farm/en/api-accs.html), [sessions](https://ggl.farm/en/api-profile.html)) |
| [references/proxy.md](references/proxy.md) | proxy states and slots, `proxy/load` line formats, reset link, edit, check, delete ([site](https://ggl.farm/en/api-proxy.html)) |
| [references/task.md](references/task.md) | task types and statuses, queue behaviour, queueing, restart, cancel ([site](https://ggl.farm/en/api-task.html)) |
| [references/browser.md](references/browser.md) | the browser, the SMS catalogue, CapMonster, the journal, `browser/quota` ([site](https://ggl.farm/en/api-browser.html)) |
| [references/faq.md](references/faq.md) | how the service behaves, not which call: adding versus registering, threads, the working cycle, 2FA, proxies, teams, choosing a server, errors, repair options |

When the FAQ (written for the web interface) and a reference differ, the reference wins.

## OpenAPI specs

Four OpenAPI 3.1 files in `spec/`, one per application ([conventions](spec/README.md)): [openapi-accs.json](spec/openapi-accs.json) (with `gang/*`), [openapi-task.json](spec/openapi-task.json), [openapi-proxy.json](spec/openapi-proxy.json), [openapi-browser.json](spec/openapi-browser.json) (with `quota`).
