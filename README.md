# lsb-roam-data-temp

Generated spawn regions for LSB, derived from the FFXI roaming dataset. **Temporary and machine
generated** — nothing here is hand-authored, and it is meant to be reviewed and moved into the real
`data/zones/` tree, not consumed as-is.

```
data/zones/<zone>/zone.yaml    # regions: geometry
data/zones/<zone>/mobs.yaml    # spawns: with a region: on the ones that got one
```

## What these files are

`zone.yaml` holds only a `regions:` block. Each region is an outline plus optional hole rings, every
vertex `[x, y, z]` in raw FFXI world coordinates, where **y is the floor under that corner** — the
polygon describes the ground surface, not a bounding volume. Overlapping regions are told apart by
whose floor is nearest. Rings are implicitly closed and have no winding requirement.

`mobs.yaml` is **not** the real LSB `mobs.yaml`. It is synthesized from `sql/mob_spawn_points.sql`
plus the roaming dataset and carries only the fields the region tooling reads (`template`, `at`,
`level`, `region`), so a zone can be opened and reviewed before its real YAML exists. Where a spawn
was given a region, its `at:` is dropped: the region replaces the fixed spawn point.

## How a region was decided

Each mob's recorded trail is rasterised at 4 yalms into the ground it covers. Mobs whose trail spans
under 25 yalms never leave their spawn and get no region. Everything else, patrol routes included, is
grouped with its neighbours by how much their territories overlap, and each group becomes one region
shared by every mob in it — a field belongs to the pack that walks it, not to one species.

Two rules keep the geometry honest, both grounded in the roam data rather than in a tuned constant:

- A polygon may not reach more than 4 yalms from ground a mob was actually recorded standing on, and
  padding may only land on cells *someone* in the zone was recorded standing on. Without the second
  rule an obstacle narrower than twice the padding vanishes: mobs pass either side, each side pads
  inward, and the rock is gone.
- A gap is only cut out as a hole if nobody in the zone was ever recorded inside it. A sparsely
  sampled flier leaves pockets all over its own range that ground mobs walk straight through; those
  are sampling artefacts, not buildings.

## Known limits

- **Fliers cover unwalkable ground.** A Pixie's region is the floor under a flight path — its trail
  spans 51 yalms vertically in West Ronfaure against about 11 for ground mobs. No amount of geometry
  fixes that; those regions need a human.
- **Region names are positional** (`<zone>_<compass>`, suffixed on collision) and are therefore not
  stable across regenerations. Do not reference them from anything until they are reviewed.
- **Coverage is limited by the capture.** A zone only gets regions where mobs were observed; sparsely
  sampled corners get nothing.

Regenerate with [xi-regions-bootstrap](https://github.com/sruon/xi-visualizer); review in the
visualizer's Regions page by pointing it at `data/zones`.
