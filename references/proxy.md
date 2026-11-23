# Proxy API Reference

## Overview

The `proxy` application manages your proxy pool without the interface: search and read proxies, add one proxy or a whole list, edit settings, check health and delete. Accounts take their proxies from this pool, so a healthy pool with free slots is a precondition for registering and farming accounts.

Base URL: `https://ggl.farm/api/proxy/{func}.json?id={apitoken}`

Authentication, the `status`/`error` envelope, the `nav` pagination block, `show=ids` and the HTTP status of errors are described in [overview.md](overview.md). GET and POST are interchangeable there; the headings below show the usual method: GET to read (`list`, `get`), POST to write (`add`, `load`, `edit`, `check`, `del`).

**User level only.** Every call sees the proxies of your own pool. There is no team (`gang`) twin and no `scope` parameter.

Examples below assume `TOKEN` holds your API token (`<user id>-<API key>`).

---

## Proxy Vocabulary

**`type` - protocol:**

| Value | Name |
|---|---|
| 0 | HTTP |
| 1 | HTTPS |
| 2 | SOCKS5 |
| 3 | SOCKS5H |

SOCKS5 resolves host names on your side, SOCKS5H on the proxy side. When a proxy is **added**, a SOCKS5 proxy with a domain instead of an IP is saved as SOCKS5H, and a SOCKS5H proxy with an IP address is saved as SOCKS5. `edit` stores the type as sent, without this conversion. The `type` parameter is always the number `0`-`3`: a name such as `socks5`, or any other unknown value, becomes `0` (HTTP).

**`status` - state of the proxy:**

| Value | Name | Meaning |
|---|---|---|
| 0 | Waiting | Added, not checked yet. The service checks new proxies automatically; `proxy/check` forces it |
| 1 | Working | The last check succeeded: the proxy answered and its exit IP and country are known |
| 2 | Error | The last check failed. `errors` counts the failed checks in a row |

Only **Working** proxies with a free slot are handed to accounts. A successful check brings an Error proxy back to Working by itself. A proxy that keeps failing for a long time is removed by the service, and the accounts that used it are left without a proxy.

**Slots.** `max` is how many accounts may share the proxy, `count` how many do now, `avail` how many slots are free (`max - count`).

**Exit data.** `ip` is the IP address seen through the proxy (`false` until the first successful check), `ip6` is `true` when it is an IPv6 address, `geo` is the two-letter lowercase country code of that IP (`zz` while unknown).

---

## Proxy Object

Returned by `list` (short form) and `get` (full form).

```json
{
  "id": 501,
  "status": 1,
  "errors": 0,
  "type": 2,
  "host": "203.0.113.7",
  "port": 1080,
  "user": "usr",
  "pass": "pwd",
  "ip": "198.51.100.9",
  "ip6": false,
  "geo": "de",
  "avail": 24,
  "count": 1,
  "max": 25,
  "url": false
}
```

| Field | Type | Meaning |
|---|---|---|
| `id` | int | Proxy ID in GGL |
| `status` | int | `0` Waiting, `1` Working, `2` Error |
| `errors` | int | Failed checks in a row, `0` after a successful check |
| `type` | int | `0` HTTP, `1` HTTPS, `2` SOCKS5, `3` SOCKS5H |
| `host` | string | IP address or domain |
| `port` | int | Port |
| `user`, `pass` | string | Login and password, empty when the proxy has none. Returned in clear |
| `ip` | string / false | Exit IP, `false` if not checked yet |
| `ip6` | bool | The exit IP is IPv6 |
| `geo` | string | Country code of the exit IP |
| `avail`, `count`, `max` | int | Free, taken and total slots |
| `url` | string / false | IP reset link, `false` if not set |

`get` adds three fields:

| Field | Type | Meaning |
|---|---|---|
| `link` | string | Connect string `proto://login:pass@host:port` (`proto` is `http`, `https`, `socks5` or `socks5h`) |
| `check` | int | When the next automatic check is due, UNIX timestamp |
| `last` | int | When the proxy was last checked, UNIX timestamp, `0` if never |

Both `list` and `get` also return `dataset`: `{ "types": ["HTTP","HTTPS","SOCKS5","SOCKS5H"], "statuses": ["Waiting","Working","Error"] }`. These are plain arrays: the value of `type` or `status` is the index. The names follow your account interface language.

Credentials that contain a quote character (`'` or `"`) are stored escaped, and a `<...>` fragment is cut out, so such a login or password will not authenticate. Avoid these characters.

---

## Reset Link

A rotating proxy can carry a link that changes its exit IP (`url` on `add`, `edit` and in a `load` line). The service calls the link when it starts work on an account that uses the proxy, so the account gets a fresh IP. The link is checked twice:

