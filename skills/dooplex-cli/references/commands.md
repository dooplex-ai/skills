# @dooplex/cli — command reference

All commands accept the global flags `--json`, `--app-url <url>`, and `--help`. With
`--json`, output is `{"ok":true,…}` (exit 0) or `{"ok":false,"error":"…"}` (exit ≠ 0).

## Auth

### `login`

Browser OAuth (loopback + PKCE). Opens `<app>/cli/authorize`; on approval, saves a
bearer token to `~/.dooplex/credentials.json` (chmod 0600).

```bash
npx @dooplex/cli login
```

### `whoami`

Show the active origin, token id, and expiry (or that it came from `DOOPLEX_CLI_TOKEN`).

### `logout`

Delete the saved token for the current origin.

### `self-update`

Standalone binary: downloads the latest release for this OS/CPU, verifies its
SHA-256 and replaces the executable in place. npm install: prints
`npm install -g @dooplex/cli@latest` and changes nothing.

```bash
npx @dooplex/cli self-update --json
# → {"ok":true,"action":"self-update","dist":"npm","command":"npm install -g @dooplex/cli@latest"}
```

## Reading (needs only membership)

### `list`

List creators the token can access.

- `--status active|paused|archived` — filter.
- `--all` — include archived.

```bash
npx @dooplex/cli list --json
# → {"ok":true,"creators":[{ "slug","displayName","status","unlimitedPriceCents",
#     "currency","defaultLanguage","timezone","websiteUrl","brandColor",
#     "replyChattiness","hasWelcomeMessage","createdAt","updatedAt" }, …],"count":N}
```

### `show <slug>`

Full detail for one creator: the fields above, plus `systemPrompt`, `channels`
(`channelType`, `displayName`, `enabled`), and `team` (`userId`, `role`).

```bash
npx @dooplex/cli show jane-doe --json
```

## Editing (needs owner-level access)

### `set <slug> [flags]`

Patch account settings. Any subset:

- `--name <text>`
- `--basic-price-cents <int>` (e.g. `499` = $4.99)
- `--unlimited-price-cents <int>` (e.g. `1999` = $19.99)
- `--currency <ISO>` (e.g. `USD`)
- `--language <locale>` (e.g. `en`, `zh-Hant`)
- `--timezone <IANA>` (e.g. `America/New_York`)
- `--website <url>`
- `--brand-color <#hex>` (e.g. `#1b2454`)

```bash
npx @dooplex/cli set jane-doe --basic-price-cents 499 --unlimited-price-cents 1999 --json
```

### `set-prompt <slug>`

Replace the twin's system prompt (versioned server-side). Exactly one of:

- `--prompt "<text>"`
- `--prompt-file <path>`

```bash
npx @dooplex/cli set-prompt jane-doe --prompt-file ./persona.txt
```

### `set-welcome <slug>`

Set or clear the `/start` welcome anchor. Exactly one of:

- `--message "<text>"`
- `--clear`

### `pause <slug>` / `resume <slug>`

Take the twin offline / bring it back. `pause` keeps existing subscriptions billing.

### `archive <slug> --yes`

Archive the creator: the public page goes offline and the twin stops replying.
Reversible. **Requires `--yes`.**

## Prompt versions (reads need only membership; rollback needs owner-level access)

### `prompt-versions <slug>`

System-prompt version history, newest first.

```bash
npx @dooplex/cli prompt-versions jane-doe --json
# → {"ok":true,"versions":[{ "id","changeSummary","isCurrent","createdByUserId","createdAt" }, …]}
```

### `prompt-show <slug> <versionId>`

One version's full editable body.

```bash
npx @dooplex/cli prompt-show jane-doe <versionId> --json
# → {"ok":true,"version":{ "id","editableBody","changeSummary","isCurrent","createdByUserId","createdAt" }}
```

### `prompt-rollback <slug> <versionId>`

Roll back to a prior version. **Owner-level only.** Bypasses the safety-drift
guard by design — the same behavior as the console's rollback (past versions were
already vetted when first saved). Does not expose the platform-guardrail toggles or
the safety-rules grid — those stay console-only.

```bash
npx @dooplex/cli prompt-rollback jane-doe <versionId> --json
```

## Knowledge (Q&A library) — no role gate beyond membership

Unlike prompt/public-page edits, these mirror `app/actions/knowledge.ts`, which has
no owner-only gate: any creator_users member (including `operator`) can edit.

