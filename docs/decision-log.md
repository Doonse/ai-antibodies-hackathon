# Decision log

Judgment is 40% of the score, and the judges ask where each number came from. This file is the
evidence. One entry per decision that changed what we submit. Append, never rewrite.

Template:

```
## [HH:MM] <decision in one line>
Options considered:
Chose: ... because ...
Evidence: <model + version, score, seed, or the paper/AMASS query>
Confidence: low | medium | high — and what would change our mind
```

Source labels used below:
`TASK p<n>` = the official task description PDF, page n.
`MAP` = the hackathon resource map PDF (first-call timings, 12 Nov).
`RECIPES` / `SKILL` = the Nebius job client docs in the Slack folder.

First measured result of ours: the **13:20** entry (abguard pass rate by CDRH3 mutation count).
Everything above it is reasoning from the sources, labelled as such.

## Authorship

Claude Code (Anthropic) is pre-installed on the team VM with credits from the organisers, and we
use it. So that the Judgment score is assessed on the right basis, authorship is explicit:

- The **13:20** and **13:25** entries, `docs/abguard.md`, and everything under `pipeline/` were
  written by **Claude Code, working as W1 under Mark's direction**. Where those entries say "I",
  it is the agent, not a team member. The measurements in them are real — run on the VM against
  the organisers' own `abguard` and our own batch 1 — and every number names the file it came from
  and a command that reproduces it.
- Entries before 13:20 are the team's, reasoned from the task PDF and the Slack material.
- The agent's two errors so far are in the record on purpose: a gate script that exited 0 on
  failure (caught and fixed before use), and a synthetic-probe recommendation that real batch-1
  data overturned within the hour (13:20, superseded by 13:25). Neither was caught by review —
  one by testing, one by better data. **Treat agent output as a strong draft that needs checking,
  not as a result.** `python3 pipeline/status.py --how` and `bash pipeline/filter/gate.sh` exist
  so any claim here can be re-derived in one command.
- Decisions deliberately **not** made by the agent, because they are ours: whether to enter the
  de novo track, what to generate next, and launching anything on the shared H100.

---

## [10:55] Enter both tracks, but build the optimisation track first

Timestamps on the first three entries are when the log was reseeded against the task PDF, not
when each point was argued.

Options considered:
1. De novo only — design a new VHH/scFv/Fab to the trastuzumab epitope.
2. Optimisation only — point mutations on trastuzumab.
3. Both, split effort.

Chose: **both tracks, with optimisation first and de novo running in parallel on W3's second
owner**, because:
- The prize fund (€3000) is split **equally** between the two tracks, €800 / €500 / €200 per
  track, so a track with fewer strong teams is worth the same as the crowded one (`TASK p9`).
- Submissions in the two tracks are **evaluated independently**, and we may submit any
  proportion of each (`TASK p8`).
- We do not choose the track. `abguard` classifies each design by identity to the nearest known
  antibody, and **a design is never moved between tracks** — miss your track's criteria and you
  are rejected, not reclassified (`TASK p3`). So "which track" is a consequence of how we
  generate, which means we have to generate deliberately on both.
- Optimisation first because the HER2-specific assets are optimisation-shaped: two Apheris
  weights fine-tuned on the trastuzumab FACS screen that **only redesign CDRH3**, plus 524,346
  measured CDRH3 variants and ~700 optimisation designs with 190 SPR affinities (`TASK p4, p5`).
  De novo has no comparable measured prior for this epitope.

Evidence: `TASK p3` (classification flowchart), `TASK p4` (supporting data counts), `TASK p5`
(weight table), `TASK p8–9` (independent evaluation, prize split).

Confidence: medium. What would change our mind: if `abguard` rejects most of our optimisation
output on novelty (see next entry), de novo becomes the cheaper route to eligible designs and we
move Abdelrahman onto it.

---

## [10:55] Primary generation weight is `her2-novel-2026-11-09`, with `her2-2026-11-07` balanced as the diversity arm

The tension: on the trastuzumab CDRH3 task, the weights that hit known binders are the weights
that fail the novelty filter. The organisers' own table (`TASK p5–6`):

| Weight and setting | Known binders | Pass novelty filter | Interface error (Å) |
|---|---|---|---|
| `antibody-2026-10-30` (not fine-tuned) | 0% | 100% | 8.2 |
| `her2-2026-11-07`, conservative | 21% | 0% | 9.4 |
| `her2-2026-11-07`, balanced | 21% | 6% | 9.1 |
| `her2-2026-11-07`, exploratory | 0.3% | 74% | 9.1 |
| `her2-novel-2026-11-09` | 4% | 86% | 6.8 |

