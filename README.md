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

Each mob's recorded trail is rasterised at 4 yalms into the ground it covers, filling the segment
between consecutive samples: samples sit a median 10 yalms apart, so rasterising the points alone
leaves a dotted line rather than a path. Mobs whose trail spans under 8 yalms never leave their spawn
and get no region. Everything else, patrol routes included, is grouped with its neighbours by how much
their territories overlap, and each group becomes one region shared by every mob in it — a field
belongs to the pack that walks it, not to one species.

Two rules keep the geometry honest, and both are answered by the zone's **navmesh** rather than by a
tuned constant:

- A polygon may not reach more than 4 yalms from ground a mob was recorded standing on, and it is
  clipped to the navmesh, so padding cannot cross a wall. Each grid cell is checked at nine points
  across it and needs seven on the navmesh: judging by the centre alone lets every boundary cell hang
  half a cell over the wall, while demanding all nine erases narrow indoor corridors, where no cell is
  ever fully covered. Where clipping severs a corridor the mobs walked through, the cells along the
  severed path are put back -- otherwise the far side holds no mob, is dropped, and a third of a
  territory disappears. Inferring walkable ground from the capture
  instead both over-reaches -- padding a trail out across a tunnel wall -- and under-covers, since
  the roam capture lights up only about 45% of a zone's walkable surface.
- A gap is cut out as a hole only when it is off the navmesh. A gap on walkable ground is somewhere
  the mobs simply were not recorded, not a building, so it is filled in.

The navmesh comes from `lsb/navmeshes/*.nav`; all 161 zones have one.

## Known limits

- **Fliers cover unwalkable ground.** A Pixie's region is the floor under a flight path — its trail
  spans 51 yalms vertically in West Ronfaure against about 11 for ground mobs. No amount of geometry
  fixes that; those regions need a human.
- **Region names are anchored, not positional.** A name is `<compass>_<index>` -- no zone prefix,
  since a name is only ever resolved against its own `zone.yaml`. The index is the zone-local mob
  index of the lowest-numbered mob in the region, and the compass reading comes from that mob's own
  recorded position. Each mob belongs to exactly one region, so the anchor
  is unique without a collision counter, and neither part moves when the geometry is retuned. Across
  a reach-cap change 99.9% of spawns keep their region name and across a simplification change 100%
  do; the residual churn is a grouping change genuinely moving a mob to a different region.
- **Coverage is limited by the capture.** A zone only gets regions where mobs were observed; sparsely
  sampled corners get nothing. The navmesh bounds a region, it does not extend one.
- **The 4-yalm grid still rounds narrow features up.** A tunnel comes out somewhat wider than it is.
  Halving the grid narrows it further but costs four to five times the vertices, which is not a trade
  worth making for a file meant to be read in a diff.

Regenerate with [xi-regions-bootstrap](https://github.com/sruon/xi-visualizer); review in the
visualizer's Regions page by pointing it at `data/zones`.