### `knowledge-list <slug>`

- `--status active|archived|all` (default `active`).

```bash
npx @dooplex/cli knowledge-list jane-doe --status all --json
# → {"ok":true,"entries":[{ "id","question","answer","topic","state","timesUsed",
#     "lastUsedAt","createdAt","updatedAt" }, …],"count":N}
```

### `knowledge-show <slug> <entryId>`

Entry detail plus its full version history.

```bash
npx @dooplex/cli knowledge-show jane-doe <entryId> --json
# → {"ok":true,"entry":{…},"versions":[{ "id","questionSnapshot","answerSnapshot",
#     "isCurrent","editedByUserId","changeSummary","createdAt" }, …]}
```

### `knowledge-update <slug> <entryId>`

- `--question <text>` and `--answer <text>` (both required).
- `--topic <text>` (optional; **omit to leave the existing topic unchanged**, pass
  `--topic ""` to clear it).
- `--summary <text>` (optional changelog note).

Re-embeds the paired corpus item so retrieval reflects the edit; if that resync fails,
the response still succeeds but includes a `warning` — the reply may use the old
answer until the entry is edited again. **Fails with a 409** if the entry is
currently archived — restore it first (editing an archived entry's timestamp would
break the archive/restore coupling with its paired corpus item).

```bash
npx @dooplex/cli knowledge-update jane-doe <entryId> \
  --question "What time do you stream?" --answer "9pm daily." --json
```

### `knowledge-archive <slug> <entryId>` / `knowledge-restore <slug> <entryId>`

Soft-delete an entry (stops it grounding twin replies) / undo that.

### `knowledge-rollback <slug> <entryId> <versionId>`

Roll back one entry's question/answer to a prior version. Like `knowledge-update`,
re-embeds the paired corpus item on success, surfaces a `warning` if that resync
fails, and **fails with a 409** if the entry is currently archived.

```bash
npx @dooplex/cli knowledge-rollback jane-doe <entryId> <versionId> --json
```

## Public page (reads need only membership; edits need owner-level access)

### `page-show <slug>`

The Linktree-style public page: taglines, background color, cover image, socials,
and published state.

```bash
npx @dooplex/cli page-show jane-doe --json
# → {"ok":true,"profile":{ "taglineCn","taglineEn","accentColor","coverImageUrl",
#     "socials":[…],"published","updatedAt" } | null}
```

### `page-set <slug> [flags]`

Any subset:

- `--tagline-en <text>` / `--tagline-cn <text>`
- `--accent-color <#hex>` (e.g. `#336699`) — the page's background theme color
- `--publish` / `--unpublish` (publishing requires at least one tagline already set)
- `--socials-json '<json array>'` or `--socials-file <path>` (mutually exclusive) — an
  array of `{ name, url, handle, … }` objects, replacing the full socials list

```bash
npx @dooplex/cli page-set jane-doe \
  --tagline-en "Live daily at 8pm" --accent-color '#336699' --publish --json
npx @dooplex/cli page-set jane-doe --socials-file ./socials.json --json
```

## Corpus (grounding library)

`corpus-*` is the twin's **retrieval memory**: documents that get chunked, embedded,
and searched before every reply. `knowledge-*` is the separate curated Q&A list. When
the task is "teach the twin about X" or "give the twin this document", it is
`corpus-add`, not `knowledge-*`.

### `corpus-list <slug>`

`--folder <id>` one folder · `--unfiled` items with no folder · omit both for the
whole library · `--query <text>` search · `--status active|archived|all`.

Returns `folders` (each with `acceptsManualContent`) and `items`. The `shape` field
says which item shape came back: `active` items carry `chunkCount`, `archived` ones
carry `archivedAt` and no chunk counts. `--folder` and `--query` scope both shapes.

### `corpus-show <slug> <itemId> [--chunks]`

Item detail plus a content preview. `--chunks` returns the actual retrievable chunk
texts — use it when debugging why the twin did or didn't ground on something.

### `corpus-add <slug> --title <text> (--content <text> | --content-file <path>)`

Also `--folder <id>` (default Unfiled) and `--priority today|recent|standard|archived`
(re-rank weights 1.3 / 1.15 / 1.0 / 0.6 — a demotion, never a filter). Ingest is
synchronous: when it returns, the chunks are embedded and searchable.

