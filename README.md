# hataori 機織 — garment / apparel robotics actor

**hataori** (機織, *hata-ori*) means weaving on a loom. The name says nothing about what
the code does, so: this repository is the **governed actor for cut-make-trim garment
work** — pattern grading, fabric cutting, garment assembly, needle-detect QC, and
finishing. It is the actor that confronts the sweatshop directly.

**LPS #2** (ADR-2606032100). ISIC **C13-14** · ISCO **7531/7532/8219** · UNSPSC **53**.
DID `did:web:etzhayyim.com:actor:hataori`. Tier B. Status **R0**.

## Start here

**→ [`docs/operator-quickstart.md`](docs/operator-quickstart.md)** — from a fresh clone
to watching the actor refuse, in five minutes. It also shows you the two places where it
does *not* refuse, which is the thing worth knowing before you trust anything it emits.

```bash
bb test     # 9 tests / 24 assertions, measured 2026-09-01
```

## Why garment work

~65 M garment workers, overwhelmingly women, wage-theft- and disaster-prone (Rana Plaza).
Mid headcount, **maximal exploitation** — and no actor existed. hataori's purpose (gate
**G9**) is to **end** sweatshop labour, not to out-compete it into a worse one: every
finished lot carries fair-labor provenance proving no displaced worker was re-employed
below the Basic-High-Income standard, with the displaced cohort registered for the
tenure-weighted Displacement Dividend (ADR-2606032130, gate **G2**).

## What is here

Five cells forming one chain — `pattern_grading → fabric_cutting → garment_assembly →
quality_inspection → finishing_packing` — declared in `manifest.edn` and `cells/*.edn`,
with five ATProto-style lexicons in `lex/*.edn`.

**Exactly one cell is implemented.** `finishing_packing` (畳 tatami) is the terminal,
constitutional cell: it emits a finished lot *only* together with a fair-labor
provenance record, and refuses on N4 (overproduction), G9 (below-BHI re-employment) and
G2 (unfunded displacement). The other four are declarations. The quickstart derives this
from the tree rather than asking you to believe it.

## Honest

Robotic sewing of limp fabric — seam manipulation — is the hard, industry-unsolved
long-tail problem (gate **G8**). The `nuidono` 縫殿 cell is `:research` maturity, not a
solved skill. R0 means design plus one coded cell: no line, no hardware, no deployment,
`:representative` seed data only.

Three of the nine declared gates (G5 cash≡0, G6 Wellbecoming, G7 outward-gating) ride on
no cell at all, and **G9's default is fail-open** — absent input is attested as
compliant. Both are measured in the quickstart, sections 5 and 4.

## Layout

| path | what |
|---|---|
| `manifest.edn` | actor identity, fleet, 9 gates, 5 non-goals, cell + lex indexes |
| `cells/*.edn` | per-cell declaration: state graph, gates carried, kotoba reads/writes |
| `src/hataori/cells/finishing_packing/` | the one implemented cell, and its tests |
| `src/hataori/methods/test_charter_gates.cljc` | lexicon conformance (G9, G1, QC) |
| `lex/*.edn` | five record lexicons |
| `kotoba/seed.edn`, `data/fleet.kotoba.edn` | `:representative` R0 seed, 5-robot fleet |
| `schema.edn` | generated Datomic/Datascript schema — do not hand-edit |
