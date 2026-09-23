---
name: dooplex-cli
description: >-
  Manage a Dooplex creator account (your AI twin) from the terminal via the
  @dooplex/cli command-line tool. Use when asked to list or inspect Dooplex
  creators, change a creator's subscription price or account settings (name,
  currency, language, timezone, website, brand color), pause /
  resume / archive a twin, update the twin's system prompt or /start welcome
  message, browse or roll back prompt versions, edit the Q&A knowledge library, add
  or manage the twin's corpus / grounding documents and their folders, read or
  write the twin's skills (playbooks), read or change the twin's guardrails,
  safety rules and fan onboarding, read conversations, a fan's transcript, or the
  twin's open questions, check whether the twin's channels are connected, edit
  scheduled drops or send a broadcast, manage console access, export a creator's
  configuration, draft/publish/announce member content packages, or
  edit the public (Linktree-style) page. Runs with `npx @dooplex/cli`; first
  authenticate with `npx @dooplex/cli login`. Self-contained — needs only Node.js
  and the published npm package.
---

# Manage a Dooplex creator account (CLI)

`@dooplex/cli` is the command-line client for [Dooplex](https://dooplex.ai). It is a
thin, authenticated client for the Dooplex platform API — it never touches a database
directly, and every operation goes through the same access checks the web console
uses, so it can only reach creators the signed-in user already has access to.

**Prerequisite:** Node.js ≥ 20. No install step — invoke with `npx @dooplex/cli`
(or `npm i -g @dooplex/cli` for a shorter `dooplex` command). Without Node, a
standalone binary installs with `curl -fsSL https://dooplex.ai/cli/install.sh | sh`
and updates with `dooplex self-update`. A once-a-day stderr notice mentions a newer
version; it never appears in `--json` output, and `DOOPLEX_NO_UPDATE_CHECK=1` disables it.

## 1. Authenticate (once)

```bash
npx @dooplex/cli login
```

This opens a browser to approve the CLI (OAuth loopback + PKCE), then saves a
long-lived bearer token to `~/.dooplex/credentials.json` (chmod `0600`). In an SSH
session no browser is opened: the CLI prints the link to open on your own computer,
and the approval page shows a code to paste back into the terminal. Verify and
manage the session:

```bash
npx @dooplex/cli whoami     # show the active identity/token
npx @dooplex/cli logout     # forget the saved token for this host
```

**Headless / CI / non-interactive agents:** don't run the browser flow. Instead have
the human mint a token interactively once, store it as a secret, and pass it via the
`DOOPLEX_CLI_TOKEN` environment variable — it overrides the saved file:

```bash
DOOPLEX_CLI_TOKEN="$DPX_TOKEN" npx @dooplex/cli list --json
```

## 2. Agent-friendly contract

Always pass `--json` when driving this programmatically:

- Success → `{"ok":true,…}` on stdout, exit code `0`.
- Failure → `{"ok":false,"error":"…"}` on stdout, **non-zero exit code**.

**The exit code is authoritative** — never infer success from stdout text. On a
`401`, the error tells you to run `login`; on a `403` you lack the required role for
that creator (edits need owner-level access; reads need only membership).

## 3. Discover the surface

```bash
npx @dooplex/cli commands --json
```

Returns every command with its arguments, required `access`
(`none|member|owner|channels`), whether it `mutates`, and whether it
`requiresConfirmation` (`--yes`), and the `flags` each command accepts — enough to
construct an invocation without reading the help text. **Prefer this over parsing `--help`** — the help
text is written to be read, the manifest is the contract.

Use `mutates` and `requiresConfirmation` to decide what is safe to run unattended:
`broadcast` and `content-send` reach an entire audience and cannot be recalled.

**Pass `--yes` as a bare flag.** `--yes=true` also works, but `--yes=<anything else>`
is refused — including the empty string, so `--yes=$CONFIRM` with the variable unset
errors instead of confirming. Do not build the flag out of a variable: decide whether
to send, then pass `--yes` or omit it.

## 4. Commands

```bash
npx @dooplex/cli list --json                 # creators you can access (--status, --all)
npx @dooplex/cli show <slug> --json          # settings + channels + team
npx @dooplex/cli set <slug> --basic-price-cents 499 --unlimited-price-cents 1999 --json
npx @dooplex/cli set-prompt <slug> --prompt-file ./persona.txt
npx @dooplex/cli set-welcome <slug> --message "Hey — welcome!"
npx @dooplex/cli pause <slug> --json         # twin offline (billing continues)
npx @dooplex/cli resume <slug> --json
npx @dooplex/cli archive <slug> --yes --json # reversible; --yes required

npx @dooplex/cli prompt-versions <slug> --json           # system-prompt history
npx @dooplex/cli prompt-rollback <slug> <versionId>      # revert to a prior version
npx @dooplex/cli knowledge-list <slug> --json             # Q&A library
npx @dooplex/cli knowledge-update <slug> <entryId> \
  --question "New Q?" --answer "New A."
npx @dooplex/cli page-set <slug> --tagline-en "Live daily at 8pm" --publish

npx @dooplex/cli corpus-folders <slug> --json             # folder ids + writability
npx @dooplex/cli corpus-add <slug> --title "Tour FAQ" \
  --content-file ./tour-faq.md --folder <folderId> --json
npx @dooplex/cli corpus-list <slug> --query tour --json
npx @dooplex/cli corpus-show <slug> <itemId> --chunks --json

npx @dooplex/cli skill-list <slug> --json                 # the twin's playbooks
npx @dooplex/cli skill-set <slug> fans tour-faq --content-file ./tour-faq.md
npx @dooplex/cli skill-delete <slug> fans tour-faq --yes

npx @dooplex/cli twin-show <slug> --json                  # guardrails, safety rules
npx @dooplex/cli twin-set <slug> --guardrail-agent on --json
npx @dooplex/cli twin-set <slug> --safety-rules-file ./rules.json --json

npx @dooplex/cli conversations <slug> --json              # what the twin has been doing
npx @dooplex/cli transcript <slug> <fanId> --json         # one fan's turns
npx @dooplex/cli questions <slug> --json                  # what it escalated

npx @dooplex/cli channels <slug> --json                   # is a bot connected?

npx @dooplex/cli drops <slug> --json                      # scheduled pushes
npx @dooplex/cli drop-set <slug> <dropId> --enabled false --json
npx @dooplex/cli broadcast <slug> --text-file ./note.md --yes --json

npx @dooplex/cli team <slug> --json                       # console access
npx @dooplex/cli export <slug> --json                     # config backup

npx @dooplex/cli content-create <slug> --title "Tour recap" --json
npx @dooplex/cli content-set <slug> <id> --body-file ./recap.md --json
npx @dooplex/cli content-publish <slug> <id> --json       # readable, silent
npx @dooplex/cli content-send <slug> <id> --yes --json    # announce — to EVERY
                                                          # reachable fan, members
                                                          # or not, unrecallable
npx @dooplex/cli content-retry <slug> <id> --yes --json   # only the ones it missed
```

**Close the loop.** After changing a prompt, skill, corpus item or guardrail, read
back with `conversations` / `transcript` rather than assuming the change landed the
way you intended — these are the only way to see the twin's actual output from the
terminal.

**Corpus vs knowledge — pick the right store.** `corpus-*` is the twin's grounding
library: documents that get chunked, embedded, and searched before every reply. To
teach the twin about something, or to hand it source material, that is `corpus-add`.
`knowledge-*` is the separate curated Q&A list — question/answer pairs the creator has
approved. Writing a document into the Q&A list, or a single FAQ answer into the
corpus, both technically succeed and both behave badly.

Full command + flag reference, JSON shapes, and worked examples:
[references/commands.md](references/commands.md).

## Guardrails

- **`set-prompt`/`prompt-rollback` change the twin's persona.** Apply an
  already-decided prompt, or revert to one that was already live; don't invent a new
  persona without the creator's sign-off.
- **`archive` refuses without `--yes`** — it takes the public page offline and
  silences the twin (reversible).
- **Knowledge edits need only membership**; prompt-rollback and public-page edits need
  owner-level access. If a command 403s, that's a role gap, not a bug.
- **Every edit command 409s on an archived creator** — `resume` it first. Reads by slug
  (`show`, `prompt-versions`, `knowledge-list`, `page-show`) still work on one, so reach
  an archived creator by slug rather than expecting `list` to surface it.
- The saved token and `DOOPLEX_CLI_TOKEN` are **secrets** — treat
  `~/.dooplex/credentials.json` accordingly.
- Prefer `--json` and check the exit code before reporting success.

## Verify a change landed

```bash
npx @dooplex/cli show <slug> --json     # re-read after a set/pause/etc.
```
