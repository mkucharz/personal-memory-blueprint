# Migrating to v4.0.0

## 4.0.0 → 4.1.0 (page layouts and icon names)

v4.1 is a presentation-only change: every object type gains a `ui.view` block
describing its own page, and every type's `ui.icon` switches from an emoji to a
name from the Memory icon catalog. No type is added, renamed, removed, or given
a new property, so existing data needs no migration.

Applying it to an existing v4.0 project is a normal re-install:

```bash
memory blueprints install . --project <project> --upgrade
```

If the project's object pages do not change afterwards, the type registry kept
the first-installed definitions ([#294](https://github.com/emergent-company/emergent.memory/issues/294)):
unassign the six packs and install again, exactly as in the v3 → v4 procedure
below. Graph data is untouched either way.

Two rendering details worth knowing:

- A gateway older than the `ui.view` feature ignores the block and still reads
  `icon`/`color`, so installing 4.1 is safe before the server is upgraded — the
  pages simply stay on the generic layout.
- An emoji in `ui.icon` renders as literal text on a current gateway, which is
  why v4.1 replaces them; a name outside the catalog renders as a generic box.

---

v4 splits the blueprint into a core pack plus five area packs and adds
`Organization`, `Obligation`, and the area types. A project already on v3.x
(and/or the older `personal-kb` pack) cannot simply run
`memory blueprints install … --upgrade`:

- An upgrade re-applies the packs but **does not change types already in the
  project's type registry**; with or without `--merge-types`, existing types
  keep their first-installed definition
  ([emergent.memory#294](https://github.com/emergent-company/emergent.memory/issues/294)).
- When several installed packs define the same type, different parts of the
  server pick different definitions
  ([#295](https://github.com/emergent-company/emergent.memory/issues/295)).

So the path is: **back up → uninstall the old packs → install v4 fresh →
re-type the data on a graph branch → review → merge**. Graph objects survive
a pack uninstall (types are not an allowlist), so no data is lost by step 2.

## 0. Prerequisites

- Memory CLI 0.91 or newer (`memory --version`).
- If your CLI config pins a project and project token, run the commands below
  with `--config` pointing at a config for the target project, or pass
  `--project` and make sure no other project's token is active.

## 1. Back up

```bash
memory blueprints dump ./backup-$(date +%F) --project <project>
memory schemas installed --project <project> > ./backup-$(date +%F)/schemas.txt
```

## 2. Uninstall the superseded packs

List what is installed, then remove every `personal-kb` and `personal-memory`
version (keep unrelated packs such as `meetings`):

```bash
memory schemas installed --project <project>
memory schemas uninstall <assignment-id> --project <project>   # repeat per old pack
```

Check that no registry row still claims the core type names (`Person`,
`Event`, `Note`, `Task`, …) for an old pack. A pack from another blueprint
that also defines one of these names (e.g. `meetings` defines `Person`) will
keep it; that is acceptable, but note it (see step 5).

## 3. Install v4 fresh

```bash
task validate:strict
task dry-run PROJECT=<project>
task install PROJECT=<project>
```

Then confirm the registry, not only the compiled view, carries the v4
definitions:

```bash
memory schemas compiled-types --verbose --project <project>
# and spot-check GET /api/schema-registry/projects/<project>/types/Person
```

## 4. Re-type the data (on a graph branch)

Do this on a branch (`graph-branch-create`, e.g. `migration/v4`), review the
diff, then merge. Same-name types (`Account`, `Guide`, `Person`, `Project`,
`Event`, `Note`) need no rename — only their properties are reshaped. Types
that split or change name are moved object by object (create the new object
with the same `key`, re-point its edges, delete the old one):

| Old (v3 / personal-kb) | New (v4) | Rule |
|---|---|---|
| `Asset` with category `Vehicle` | `Vehicle` | plate, make/model from name/details |
| `Asset` with category `Server` | `Server` | hostname, IPs, role; drop any key or credential mention |
| `Asset` that is an institution (school, office) | `Organization` | `org_kind` from context |
| `Account` with category `Domain Registration` | `Domain` | `name` = FQDN; renewal → `Obligation` |
| `Account` with category `Insurance` | `InsurancePolicy` | insurer → `Organization` via `issued_by`; renewal → `Obligation` |
| `Account` for a tax/ZUS payment | `Account` (`account_kind` tax_payment / social_insurance) | the payment deadline → `Obligation` |
| `Account` describing a business partner | `Organization` (+ `Contract` if an agreement exists) | |
| `Project` describing a company | `LegalEntity` | tax/registration numbers, `status`, `closed_on` |
| `Event` describing a recurring payment, filing, or renewal | `Obligation` | `recurrence`, `next_due`, `auto_pay`; delete the Event |
| `Document` describing a court case | `LegalCase` | `case_number` from the name |
| other `Document` | `File` | |
| `Email` | `Note` (`source: email`) | keep `mentions` / `sourced_from` edges |
| `Guide` with empty properties | `Guide` | fill `content` from the source document, or mark for review |

Also in this step:
- Give every object a `key` following the conventions in the README.
- Merge duplicates (same person / organisation / account recorded twice):
  keep the richer object, move edges, delete the other.
- Set `Person.is_self` on the user and `relationship` where known.
- Close what has ended (`Account.status=closed`, `LegalEntity.status=deregistered`)
  instead of deleting it.
- Re-point existing edges whose names v4 declares (`owns`, `works_on`,
  `manages`, `advises`, `related_to`, `mentions`, `sourced_from`, `used_by`,
  `documented_in`, `insures`); rename `uses` (Project → Account) to the
  reverse `used_by`, and `sources_from` to `sourced_from`.

## 5. Verify and merge

- `memory schemas compiled-types --verbose`: every v4 type listed once and not
  shadowed (or shadowed only by a pack you chose to keep, e.g. `meetings` →
  `Person`).
- No objects left on the retired types (`Asset`, `Email`, `Document`).
- Every `Obligation` with status=active has `next_due`.
- Run `obligations-watcher` once by hand and check the tasks it creates.
- Merge the branch.
