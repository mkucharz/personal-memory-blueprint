# personal-memory-blueprint

Emergent Memory blueprint for a personal AI agent's knowledge graph — the AI
agent's long-term memory about the user's life.

It installs one schema pack (`personal-memory`, v3.1.0) and three agents. The
pack is the combined personal-memory + personal-kb schema: it carries the `Fact`
type and the general `Note` / `Event` names (renamed from `Memo` /
`CalendarEvent`), and uses the platform's semantic property types so values are
validated and normalized on write.

```bash
memory blueprints install https://github.com/mkucharz/personal-memory-blueprint
```

Or in the web UI: **Blueprints → Import from GitHub**, paste the same URL.

## What this blueprint provides

A template pack and three agents that together build, maintain, and query a rich
personal knowledge graph from emails, events, contacts, finances, and
conversations.

```
project.yaml            blueprint identity + the project description
schemas/
  personal-memory.yaml  the pack: 11 object types, 8 relationship types, migrations
agents/
  memory-extractor.yaml builds and maintains the graph
  daily-briefing.yaml   read-only morning briefing
  personal-kb-agent.yaml saves and recalls on request
Taskfile.yml            the local authoring loop (validate / inspect / install)
```

## Graph object types

| Type | Description |
|---|---|
| `Person` | A person in the user's life — family, friend, colleague |
| `Task` | A to-do item or action to complete |
| `Project` | A collection of related tasks and goals |
| `Event` | Something that happened or is planned; also calendar synced |
| `FinancialTransaction` | A bank or budget transaction |
| `Contact` | A raw address-book contact (Apple Contacts, Gmail) |
| `Place` | A physical location, city, venue, or address |
| `Note` | A free-form note, idea, or captured memory |
| `File` | A supporting document or attachment referenced by another object |
| `Fact` | A standalone durable fact |
| `Habit` | A recurring behaviour the user is tracking |

Every type declares `labels` (search/list grouping) and `ui` (icon + colour), so
the graph is readable in the UI without per-project configuration.

## Relationships

| Relationship | From → To | Meaning |
|---|---|---|
| `assigned_to` | Task → Person | Who is responsible |
| `belongs_to_project` | Task → Project | Task's parent project |
| `involves_person` | Event → Person | Attendees / participants |
| `located_at` | Event → Place | Physical venue |
| `related_to_contact` | Person → Contact | Person's address-book record |
| `has_transaction` | Project → FinancialTransaction | Related financial activity |
| `references_file` | Note → File | Supporting document |
| `triggered_task` | Event → Task | Follow-up tasks from meetings |

`involves_person` carries **edge properties**: `role` (`organizer`, `required`,
`optional`, `resource`) and `response_status` (`accepted`, `declined`,
`tentative`, `needsAction`, `none`). The calendar's raw attendee objects stay on
the event as the `attendees` property, so the source payload is never lost.

## Agents

| Agent | Description |
|---|---|
| `memory-extractor` | Extracts structured objects from emails, events, and conversations. The primary graph-building agent. |
| `daily-briefing` | Read-only briefing: today's events, overdue tasks, habit streaks, finance alerts, people context. No schedule — run it on demand. |
| `personal-kb-agent` | Saves and recalls information on request — the user-facing knowledge base assistant (`visibility: external`). |

No agent pins a model: each inherits the project's default model at run time, so
the blueprint works on any deployment and a model change is a project setting,
not a blueprint edit. Installing through the web UI therefore has no model
mapping step.

## Authoring — the local loop

`memory` is the [Memory CLI](https://github.com/emergent-company/emergent.memory).
Everything below is offline except `install` / `upgrade`.

```bash
task validate          # packs, agents, types, relationship properties, migrations
task validate:strict   # same, with warnings promoted to failures
task inspect           # what would be installed
task install PROJECT=<project>          # first install
task upgrade PROJECT=<project>          # re-apply after a change
task dry-run PROJECT=<project>          # preview against a project
```

The intended loop is: edit → `task validate` → `task upgrade PROJECT=<project>`
→ commit. Re-applying refreshes the pack's definitions in place, so a label,
description, type, bound or new property does **not** need a version bump; the
pack checksum moves, which is what the blueprint's "update available" signal
reads. Bump `version` in both `schemas/personal-memory.yaml` and `project.yaml`
(and tag `vX.Y.Z`) when the change is worth pinning, or when it needs a
migration.

### Property types this pack uses

The pack leans on the platform's semantic types rather than documenting a
format in prose — they are validated and normalized on write:

| Type | Used for | Stored as |
|---|---|---|
| `email` | `Person.emails[]`, `Event.organizer_email` | lower-case address |
| `phone` | `Person.phone`, `Place.phone`, `Person.phones[]` | as typed (7–15 digits) |
| `country` | `Place.country` | ISO 3166-1 two-letter code |
| `currency` | `FinancialTransaction.currency` | ISO 4217 code |
| `time` | `Habit.target_time` | `HH:MM` |
| `date` | birthdays, due dates, event start/end, completions | RFC 3339 |
| `url` | calendar links, websites, `File.url` | absolute http(s) URL |
| `array` | tags, emails, phones, attendees, recurrence | real array, element type enforced |
| `number` + `minimum`/`maximum` | `Place.lat`/`lng`/`rating`, streaks, file size | bounded on write |

Lists are real arrays now (`tags`, `emails`, `phones`, `attendees`, `types`,
`groups`, `recurrence`), not a `string` documented as "JSON array". An object
that still holds the old serialized string is coerced to an array the next time
it is written.

## Upgrade path

Projects on the bundled `personal-memory` v2.x pack upgrade through the pack's
`migrations` block, which renames `Memo` → `Note` and `CalendarEvent` → `Event`
across live graph data.

```yaml
migrations:
  from_version: "2.1.0"
  type_renames:
    - from: Memo
      to: Note
    - from: CalendarEvent
      to: Event
```

`from` names the old type; `to` must exist in the new pack (the CLI's
`validate` and the server both enforce that). Property renames use
`property_renames` with `type_name` / `from` / `to`, and intentional drops use
`removed_properties`.

> Note: a project that has **both** the old bundled `personal-memory` and the
> registry `personal-kb` applied will end up with `Note`/`Event` from both. This
> pack supersedes both; remove/replace the old installs when adopting it.

## Data sources

| Tool category | Types written |
|---|---|
| `calendar_*` (Google Calendar) | `Event` (with `involves_person` edges) |
| `apple_*` (Apple Contacts + Reminders) | `Contact`, `Task` |
| `places_*` (Google Places) | `Place` |
| `actualbudget_*` / `enablebanking_*` | `FinancialTransaction` |
| `memory-extractor` agent | `Person`, `Task`, `Project`, `Note`, `Fact`, `File` |

## Design notes

- `Event` is a generic type: calendar sync tools populate the calendar-specific
  fields (`event_id`, `calendar_id`, `start`, `end`, `attendees`, …); the agent
  uses the generic `name` / `date` / `notes` fields.
- `Person` and `Contact` are intentionally separate: `Contact` is a raw address
  book record; `Person` is the enriched knowledge-graph entity built over time.
- Personal-kb compatibility fields are accepted as aliases on the merged types
  (`phone`/`phones`, `employer`/`organization`, `occupation`/`job_title`,
  `Note.name`/`Note.title`).
- `Contact.emails` / `Contact.phones` keep the address-book `{label, value}`
  shape as an array of objects — it is a faithful copy of the source record,
  while `Person` holds the normalized values.
