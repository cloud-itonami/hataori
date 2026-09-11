# hataori 機織 — operator quickstart

This walks you from a fresh clone to **seeing the actor refuse**, in about five minutes.
Every command below was run against this tree before it was written down.

hataori is at **R0**: design plus one coded cell. There is no line, no hardware, no
deployment. What you can actually exercise is the *constitutional* part — the terminal
cell that decides whether a finished garment lot may be attested at all.

> **On the numbers in this file.** Where an expected output is quoted, it is what the
> command printed on **2026-09-01**. Re-run the command; do not cite the number. Counts
> are derived from `manifest.edn` and the tree, so they move when the actor grows.

## 0. What you need

```bash
bb --version        # Babashka. This is the only runtime the suite runs under today.
```

Babashka is the honest answer and also an awkward one. The workspace-wide rule
(`CLAUDE.md`, ADR-2607173000) retires `bb` as a script host in favour of `nbb`, and
forbids new `bb.edn` / `.sh` files. This repo predates that and still carries
`bb.edn` + `run_tests.sh`.

**`nbb` will not run this suite as-is**, and it is worth knowing why before you try:

- `src/hataori/methods/test_charter_gates.cljk` reads `lex/*.edn` from disk inside a
  `#?(:clj ...)` block. Under `nbb` the `:cljs` branch is taken, `lex` is never
  defined, and the gate tests cannot run.
- `src/hataori/cells/finishing_packing/test_state_machine.cljk` asserts with
  `(thrown-with-msg? clojure.lang.ExceptionInfo ...)`, which is a JVM class name.

The cell **implementation** (`state_machine.cljc`) has no reader conditionals and is
portable. Only the two test namespaces are JVM-shaped. Porting them is real work with a
real gate, and is deliberately not done here — see the bottom of this file.

## 1. Run the suite

```bash
kbb -M:test          # or: ./run_tests.sh — bb.edn's `test` task just shells out to it
```

Measured 2026-09-01: `Ran 9 tests containing 24 assertions. 0 failures, 0 errors.`

`repository-contracts.edn` declares `:repository/test-command "kbb -M:test"`. That claim
holds — it is the command above, and it exits 0.

## 2. Drive the terminal cell

`finishing_packing` (畳 tatami) is the only cell with an implementation. Everything
else in `cells/` is a declaration. Drive it end to end:

```bash
kbb -cp src -e '
(require (quote [hataori.cells.finishing-packing.state-machine :as sm]))
(let [ok (sm/handle {"quantity" 50 "made_to_need_ceiling" 50
                     "offcut_waste_permille" 80
                     "displaced_cohort_id" "hataori-C13-7531-global-2026"
                     "dividend_attested" true})]
  (println "PHASE:" (get-in ok ["cell_state" "phase"]))
  (println "PAYLOAD:" (pr-str (get-in ok ["cell_state" "payload"]))))'
```

The phase reaches `lot_attested` and the payload carries **two** records: the
`finished_lot` and, inseparably, a `fair_labor_provenance`. That pairing is the whole
point of the cell — a lot cannot be emitted without the provenance beside it.

The inputs above come from `kotoba/seed.edn`, the `:representative` R0 seed — with one
addition. The seed supplies the quantity (50), the offcut waste (80‰), the cohort id and
the dividend attestation, but **it carries no `made_to_need_ceiling`**. The `50` above is
mine. Feeding the seed straight through leaves N4 with nothing to compare against; see
section 4.

## 3. Watch it refuse

This is the part worth your five minutes. Three constitutional gates are enforced in
code, and each names itself when it refuses:

```bash
kbb -cp src -e '
(require (quote [hataori.cells.finishing-packing.state-machine :as sm]))
(defn try! [label f]
  (println label
    (try (f) "NO REFUSAL — this is a hole"
      (catch clojure.lang.ExceptionInfo e
        (str "REFUSED " (pr-str (:hataori/violation (ex-data e))) " — " (ex-message e))))))
(try! "N4: " #(sm/handle {"quantity" 200 "made_to_need_ceiling" 100
                          "displaced_cohort_id" "c" "dividend_attested" true}))
(try! "G9: " #(sm/handle {"quantity" 50 "made_to_need_ceiling" 50
                          "displaced_cohort_id" "c" "dividend_attested" true
                          "no_worker_below_bhi" false}))
(try! "G2a:" #(sm/handle {"quantity" 50 "made_to_need_ceiling" 50
                          "displaced_cohort_id" "c" "dividend_attested" false}))
(try! "G2b:" #(sm/handle {"quantity" 50 "made_to_need_ceiling" 50
                          "displaced_cohort_id" "" "dividend_attested" true}))'
```