- **When saved.** Only `http://` and `https://` links are kept. A link to an IPv6 address, or to an IPv4 address that is not public (local, private, link-local including the cloud metadata address, CGNAT) is not saved: the proxy is still added or changed, with no reset link and no error.
- **When called.** A host name is saved as it is, but the service does not call a link whose name resolves to such an address, or to no IPv4 address at all. Every redirect is checked the same way, three at most. The reply of the link is ignored.

---

## Proxy Line Formats (`proxy/load`)

`load` takes one proxy per line and detects the format of each line by itself. Only the host and the port are required; login and password may be left out.

| Format | Example | Notes |
|---|---|---|
| `protocol://user:pass@host:port` | `socks5://user:pass@203.0.113.7:1080` | The recommended form |
| `protocol://host:port:user:pass` | `socks5://203.0.113.7:1080:user:pass` | Host is an IPv4 address or a domain name |
| `user:pass@host:port` | `user:pass@203.0.113.7:3128` | No protocol: HTTP |
| `host:port:user:pass` | `203.0.113.7:3128:user:pass` | No protocol: HTTP |
| `host:port` | `203.0.113.7:3128` | No protocol, no login: HTTP |
| `protocol://host:port` | `socks5://203.0.113.7:1080` | No login |

- **Protocols:** `http`, `https`, `socks5`, `socks5h`; `socks` is an alias for `socks5`. A line without a protocol is HTTP, and an unrecognised protocol is saved as HTTP too, so write the protocol for SOCKS proxies.
- **Slots and reset link** go after the proxy, separated by two pipes: `proxy||max` or `proxy||max||resetlink`, for example `socks5://user:pass@203.0.113.7:1080||25||https://rotate.example.net/abc`. Without them `max` is 1 and there is no reset link. The practical maximum for `max` is 32767. The link follows the [Reset Link](#reset-link) rules.
- Blank lines are skipped. Lines are separated by `\n` (`\r\n` is fine).
- A line that gives no host or no port counts as `rejected`.
- A proxy that is already registered is a `duplicate`: it is skipped and **not updated**, so its slots and reset link stay as they were. A line that cannot be saved at all (for example a port above 65535) is counted as a duplicate too.

---

## proxy/list - Search the pool

```
GET https://ggl.farm/api/proxy/list.json?id={apitoken}
```

Returns the proxies that match all given filters, ascending by ID. Each entry is the short Proxy Object.

**Parameters** (all optional, combined with AND):

| Name | Type | Description |
|---|---|---|
| `status` | int | `0` Waiting, `1` Working, `2` Error |
| `ip6` | int | `0` IPv4 exit IP, `1` IPv6 exit IP |
| `geo` | string | Country code of the exit IP, e.g. `de` |
| `type` | int | `0` HTTP, `1` HTTPS, `2` SOCKS5, `3` SOCKS5H. Numbers only: a name counts as `0` |
| `host` | string | Host, matched after the value is lower-cased and stripped of everything but letters, digits, `.`, `-` and `_` (a colon included): it is not matched as typed |
| `port` | int | Exact port |
| `user` | string | Exact login |
| `pass` | string | Exact password |
| `s` | string | Free-text search over host, port, login, password and reset link |
| `page` | int | Page number, from 1. Default 1 |
| `show` | int / `ids` | Proxies per page, default 100, no upper limit. The value `ids` switches to ID-only mode |

`status=0`, `ip6=0` and `type=0` are valid filters, and a present but empty `status`, `ip6` or `type` counts as `0`. An empty `geo`, `host`, `port`, `user`, `pass` or `s` is ignored.

**Response:**

```json
{
  "status": "ok",
  "nav": { "total": 237, "page": 1, "pages": 3, "show": 100 },
  "dataset": {
    "types": ["HTTP", "HTTPS", "SOCKS5", "SOCKS5H"],
    "statuses": ["Waiting", "Working", "Error"]
  },
  "data": {
    "501": { "id": 501, "status": 1, "errors": 0, "type": 2, "host": "203.0.113.7", "port": 1080,
             "user": "usr", "pass": "pwd", "ip": "198.51.100.9", "ip6": false, "geo": "de",
             "avail": 24, "count": 1, "max": 25, "url": false }
  }
}
```

`data` is an **object keyed by proxy ID**; when nothing matches it is an empty array `[]`.

**ID-only mode** (`show=ids`): no paging, no `nav`, no `dataset`; `data` is the sorted array of **all** matching IDs, ready to feed into `check` or `del`:

```json
{ "status": "ok", "data": [ 501, 502, 515, 516 ] }
```

**Errors:** `unpaid`, `bad-param` (a request value is an array; the reply also carries `param`, the name of that parameter)

```bash
# Working SOCKS5 proxies in Germany, 50 per page
curl -s "https://ggl.farm/api/proxy/list.json?id=$TOKEN&status=1&type=2&geo=de&show=50&page=1"

# IDs of every failing proxy
curl -s "https://ggl.farm/api/proxy/list.json?id=$TOKEN&status=2&show=ids"
```

---

## proxy/get - One proxy

```
GET https://ggl.farm/api/proxy/get.json?id={apitoken}&proxy={id}
```

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `proxy` | int | yes | Proxy ID in GGL |

**Response:** `dataset` plus `data`, the full Proxy Object (list fields plus `link`, `check`, `last`):

```json
{
  "status": "ok",
  "dataset": { "types": ["HTTP", "HTTPS", "SOCKS5", "SOCKS5H"], "statuses": ["Waiting", "Working", "Error"] },
  "data": {
    "id": 501, "status": 1, "errors": 0, "type": 2,
    "host": "203.0.113.7", "port": 1080, "user": "usr", "pass": "pwd",
    "link": "socks5://usr:pwd@203.0.113.7:1080", "url": false,
    "ip": "198.51.100.9", "ip6": false, "geo": "de",
    "avail": 24, "count": 1, "max": 25,
    "check": 1763010000, "last": 1762976800
  }
}
```

**Errors:** `no-id`, `bad-id` (not found among your proxies), `unpaid`

---

## proxy/add - Add one proxy

```
POST https://ggl.farm/api/proxy/add.json?id={apitoken}
```

Adds a single proxy from fields, like the add-proxy form. For many proxies use `proxy/load`. The new proxy starts in `Waiting` status. Adding never updates an existing proxy. A successful call recounts `count` and `avail` of every proxy in your pool.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `host` | string | yes | IP address or domain |
| `port` | int | yes | Port, `1`-`65535` |
| `type` | int | no | `0` HTTP (default), `1` HTTPS, `2` SOCKS5, `3` SOCKS5H. An unknown value, or a name, is replaced with HTTP |
| `user` | string | no | Login |
| `pass` | string | no | Password |
| `max` | int | no | Slots: how many accounts may share the proxy. At least 1 (default 1) |
| `url` | string | no | IP reset link, see [Reset Link](#reset-link). A link that does not pass is saved as empty and the proxy is added without it |

**Response:** `{ "status": "ok", "id": 502 }` - the ID of the new proxy.

**Errors:** `bad-param` (a request value is an array; the reply also carries `param`), `no-host`, `no-port`, `duplicate` (the same proxy is already in the system, or the save itself failed), `add` (the proxy could not be saved), `unpaid`

```bash
curl -s -X POST "https://ggl.farm/api/proxy/add.json?id=$TOKEN" \
  -d "host=203.0.113.7" -d "port=1080" -d "type=2" \
  -d "user=usr" -d "pass=pwd" -d "max=25"
```

---

## proxy/load - Bulk add from a list

```
POST https://ggl.farm/api/proxy/load.json?id={apitoken}
```

Uploads a list of proxies, one per line, the same as the upload-list form. Formats are in [Proxy Line Formats](#proxy-line-formats-proxyload) above. Re-uploading is safe: duplicates are skipped and nothing existing is changed. Every call recounts `count` and `avail` of every proxy in your pool, even when no line was added. The call succeeds (`status: ok`) even if every line was a duplicate or was rejected; read the counters.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `proxy` | string | yes | The list, one proxy per line |

**Response:**

```json
{ "status": "ok", "added": 3, "duplicates": 1, "rejected": 0, "data": [ 502, 503, 504 ] }
```

| Field | Meaning |
|---|---|
| `added` | Proxies from the list that were added |
| `duplicates` | Proxies skipped because they already exist (including repeats inside the list) or could not be saved |
| `rejected` | Lines that could not be parsed (no host or no port) |
| `data` | IDs of the added proxies, in list order |

**Errors:** `no-list` (`proxy` missing or empty), `bad-param` (a request value is an array; the reply also carries `param`), `unpaid`

```bash
# Mixed formats in one request; the first line carries 25 slots and a reset link
curl -s -X POST "https://ggl.farm/api/proxy/load.json?id=$TOKEN" \
  --data-urlencode "proxy=socks5://user:pass@203.0.113.7:1080||25||https://rotate.example.net/abc
203.0.113.8:1080:login:secret
http://login:secret@203.0.113.9:8080
https://203.0.113.10:8443"
```

---

## proxy/edit - Change a proxy

```
POST https://ggl.farm/api/proxy/edit.json?id={apitoken}&proxy={id}
```

Changes the settings of one proxy, like the edit form. **Send every field, including the unchanged ones:** a missing field is cleared (login, password, reset link), `type` falls back to `0` (HTTP) and `max` to `1`.

When the host, port, login, password or type changes, the new settings are pushed to the antidetect browser profiles that use the proxy automatically. Editing does not re-check the proxy and does not change its status: run `proxy/check` afterwards.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `proxy` | int | yes | Proxy ID in GGL |
| `host` | string | yes | IP address or domain |
| `port` | int | yes | Port, `1`-`65535` |
| `type` | int | no | `0` HTTP, `1` HTTPS, `2` SOCKS5, `3` SOCKS5H. Stored as sent; an unknown value, or a name, is replaced with HTTP |
| `user` | string | no | Login |
| `pass` | string | no | Password |
| `max` | int | no | Slots, at least 1. Free slots are recalculated from it |
| `url` | string | no | IP reset link, see [Reset Link](#reset-link). A link that does not pass is saved as empty |

**Response:** `{ "status": "ok" }`

**Errors:** `no-id`, `bad-id` (not found among your proxies), `bad-param` (a request value is an array; the reply also carries `param`), `no-host`, `no-port`, `edit` (could not be saved, for example the edited proxy would become identical to another one), `unpaid`

```bash
curl -s -X POST "https://ggl.farm/api/proxy/edit.json?id=$TOKEN&proxy=501" \
  -d "host=203.0.113.7" -d "port=1080" -d "type=2" \
  -d "user=usr" -d "pass=newpwd" -d "max=50" -d "url=https://rotate.example.net/abc"
```

---

## proxy/check - Check health

```
POST https://ggl.farm/api/proxy/check.json?id={apitoken}&ids={ids}
```

Runs a health check right now, like the Check button in the proxy list. The proxies are checked **one by one inside the request** and each can take up to ten seconds, so set a generous client timeout and send small batches. On success the proxy becomes Working and gets its exit IP and country; on failure it becomes Error and `errors` goes up by one.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `ids` | array / string | yes | Proxy IDs, an array or a comma-separated string (`501,502,503`). IDs that are not yours are ignored. At most 25 of your proxies per call. The only parameter that may be an array |

**Response:** `data` is an object `{ "<proxy id>": <new status> }`, the status being `1` (Working) or `2` (Error):

```json
{ "status": "ok", "data": { "501": 1, "502": 2 } }
```

**Errors:** `no-ids` (the list is missing or empty, or none of the IDs is yours), `many-ids` (more than 25 proxies), `bad-param` (any other parameter is an array; the reply also carries `param`), `unpaid`

```bash
curl -s -m 300 -X POST "https://ggl.farm/api/proxy/check.json?id=$TOKEN" -d "ids=501,502,503"
```

---

## proxy/del - Delete proxies

```
POST https://ggl.farm/api/proxy/del.json?id={apitoken}&ids={ids}
```

Deletes proxies from the service. **Accounts linked to a deleted proxy are left without a proxy and do not run until you link new proxies to them.** There is no confirmation and no undo.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `ids` | array / string | yes | Proxy IDs, an array or a comma-separated string. IDs that are not yours are ignored. No limit per call. The only parameter that may be an array |

**Response:** `{ "status": "ok" }` (no per-ID result)

**Errors:** `no-ids` (the list is missing or empty, or none of the IDs is yours), `bad-param` (any other parameter is an array; the reply also carries `param`), `unpaid`

```bash
# Find the proxies in Error status, then delete two of them
curl -s "https://ggl.farm/api/proxy/list.json?id=$TOKEN&status=2&show=ids"
curl -s -X POST "https://ggl.farm/api/proxy/del.json?id=$TOKEN" -d "ids=516,517"
```

---

## Errors Reference

All errors come as `{"status":"error","error":"<code>"}`. The HTTP status is the one described in [overview.md](overview.md).

| Error | HTTP | Functions | Meaning |
|---|---|---|---|
| `no-id` | 404 | get, edit | Proxy ID (`proxy`) not specified |
| `bad-id` | 404 | get, edit | Proxy not found among yours |
| `no-host` | 400 | add, edit | Host not specified |
| `no-port` | 400 | add, edit | Port not specified or zero |
| `duplicate` | 400 | add | This proxy is already in the system, or the save itself failed |
| `add` | 400 | add | The proxy could not be saved |
| `bad-param` | 400 | list, add, load, edit, check, del | A request value is an array (`ids` is exempt); the reply also carries `param` with the name of that parameter |
| `edit` | 400 | edit | The proxy could not be saved |
| `no-list` | 400 | load | The proxy list is missing or empty |
| `no-ids` | 400 | check, del | `ids` missing, not recognised, or none of them is your proxy |
| `many-ids` | 400 | check | More than 25 proxies in one request |
| `unpaid` | 402 | all | Subscription not paid |
