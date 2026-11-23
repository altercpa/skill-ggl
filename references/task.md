# Tasks API Reference

## Overview

The `task` application shows what the robot is doing on your accounts and lets you steer the queue: list tasks, queue a login or 2FA, cancel a task, restart a task that ended with a Problem or an Error. Each account has its own queue and runs **one task at a time**, strictly in the order the tasks were added.

Base URL: `https://ggl.farm/api/task/{func}.json?id={apitoken}`

Authentication, the `status`/`error` envelope, the `nav` pagination block and the HTTP status of errors are described in [overview.md](overview.md). GET and POST are interchangeable there; the headings below show the usual method: GET to read (`list`), POST to write (`add`, `del`, `restart`).

**User level only.** Every call sees the tasks of your own accounts. There is no team (`gang`) twin and no `scope` parameter; a team member sees only the tasks of their own accounts, not those of teammates.

Examples below assume `TOKEN` holds your API token (`<user id>-<API key>`).

---

## Task Vocabulary

**`type` - what the task does** (the key of `dataset.types`):

| Value | Name | Meaning | Name accepted by `list` / `add` |
|---|---|---|---|
| 1 | Registration | Registers the account. Created by the service, **cannot be queued or deleted through the API** | `create` (`list` only) |
| 2 | Farm | Farming runs on its own schedule and never appears as a queue row | `farm` (`list` only) |
| 3 | Statistics | Statistics runs on its own schedule and never appears as a queue row | `stats` (`list` only) |
| 4 | 2FA | Turns on two-factor authentication | `make2fa` |
| 5 | Login | Logs into the account | `login` |

In practice `list` returns tasks of type 1, 4 and 5. `add` queues only `make2fa` and `login`. The type names are returned in the language of your account interface.

**`status` - where the task is** (the index of `dataset.statuses`):

| Value | Name | Meaning |
|---|---|---|
| 0 | In queue | Held behind an earlier unfinished task of the same account |
| 1 | Waiting | Ready; starts when its `delay` time has passed and the account is free |
| 2 | Working | The robot is executing it |
| 3 | Problem | Halted. **The account's queue is paused** until the task is restarted or deleted |
| 4 | Error | Halted, same as Problem |
| 5 | Success | Finished. Stays in the account history; cannot be restarted or deleted |

