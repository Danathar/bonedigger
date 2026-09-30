# bonedigger — fleet-aware confirmations (v2 spec)

Load when working on multi-machine issue tracking: the opt-in fleet tag on `ujust report` / `ujust report --confirm`, distinct-machine counting on issues, or the `ujust fleet-status` command. Tracks [bonedigger#4](https://github.com/projectbluefin/bonedigger/issues/4).

**Status: proposed.** Issue #4 requires security/design review to settle identifier construction before any client implementation. This document is the design under review, not an approved implementation. Do not implement it in `common` until a maintainer signs off on [Identifier construction](#identifier-construction) and [Open review questions](#open-review-questions).

This is a v2 outcome. It does not change the v1 single-user / single-machine `ujust report` or `--confirm` contract: with fleet features off (the default), the report and confirm payloads are byte-for-byte what v1 sends.

## Goal

Someone who runs several Bluefin machines can see that an issue affects N of *their own* devices, and not mistake N confirmations from one person for N independent users. Fleet correlation is opt-in, removable, and uses GitHub Issues as the only backend. No central server, no new credential system.

## Non-goals

- No hostname collection.
- No background or implicit enrollment; every tag is the result of an explicit user action.
- No central telemetry service.
- No use of `/etc/machine-id`, hostname, MAC, DMI serial or any other hardware/OS identifier as input to the tag.
- No change to v1 behaviour for users who do not opt in.

## Ownership

| Concern | Owner |
|---------|-------|
| This spec, comment marker format, counting contract | `projectbluefin/bonedigger` |
| Client: `--fleet-*` flags, `ujust fleet-status`, local state file | `projectbluefin/common` (`60-bonedigger.just` + `/usr/libexec/bonedigger-report`) |
| Rendering counts on issues; confirm-based priority escalation | Hive / the downstream intake workflow (bonedigger no longer ships lifecycle workflows) |

Per `AGENTS.md`: do not add a sync workflow or copy client code into this repo.

## Identifier construction

### Decision

Fleet identity is **two random tokens created locally on explicit opt-in**, not derived from anything on the machine.

| Token | Generation | Meaning |
|-------|-----------|---------|
| `fleet` | 64 random bits (`openssl rand -hex 8` or `/dev/urandom`), created by the first `--fleet-enroll` | Groups machines the same person chose to group |
| `machine` | 64 random bits, created per machine by `--fleet-enroll` or `--fleet-join` | Distinguishes machines inside a fleet |

Both are lowercase hex, 16 characters. Neither is a hash, so there is nothing to reverse and nothing to dictionary-attack.

**Validation (MUST).** Every `fleet` and `machine` value is untrusted input until checked against `^[0-9a-f]{16}$`:

- `--fleet-join <fleet>` MUST reject any argument that does not match, before writing `fleet.json` or previewing anything.
- Loading `fleet.json` MUST re-validate both values and refuse to emit a tag if either fails (treat the file as corrupt, report it, post untagged).
- No value that failed validation is ever interpolated into a preview, comment marker or issue body.

This is not only a hygiene rule: the marker lives inside an HTML comment, so an unvalidated value containing `-->` would close the marker early and render attacker- or typo-supplied markdown inside the user's own posted comment.

State lives in one user-owned file, `${XDG_CONFIG_HOME:-$HOME/.config}/bonedigger/fleet.json`, mode `0600`, containing `{"version":1,"fleet":"…","machine":"…"}`. Its absence means "not enrolled".

Adding a second machine to a fleet is explicit and manual: run `ujust report --fleet-join <fleet>` on it (the user copies the 16-character fleet value). This generates a new `machine` token locally. No secret or credential is transferred, and fleet is not a secret — it only groups.

### Why not derive it from machine-id

Rejected: `hash(machine-id)` / a "hashed family" of it. Reasons:

- A stable global value would appear in every issue the machine touches and links them across users, images and repos with no user action to break the link.
- Anyone who has ever seen the machine-id (it appears in journals, crash reports, some log bundles) can confirm a match; systemd documents machine-id as confidential and recommends application-specific derivation.
- Removal is impossible: the value is recomputed identically after deletion.
- A random token is strictly better on every axis the issue names (not reversible, removable, no cross-user linkability) and costs nothing, because the only property needed is per-machine distinctness.

Also rejected: HMAC keyed with machine-id (e.g. `systemd-id128 --app-specific`). It is not reversible and not cross-app linkable, but it is recomputed deterministically, so it cannot be revoked, and it adds no value over a random token because machine-id does not survive a reinstall anyway.

Trade-offs of the random design (accepted, listed for review):

- Reinstall or deleting `fleet.json` yields a new machine token, so that box counts as a new machine on later issues. Undercounting is impossible; a small overcount is possible and only for a user who chose to enroll.
- Copying `fleet.json` between two machines makes them count as one. The user did that.

### Trust and forgery

The comment author is authenticated by GitHub. Counting therefore keys on the triple **(comment author login, fleet, machine)**, never on the tags alone. Consequences:

- A third party who copies a public tag into their own comment is counted under *their* login, as one extra machine at most. They cannot inflate or merge into someone else's fleet.
- Two different GitHub accounts never share a fleet count, even with identical tag values.
- Counts are evidence for a human decision. Because forging one's own extra machines is trivially possible, fleet counts must never be used as an automatic escalation input (see [Escalation](#confirmation-escalation-stays-truthful)).

## Opt-in and removal

Nothing is sent unless `fleet.json` exists. Enrollment never happens as a side effect of `report`, `--confirm`, `fleet-status` or an upgrade.

| Command | Effect |
|---------|--------|
| `ujust report --fleet-enroll` | Explain what will be posted, ask `gum confirm`, create `fleet.json` with fresh tokens |
| `ujust report --fleet-join <fleet>` | Same, reusing the given `fleet` value (rejected unless it matches `^[0-9a-f]{16}$`) with a new `machine` token |
| `ujust report --fleet-forget` | Delete `fleet.json`. Later reports and confirmations carry no tag |
| `ujust report --fleet-forget --purge` | As above, then strip the marker from the user's own past comments (see below) |
| `ujust report --no-fleet` | Skip the tag for one invocation without un-enrolling |

The full tag is shown in the local preview before anything is posted, exactly as it will appear, and is included only in the existing `gh issue create` / `gh issue comment` body — it adds no new network destination.

### Deletion semantics

Tags are posted publicly as part of the user's own comment or issue body, so they are the user's content:

- `--fleet-forget` stops future disclosure. It cannot recall what was already posted; the command says so.
- `--purge` **edits** the user's own marker-bearing issue bodies and comments to remove only the marker line. GitHub has no comment-search API, and marker text inside an HTML comment is not reliably indexed by issue search, so discovery cannot be a single query. The mechanism is:
  1. `gh search issues --author @me` and `gh search issues --commenter @me` (**unscoped — no `--repo` and no routing-table filter**, `--state all`) to get candidate issues — these search by participation, not by marker text. The search must not be narrowed to the trackers in the routing table: `parse_confirm_target` in `common`'s `/usr/libexec/bonedigger-report` accepts any `https://github.com/OWNER/REPO/issues/N` target, and `fleet-status` has a `--repo owner/name` override, so a tagged comment can exist in any repository the user can write to. Scoping discovery would silently leave those tags in place;
  2. for each candidate, `gh api --paginate repos/OWNER/REPO/issues/N/comments` plus the issue body, keep only items whose author login equals the authenticated login and whose body matches the marker regex;
  3. `gh issue comment --edit-last` is not sufficient; edit each match by id via `gh api --method PATCH repos/OWNER/REPO/issues/comments/<id>` (or `.../issues/N` for a body).

  Editing rather than deleting keeps the confirmation and its digest evidence; the item then counts as an untagged confirmation. GitHub retains edit history of comments; the command must say that, and that a user who needs the history gone must delete the comment through GitHub. Known limit: `gh search issues` only reaches issues the search index returns for that login, so `--purge` is best-effort and must report how many items it examined and edited rather than claiming completeness (see [Open review questions](#open-review-questions)).
- Rotation: `--fleet-forget` followed by `--fleet-enroll` produces new, unlinkable tokens.

## Comment marker

A single HTML comment on its own line at the end of the confirmation comment (or issue body for a report):

```
<!-- bonedigger-fleet: v1 fleet=<16 hex> machine=<16 hex> -->
```

Parsers must ignore any comment whose marker does not match this exact pattern, and must ignore unknown versions. The preview also renders a visible line (`Fleet: enrolled, fleet <fleet>, machine <machine>`) so the reporter sees what is disclosed; HTML comments are otherwise invisible in the rendered issue.

## Counting affected machines

Counts are derived read-side from the issue's comments; no state is stored anywhere but the comments themselves.

- **confirmations** — number of confirmation comments (unchanged v1 meaning). A confirmation comment is one whose body starts with the header ``**System fingerprint** (via `ujust report --confirm`)`` emitted by `confirm_report` in `common`'s `/usr/libexec/bonedigger-report`. Triage, discussion and bot comments on the same issue are not confirmations and must not be counted. The issue body is **not** a confirmation, so the reporter never inflates this number.
- **machines** — number of distinct `(author, fleet, machine)` triples among tagged confirmations **and the tagged issue body**. The reporter's own machine is an affected machine, so its marker counts once, attributed to the issue author.
- **fleets** — number of distinct `(author, fleet)` pairs over the same set.
- Untagged confirmations (opt-out users) are counted in `confirmations` only. Their issue data is never altered.

Presentation on the issue: `N confirmations · M distinct machines in F fleets (K untagged)`, where `K` is the `untagged` field below — confirmation comments carrying no marker. `tagged` counts the issue body's marker too, so `tagged + untagged` can exceed `confirmations`.

Reference implementation (input is `{"issue": <issue object>, "comments": [<comment objects>]}`, e.g. from `gh api repos/OWNER/REPO/issues/N` and `gh api --paginate repos/OWNER/REPO/issues/N/comments`; output verified against the acceptance cases below):

```jq
def is_confirmation:
  test("^\\*\\*System fingerprint\\*\\* \\(via `ujust report --confirm`\\)");
def tag:
  capture("<!-- bonedigger-fleet: v1 fleet=(?<fleet>[0-9a-f]{16}) machine=(?<machine>[0-9a-f]{16}) -->") // null;
[ .comments[] | select((.body // "") | is_confirmation) | {author: .user.login, tag: ((.body // "") | tag)} ] as $c
| ($c | map(select(.tag != null))) as $ct
| ([ {author: .issue.user.login, tag: ((.issue.body // "") | tag)} ] | map(select(.tag != null))) as $bt
| ($ct + $bt) as $t
| {
    confirmations: ($c | length),
    tagged: ($t | length),
    untagged: (($c | length) - ($ct | length)),
    machines: ($t | map([.author, .tag.fleet, .tag.machine]) | unique | length),
    fleets: ($t | map([.author, .tag.fleet]) | unique | length)
  }
```

`gh api --paginate` emits one JSON array per page; merge them first with `jq -s add`, then combine with the issue object (e.g. `jq -n --slurpfile i issue.json --slurpfile c comments.json '{issue: $i[0], comments: ($c | add)}'`) before applying the filter with `jq -f`.

The `is_confirmation` guard is what keeps the input honest: `repos/OWNER/REPO/issues/N/comments` returns every comment, including triage and bot chatter, and counting those would inflate `confirmations`. A consumer that already receives a pre-filtered confirmation-only list can drop the `select`, but must then guarantee the filtering elsewhere. The issue body deliberately bypasses the guard: it is read for its marker only, contributes to `machines` / `fleets`, and never to `confirmations`.

The GitHub API returns `body: null` for an issue opened with an empty body, and for a comment left with no text; `test` and `capture` abort the whole filter on a non-string input, so every body — issue and comment alike — is defaulted with `// ""` before matching. An untagged or empty body simply yields no triple, and a null comment body is not a confirmation.

## Surfacing fleet-relevant issues — `ujust fleet-status`

Read-only, local, no enrollment.

1. Read the booted image digest from `bootc status --json` (same source as the v1 fingerprint). An admin inspecting another machine runs the command on that machine, or passes `--digest sha256:<64 hex>` explicitly. There is no SSH access, no inventory file and no machine registry.
2. Search open issues in the image's tracker (same routing table as `--confirm`; `--repo owner/name` overrides) for the full digest string with `gh search issues … --state open`.
3. Fetch each candidate and **verify locally** that the digest appears in the issue body or a comment. Print only verified issues, each with its matching line as digest evidence, plus the current `confirmations · machines` line from [Counting](#counting-affected-machines).
4. Print nothing about closed issues and never write to GitHub. It needs only a signed-in `gh`, like `--confirm`.

**Validation (MUST).** `--digest` MUST match `^sha256:[0-9a-f]{64}$` and `--repo` MUST match `^[A-Za-z0-9._-]+/[A-Za-z0-9._-]+$`; a value that fails is rejected before it is interpolated into the `gh search issues` query or a repository path, for the same reason the fleet tokens are validated. The digest read from `bootc status --json` in step 1 MUST pass the same `^sha256:[0-9a-f]{64}$` check: the v1 client records `unknown` when bootc or jq is missing, and that value, or anything else that fails the check, stops the command with a message instead of being searched for.

It does not read or require `fleet.json`, and never creates it.

Known limit: digests inside a gist attachment are not searchable by GitHub issue search. Only digests present in issue or comment text match, which is exactly what `--confirm` comments and the report summary block put there.

## Confirmation escalation stays truthful

Confirmation-count escalation is a downstream intake concern, not something this repo ships today: bonedigger at this commit contains only `sync-templates.yml`, and `common`'s `docs/skills/bonedigger/references/full-loop.md` describes confirmations as human evidence rather than a label transition. No threshold configuration exists in either repo. The contract below is therefore what any such escalation **must** honour if and when it is implemented (the 3 and 5 confirmation thresholds referenced in issue #4 discussion are the intended values, not an implemented rule):

- It counts `confirmations` exactly as defined in [Counting](#counting-affected-machines) — confirmation comments only. `machines` and `fleets` are display-only and must not be fed into, substituted for, or subtracted from that count.
- Enabling fleet tags on some or all reports changes nothing about when a threshold is crossed; disabling them likewise.
- A fleet with 10 machines that confirmed once each is 10 confirmations and crosses a threshold exactly as 10 unrelated reporters would. The fleet line exists so a human reading the issue can discount them as one reporter; the priority call stays human.

## Acceptance mapping

| Acceptance criterion | How it is satisfied | Verification |
|----------------------|--------------------|--------------|
| Report without opting in sends no fleet tag; tag not reversible to a hostname and removable | Tag only emitted if `fleet.json` exists; tokens are random, not derived; `--fleet-forget[--purge]` | Client test: no `bonedigger-fleet` string in body without `fleet.json`; tokens uncorrelated with hostname/machine-id; after `--fleet-forget` no marker |
| Three confirmations from one device = 1 machine; three opted-in machines = 3 | `(author, fleet, machine)` distinct count over tagged confirmations plus the tagged issue body | Run the jq above: 3 comments with the same triple (untagged issue body) → `machines: 1`; 3 distinct `machine` values → `machines: 3`, including the case where one of the three is the reporter's tagged issue body plus 2 confirmations; an empty-body issue or a bodiless comment (`{"issue":{"user":{"login":"a"},"body":null},"comments":[{"user":{"login":"d"},"body":null}]}`) must exit 0 with `confirmations: 0, machines: 0, fleets: 0` rather than erroring |
| `ujust fleet-status` lists only relevant open issues with digest evidence; no consent-less enrollment | Local digest search + local verification; never writes `fleet.json` | Client test: closed and non-matching issues omitted; `fleet.json` absent afterward |
| Confirmation-count escalation behaves identically with fleet correlation on or off | Fleet counts are display-only | Escalation input is `confirmations`; same fixture with and without markers yields the same value |

## Open review questions

These need a maintainer / security decision before implementation:

1. Approve random per-machine tokens as the identifier construction (vs. any machine-derived design).
2. Approve 64-bit token length. Collisions matter only within one `(author, fleet)`, where the birthday bound at 64 bits is negligible for any realistic fleet.
3. Approve edit-not-delete as the `--purge` behaviour, given GitHub retains comment edit history.
4. Decide where the `machines` / `fleets` line is rendered (Hive-maintained issue summary vs. a local `ujust` view only).
5. Accept that `--purge` discovery is best-effort (GitHub has no comment-search API; participation search is the only entry point), or drop `--purge` in favour of pointing users at their GitHub comment list.

## Related

- [`bonedigger-ujust`](bonedigger-ujust.md) — v1 `ujust report` contract this builds on.
- [`bonedigger-overview`](bonedigger-overview.md) — architecture and privacy model.
- Client implementation and routing table for `--confirm`: `projectbluefin/common`, `docs/skills/bonedigger/references/full-loop.md`.