Options considered:
1. `her2-2026-11-07` conservative — best shot at real binders, **0% pass novelty**: every design
   is rejected before it is measured. Unusable as a submission source.
2. `antibody-2026-10-30` — everything passes novelty, 0% known binders, worst interface error of
   the HER2-relevant rows at 8.2 Å.
3. `her2-novel-2026-11-09` — 86% pass novelty and the **best interface error in the table,
   6.8 Å**, while still recovering 4% known binders.

Chose: **`her2-novel-2026-11-09` as the primary**, because it is the only row that is good on both
axes at once, and **`her2-2026-11-07` balanced (`--sequence-tsr-k 2`) as a second, smaller arm**
for loop diversity, accepting that ~94% of that arm is discarded. `antibody-2026-10-30` is held in
reserve for anything that needs a CDR other than CDRH3 — the HER2 weights redesign CDRH3 only
(`TASK p6`).

Operational consequence, and the reason this is a decision and not a note: **a known binder fails
the novelty filter, so we filter before we rank** (`TASK p6`). Novelty check is step one of the
scoring pipeline, not a final sanity check. Ranking a pool that still contains known binders
wastes the compute we do not have.

Evidence: `TASK p5–6` weight table and the two sentences after it.

Confidence: high on the ordering (it is the organisers' own measurement), low on the absolute
rates for our settings — the table is their run, not ours. What would change our mind: our own
abguard pass rate on batch 1. If `her2-novel-2026-11-09` passes well below 86% for us, we shift
volume to the exploratory setting (74% pass).

---

## [10:55] Rank on `min_ipae`, `iptm` and `binder_plddt`, from a template-taking refold

Options considered:
1. Use RFD4-Proteina `refold-score`'s own pass criteria as the gate. **Not available**: for the
   HER2 exercise "the pass column stays empty" (`TASK p6`).
2. Rank on a single confidence number.
3. Rank on the three scores the organisers point at, from a refold done with a model that takes a
   template.

Chose: option 3. Pipeline is `refold-prepare` → fold with a Predict Structure model → `refold-score`,
and we rank on **`min_ipae`, `iptm`, `binder_plddt`** (`TASK p5–6`).

Two mechanics that follow from the sources and are easy to get wrong:

- **Refold with a template-taking model.** Template support per `TASK p5`: OpenFold3 (OpenBind,
  preview 2) yes, Protenix v2/v1 yes, Boltz-2 yes, Mol-1 base yes, Boltz-1 **no**, ESMFold2 **no**.
  ESMFold2 works but folds the target unguided, which is an unfair check for a binder (`TASK p6`).
  So ESMFold2 is a second opinion, never the ranking fold.
- **Boltz2 on the Nebius platform returns no ipTM** (`RECIPES` §2: the 1ZVY pair gave confidence
  0.77 and pTM 0.57, with no ipTM in the output). Any `iptm` we report comes from `refold-score`
  or from OpenFold3 — not from a Boltz2 run. If we need an interface number out of Boltz2 we
  count cross-chain contacts in the returned mmCIF and label it as that.

Standing caveat, carried from `docs/hackathon-brief.md`: none of pLDDT, PAE, ipTM or ipSAE is a
KD. These three scores order candidates; they do not predict the SPR number we are judged on.
Thermostability in particular is not in this triple at all and needs its own evidence per design.

Evidence: `TASK p5` (template column), `TASK p6` (empty pass column, ESMFold2 caveat, the three
score names), `RECIPES` §2 (no ipTM from Boltz2).

Confidence: high — this is the organisers' stated ranking basis for this exercise. What would
change our mind: a pass column appearing in an actual `refold-score` output, or an organiser
saying otherwise on the day.

---

## [13:20] Generate CDRH3 at 5 mutations, not 3 — measured on abguard, not inferred

This is the first entry backed by a result of ours rather than by the task PDF.

Context: `abguard` was reported missing from the VM at ~13:00, which would have put the
submission at risk. It was never missing — it is a `uv` tool at `~/.local/bin/abguard`, and
`~/.local/bin` is on `PATH` in a **login shell only**, so every non-interactive check said
"command not found". Full write-up in [`abguard.md`](abguard.md).

Options considered: build to the ">2 mutations" bar the task PDF implies (3 mutations) and spend
the H100 budget on volume; or measure the real pass rate first and set the target from it.

Chose: **design CDRH3 at 5 mutations from trastuzumab**, because 3 is necessary but nowhere near
sufficient, and we can now prove it.

Evidence: `abguard team-validate` against its bundled reference (610,109 CDRH3s across
`trastuzumab`, `trastuzumab_her2_spr_all_768` and `tz_her2_facs_3classes_524346`). 100 synthetic
trastuzumab CDRH3 variants, 25 at each mutation count, seed 20261114, cysteine excluded from the
substitution alphabet:

| CDRH3 mutations from trastuzumab | Passed abguard |
|---|---|
| 3 | 3/25 (12%) |
| 4 | 16/25 (64%) |
| 5 | 23/25 (92%) |
| 6 | 23/25 (92%) |

Reproduce: `bash pipeline/filter/gate.sh designs/probe.csv` on the VM.

Why 3 fails so often: the mutation distance is counted against **every** provided loop, not just
the parent. A design 3 mutations from trastuzumab typically lands 1–2 mutations from some *other*
measured loop in the 524k FACS screen, and is rejected on that. The screen densely covers the
neighbourhood around trastuzumab's CDRH3 — which is exactly the neighbourhood the HER2 fine-tuned
weights were trained to generate in.

**This is the mechanism behind the PDF's novelty table** (`her2-2026-11-07` conservative: 0% pass,
balanced: 6%). We had read those numbers as a property of the weights; they are a property of the
reference set the weights were trained on. That strengthens the 10:55 decision to lead with
`her2-novel-2026-11-09` — it was trained on high/low binder *pairs* rather than to reproduce the
screen, which is why it reaches 86% — and it is now an argued conclusion rather than a bet on a
table we did not generate.

