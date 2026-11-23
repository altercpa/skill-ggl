# GGL.farm - OpenAPI 3.1.0 Specifications

Four self-contained OpenAPI 3.1.0 files, one per API application.

## Files

| File | Covers | Endpoints |
|---|---|---|
| `openapi-accs.json` | Accounts: list, get, make, push, add, edit, fix, del, browser status/folders/profiles, manual browser sessions (start, ping, stop) + the team-level `gang/*` twins | 15 + 4 |
| `openapi-task.json` | Tasks: list, add, del, restart | 4 |
| `openapi-proxy.json` | Proxies: list, get, add, load, edit, check, del | 7 |
| `openapi-browser.json` | Integration: anti-detect browser, SMS service and CapMonster settings, service list, integration journal, thread load of your plan | 7 |

## Key conventions

- **Server**: `https://ggl.farm/api`. Paths in the files are relative to it (`/accs/list.json` is `https://ggl.farm/api/accs/list.json`).
- **Success is HTTP 200** with `status: "ok"`. Errors carry `status: "error"` and an HTTP status derived from the error code (401 `key`, 402 `unpaid`, 403 `ban`, 404 `no-id`/`bad-id`/`bad-status`, 418 `bot`, 500 `func`/`app`/`db`, otherwise 400); the `X-API-Status` / `X-API-Error` headers repeat them.
- **Auth**: `?id={user_id}-{api_key}` query parameter (apiKey scheme named `id`) on every operation. A form POST may send it as the `authid` field instead; with a JSON body keep it in the query string (`authid` is not supported in a JSON body).
- **Array values**: a parameter value that is an array answers `bad-param` with the parameter name in `param`; `ids` of `fix`, `del` (and of the proxy `check`, `del`) is exempt. Functions that read only an ID do not check.
- **Methods**: GET and POST are interchangeable; parameters from the query string and the body are merged and the body wins. Read operations are described as GET, write operations as POST with `application/x-www-form-urlencoded` and `application/json` bodies. `accs/start`, `accs/ping` and `accs/stop` change state but are described as GET, like their published URLs.
- **Team level**: there is no `scope` parameter. Team (`gang`) twins exist only for the account functions `list`, `get`, `fix`, `del` and live in `openapi-accs.json` under `/gang/...`; everything else is user-level.
- **Pagination**: `accs/list` and `browser/log` reply with flat `page` / `show` / `total` (echoing the request); `task/list` and `proxy/list` reply with a `nav` block. `show=ids` returns only the matching IDs.
- **Dictionaries**: `task` and `proxy` replies carry a `dataset` block that maps numeric codes to names: plain arrays (the code is the index) for proxies and for task statuses, an object keyed `1`-`5` for task types. The account vocabularies (state, error, task, farm period) are enums in `openapi-accs.json`.
- Each file defines the shared error responses (`BadRequest`, `Unauthorized`, `PaymentRequired`, `Forbidden`, `ServerError`, and `NotFound` where object IDs apply) and an `ErrorResponse` schema. `openapi-accs.json` lists the error codes as an `enum`; the other files list them in the `error` description.

## Viewing

Use any OpenAPI 3.1 viewer. Quick options:

```bash
# Swagger UI via Docker
docker run -p 8080:8080 -e SWAGGER_JSON=/spec/openapi-accs.json \
  -v $(pwd):/spec swaggerapi/swagger-ui

# Redoc CLI
npx @redocly/cli preview-docs openapi-accs.json
```

Or paste into [editor.swagger.io](https://editor.swagger.io) or [Redocly Playground](https://redocly.com/demo/).