```bash
npx @dooplex/cli corpus-add jane-doe --title "2026 tour FAQ" \
  --content-file ./tour-faq.md --folder "$FOLDER_ID" --json
```

### `corpus-update <slug> <itemId>`

`--title <text>` renames; `--folder <id>` / `--unfiled` refiles. Omitting a flag
leaves that field alone — `--unfiled` is the only way to clear the folder.

Content is capped at 10 MB per item; past that the call returns `413` and nothing is
written. Split a larger document across several corpus items.

### `corpus-archive` / `corpus-restore <slug> <itemId>`

Archiving keeps the vectors but excludes the item from chunk hydration, so it can
never reach a reply. It also disappears from the default listing: find it again with
`corpus-list <slug> --status archived`. `corpus-restore` requires `--folder <id>`,
because an item is refiled as it un-archives. It only ever un-archives: aimed at an
item that is still live it returns `409` rather than quietly moving it — use
`corpus-update --folder` for that.

### `corpus-folders <slug>` + `corpus-folder-create|rename|delete`

`corpus-folders` lists ids, counts, and `acceptsManualContent`. **Check that flag
before writing** — folders owned by Google Drive, member packages, or YouTube reject
manual content with a 400, because mixing sources breaks their next sync.
`corpus-folder-delete` needs `--yes` and moves the folder's items to Unfiled.

## Content packages

The creator→member publishing loop. Writes need owner-level access.

### `content-list` / `content-show` / `content-create` / `content-set`

`content-create <slug> --title <text>` makes a **members-only draft** — nothing is
visible to a fan until an explicit publish.

`content-set <slug> <id>` takes `--title`, `--description`, `--tier public|members`,
`--preview-chars <n>`, `--channels a,b`, and the body via `--body <text>` /
`--body-file <path>` with `--role body|source_notes` (default `body`).

`content-show` reports `accessSettingsLocked` and, per text role, whether it is
`indexed`. Both matter before a write:

- **Tier and preview length freeze at publication** — changing them afterwards is a
  `409`, not a silent no-op.
- **A text save can succeed un-indexed.** The response carries `indexingReady: false`
  and a warning; the text is not searchable and a send will be refused until it is.

### `content-publish` / `content-send` / `content-retry` / `content-unpublish` / `content-archive`

**Publishing and announcing are deliberately separate.** `content-publish` makes the
package readable and announces nothing. `--notify` adds the announcement, and both it
and `content-send` require `--yes`, because an announcement cannot be recalled.

**Check what the response says about delivery before calling it done.** A publish
returns `ok: true` even when the announcement it was asked to make reached nobody —
the package IS published; the send is what failed. Read:

| Field                                  | Meaning                                                            |
| -------------------------------------- | ------------------------------------------------------------------ |
| `announcementBlocked` + `warning`      | `--notify` was requested and went nowhere. A send is still owed.   |
| `partial` (`enqueued` < `targetCount`) | Some recipients were accepted, some were not.                      |
| `blockedChannels`                      | A channel attempted nothing (unavailable, or text still indexing). |

Two different recoveries, and picking the wrong one leaves the audience unreached:

- **A channel that attempted nothing** (`blockedChannels`, `announcementBlocked`) never
  stamped its initial claim. Fix the cause, then run **`content-send`** — it covers the
  channels that never fired and reports `alreadyNotified` for the ones that did.
- **Recipients a claimed send failed to reach** (`partial`, `enqueued < targetCount`)
  are only reachable through **`content-retry <slug> <id> --yes`**. A second
  `content-send` finds the claim stamped and touches none of them.

**The announcement audience is every reachable fan, not only members.** It is an
acquisition message — `fireMemberPackage` targets audience `all`, and the Mini App
applies `access_tier` only once the fan opens the link, showing non-members a
subscription prompt. Do not treat `--notify` as a members-only send. Announcing twice is idempotent — the response reports
`alreadyNotified` / `alreadyFired` rather than sending again.

### Not covered: asset upload

Images, video and subtitles use direct-upload URLs and multipart bodies. Those belong
to the browser; use the console for them. Everything else in the lifecycle is here.

## Team and export

### `team <slug>` / `team-add` / `team-remove`

`team-add --email <address> --role owner|operator`. If the address has no Dooplex
account yet, the grant is recorded as pending **with the role you asked for** and
applied on their first sign-in — the response carries `pending: true` and `role`.

