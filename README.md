# Recovered Trace Links — SOTA baselines (ArDoCo dataset)

Persisted, normalized recovered trace links for the doc-to-code and doc-to-model
traceability tasks, collected from the SOTA replication packages so they survive
`mvn clean` and are directly comparable.

> **Filling the paper?** See the cross-system results index:
> `transarc-emp/reports/RESULTS_INDEX.md` — per-system locations + RQ1/RQ2 numbers.

## Tasks / layout

| Folder | ArDoCo task | Meaning | target_id is |
|--------|-------------|---------|--------------|
| `doc-code/`  | SAD-Code | architecture-documentation sentence → source-code file | code file path |
| `model-doc/` | SAD-SAM  | architecture-documentation sentence → architecture-model element | PCM model element id (`_...`) |

Full ArDoCo dataset = 5 projects: `bigbluebutton`, `jabref`, `mediastore`, `teammates`, `teastore`.

## File format (normalized)

CSV with header `sentence_id,target_id`; one recovered link per row. Row count =
predicted-positive count (TP + FP) for that run.

Naming: `<approach>-<project>[-<model>].csv`.

## Sources & provenance

| Approach | Task(s) | Model | Origin | Note |
|----------|---------|-------|--------|------|
| **LiSSA**    | doc-code, model-doc | gpt-4o-mini-2024-07-18 **and** gpt-5-mini (flex), seed 133742243 | `lissa-replication/results/tracelinks/{d2c,d2m}/` (committed) | two model variants each |
| **Artemis**  | doc-code, model-doc | gpt-5.4 (flex) | TAAS25 `…/tlr/tests-tlr/target/raw-tracelinks/*.tsv` (build output) | ArTEMiS = SAD-SAM via NER; doc-code is ArTEMiS+TransArC |
| **TransArC** | doc-code | none (deterministic) | **canonical** `transarc-emp/results/*/sad-code/sadCodeTlr_*.csv` | classic SAD→SAM→Code; ICSE24 self-contained run reproduces 4/5 — see caveat 4 |
| **SWATTR**   | model-doc | none (deterministic) | **canonical** `transarc-emp/results/*/sad-sam/sadSamTlr_*.csv` | classic heuristic SAD-SAM; ICSE24 self-contained run reproduces 4/5 — see caveat 4 |

`manifest.csv` lists every persisted file with its row count.

## Coverage status (36 files)

- **model-doc (SAD-SAM):** LiSSA 5/5 ✅, Artemis 5/5 ✅, SWATTR 5/5 ✅.
- **doc-code (SAD-Code):** Artemis 5/5 ✅, TransArC 5/5 ✅, LiSSA **3/5** (⚠️ missing `jabref`, `teammates` — gold standards exist; LiSSA needs an OpenAI key to generate).

## Normalization

All files are canonicalized by `normalize.py` (idempotent): header `sentence_id,target_id`,
integer sentence id, rows deduplicated and sorted by `(sentence_id, target_id)`.

**Sentence indexing is uniform — verified, not assumed.** Every file's sentence ids are
**1-based and aligned to the ArDoCo gold standard**. This was confirmed empirically per
file by testing shifts {−1, 0, +1} against the gold standard and taking the one that
maximizes gold overlap (TP): shift **0** wins decisively for every Artemis, LiSSA,
SWATTR, and TransArC file, both tasks. (Internally `SentenceEntity#getSentenceNumber()`
is 0-based and the ArDoCo dumps add +1; the NER occurrence used by Artemis model-doc is
already 1-based — so all approaches land on the same 1-based convention. No shifting is
applied.)

## Caveats before computing metrics

1. **Model mismatch.** LiSSA has two model runs (gpt-4o-mini, gpt-5-mini); Artemis is a
   single gpt-5.4 run; SWATTR/TransArC are deterministic. Pair comparably
   (e.g. LiSSA gpt-5-mini ↔ Artemis gpt-5.4).
2. **IDs align across approaches:** model-doc uses the same PCM element ids; doc-code uses
   the same code file paths.
