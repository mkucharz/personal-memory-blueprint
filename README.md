# personal-memory-blueprint

Emergent Memory blueprint for a personal AI agent's knowledge graph — the
agent's long-term memory about the user's life, built so that no deadline is
missed and the facts behind each one are one search away.

It installs **seven schema packs** — one core pack and six area packs — and
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
  pm-money.yaml           Account, Invoice
  pm-assets.yaml          Vehicle, Property, InsurancePolicy, Domain, Server
  pm-legal.yaml           LegalCase
  pm-knowledge.yaml       Guide
  pm-travel.yaml          IdentityDocument, Trip, Booking, Claim
  pm-documents.yaml       Document (document registry)
  pm-belongings.yaml      Item (belongings registry)
  pm-review.yaml          OpenQuestion (questions for the user)
  pm-loans.yaml           Loan (mortgages and other debts)
  pm-maintenance.yaml     ServiceRecord (inspections, services, repairs)
  pm-home.yaml            UtilityService (electricity, water, internet, ... at a property)
agents/
  memory-extractor.yaml     builds and maintains the graph
  obligations-watcher.yaml  daily: upcoming obligations → tasks, roll forward when done
  daily-briefing.yaml       read-only morning briefing
  personal-kb-agent.yaml    saves and recalls on request
Taskfile.yml              the local authoring loop (validate / inspect / install)
MIGRATION.md              moving a v3.x project (and personal-kb data) to v4
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
   `has_obligation`, `sourced_from`, `held_with`, `used_by`, `located_at`,
   `involves_person`, `issued_by` and `has_transaction` are shared this way.
   Within one pack a link name is declared once (list several `sourceTypes`
   instead of repeating the name).
3. **Due dates live on `Obligation`.** Other types link to their obligations
   with `has_obligation` instead of carrying their own due-date fields.
4. **No secrets.** No passwords, keys, tokens, or full card or account
   numbers — `Account.number_hint` holds at most the last four digits;
   `Server` is an inventory only. Identity document numbers are optional and
   stored only when the user wants them.

## Object types

### Core — `personal-memory` 4.0.0

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
| `pm-companies` 1.0.0 | `LegalEntity`, `Contract` | Companies the user owns or runs (JDG, sp. z o.o., AS) and their agreements |
| `pm-money` 1.1.0 | `Account`, `Invoice` | Bank, fintech, tax/ZUS payment, and online service accounts; invoices issued and received |
| `pm-assets` 1.0.0 | `Vehicle`, `Property`, `InsurancePolicy`, `Domain`, `Server` | Things that must stay legal, insured, and renewed |
| `pm-legal` 1.0.0 | `LegalCase` | Court and administrative cases |
| `pm-knowledge` 1.0.0 | `Guide` | Maintained how-tos, checklists, calendars, profiles |
| `pm-travel` 1.0.0 | `IdentityDocument`, `Trip`, `Booking`, `Claim` | Document expiries, journeys and reservations, complaints and compensation claims |
| `pm-documents` 1.1.0 | `Document` | Registry of important documents: kind, date, whether the paper original is held and where, best digital copy (Drive account / folder, email); `concerns`, `issued_by`, `party_to_document`, `has_copy`, `supersedes`. Also the map for Drive and email cleanup |
| `pm-review` 1.0.0 | `OpenQuestion` | The KB's own to-do list: questions agents could not answer from sources (which vehicle, still owned?, signed?). Linked with `asks_about`; closed with the user's answer after the linked records are updated. The daily briefing shows the top 3 |
| `pm-loans` 1.0.0 | `Loan` | Mortgages and other debts: agreement number, principal, indexation (e.g. CHF), status incl. annulled / in dispute; `lent_by`, `borrowed_by` (role), `secured_by` a property, `disputed_in` a court case |
| `pm-maintenance` 1.0.0 | `ServiceRecord` | Service history: technical inspections, heat-pump and boiler inspections (warranty conditions), repairs; `service_of` a vehicle, property, item or server, `serviced_by`, next due date as an Obligation |
| `pm-home` 1.0.0 | `UtilityService` | Supply contracts at a property (electricity, water, waste, gas, internet, mobile): supplier, customer/meter numbers, contract holder and whether it must be transferred, billing cycle, last meter reading; `supplied_by`, `serves`, `held_by`, `billed_through` |
| `pm-belongings` 1.1.0 | `Item` | Registry of belongings: hardware, tools, appliances, gear; owner (person or company for business assets), `kept_at` a property or vehicle, serials, purchase and warranty data, `lent_to`, `documented_by` a Document |

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
| `issued_by` / `billed_to` | Invoice → LegalEntity, Organization, Person | Seller and buyer of an invoice |
| `under_contract` / `settles` | Invoice → Contract / Obligation | The agreement billed under; the invoicing duty it fulfils |
| `holds` | Person → IdentityDocument | Whose document it is |
| `part_of_trip` / `booked_with` | Booking → Trip / Organization | Trip structure and provider |
| `claim_against` / `claim_for` | Claim → Organization / Booking, Trip, InsurancePolicy, Account | Who a claim is against and what it is about |
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
`srv-<hostname>`, `case-<case number>`, `guide-<slug>`, `task-<slug>`,
`inv-<seller>-<number>`, `iddoc-<holder>-<kind>`, `trip-<yyyy-mm>-<destination>`,
`booking-<provider>-<reference>`, `claim-<counterparty>-<reference>`.

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

**Adding a new type to a pack that is already installed** (e.g. `Invoice` in
`pm-money` 1.1.0) has the same problem: the upgrade updates the pack but never
registers the new type. Remove that one pack's assignment and re-run the
install, which registers the pack as new; objects of its existing types are
kept:

```bash
memory schemas installed --project <project>            # find the pack's assignment ID
memory schemas uninstall <assignment-id> --project <project>
task upgrade PROJECT=<project>
```

A brand-new pack (e.g. `pm-travel`) installs normally with `task upgrade`.

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