Two corrections to our own docs fall out of this, both now fixed in `README.md` and `workplan.md`:

- **There is no "< 70% CDRH3 identity to trastuzumab" requirement.** `abguard` never computes a
  CDRH3 identity threshold; the ≥70% figure only classifies a design as a trastuzumab variant
  rather than de novo. Our variants sit at 98–99% whole-sequence identity and are valid
  optimisations. Designing to the 70% reading would have meant rewriting a third of the loop for
  nothing, and would have cost us the optimisation track.
- **The 250 aa limit is per chain**, not VH + VL combined.

Confidence: high on the ordering and on the mechanism — it is the organisers' own tool and its
own reference, run by us. Medium on the absolute rates: the probe is random substitution, and
RFD4-Proteina's loops are not random, so our real pass rate will differ. What would change our
mind: the measured pass rate on generation batch 1. If `her2-novel-2026-11-09` at 5 mutations
comes in well under 92%, we raise the target to 6 rather than lower the mutation count.

Consequence for the plan: `pipeline/filter/novelty.py` (Abdelrahman, 13:15) is cancelled — it
would have reimplemented this against 525,063 loops and to the wrong threshold. The gate is
`pipeline/filter/gate.sh`, and `rank.py` joins `reports/novelty-report.csv` on `sequence_id`.

---

## [13:25] Supersedes 13:20 — target ≥ 6 CDRH3 mutations. 3–4 is a dead zone, and 5 is marginal

Batch 1 finished enough designs to measure the gate on real RFD4-Proteina output instead of my
synthetic probe. **The probe was wrong in the direction that mattered**, so this entry replaces the
13:20 recommendation. Keeping both because the error is informative.

Run: `~/mark/runs/b1-novel-s1`, `her2-novel-2026-11-09`, seed 1, 129 designs extracted mid-run
(`designs/raw/batch1-b1-novel-s1.csv`). Gated with `bash pipeline/filter/gate.sh`.

**91/129 eligible — 70.5%.** Zero hard errors: no format, length, alphabet, cysteine or `immunum`
failures in 129 designs. **All 38 rejections are novelty**, and the breakdown is the finding:

| CDRH3 mutations from trastuzumab | Passed |
|---|---|
| 2 | 0/3 |
| 3 | **0/5** |
| 4 | **0/7** |
| 5 | 4/9 (44%) |
| 6 | 15/18 (83%) |
| 7 | 44/56 (79%) |
| 8 | 25/28 (89%) |
| 9 | 3/3 |

**35 of the 38 rejections were ≥3 mutations from trastuzumab** and were killed by proximity to a
*different* measured loop. Only 3 failed on trastuzumab distance.