An unclaimed invitation is a live grant, honoured whenever that address first signs in.
`team` lists them under `pendingInvitations`, and
`team-remove <slug> --email <address> --yes` withdraws one. Use it for a typo: nothing
else takes the grant back.

The **last owner** can be neither removed nor demoted (`409` either way): it would leave
the creator with nobody who can manage it, and there is no console team UI to undo it.
`team-add <own address> --role operator` is refused for the same reason `team-remove`
is. `--role` accepts `owner` and `operator`.

### `export <slug>`

The creator's configuration as JSON — settings, prompt, behavior, public page,
channels (redacted), team and unclaimed invitations. **No fan data, no credentials, no
corpus bodies.** It is a config backup to diff or restore from, not a data dump.

Team members carry their **email** as well as their user id, because `team-add` restores
by email — an export keyed only on Auth.js user ids cannot be replayed.

## Drops and broadcast

### `drops <slug>`

Scheduled drops with `kind`, `cadenceLabel`, `cadenceCron`, `access`, `bodySource`,
`enabled`. Cron expressions are interpreted in the **creator's** timezone, so the same
expression is a different wall-clock time for two creators.

### `drop-set <slug> <dropId>` — owner-level

`--title <text>` · `--body <text>` / `--body-file <path>` · `--access
free|preview|members` · `--enabled true|false`. Every field is validated before the
first write.

A `welcome` drop is the `/start` message and can never be gated: `--access members`
returns `409`. A drop id belonging to another creator returns `404` rather than
reporting a no-op success.

### `broadcast <slug> --yes` — owner-level

