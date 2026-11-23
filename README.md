# GGL.farm API — a skill for Claude

**Give your AI assistant the complete GGL.farm API — accounts, proxies, tasks and integration settings, every endpoint, parameter and error code, with OpenAPI specs — and the customer FAQ that explains how the farm behaves.**

[![License: MIT](https://img.shields.io/github/license/altercpa/skill-ggl)](LICENSE)
[![Latest release](https://img.shields.io/github/v/release/altercpa/skill-ggl)](https://github.com/altercpa/skill-ggl/releases/latest)

**English** · [Русский](README.ru.md)

![Purr the cat waters a young plant in a greenhouse of potted seedlings under glowing icons for the browser, proxies, tasks, SMS and integration — cover illustration for the GGL.farm API skill](.github/assets/cover.jpg)

## What is it

[GGL.farm](https://ggl.farm) is a Google-account farm service: it keeps your accounts and proxies, runs a
task queue and works through **your own antidetect browser** (Vision, OctoBrowser or AdsPower) on your own
server. The API repeats the web interface, so a farm of hundreds of accounts can be driven from a script.

This is a [Claude](https://claude.com) **skill** that teaches Claude that API. Ask for a script, an
integration or a bulk operation in plain words — Claude writes calls with the right endpoints, parameters
and error handling instead of guessing. Ask how the farm itself behaves — what a thread is, what error `902`
means, what the repair form does — and Claude reads the FAQ instead of guessing that too.

## What it covers

| Area | What you can do |
|---|---|
| **Accounts** | list and search, read one account, register new ones, connect existing ones by login and password or by browser profile, edit, repair and delete in bulk, open a profile for your own Playwright, Puppeteer or Selenium automation |
| **Proxies** | search the pool, add one proxy or a whole list in many line formats, set slots, check health, edit, delete |
| **Tasks** | queue a login or a 2FA setup, list the queue, restart a task that ended with a problem or an error, cancel one |
| **Integration** | connect the antidetect browser, choose and configure the SMS service, set the CapMonster key, read the journal of requests sent to your browser, see the thread load of your plan |
| **Teams** | the team administrator's view of every member's accounts: list, read, repair, delete |
| **How the farm works** | the customer FAQ — adding versus registering, threads and limits, the working cycle, 2FA, proxies, SMS, teams, choosing a server, error codes, the repair form |

## Install

### Claude Code

Download [`altercpa-ggl-api-skill.zip`](https://github.com/altercpa/skill-ggl/releases/latest/download/altercpa-ggl-api-skill.zip)
and unpack it into a skills folder:

```sh
unzip altercpa-ggl-api-skill.zip -d ~/.claude/skills/     # every project
unzip altercpa-ggl-api-skill.zip -d .claude/skills/       # this project only
```

On Windows unpack it into `%USERPROFILE%\.claude\skills\`.

Prefer git? Clone straight into the skills folder, and `git pull` is the whole update:

```sh
git clone https://github.com/altercpa/skill-ggl.git ~/.claude/skills/altercpa-ggl-api
```

### Claude.ai and Claude Desktop

In Claude's settings add a custom skill and upload the same zip.

## Try it

Nothing to call by hand — Claude loads the skill on its own whenever a request touches GGL.farm or its API.
For example:

- *Write a Python script that loads my proxy list into GGL.farm, checks the proxies and deletes the ones in error.*
- *Add the accounts from this file to GGL.farm with the tag `batch7` and turn farming on.*
- *Which of my accounts have error 902, and how do I fix them all in one call?*
- *Open the browser profile of account 501 over CDP with Playwright and take a screenshot.*
- *What does error 102 mean and what do I do about it?*

Calls follow one pattern; the token is `<user id>-<API key>` from your profile page at [ggl.farm](https://ggl.farm):

```
https://ggl.farm/api/{app}/{func}.json?id={token}
```

## Update

Releases are tagged `vYYYY.MM.DD`. Watch the repository (*Watch → Custom → Releases*) to be
notified, then download the new zip — or run `git pull` if you cloned.

## What is inside

| Path | What |
|---|---|
| [`SKILL.md`](SKILL.md) | the skill's entry point |
| [`references/`](references) | the reference, one file per API application, an overview of the shared rules, and the customer FAQ |
| [`spec/`](spec) | OpenAPI 3.1 specifications, one file per application |

## License

[MIT](LICENSE) — use it, copy it, change it, ship it.

The mascot artwork in [`.github/assets/`](.github/assets) is © AlterCPA and is not covered by the MIT license.