So the 13:20 advice to "design to 5" was the wrong lever, and the synthetic numbers (3 → 12%,
4 → 64%, 5 → 92%) do not transfer. Why the probe over-estimated: it substituted residues at random,
which scatters designs across the whole sequence space, whereas RFD4 draws loops toward the same
high-affinity motifs the FACS screen already explored. **At 3–4 mutations the real generator lands
inside the densest part of the reference every time — 0/12.** Random mutants do not.

Chose: **do not steer generation toward fewer mutations.** `her2-novel-2026-11-09` already produces
6–8 mutations unprompted (102/129 at ≥6) and that is precisely why it passes; that distribution is
the weight's value, not something to correct. Treat **≥6 as the floor**, 5 as marginal, 3–4 as
unusable. Over-generate ~1.4× the slots we need and let the gate decide.

Confidence: high — our own designs, our own run, the organisers' own tool. Caveats recorded: one
seed, one weight, and the run was still generating when sampled, so this is 129 of a larger batch.
What would change our mind: a second seed landing materially off 70%.

Two smaller things from the same run:

- **91 survivors contain only 85 unique heavy chains** — 6 duplicate pairs inside one batch.
  `abguard competition-validate` rejects sequences shared *between* teams, so duplicates are pure
  waste in a 20-slot list. Dedupe on `heavy_sequence` before ranking.
- 744 LAP liability occurrences across 129 designs (~5.8 each), none of them hard errors.
  `reports/batch1-liability.json` is the per-design detail; W4 should rank on it, since liabilities
  are a developability signal and developability is not one of the three judged axes but does
  decide whether a design expresses at all.

Our 70.5% sits below the PDF's 86% for this weight. Not alarming at n=129 on one seed, but it means
the 20-design list needs ~29 generated candidates minimum, and more if we also dedupe.

---

## [14:05] NetSolP is a floor and a tiebreak, not a filter — calibrated against 190 measured binders

The organisers name NetSolP as one of four developability tools (`TASK p7`), so we ran it. The
question was what threshold to filter on. The answer is that **no threshold is defensible**, and we
can show that with the organisers' own SPR data.

Options considered: pick a plausible cut (0.5 is the natural midpoint of a probability); use the
distribution of our own designs and cut at some percentile; or calibrate against molecules already
known to express and bind, and only reject outside that range.

Chose: **calibrate.** Scored the 200 batch-1 designs and the 190 SPR-quantified trastuzumab
variants — molecules that demonstrably express *and* bind HER2.

| | n | min | p5 | median | p95 | max | sd |
|---|---|---|---|---|---|---|---|
| our designs | 200 | 0.4958 | 0.5110 | 0.5302 | 0.5489 | 0.5489 | 0.0113 |
| **measured binders** | 190 | 0.4960 | 0.5179 | 0.5426 | 0.5489 | 0.5489 | 0.0103 |

The distributions are the same to within noise, and the entire dynamic range is 0.05 wide. So any
cut that removes a meaningful share of designs removes a comparable share of **known good
molecules**: 0.515 drops 12% of designs and 4.2% of measured binders; 0.530 drops 47% of designs
and 14.2% of measured binders. And it carries no affinity signal —
**Pearson r(netsolp_min, KD nM) = −0.04** over the 190 binders.

This is not a failure of NetSolP. On diverse proteins it separates properly (SIV integrase 0.283,
human leptin 0.460, HhaI methyltransferase 0.765). It simply cannot resolve CDRH3 point variants of
one antibody: every design shares the trastuzumab framework and differs in ~10 of ~230 residues.
Our light chain is fixed trastuzumab VL and scores a constant 0.5489, so `netsolp_solubility_min`
is effectively the heavy-chain score.

So the rule we apply: **a hard floor at 0.4960, the minimum over the measured set** — a design
scoring below every molecule known to work is outside the supported range. On batch 1 that rejects
exactly one design, `her2_cdrh3_124` (0.4958). Above the floor, `netsolp_solubility_min` is a
ranking tiebreak only, never a gate.

Evidence: `reports/netsolp-calibration-spr190.csv` (the 190 scored binders),
`reports/batch1-solubility.csv`, `pipeline/filter/solubility.py` (the calibration is in its
docstring). NetSolP 1.0 quantized ONNX, Distilled model, installed on the VM at `~/tools/netsolp`
by a teammate; 3 designs score in 4.5 s on CPU, so it never competes with the GPU.