**`error` - task error code** (`0` = no error). It shares most codes with the account `error` field, see Vocabularies in [accs.md](accs.md#vocabularies). The codes you will meet on tasks are mostly `901` (no proxy), `902` (proxy error), `903`-`905` (browser profile) and `906` (timeout; a task error only, it never appears on an account). A task can carry such an error and still be `Waiting`: the service postpones it and tries again later.

A task queued on a dead account ends in Problem when its turn comes.

---

## Task Object

```json
{
  "id": 1042,
  "acc": 666,
  "type": 4,
  "status": 1,
  "error": 0,
  "retry": 0,
  "create": 1754850000,
  "delay": 1754864400,
  "start": 0,
  "finish": 0
}
```

| Field | Type | Meaning |
|---|---|---|
| `id` | int | Task ID |
| `acc` | int | ID of the account the task belongs to |
| `type` | int | Task type, see the vocabulary above |
| `status` | int | Task status, see the vocabulary above |
| `error` | int | Error code of the last run, `0` = none |
| `retry` | int | How many times the task was re-launched (restarts included) |
| `create` | int | When the task was queued, UNIX timestamp |
| `delay` | int | Earliest start time, UNIX timestamp. `0` = can run at once. The service also moves it forward when it postpones the task (no proxy, thread limit) |
| `start` | int | Last launch, UNIX timestamp, `0` = never launched |
| `finish` | int | Last completion, UNIX timestamp, `0` = never completed |

---

## task/list - List tasks

```
GET https://ggl.farm/api/task/list.json?id={apitoken}
```

With `acc` you get the tasks of one account; without it, the tasks of all your accounts. Tasks come in ascending ID order (the order they were added).

**Parameters** (all optional, combined with AND):

| Name | Type | Description |
|---|---|---|
| `acc` | int | Account ID in GGL. Only the tasks of this account |
| `type` | string / int | Task type by name (`create`, `farm`, `stats`, `make2fa`, `login`, case-insensitive) or by number (`1`-`5`). Any other value is read by its leading digits: `2fa` counts as `2` (Farm), not as `make2fa`; a word with no leading digits counts as `0` and matches nothing |
| `status` | int | Task status `0`-`5`. `status=0` is a valid filter (In queue); a non-numeric value counts as `0` |
| `page` | int | Page number, from 1. Default 1 |
| `show` | int | Tasks per page. Default 100 (also for `0`), no upper limit |

**Response:**

```json
{
  "status": "ok",
  "nav": { "total": 3, "page": 1, "pages": 1, "show": 100 },
  "dataset": {
    "types":    { "1": "Registration", "2": "Farm", "3": "Statistics", "4": "2FA", "5": "Login" },
    "statuses": [ "In queue", "Waiting", "Working", "Problem", "Error", "Success" ]
  },
  "data": {
    "1041": { "id": 1041, "acc": 666, "type": 5, "status": 5, "error": 0, "retry": 0,
              "create": 1754846400, "delay": 0, "start": 1754846500, "finish": 1754849400 },
    "1042": { "id": 1042, "acc": 666, "type": 4, "status": 1, "error": 0, "retry": 0,
              "create": 1754850000, "delay": 1754864400, "start": 0, "finish": 0 },
    "1055": { "id": 1055, "acc": 671, "type": 4, "status": 4, "error": 902, "retry": 2,
              "create": 1754760000, "delay": 1754790000, "start": 1754789000, "finish": 1754789600 }
  }
}
```

- `nav`: `total` matching tasks, current `page`, `pages` in total, `show` per page.
- `dataset`: the dictionaries for `type` and `status`. `types` is an object keyed by the type number; `statuses` is a plain array, the status number is the index. Match on the number, not the name.
- `data`: an **object keyed by task ID**. When nothing matches, `data` is an empty array `[]`, not `{}`.

**Errors:** `bad-id` (the `acc` account is not found among yours), `bad-param` (a request value is an array; the reply also carries `param`, the name of that parameter), `unpaid`

```bash
# Everything that is stuck: Problem, then Error
curl -s "https://ggl.farm/api/task/list.json?id=$TOKEN&status=3"
curl -s "https://ggl.farm/api/task/list.json?id=$TOKEN&status=4"

# Pending 2FA tasks of one account
curl -s "https://ggl.farm/api/task/list.json?id=$TOKEN&acc=666&type=make2fa&status=1"
```

---

## task/add - Queue a task

```
POST https://ggl.farm/api/task/add.json?id={apitoken}
```

Queues a task on an existing account of yours, the same as the task buttons in the account card. The robot runs it in the order of the account queue. If an **unfinished** task of the same type is already on the account, nothing is added and `duplicate` is returned. A task stuck in Problem or Error counts as unfinished: restart or delete it first.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `acc` | int | yes | Account ID in GGL |
| `type` | string | yes | `make2fa` (turn on two-factor authentication) or `login` (log into the account). Names only: numbers and any other value, `2fa` included, give `bad-type`; an empty value or `0` gives `no-type` |
| `delay` | float | no | Hours to wait before the task may start, fractions allowed (`2.5`), from `0` to `8760` (a year): a negative or non-numeric value counts as 0, a bigger one is cut to 8760. Applied to `make2fa` only; a `login` task is always queued for immediate execution |

**Response:** `{ "status": "ok", "id": 1042 }` - the ID of the new task.

The task starts as `Waiting` when the account has no unfinished task ahead of it, otherwise as `In queue`.

**Errors:** `no-id` (no `acc`), `bad-id` (account not found among yours), `no-type` (`type` missing, empty or `0`), `bad-type` (anything but `make2fa` / `login`), `bad-param` (a request value is an array; the reply also carries `param`), `duplicate`, `unpaid`

```bash
# Enable 2FA in two and a half hours
curl -s -X POST "https://ggl.farm/api/task/add.json?id=$TOKEN" \
  -d "acc=666" -d "type=make2fa" -d "delay=2.5"

# Log into the account right now (JSON body)
curl -s -X POST "https://ggl.farm/api/task/add.json?id=$TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"acc":666,"type":"login"}'
```

---

## task/del - Cancel a task

```
POST https://ggl.farm/api/task/del.json?id={apitoken}&task={id}
```

Removes the task from the account queue. The queue closes the gap: the task that followed the deleted one moves up, and it becomes Waiting when the deleted task was at the head of the queue. A task in Working status can be deleted, but the run already in progress is not stopped at that moment.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `task` | int | yes | Task ID in GGL |

**Response:** `{ "status": "ok" }`

Not deletable: Registration tasks (type 1) and tasks in Success status, which stay in the account history.

**Errors:** `no-id`, `bad-id` (task not found among yours), `system` (a Registration task), `bad-status` (a Success task), `unpaid`

---

## task/restart - Restart a task

```
POST https://ggl.farm/api/task/restart.json?id={apitoken}&task={id}
```

Puts a halted task back in the queue: the task returns to `Waiting` and its `retry` counter goes up by one. Only tasks in Problem or Error status can be restarted. Registration tasks that failed can be restarted too.

Fix the cause first, for example replace a proxy that does not work (see [proxy.md](proxy.md)). There is no automatic limit on restarts, so a task restarted without a fix simply fails again.

**Parameters:**

| Name | Type | Required | Description |
|---|---|---|---|
| `task` | int | yes | Task ID in GGL |

**Response:** `{ "status": "ok" }`

**Errors:** `no-id`, `bad-id` (task not found among yours), `bad-status` (the task is not in Problem or Error status), `unpaid`

```bash
# Recovery loop: find the failed tasks, fix the cause, restart each one
curl -s "https://ggl.farm/api/task/list.json?id=$TOKEN&status=4"
curl -s -X POST "https://ggl.farm/api/task/restart.json?id=$TOKEN&task=1055"
```

---

## Errors Reference

All errors come as `{"status":"error","error":"<code>"}`. The HTTP status is the one described in [overview.md](overview.md).

| Error | HTTP | Functions | Meaning |
|---|---|---|---|
| `no-id` | 404 | add, del, restart | Account (`acc`) or task (`task`) ID not specified |
| `bad-id` | 404 | list, add, del, restart | Account or task not found among yours |
| `bad-status` | 404 | del, restart | The task status does not allow it: only non-Success tasks can be deleted, only Problem and Error tasks can be restarted |
| `no-type` | 400 | add | `type` not specified, empty or `0` |
| `bad-type` | 400 | add | Task type is not `make2fa` or `login` (a number is not accepted) |
| `bad-param` | 400 | list, add | A request value is an array; the reply also carries `param` with the name of that parameter |
| `duplicate` | 400 | add | An unfinished task of this type is already on the account |
| `system` | 400 | del | Registration task, cannot be deleted |
| `unpaid` | 402 | all | Subscription not paid |