Measured 2026-09-01 — all four refuse, each with its own `:hataori/violation` tag:

| probe | tag | means |
|---|---|---|
| quantity above the made-to-need ceiling | `:n4` | no fast-fashion overproduction |
| a displaced worker below the Basic-High-Income floor | `:g9` | the gate the actor exists for |
| dividend not attested | `:g2` | no live displacement without a funded cohort |
| cohort id empty | `:g2` | same gate, other half |

**Check the tag, not just that it threw.** A test that only asserts "something was
thrown" counts a refusal for an unrelated reason as a success. The `ex-data` carries
`:hataori/violation`, so pin that.

## 4. Watch it *not* refuse

Run the same cell with the gate inputs simply **absent**, rather than false:

```bash
kbb -cp src -e '
(require (quote [hataori.cells.finishing-packing.state-machine :as sm]))
;; no_worker_below_bhi is never supplied by anyone
(let [r (sm/handle {"quantity" 50 "made_to_need_ceiling" 50
                    "displaced_cohort_id" "c" "dividend_attested" true})]
  (println "G9 absent  ->" (get-in r ["cell_state" "phase"]))
  (println "  provenance:" (pr-str (get-in r ["cell_state" "payload" "fair_labor_provenance"]))))
;; made_to_need_ceiling is never supplied
(let [r (sm/handle {"quantity" 999999 "displaced_cohort_id" "c" "dividend_attested" true})]
  (println "N4 ceiling absent ->" (get-in r ["cell_state" "payload" "finished_lot" "quantity"])))
;; dividend + cohort never supplied
(println "G2 absent  ->"
  (try (do (sm/handle {"quantity" 50 "made_to_need_ceiling" 50}) "attested")
       (catch clojure.lang.ExceptionInfo e (str "REFUSED " (pr-str (:hataori/violation (ex-data e)))))))'
```

Measured 2026-09-01, and this is the most important thing in this document:

- **G9 is fail-open.** With `no_worker_below_bhi` absent, the lot is attested *and the
  emitted provenance record says `"noWorkerBelowBhi" true`*. The cell writes an
  attestation nobody made. The default lives in `defaults` in `state_machine.cljc`
  (`"no_worker_below_bhi" true`) and is read with `(get state "no_worker_below_bhi" true)`.
- **N4's ceiling defaults to the quantity.** With `made_to_need_ceiling` absent, the
  ceiling becomes whatever was produced, so 999,999 units pass as "made to need". This
  one is deliberate and pinned by `test-phase-progression-and-ceiling-default`, but the
  consequence is that N4 blocks nothing unless a caller supplies a ceiling.
- **G2 is fail-closed.** Absent dividend and absent cohort both refuse. This is the
  behaviour the other two should have.

So the actor's headline gate — the one its charter, README, and DID all name as its
reason for existing — is the one that says yes when nobody answered. An operator
should read a `fair_labor_provenance` record as *"no one asserted otherwise"*, not as
*"this was attested"*, until that default is inverted.

This is a finding, not a fix. Fixing it means making the absent case refuse and
pinning that refusal by its `:hataori/violation` tag — a change to the cell plus tests,
which is not what this document is.

## 5. Read the shape

Five cells, one chain. Derive it rather than trusting this paragraph:

```bash
kbb -e '
(require (quote [clojure.edn :as edn]))
(let [cells (for [id (map :cell/id (:actor/cells (edn/read-string (slurp "manifest.edn"))))]
              (let [c (first (edn/read-string (slurp (str "cells/" id ".edn"))))]
                (assoc (edn/read-string (:cell/kotoba c)) :id id :gates (:cell/gates c))))]
  (doseq [c cells]
    (println (format "%-18s reads %-38s writes %s  gates %s"
                     (:id c) (pr-str (:reads c)) (pr-str (:writes c)) (pr-str (:gates c)))))
  (let [w (set (mapcat :writes cells)) r (set (mapcat :reads cells))]
    (println "external inputs: " (pr-str (sort (remove w r))))
    (println "terminal outputs:" (pr-str (sort (remove r w))))))'
```

Measured 2026-09-01: the five `:cell/kotoba` read/write sets chain head to tail —
`pattern_grading → fabric_cutting → garment_assembly → quality_inspection →
finishing_packing` — with three external inputs (`:pattern/id`, `:size/range`,
`:fabric/lot`) and two terminal outputs (`:finished/lot`, `:labor/provenance`).
Nothing dangles in the middle.

To see what is coded versus declared, and which gates ride on a cell at all:

```bash
kbb -e '
(require (quote [clojure.edn :as edn]))
(let [m (edn/read-string (slurp "manifest.edn"))
      cell #(first (edn/read-string (slurp (str "cells/" % ".edn"))))
      ids (map :cell/id (:actor/cells m))
      on (set (mapcat #(:cell/gates (cell %)) ids))]
  (println "cells with an implementation:" (pr-str (vec (filter #(:cell/entry (cell %)) ids))))
  (println "gates declared:" (pr-str (vec (map :gate/id (:actor/gates m)))))
  (println "gates on no cell:" (pr-str (vec (remove on (map :gate/id (:actor/gates m)))))))'
```

Measured 2026-09-01: exactly one cell (`finishing_packing`) has a `:cell/entry`, and
three of the nine declared gates (**G5** cash≡0, **G6** Wellbecoming, **G7**
outward-gating) ride on no cell. That is consistent with R0 — those three are
deployment- and treasury-shaped, not cell-shaped — but it means the manifest's gate
list is not a list of things this tree enforces. Counting what is actually executed:

- **Refused by cell code**: `G2`, `G9`, and `N4` — the last a non-goal rather than a
  gate. These are the three tags in section 3; `grep -o ':hataori/violation :[a-z0-9]*'
  src/hataori/cells/finishing_packing/state_machine.cljk` is the whole list.
- **Checked at the lexicon level**: `G1` and the needle-detect QC enum, by
  `test_charter_gates.cljc` (which also re-pins `G9`).
- **Declared only**: `G3` witness quorum, `G4` Murakumo-only, `G8` sourcing honesty ride
  on cells but nothing in this tree executes them; `G5`, `G6`, `G7` ride on no cell at
  all.

So six of nine gates ride on a cell, but two of the nine are enforced by running code.
The rest are obligations on a future cell wave, not properties of this tree.

The lexicon side is checked separately. `src/hataori/methods/test_charter_gates.cljk`
pins **G9** (`fairLaborProvenance` requires `displacedCohortId` and
`noWorkerBelowBhi`, the latter `const true`), **G1** (`cuttingPlanAttestation.patented`
`const false`), and the needle-detect QC result enum. Note that the lexicon pins
`noWorkerBelowBhi` as a *const true* — the schema cannot express "false"; refusal has
to happen in the cell, which is section 4's problem.

## 6. What this repo claims outward

Two addresses appear throughout the manifest and README. They do not behave the same
way, and neither is checked by the suite:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://etzhayyim.com/actor/hataori/did.json
curl -s -o /dev/null -w '%{http_code}\n' https://hataori.etzhayyim.com/
```

Measured 2026-09-01:

- `did:web:etzhayyim.com:actor:hataori` — the `:actor/did` — **resolves, 200**. Read the
  body before relying on it: `verificationMethod` is `[]`, the one service's
  `serviceEndpoint` is `null`, and `_meta.phase` says `"α P1 scaffold"`. The DID exists;
  it cannot yet verify a signature.
- `hataori.etzhayyim.com` — the `:actor/domain`, and the host inside every `lot/*` id
  the cell emits — **does not resolve at all** (DNS failure, curl exit 6). Lot ids like
  `did:web:hataori.etzhayyim.com/lot/demo-0001` are therefore not dereferenceable today.

Re-measure rather than quoting these. This repo owns neither the DNS nor the published
DID document, so both can change without a commit here.

## 7. Stale claims to know about

`repository-contracts.edn` describes a location this repo no longer has:

- `:repository/name "com-etzhayyim-hataori"` and
  `:repository/west-path "orgs/etzhayyim/com-etzhayyim-hataori"`. The west manifest
  registers this repo as `hataori` at `orgs/cloud-itonami/hataori`; the path in the
  contract file does not exist and the string appears nowhere in `manifest/west.yml`.
  Verify from a superproject checkout with
  `grep -A3 'name: hataori$' manifest/west.yml`.
- `:repository/legacy-path "20-actors/hataori"` is history, and correctly labelled as
  such.

`:repository/runtime :babashka` and `:repository/test-command "kbb -M:test"` are both
accurate — see section 1.

## What this document deliberately does not do

- It does not fix the fail-open G9 default in section 4. That needs a change to the
  cell and a test that pins the refusal by its `:hataori/violation` tag, which is a
  different axis of work.
- It does not port the two test namespaces off the JVM (section 0), which is what
  would let the suite run under `nbb` and retire `bb.edn` / `run_tests.sh`.
- It does not correct `repository-contracts.edn` (section 7).
- It does not check any of the outward addresses in section 6 from the suite. Whether
  they should be checked at all is a real question: they are claims about the world,
  not about this tree, so a red suite would not tell you what to change here.
