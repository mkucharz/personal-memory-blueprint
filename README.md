# personal-memory-blueprint

Emergent Memory blueprint for a personal AI agent's knowledge graph — the
agent's long-term memory about the user's life, built so that no deadline is
missed and the facts behind each one are one search away.

It installs **six schema packs** — one core pack and five area packs — and
**four agents**.

```bash
memory blueprints install https://github.com/mkucharz/personal-memory-blueprint
```

Or in the web UI: **Blueprints → Import from GitHub**, paste the same URL.

> Upgrading an existing project from v3.x: do **not** use `--upgrade` — see
> [MIGRATION.md](MIGRATION.md). Blueprint upgrades currently do not change
> types that already exist in a project
> ([emergent.memory#294](https://github.com/emergent-company/emergent.memory/issues/294)).

## Layout

```
project.yaml              blueprint identity + the project description
schemas/
  personal-memory.yaml    core pack — shared types every area links to
  pm-companies.yaml       LegalEntity, Contract
  pm-money.yaml           Account
  pm-assets.yaml          Vehicle, Property, InsurancePolicy, Domain, Server
  pm-legal.yaml           LegalCase
  pm-knowledge.yaml       Guide
agents/
  memory-extractor.yaml     builds and maintains the graph
  obligations-watcher.yaml  daily: upcoming obligations → tasks, roll forward when done
  daily-briefing.yaml       read-only morning briefing
  personal-kb-agent.yaml    saves and recalls on request
Taskfile.yml              the local authoring loop (validate / inspect / install)
MIGRATION.md              moving a v3.x project (and personal-kb data) to v4
examples/dev-seed/        fictional demo graph — NOT part of the blueprint
```

## Pack rules

These keep the packs installable together without type-merge conflicts
([emergent.memory#293](https://github.com/emergent-company/emergent.memory/issues/293),
[#295](https://github.com/emergent-company/emergent.memory/issues/295)):

1. **Every object type is declared in exactly one pack.** Area packs link to
   core types by name (`sourceType: Person`) and never redeclare them. The CLI
   validator (0.91+) accepts links to types owned by another pack in the same
   blueprint.
2. **A link name declared in several packs carries identical `properties` in
   each.** The server keeps one property schema per link name, so differing
   definitions would be picked arbitrarily. `owns`, `related_to`, `party_to`,
   `has_obligation`, `sourced_from`, `held_with`, `used_by`, `located_at` and
   `has_transaction` are shared this way.
3. **Due dates live on `Obligation`.** Other types link to their obligations
   with `has_obligation` instead of carrying their own due-date fields.
4. **No secrets.** No passwords, keys, tokens, full card or account numbers,
   or identity document numbers — `Account.number_hint` holds at most the last
   four digits; `Server` is an inventory only.

## Typed pages (`ui.view`)

Every object type declares how its own page is laid out, next to the data type
itself — there is no per-project page configuration and no separate registry.
The block sits in the `ui` object that already carries the type's icon and
colour:

```yaml
ui:
  icon: "home"                    # a name from the Memory icon catalog
  color: "#A16207"
  view:
    inline: [property_kind, tenure]   # how a Property looks as a neighbour row
    sections:
      - title: Insurance
        relation: insures             # a relationship type this blueprint declares
        dir: in                       # in | out | both
      - title: Obligations
        relation: has_obligation
        dir: out
        limit: 5
```

- `inline` is the depth-1 preview shown when this type appears on another
  object's page.
- `sections` are the relationship groups on this type's own page, in reading
  order. `dir` is from this type's side: `out` = edges leaving it, `in` = edges
  arriving at it.
- A section that outgrows its `limit` collapses behind a "Show N more" control
  that also works without JavaScript; "Show all" stops at 200 rows and links to
  a filtered browse when there are more.
- Relations the view does not name still appear in a trailing "More
  connections" group, so adopting a view never hides an edge. That is why
  `sourced_from` is deliberately not claimed anywhere: provenance edges stay
  reachable without crowding the useful ones.
- Every `relation` must be declared by an installed pack; `task
  validate:strict` fails when one is not.

Icons are **names from the closed Memory icon catalog**, not emoji: the web UI
compiles a fixed set of glyphs, so a name outside it renders as a generic box
and an emoji renders as literal text. As of v4.1 every type here uses a catalog
name; the catalog is available over MCP as `schema-icon-list` and is defined in
`apps/server/pkg/sdk/schemaicon/schemaicon.go` of the Memory server repo.

## Object types

### Core — `personal-memory` 4.1.0

| Type | Description |
|---|---|
| `Person` | A person in the user's life; `is_self` marks the user, `relationship` is an enum |
| `Organization` | An outside organisation: bank, insurer, vendor, court, office, firm (**new**) |
| `Obligation` | Anything with a due date: filing, payment, renewal, expiry, subscription (**new**) |
| `Task` | A to-do item or action to complete |
| `Project` | A collection of related tasks and goals |
| `Event` | Something that happened or is planned; also calendar synced |
| `FinancialTransaction` | A bank or budget transaction |
| `Contact` | A raw address-book contact (Apple Contacts, Gmail) |
| `Place` | A physical location, city, venue, or address |
| `Note` | A free-form note, idea, captured memory, or imported email |
| `File` | A supporting document or attachment |
| `Fact` | A standalone durable fact |
| `Habit` | A recurring behaviour the user is tracking |

### Areas

| Pack | Types | Purpose |
|---|---|---|
| `pm-companies` 1.1.0 | `LegalEntity`, `Contract` | Companies the user owns or runs (JDG, sp. z o.o., AS) and their agreements |
| `pm-money` 1.1.0 | `Account` | Bank, fintech, tax/ZUS payment, and online service accounts |
| `pm-assets` 1.1.0 | `Vehicle`, `Property`, `InsurancePolicy`, `Domain`, `Server` | Things that must stay legal, insured, and renewed |
| `pm-legal` 1.1.0 | `LegalCase` | Court and administrative cases |
| `pm-knowledge` 1.1.0 | `Guide` | Maintained how-tos, checklists, calendars, profiles |

## Key links

| Link | From → To | Meaning |
|---|---|---|
| `has_obligation` | any subject → Obligation | The deadline belongs to this person, company, contract, account, asset, policy, or case |
| `payable_to` | Obligation → Organization | Who is paid / filed with / provides it |
| `paid_from` | Obligation → Account | Which account pays it |
| `triggered_task` | Obligation, Event, Note → Task | Task created for an occurrence; done task = obligation done |
| `owns` | Person, LegalEntity → anything owned | Ownership, with `share_percent` / `holder_role` |
| `board_role_in` | Person → LegalEntity | Formal role (board chair, director), `sole_signing` |
| `serviced_by` | LegalEntity → Organization, Person | Accountant, bank, law firm for a company |
| `party_to` | Person, LegalEntity, Organization → Contract, LegalCase | Party with `role` |
| `held_with` | Account, Domain, Server → Organization | Provider / registrar / host |
| `insures` / `issued_by` / `brokered_by` | InsurancePolicy → asset / insurer / agent | Insurance structure |
| `sourced_from` | anything → Note, File | Provenance (imported email, scan) |

The full list, with descriptions, is in each pack file.

## Agents

| Agent | Trigger | Description |
|---|---|---|
| `memory-extractor` | manual | Extracts structured objects from emails, events, and conversations; sets stable `key`s; never stores secrets |
| `obligations-watcher` | daily 05:00 (server time) | Creates one Task per upcoming Obligation occurrence; rolls the Obligation forward when that Task is done; reports gaps |
| `daily-briefing` | manual | Read-only briefing: events, deadlines in the next 14 days, tasks, people, finance, habits, notes |
| `personal-kb-agent` | chat | Saves and recalls on request (`visibility: external`) |

No agent pins a model: each inherits the project's default model at run time.

## Keys

Agents give every object a stable lower-case slug `key` and search by it
before creating, so re-runs update instead of duplicating:
`self`, `first-last`, `org-<name>`, `le-<country>-<name>`, `acct-<provider>-<qualifier>`,
`obl-<subject>-<what>`, `veh-<plate>`, `pol-<insurer>-<subject>`, `dom-<fqdn>`,
`srv-<hostname>`, `case-<case number>`, `guide-<slug>`, `task-<slug>`.

## Authoring — the local loop

`memory` is the [Memory CLI](https://github.com/emergent-company/emergent.memory)
(0.91+). Everything below is offline except `install` / `upgrade`.

```bash
task validate          # packs, agents, types, relationship properties, migrations
task validate:strict   # same, with warnings promoted to failures (CI gate)
task inspect           # what would be installed
task dry-run PROJECT=<project>
task install PROJECT=<project>          # first install into an empty project
```

`task upgrade` re-applies the blueprint, but until
[#294](https://github.com/emergent-company/emergent.memory/issues/294) is fixed
it only adds **new** types; changes to existing types are not applied to the
project's type registry. For a change to an existing type, follow the
uninstall → install → migrate procedure in [MIGRATION.md](MIGRATION.md).

### Property types

The CLI's strict validator (0.91) recognises only the canonical JSON types, so
semantic values use `type: string` with a `format`. The server validates and
normalises those formats on write exactly like the semantic types:

| Declared as | Used for | Stored as |
|---|---|---|
| `string` + `format: email` | emails, login emails | lower-case address |
| `string` + `format: phone` | phones | as typed (7–15 digits) |
| `string` + `format: country` | countries | ISO 3166-1 two-letter code |
| `string` + `format: currency` | currencies | ISO 4217 code |
| `string` + `format: time` | `Habit.target_time` | `HH:MM` |
| `date` | birthdays, due dates, start/end dates | RFC 3339 |
| `url` | links, websites | absolute http(s) URL |
| `array` | tags, emails, phones, IPs, recurrence | real array |
| `number`/`integer` + bounds | shares, mileage, lead times | bounded on write |

`enum` on a string property documents the allowed values and drives UI pickers.

## Data sources

| Source | Types written |
|---|---|
| Gmail (via `memory-extractor`) | `Note` (email), `Organization`, `Account`, `Obligation`, `Person`, `Task`, `LegalCase` |
| Google Calendar | `Event` (with `involves_person` edges) |
| Apple Contacts + Reminders | `Contact`, `Task` |
| Google Places | `Place` |
| Actual Budget / Enable Banking | `FinancialTransaction` |
| inFakt | `Obligation` (tax and ZUS), `Account`, `Organization` |