Confidence: high that no higher cut is defensible — it is a direct comparison against measured
binders on the organisers' own data. Medium on the floor's exact value: it is one dataset's minimum,
so it will move slightly with a different reference set. What would change our mind: a reference set
of measured NON-expressing variants. We have binders only, so we can bound the bottom but cannot
calibrate discrimination. If the organisers release the non-expressing arm of the FACS screen, this
becomes a real filter.

Caveat for Execution: `~/tools/netsolp` lives on the VM, not in this repo, and **the VM is wiped
tonight**. The calibration CSV and the scored outputs are committed; the tool itself is not. Either
vendor the installer or record the exact provenance before 17:40.

## [Added on re-analysis] One authoritative gate run over all 400 designs — and `sequence_id` is only meaningful inside one run

Context: the repo held one batch (200) while a second finished batch (200) sat ungated in
`~/mark/runs/b2-explor-s2-fixed`, and two different counts of "passed abguard" were in
circulation — **140** and **14**. Both were correct and measured different things, which is
exactly the failure mode a single gate run exists to prevent.

Chose: **re-extract both batches with batch-prefixed ids and gate all 400 in one run.**
`reports/all-batches-eligible.csv` is now the only eligible set anyone quotes.

### The two counts, reconciled

- **140** = batch-1 yield, 140/200 of the generated designs.
- **14** = how much of a hand-picked shortlist of 20 survived (`~/josef/submission_b1_valid.csv`).
  Reproduced exactly: `abguard team-validate --drop` gives 14/20, same six rejections.

Neither is the submission number. The funnel is 400 generated → 210 eligible → 191 unique.

### The finding that forced the id prefix

`her2_cdrh3_12` and `her2_cdrh3_17` passed in the shortlist run and failed in the batch run.
They are **different molecules with the same id**:

| id | shortlist CDRH3 | batch-1 CDRH3 |
|---|---|---|
| `her2_cdrh3_12` | `SRDLSLSVYAFDY` (passes, 3+ mutations) | `SRDLSFSVYAFDY` (fails, 2 from `SRYLFFSVYAFDY`) |
| `her2_cdrh3_17` | `SRSLDGYYNYFDY` (passes) | `SRYSSGTDYAFDY` (fails, 2 from `SRYSSTDFYAFDY`) |

Cause: the shortlist came from a **third, separate generation run** —
`~/josef/runs/her2-b1/design`, `her2-novel`, seed 1, 100 replicates — whose samples are also
named `her2_cdrh3_0..99`. RFD4-Proteina numbers samples per run, so the same id names a
different molecule in every run. Same weight and same seed did **not** reproduce the same
molecules at a different `--num-replicates`.

Consequence: a bare `her2_cdrh3_N` is not a design identifier and must never be joined on.
Every id now carries its run (`b1-novel-s1-her2_cdrh3_12`). Pre-prefix files are in
`designs/raw/superseded/` and `reports/superseded/`, kept for audit, excluded from counts.

### Measured, all 400, one run, bundled reference

| | b1 `her2-novel-2026-11-09` s1 | b2 `her2-2026-11-07` exploratory s2 | all |
|---|---|---|---|
| generated | 200 | 200 | 400 |
| eligible | **140 (70.0%)** | **70 (35.0%)** | **210 (52.5%)** |
| unique heavy chains (generated) | 170 | 200 | 370 |

- 187 novelty rejections: **60 in b1, 127 in b2.** The exploratory arm is twice as diverse by
  unique-chain count (200/200 vs 170/200) and half as eligible. Diversity it buys lands *near
  other measured loops in the 524k screen*, not away from them.
- 3 unpaired-cysteine hard errors — the first hard errors we have seen; b1 alone had none.
- 45 repeated heavy/light pairs, all within b1. Dedupe on `heavy_sequence`, not on id.
- 2 designs below the NetSolP floor (`b1-novel-s1-her2_cdrh3_124`, `b2-explor-s2-her2_cdrh3_89`).

Still open: 191 unique eligible designs, **0 scored in the repo.** Structural scores exist for
40 b1 designs (`~/mark/runs/scored-of3/refold_metrics.csv`, OpenFold3) and 20 (Josef's, Boltz-2)
and are not joined in. The two predictors disagree sharply on the same designs — Boltz-2 ipTM
0.86–0.90 against OpenFold3 0.57–0.60 — which is a recorded uncertainty, not a tiebreak.
