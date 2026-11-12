# Demo graph (fictional)

A small, completely fictional personal-memory graph for trying the blueprint
out — every person, company, account and case below is invented, and no value
here came from a real person's data.

It carries 60 objects covering **all 23 types** and 112 relationships covering
**all 36 link types**, shaped so that every `ui.view` section has something to
show: a person who owns assets and a company, a truck with insurance and a
renewal, a flat with rent and a deposit dispute, a court case with a hearing and
an appeal, an account with transactions, and a runbook three objects are
documented in.

## It is not part of the blueprint

This directory sits **outside** `seed/` on purpose. `memory blueprints install`
auto-discovers `<blueprint>/seed/objects/` and `<blueprint>/seed/relationships/`
and would apply that data to every project that installs the blueprint. Demo
data does not belong in a real knowledge graph, so it lives here and is applied
explicitly, per project.

## Applying it

The layout is the `memory blueprints dump` shape, which
`memory graph objects create-batch` accepts directly:

```bash
# against the project you want the demo data in
memory graph objects create-batch --file examples/dev-seed --project <project>
```

Records with a `key` are upserted, so the command is safe to re-run. Every
object here has a key; relationships are keyed by `(type, srcKey, dstKey)`.

Objects are seeded **after** the blueprint's packs are installed, because the
seed rows name object types and link types the packs declare.

## Removing it

The demo data is meant for a throwaway project — delete the project when you are
done with it. If you seeded a project you intend to keep, delete the demo
objects by id (`memory graph objects list --project <project>`, then
`memory graph objects delete <id>`); every object here carries a demo `key`, so
they are easy to recognise.
