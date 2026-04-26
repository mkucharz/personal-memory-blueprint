# personal-memory-blueprint

Emergent Memory blueprint for Diane's personal knowledge graph — the AI agent's
long-term memory about the user's life.

## What this blueprint provides

A template pack and two agents that together build and maintain a rich personal
knowledge graph from emails, calendar events, contacts, finances, and conversations.

## Graph object types

| Type | Description | Key |
|---|---|---|
| `person` | A person in the user's life — family, friend, colleague | display_name / email |
| `task` | A to-do item or action to complete | auto |
| `project` | A collection of related tasks and goals | name |
| `calendar_event` | A Google Calendar event | `{account}:{google_event_id}` |
| `financial_transaction` | A bank or budget transaction | `{source}:{external_id}` |
| `contact` | An address book contact (Apple Contacts, etc.) | `{source}:{source_id}` |
| `place` | A physical location from Google Places | `place_id` |
| `note` | A free-form note or captured memory | auto |
| `habit` | A recurring behaviour the user is tracking | name |

## Relationships

| Relationship | From → To | Meaning |
|---|---|---|
| `assigned_to` | task → person | Who is responsible |
| `belongs_to_project` | task → project | Task's parent project |
| `involves_person` | calendar_event → person | Attendees / participants |
| `located_at` | calendar_event → place | Physical venue |
| `related_to_contact` | person → contact | Links person to address book record |
| `has_transaction` | project → financial_transaction | Related financial activity |
| `references_file` | note → file | Supporting document |
| `triggered_task` | calendar_event → task | Follow-up tasks from meetings |

## Agents

| Agent | Description |
|---|---|
| `memory-extractor` | Extracts structured objects from emails, events, and conversations. Runs continuously or on-demand. |
| `daily-briefing` | Read-only morning briefing: today's events, overdue tasks, habit streaks, finance alerts, people context. |

## Usage

### Install the pack into an Emergent project

```bash
memory blueprints https://github.com/mkucharz/personal-memory-blueprint
```

## Data sources

These tools populate the personal graph:

| Tool category | Types written |
|---|---|
| `calendar_*` (Google Calendar) | `calendar_event` |
| `apple_*` (Apple Contacts + Reminders) | `contact`, `task` |
| `places_*` (Google Places) | `place` |
| `actualbudget_*` / `enablebanking_*` | `financial_transaction` |
| `memory-extractor` agent | `person`, `task`, `project`, `note` |

## Notes

- `calendar_event` type names are designed to be written by Diane's calendar sync tool
  once implemented (mirrors the `EventInfo` struct from `google/calendar/client.go`).
- `financial_transaction` covers both Actual Budget and Enable Banking via the `source` field.
- The `person` and `contact` types are intentionally separate: `contact` is a raw address
  book record; `person` is the enriched knowledge graph entity Diane builds over time.
- Tool delivery into Emergent sandbox is pending — see Diane's migration plan for status.
