# personal-memory-blueprint

Emergent Memory blueprint for a personal AI agent's knowledge graph — the AI
agent's long-term memory about the user's life.

This is the **combined** personal-memory + personal-kb pack (v3.0.0). It merges
the former bundled `personal-memory` pack with the registry-only `personal-kb`
schema: it adds the `Fact` type and standardizes on the general names `Note`
(renamed from `Memo`) and `Event` (renamed from `CalendarEvent`).

## What this blueprint provides

A template pack and three agents that together build, maintain, and query a rich
personal knowledge graph from emails, events, contacts, finances, and
conversations.

## Graph object types

| Type | Description | Key |
|---|---|---|
| `Person` | A person in the user's life — family, friend, colleague | display_name / email |
| `Task` | A to-do item or action to complete | auto |
| `Project` | A collection of related tasks and goals | name |
| `Event` | Something that happened or is planned; also calendar synced | event_id / auto |
| `FinancialTransaction` | A bank or budget transaction | `{source}:{external_id}` |
| `Contact` | A raw address-book contact (Apple Contacts, Gmail) | `{source}:{source_id}` |
| `Place` | A physical location, city, venue, or address | place_id / name |
| `Note` | A free-form note, idea, or captured memory | auto |
| `File` | A supporting document or attachment referenced by another object | name |
| `Fact` | A standalone durable fact | name |
| `Habit` | A recurring behaviour the user is tracking | name |

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

## Agents

| Agent | Description |
|---|---|
| `memory-extractor` | Extracts structured objects from emails, events, and conversations. Runs continuously or on-demand. |
| `daily-briefing` | Read-only morning briefing: today's events, overdue tasks, habit streaks, finance alerts, people context. |
| `personal-kb-agent` | Saves and recalls information on request — the personal knowledge base assistant. |

## Usage

### Install the blueprint into an Emergent project

CLI:

```bash
memory blueprints https://github.com/mkucharz/personal-memory-blueprint
```

Or in the web UI: **Blueprints → Import from GitHub**, paste
`https://github.com/mkucharz/personal-memory-blueprint`.

### Upgrade path

Existing projects on `personal-memory` v2.x upgrade to v3.0.0 via the pack's
migration block, which renames `Memo` → `Note` and `CalendarEvent` → `Event`
across live graph data.

> Note: a project that has **both** the old bundled `personal-memory` and the
> registry `personal-kb` applied will end up with `Note`/`Event` from both. The
> v3.0.0 pack supersedes both; remove/replace the old installs when adopting it.

## Data sources

| Tool category | Types written |
|---|---|
| `calendar_*` (Google Calendar) | `Event` |
| `apple_*` (Apple Contacts + Reminders) | `Contact`, `Task` |
| `places_*` (Google Places) | `Place` |
| `actualbudget_*` / `enablebanking_*` | `FinancialTransaction` |
| `memory-extractor` agent | `Person`, `Task`, `Project`, `Note`, `Fact` |

## Notes

- `Event` is a generic type: calendar sync tools populate its calendar-specific
  fields (`event_id`, `calendar_id`, `start`, `end`, `attendees`, …); the agent
  uses the generic `name` / `date` / `notes` fields.
- `Person` and `Contact` are intentionally separate: `Contact` is a raw address
  book record; `Person` is the enriched knowledge-graph entity built over time.
- Personal-kb compatibility fields are accepted as aliases on the merged types
  (`phone`/`phones`, `employer`/`organization`, `occupation`/`job_title`,
  `Note.name`/`Note.title`).