`--text <msg>` / `--text-file <path>`, `--mode exact|personalized`, `--broadcast-id
<id>`. Capped at 4096 characters (Telegram's per-message limit).

**`broadcastId` is the dedup key and is required by the API** — there is no
server-generated default, deliberately. Re-sending the same id is a no-op, so a retry
after a timeout is safe; a fresh id is a second broadcast to the entire audience,
which cannot be recalled. **If your caller may retry, pass `--broadcast-id` explicitly
and reuse the same value.** The CLI mints one per invocation otherwise — and prints
it to **stderr before the request is sent**, so that even if the response is lost to a
timeout you still hold the accepted dedup key and can retry safely. Rerunning without
it mints a fresh id, which is a second broadcast to the whole audience.

Outcomes that are not "sent" — already broadcast, no reachable fans, queue not
provisioned, partial enqueue — come back as `409`, not `400`: the request was fine,
the world's state is the answer. Do not reshape and retry.

## Channels

Until a bot is connected the twin cannot receive or send anything. Connecting one is
handled by the Dooplex team as part of onboarding; this read tells you whether it has
happened.

### `channels <slug>`

Connected channel accounts: `id`, `channelType`, `displayName` (the verified bot
handle), `enabled`. The encrypted bot token and the webhook secret are never
returned.

## Observability (read-only)

The other half of the loop. After changing a prompt, a skill, or the corpus, these are
how you check what the twin actually did — without them an agent is writing blind.

### `conversations <slug> [--limit <n>] [--cursor <c>]`

Fan threads ordered by most recent activity, plus `stats` (`activeThreads`,
`totalMessages`). `--limit` is 1–200 (default 50). When `hasMore` is true, pass
`nextCursor` back as `--cursor`.

Rows come from the `fan_threads` read-model, not from message bodies — since migration
0022 the transcript itself lives only in each fan's Durable Object.

### `transcript <slug> <fanId> [--limit <n>] [--before-seq <n>] [--session-only]`

One fan: `fan` (identity), `profile` (what the agent has learned about them),
`entitlement` (access level, active, quota), and `messages` oldest-first.

**Paging is keyset, not offset.** A response with `hasMore: true` also carries
`oldestSeq`; the next older page is `--before-seq <oldestSeq>`. Sequence numbers are
append-only and monotonic, so a page stays correct while new turns arrive — an offset
would silently skip or repeat turns.

`--session-only` limits the read to the fan's current session.

`profile` is the agent's memory notes about the fan (`text`, markdown) with the
`updatedAt` of the last memory write, or `null` when the agent has saved nothing yet.

**An unreadable transcript is a `503`, not an empty list.** If the transcript store
cannot be reached (a missing internal secret, a failed internal call), the response
carries `reason` and says so explicitly rather than returning `messages: []` — an
empty array here would be indistinguishable from a fan who has never spoken, and
would have you validate a config change against silence that is really an outage.

### `questions <slug>`

Returns **two separate lists**, because two different things are commonly called "the
question queue" and conflating them is misleading:

- **`escalations`** — pending rows the twin could **not verify** and handed to the
  creator (`unanswered_questions`, `kind='low_confidence'`). Each carries the fan's
  question, the twin's `draftAnswer` and its `confidence`. This is "the twin got
  stuck", and it is usually what you want after changing a prompt or the corpus.
- **`suggestions`** — the research agent's aggregated trending / knowledge-gap /
  stale-content topics (`ra_prompts`). Proactive ideas; the twin never failed at
  these. `payload` is parsed JSON, not a string.

Reading only `suggestions` would show an empty queue while real escalations were
pending, and would count topic ideas as work the twin could not do.

### Tenancy

All three are creator-scoped at the resolver. A fan id from another creator returns
`404` under a slug you can reach (never that fan's data) and `403` under the other
creator's slug.

## Twin behavior config

`set-prompt` writes the prompt body. `twin-show` / `twin-set` cover the rest of what
shapes behavior.

### `twin-show <slug>`

Returns `platformGuardrails` (category → boolean), `guardrailCategories`
(`{ optional, required }`), `guardrailAgentEnabled`, `safetyRules`, `fanOnboarding`
(`{ greeting, items }`), and `limits`. Membership is enough to read.

**Read this before any write** — `twin-set` replaces lists wholesale.

### `twin-set <slug> [flags]` — owner-level

| Flag                                      | Effect                                                  |
| ----------------------------------------- | ------------------------------------------------------- |
| `--guardrail-agent on\|off`               | Whether the post-generation guardrail agent runs.       |
| `--on <cat>` / `--off <cat>`              | Toggle one platform guardrail category.                 |
| `--guardrails-json` / `--guardrails-file` | Several categories at once (JSON object cat → boolean). |
| `--safety-rules-file <path>`              | JSON array — **replaces** the whole list.               |
| `--onboarding-file <path>`                | JSON `{ greeting, items }` — **replaces** both.         |

Mixing `--guardrails-json` with `--on`/`--off` is an error, not a precedence rule:
on a safety surface, guessing which one wins is worse than failing.

Guardrail patches **merge** onto the stored settings, so naming one category leaves
the others alone. An unrecognized category name returns `400` with the valid set in
`categories` — it never reports a no-op success.

The whole request is validated before anything is written. A `422` on the safety
rules therefore leaves a guardrail change in the same command **unapplied**: a failure
response really does mean nothing moved. The greeting counts toward the onboarding
entry cap, so greeting + 12 items is a `400` rather than a silently dropped item.

```bash
# Replace the safety-rule list: read, edit, write back.
npx @dooplex/cli twin-show jane-doe --json | jq '.safetyRules' > rules.json
# ...edit rules.json ([{ "label": "...", "description": "...", "enabled": true }])
npx @dooplex/cli twin-set jane-doe --safety-rules-file ./rules.json --json
```

### Statuses that mean different things

| Status | Meaning                                                                                                                                                                                                     |
| ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `422`  | A safety rule was **refused by the drift classifier** (`blockedRule` names which platform rule it would weaken). The request was well-formed. **Do not retry the same text** — rewrite the rule or drop it. |
| `400`  | Malformed request: bad category, bad onboarding item type, oversized rule, or an empty patch. Fix the shape.                                                                                                |
| `403`  | The token lacks owner-level access on this creator.                                                                                                                                                         |

Onboarding input is validated strictly, because `serializeOnboardingItems` is
tolerant by design for form posts and would otherwise hand an agent a `200` plus
quietly reshaped content. All of these are `400`s:

- an unknown item `type`;
- `{"type":"greeting"}` inside `items` — the greeting is the separate
  `fanOnboarding.greeting` field, and greeting entries are stripped from `items`;
- content (or a greeting) longer than the 600-character cap, which would be truncated;
- greeting + items exceeding 12 entries in total, where the last would be dropped;
- a `null` body or a `null` `fanOnboarding`.

When one request both enables a guardrail category and replaces the safety rules, the
rules are vetted against the **pending** guardrails, not the stored ones — so a rule
cannot slip through under a category the same request is turning on.

## Skills (the twin's playbooks)

A skill is a `SKILL.md`. Its frontmatter `description:` is the one-line "when to use"
the agent carries every turn; the body is fetched only when the agent decides the
skill applies. Zones: `self` (research agent's own playbooks) and `fans` (loaded into
fan-facing turns). Pick deliberately — the two steer completely different turns.

### `skill-list <slug> [--audience self|fans] [--bodies]`

Returns `{ skills: { self: [...], fans: [...] }, limits }`. Each entry is
`{ name, description }`; `--bodies` adds `content` and `truncated`. Bodies are opt-in
because ten 16k-char skills is a lot of JSON.

`truncated: true` means the `content` is a **prefix** of an over-limit body, not the
file. Writing a truncated body back discards its remainder — on either the list or
the single-skill read.

### `skill-show <slug> <zone> <name>`

Prints one body. A miss returns `404` **with an `available` array** of the zone's real
names — enough to correct a typo without a second listing call. `--json` surfaces
that array alongside `error`, so the recovery path is usable programmatically.

If the response carries `truncated: true`, the content is a **prefix** of a body that
exceeds the read limit, not the file. Never edit a truncated body and write it back —
that discards everything past the cut.

### `skill-set <slug> <zone> <name> (--content <text> | --content-file <path>)`

Creates or overwrites. **Owner-level only** — same bar as `set-prompt`, since
a skill steers behavior the same way. `created: true` in the response distinguishes a
new skill from an overwrite.

```bash
npx @dooplex/cli skill-set jane-doe fans tour-faq --content-file ./tour-faq.md --json
```

**Passing a body inline:** a `SKILL.md` starts with a `---` frontmatter fence, which
the bare `--content <text>` form would read as another flag. Use `--content=<text>`,
or `--content-file` (which is the better habit for multi-line content anyway). The CLI
detects the swallowed value and tells you which form to use.

Start the file with frontmatter, or the index line falls back to the first body line:

```markdown
---
description: Use when a fan asks about 2026 tour dates or ticket links.
---

Check the corpus for the tour FAQ before answering...
```

### `skill-delete <slug> <zone> <name> --yes`

Requires `--yes`: skills have no version history, so the body is unrecoverable.

### Failure modes worth distinguishing

| Status | Meaning                                                                                                                                                                                                                                                              |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `409`  | Zone is at its 10-skill cap — delete one first; do not retry as-is.                                                                                                                                                                                                  |
| `400`  | Bad zone, bad name, blank or oversized body — fix the request.                                                                                                                                                                                                       |
| `404`  | No such skill in that zone (read `available` from the body).                                                                                                                                                                                                         |
| `502`  | The workspace was unreachable and the outcome is **unknown** — the change may or may not have landed. The request itself was fine, so **re-read with `skill-list`/`skill-show` before retrying**; a blind retry of a delete can 404 on work that actually succeeded. |

## Discovery

### `commands`

The machine-readable command surface: every command with `group`, `summary`, `args`,
`flags`, `access` (`none|member|owner|channels`), `mutates`, and
`requiresConfirmation`.

**Read this instead of parsing `--help`.** The help text is prose; this is the
contract, and a test keeps it in step with what the CLI actually dispatches.

`requiresConfirmation` is present on every command, `true` or `false` — you never have
to infer it from the field's absence.

`mutates` plus `requiresConfirmation` are how an unattended caller decides what it may
run on its own. `broadcast` and `content-send` reach an entire audience and cannot be
recalled — both are flagged.

### `version` / `--version`

The installed CLI version.

## Environment

- `DOOPLEX_CLI_TOKEN` — bearer token to use instead of the saved file (CI/agents).
- `DOOPLEX_APP_URL` — default platform origin when `--app-url` is not passed
  (default `https://dooplex.ai`).

## Error handling

| Exit                                           | Meaning                           | Do                                       |
| ---------------------------------------------- | --------------------------------- | ---------------------------------------- |
| `0`                                            | success                           | read the `--json` payload                |
| ≠0, error mentions `login`                     | not authenticated / token invalid | run `login` (or set `DOOPLEX_CLI_TOKEN`) |
| ≠0, "do not have access" / "requires an owner" | wrong tenant or insufficient role | stop; the token can't do this            |
| ≠0, other                                      | validation or network error       | surface `error` verbatim                 |
