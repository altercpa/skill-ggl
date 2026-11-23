# GGL.farm FAQ

The customer FAQ of GGL.farm: how the service behaves and what its forms and errors mean. It speaks in terms of the web interface; where an option has an API counterpart, the call or flag is named, so an answer maps straight to a request. Plans, prices and thread limits are on https://ggl.farm and are not repeated here. **Where this page and the API references differ, the references win** ([overview.md](overview.md), [accs.md](accs.md), [task.md](task.md), [proxy.md](proxy.md), [browser.md](browser.md)).

The same questions, with video demos of the robot at work, are on the site: https://ggl.farm/en/faq.html. Contacts and support: https://ggl.farm/en/contact.html.

---

## Adding accounts

The main work with the service is loading ready accounts. Buy or dig up raw ones: the robot creates a profile in the antidetect browser, picks a proxy from your list and logs in. You bring the login, the password and a 2FA key or a recovery e-mail.

- **One by one or as a list.** In the interface a list is one account per line, fields separated by spaces or tabs: login, password, 2FA key, recovery e-mail, first name, last name. The names are optional; no 2FA key is written as a dash. Examples: `john.doe@example.com pass G63ADQ reserve@example.org John Doe` and `john.doe@example.com pass - reserve@example.org`.
- After loading, the account shows up in the list as **Waiting**. Within a minute the service creates the profile and queues the login task; from then on it manages without you. If the login fails, the account gets an error: see [Errors](#errors).
- Through the API one account is added per call: `accs/add` creates the profile for you, `accs/push` attaches an account that already has a profile in your antidetect browser. `accs/add` returns no ID: tag the batch and find it with `accs/list`.

## Registering accounts

The robot can also register new accounts: it invents a persona, creates a profile in the antidetect browser and enters the SMS code (`accs/make`). This is a bonus, not the main function: Google blocks registration, and without very good proxies and phone numbers there is little point trying. It is not promised to work.

## Threads and limits

The plan limits the number of **threads**, not of accounts, so you can load as many accounts as you like. A thread is one running antidetect browser window: one account in work at a time. With once-a-day farming one thread carries dozens of accounts, because a visit takes about half an hour.

The load is shown on the main page of the cabinet, and the API gives the same block as `browser/quota`: how many threads are busy and how long the queues of tasks, farming and statistics are. When you hit the limit, do not rush to change the plan: turn off farming on the accounts you do not need right now, and the queue frees up. Plans and their thread limits: https://ggl.farm.

## The account's working cycle

- **Pause** an account for a launch by turning farming off (`farmoff`): it drops out of service without taking threads. When you are done, turn farming back on (`farmon`) and the account returns to work; if its farming date was missed, it joins the priority queue.
- **Take it out of service** for longer with *Dead these accounts* (`kill`) and bring it back with *Resurrect these accounts* (`ressurect`), see [the repair form](#account-repair-form).
- Accounts live in the profiles of the antidetect browser on your server; the stored account data is visible to you and, in a team, to the team's administrators only.

## Enabling 2FA

2FA is up to you; the robot does not turn it on by itself. Queue a 2FA task (`task/add` with `type=make2fa`): it logs into the account, activates 2FA and saves the TOTP key in the account card. From then on the account opens both by the key and by the recovery e-mail. The key and the password are always visible in the card.

## Proxies

Bring your own proxies: the service takes them from your list and assigns them to accounts, from the same country where possible. One proxy per account is best. A proxy has **slots**, the number of accounts it carries at once; when the slots run out, add proxies ([proxy.md](proxy.md)).

HTTP, HTTPS, SOCKS5 and SOCKS5H proxies work. Mobile proxies work badly. If a proxy is already configured inside the antidetect browser itself, detach it from the account in the service with the repair form (`unproxy`): the browser then manages its own proxy.

## SMS service

For logins and confirmations connect your own SMS service: pick a provider in the integration settings and enter its API key (`browser/services`, `browser/sms`, see [browser.md](browser.md)). The SMS country can be set per account and defaults to the account country (`smsgeo`). Farming itself runs without an SMS service, but logging into accounts almost certainly needs one.

## Team work

Every plan has team work: gather a team and work with a shared pool of accounts. One server is one GGL account, so a plan is convenient to split across servers: the same Elite plan spreads over two or three servers, each handling its own pool of accounts, and the team shares one thread limit. The team administrator sees the whole pool through the `gang/*` functions ([accs.md](accs.md#gang---team-level)).

## Working with an account by hand

The profiles live in your antidetect browser, so an account can always be opened by hand right there; the service does not hold it hostage. To watch the robot work, connect to the server over RDP: it logs into accounts and farms in front of you. For your own automation on top of an account (Playwright, Puppeteer, Selenium) use the manual session `accs/start` ([accs.md](accs.md#manual-browser-session-accsstart-accsping-accsstop), site page https://ggl.farm/en/api-profile.html).

## Working through the API

Accounts, tasks, proxies and integration settings are driven by plain GET and POST requests and answered in JSON; your API key is on your profile page. [overview.md](overview.md) is the entry point of this reference. On the site: the function catalog https://ggl.farm/en/api.html and a page per application: [accounts](https://ggl.farm/en/api-accs.html), [browser profile launch](https://ggl.farm/en/api-profile.html), [tasks](https://ggl.farm/en/api-task.html), [proxies](https://ggl.farm/en/api-proxy.html), [integration](https://ggl.farm/en/api-browser.html).

## Choosing a server

An antidetect browser is hungry and needs a server of its own: a rented VPS or dedicated server. The best option is a plain Windows Server 2022, Windows Server 2019 will do. Linux is possible, but the RDP access is then yours to set up.

Size the server to the plan: **Start** needs 2 cores and 4 GB of memory, **Standard** 4 cores and 8 GB, **Profi** 12 cores and 64 GB, **Elite** two servers of 16 cores and 128 GB each.

Choose thoughtfully, but do not take a server with a reserve: when you need another one, take it, set it up, enter it in the integration settings and stop using the old one. The setup is described in the Integration section ([browser.md](browser.md)); your antidetect browser may have installation notes of its own, see its manual.

## Errors

Errors are of two kinds: temporary ones go away by themselves, final ones need your action. The full table of codes with the action for each is in [accs.md](accs.md#vocabularies). In short:

| Group | Codes | What to do |
|---|---|---|
| Final | `101` registration failed, `200` dead account, `201` profile not found | delete the account (and its profile) and create a new one; `201` means the profile was deleted in the browser |
| Needs you | `102` no phone from the SMS service | top up the SMS balance or change the SMS country of the account, then restart the stopped tasks |
| Temporary | `901` proxy not set, `902` proxy error, `903` profile did not start, `904` profile already in use, `905` profile not created | they clear by themselves once the account works again, so wait; fix the cause where you can. `901`: give the account a new proxy with the repair form (the error clears at its next launch). `902`: if the proxy is alive the error goes away within hours, if it is dead replace it. `903`: update the antidetect browser, check that the server is up and the browser running. `904`: the service stops the profile and retries in a few minutes. `905`: check that the browser has free profiles. `906`: the connection to the account was lost, usually the proxy; harmless for farming |
| Advertising cabinet | `300` | the specifics are in the account description (`meta` in `accs/get`) |

After fixing the cause of `102` or of a proxy error, restart the stopped tasks through the repair form (`restart`) or with `task/restart`.

## Account settings form

Geo parameters and tags of the account (`accs/edit`). Change with care.

- **Country**: the main country of the account. Do not change it after the account was created or connected; this may lead to unforeseen consequences.
- **Country for SMS**: where phone numbers come from. By default it is the account country. Change it when registering or logging in fails for lack of numbers.
- **Tags** group and search accounts. A tag is lowercase Latin letters and digits, at least 3 characters, no special characters; tags are separated by a space. The interface has three tag fields: add tags to the current list (bulk editing only), set the tags and replace the list (bulk and single editing), delete all tags (bulk only). Through the API, `tags` of `accs/edit` replaces the list and an empty value clears it; `accs/make` and `accs/add` take one tag.

## Account repair form

Bulk repair of accounts and routine actions (`accs/fix`, `gang/fix` for a team). Each option is one independent flag.

| Option | Flag | Effect |
|---|---|---|
| Replace broken proxies | `fixproxy` | replace a proxy that does not answer, give one to an account that has none; same-country proxies first |
| Replace all proxies | `newproxy` | assign new proxies to all chosen accounts; a proxy that matched and worked may be kept or move to another account |
| Allow proxies from another country | `anyproxy` | use other countries when your own are not enough |
| Detach the proxy from the account in the service | `unproxy` | the browser manages the proxy on its own; has no effect together with the two replace options |
| Hide all errors | `fixerror` | the errors are not repaired, only hidden from the list |
| Dead these accounts | `kill` | status Inactive: out of service, no farming, statistics or tasks; their waiting and running tasks become Problem |
| Resurrect these accounts | `ressurect` | status Active again; stopped tasks do not restart by themselves, restart them afterwards |
| Restart problematic tasks | `restart` | re-run a task in Problem or Error status (one per account per call); use it only after the cause is gone |
| Delete problematic tasks | `noproblem` | delete a task in Problem status (one per account per call) |
| Delete erroneous tasks | `noerrors` | delete a task in Error status (one per account per call) |
| Fix the task queue | `requeue` | move the head of the queue to the current task; helps when the queue is stuck |
| Clear task queue | `unqueue` | delete tasks In queue and Waiting (registration tasks stay); Problem, Error and running tasks stay |
| Enable farming | `farmon` | farming on, settings and the next-run date unchanged |
| Turn farming off | `farmoff` | farming off, settings unchanged; frees threads and the queue instead of a bigger plan; turned back on, an account that missed its date joins the priority queue, and many of them can clog the queue |
| Start farming now | `farmnow` | an out-of-turn farm visit, also turns farming on |
| Enable statistics | `statson` | turn on the upload of advertising cabinet statistics |
| Turn statistics off | `statsoff` | turn the upload off |
| Update the statistics | `update` | an out-of-turn statistics update, also turns statistics on |
| Force stop the browser profile | `forcestop` | stop a hung profile even while a task runs; the running task goes back to Waiting |
| Synchronize profile settings | `sync` | refresh the description and settings of the profile in the browser, useful after changing a name, tags or other data |
| Upload fresh cookies | `cookie` | refresh the cookies stored for profile export; works with Vision and AdsPower only |
| Check the account exists in the browser | `existing` | a missing profile makes the account inactive with error `201`; for cleanup after tidying the browser by hand |