3. **doc-code gold granularity.** The `sad-code` gold standard lists *directory /
   compilation-unit prefixes* (e.g. `…/registry/`), while recovered links are full file
   paths. Match by prefix (and enroll the gold to compilation units) when scoring — a
   raw row-equality intersection will undercount.
4. **SWATTR/TransArC bigbluebutton provenance (decided: canonical).** Two runs exist;
   they are **identical for 4/5 projects** and differ only on bbb. The folder holds the
   **canonical** `transarc-emp/results` run (bbb SWATTR link F1 `.79`, TransArC file F1
   `.83`; macro ~`.80`; matches the paper headline & `reports/*_rq1_*.csv`). The
   **self-contained ICSE24 run** (`run-transarc-icse24.sh`, reproducible from Central with
   no unpublished deps) scores bbb lower (SWATTR `.29`, TransArC `.35`; macro ~`.70`) — it
   recovers different bbb links, *not* an id/model-version mismatch. ICSE24 bbb variants
   are kept in `baseline-repos/transarc-icse24/.../target/raw-links/` if needed.

## Evaluation (mini-src)

Scored with the paper's own `transarc-emp/mini-src/metrics.py` (stdlib, auto-detects the
`recovered-links` dialect). Panels saved in `eval/`. Macro averages over the 5 projects:

| Task | Approach | F1 | coverage | noise |
|------|----------|----|----------|-------|
| model-doc (link F1) | artemis (gpt-5.4) | **.836** | .79 | .07 |
| model-doc | swattr | **.799** | .79 | .14 |
| model-doc | lissa (gpt-5-mini) | .425 | .84 | .60 |
| model-doc | lissa (gpt-4o-mini) | .280 | .89 | .73 |
| doc-code (file F1) | artemis (gpt-5.4) | **.849** | .71 | .06 |
| doc-code | transarc | **.803** | .75 | .13 |
| doc-code | lissa (gpt-5-mini, 3 proj) | ~.20 | — | — |

**Validation (numbers are correct):**
- SWATTR/TransArC reproduce the paper headline (~.80) **and** the ArDoCo built-in eval
  (`TraceLinkEvaluationIT#evaluateSadSamTlrIT`/`#evaluateSadSamCodeTlrIT`) passes **10/10**
  against the packages' published `ExpectedResults`.
- LiSSA reproduces its replication's reported per-project F1 **exactly** (sad-sam: MS .307,
  TS .258, TM .106, BBB .173, JR .554; avg .280 — cf. `lissa-replication/results/COMPARISON.md`).

## Regeneration

Artemis raw dumps (needs `OPENAI_API_KEY`):
```
cd Replication-Package-TAAS25_.../Replication-Package-TAAS25/tlr
mvn -pl tests-tlr -am test-compile failsafe:integration-test -Dit.test=RawTraceLinksIT
```

SWATTR + TransArC baselines (deterministic, no API key) — from the **self-contained
ICSE24 package** (`baseline-repos/transarc-icse24`, no NER, no unpublished snapshot deps).
The TAAS25 tooling can't build these here: it needs unpublished dev snapshots
(`io.github.ardoco:named-architecture-entity-recognition:1.0.0-SNAPSHOT`, only `2.0.0`
on Central). The ICSE24 package bundles its own ArDoCo core, so it builds from Central.
```
bash recovered-links/run-transarc-icse24.sh
# builds ardoco+arcotl reactor (rev 0.18.0-ICSE24-SNAPSHOT), runs RawLinkDumpIT
# outputs: …/ardoco+arcotl/tests/tests-tlr/target/raw-links/{SWATTR-*-sad-sam.csv, TRANSARC-*-sad-code.csv}
```
Then fold into the folder: SWATTR — swap columns (`modelElementId,sentence` →
`sentence_id,target_id`) into `model-doc/`; TransArC — `sentence,code` as-is into
`doc-code/`; finally run `python3 normalize.py`. Dump test:
`baseline-repos/transarc-icse24/ardoco+arcotl/tests/tests-tlr/src/test/java/.../integration/RawLinkDumpIT.java`.
