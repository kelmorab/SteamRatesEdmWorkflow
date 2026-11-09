# Claude working notes – SteamRatesEdmWorkflow modernization

Human-facing overview with flow charts: `docs/WORKFLOW.md` (keep both in sync).
Project brief from the user: `initial_notes.txt`. Example recipe: `README.md`.

## Project goals (from the user, 2026-11-09)
- Refactor/rewrite for readability, with good comments everywhere.
- **One config file** per measurement, and **one working directory per measurement** (parallel runs from one checkout).
- **L1 rates** in addition to HLT rates.
- Step 1 (`Prod/`, own ntuple production) will probably be replaced by a **common L1+HLT ntuple format** made in another repo (details pending). Focus is on step 2 (counting).
- Outputs in **JSON** (later: a website to store and compare measurements; maybe later also to launch measurements from a web server). No lxplus-only or interactive assumptions in the core.
- Correlation / overlap studies ("maps") should be fast, discoverable, with nice and customizable plots.
- Rates of individual **modules/filters** inside HLT paths.
- **Share code between data and MC** as much as possible. **MC will be the main future use case** (HL-LHC / Phase-2).
- Fix the **duplicate files in input file lists** problem.

## Environment
- Repo: `/eos/home-k/kelmorab/STEAM/hackathon/CMSSW_16_0_6_patch1/src/SteamRatesEdmWorkflow` (git; remote `cms-steam/SteamRatesEdmWorkflow`).
- `cmsenv` in `CMSSW_16_0_6_patch1/src` gives python3 with ROOT 6.36, uproot 5.6.6, awkward 2.8.9, numpy 1.26, pandas 2.3.3, matplotlib 3.9.4, FWLite.
- Node 16 at `/usr/bin/node`. The npm cache on AFS fails (EXDEV); use `npm_config_cache=<scratch>`. mermaid-cli 10.9.1 works with `-p` puppeteer config `{"args":["--no-sandbox"]}`.

## Example data
- Step-1 ntuples: `/eos/home-k/kelmorab/STEAM/rates_2026/402360_GRunV59/hlt_*.root`
  - 1136 files, about 5k events and 1 LS each (8 EphemeralHLTPhysics PDs × 142 LS), 2.7 GB in total.
  - Event content is **only** `edm::TriggerResults "TriggerResults" "" "MYHLT"`.
- Old step-2 output (nomaps): `/afs/cern.ch/work/k/kelmorab/STEAM/forRates/rates_2026C/CMSSW_16_0_6/src/SteamRatesEdmWorkflow/Rates/402360_HLTV1p2/Results/Data/`
  - N_LS 1136, 5,721,135 events, hlt_ps 480 (60 × 8), lumi 2.2 → 2.2, scale 0.018127.
  - A counting job takes about 1 min.
- Other runs: `/eos/home-k/kelmorab/STEAM/rates_2026/` (402360_GRunV51/V62, HI_404523_*).
- A new baseline measurement (run 403663, recipe in README) is being run by the user. **Wait for the user to say it's done**, then use it as the reference for regression tests.

## Current architecture (short)
- **Prod:**
  - `hltGetConfiguration` → `hlt.py` → `edmConfigDump` → `hlt_config.py`
  - `run_steamflow_cfg.py` (imports `hlt_config`; hardcoded `from list_cff_Run3 import inputFileNames`; keeps only TriggerResults::MYHLT)
  - `cmsCondorData.py` → `Jobs/`, `sub_total.jobb` → `hlt_i.root`
- **Rates:**
  - `config_makeCondorJobsData.py` → `condorScriptForRatesData.py`, which does `rm -rf Jobs`, overwrites `filesInputData.py` via `make_ratesFilesInputData.py`, and writes `Jobs/sub_raw/sub_i.sh`, `Jobs/Job_i/`, `sub_total.jobb` (which wipes `Results/Data/Raw`)
  - per job: `triggerCountsFromTriggerResults.py` → `Jobs/output.*.i.csv` → `handleFileTransfer.py` → `Results/Data/Raw/<Cat>/`
  - `config_mergeOutputsData.py` → `mergeOutputs.py` (scale = lumi_t/lumi_in × hlt_ps / (N_LS × 23.31 s)) → `Results/Data/output.*.csv` (→ `Draw.py` with figures)
- **Maps:**
  - `STEAM_maps/make_MenuHLT.py` → hltConfigFromDB → hltDumpStream (cx_Oracle) → csv_to_tsv → `makeMaps.py` → `Menu_HLT.py`
  - copied by hand into `Rates/`
- **MC:** near-copies (`cmsCondorMC.py`, `*MC.py`, `prepareMergeOutputsMC.py`). The counter detects MC when `-j` isn't .txt/.json. Normalization: lumi × σ[pb] × 0.01 / N_events.

## Known bugs / pitfalls (file:line as of commit 37fe714)
1. `Rates/triggerCountsFromTriggerResults.py:28` `check_json` re-reads and parses the JSON **per event**.
2. `triggerCountsFromTriggerResults.py:368-369` `tnames.append` per event → unbounded list; `in tnames` at :457 gets slower and slower.
3. `Rates/aux.py:68-111` `belongsToPAG` / `physicsStreamOK` / `scoutingStreamOK` / `parkingStreamOK`: linear scan over ~750 map keys per fired path per event. These (plus `x in list(d.keys())`) are why maps mode is slow.
4. `rstrip("0123456789")` used for version stripping mangles names ending in digits (`Dataset_AlCaP0` → `Dataset_AlCaP`; ScoutingPF0 and ScoutingPF1 collide). Use `re.sub(r"_v\d+$", "_v", name)`.
5. "Pure" reference set: fired `HLT_*` paths (nomaps) vs. fired physics-stream paths (maps). `triggerstoexclude.py` paths never get a pure count.
6. `allmaps` crashes: `newDatasetMap` is in no `Menu_HLT.py`. `compareWBM.py` needs nonexistent `psL1TMap`/`psHLTMap`. `STEAM_maps/adjustPS.py` imports nonexistent `outputMaps`.
7. **Duplicate input files:** `Prod/createFileList.py:37-43` runs one DAS query per LS and appends without dedupe. A file covering k LS is listed k times. Commit 68d5b60 deduped the EHP lists by hand. `Rates/createFileList.py` is a stale copy.
8. **N_LS normalization:**
   - N_LS is counted per job and summed in `mergeOutputs.py:63-75` with no cross-job dedupe.
   - Correct only if each (PD, LS) is in exactly one file, **and** hlt_ps includes the number of parallel PDs (×8). Duplicated files double counts and N_LS; a split LS lowers the rate.
   - Need: per-job (run, LS) sets, dedupe at merge, and an explicit n_datasets parameter.
9. `aux.makeListsOfRawOutputs` (aux.py:172) bad-job matching uses substrings of the file number.
10. Shared mutable state: `Jobs/`, `filesInputData.py`, `Results/`, `sub_total.jobb`, `condor_cluster.sub`, all relative to cwd.
11. Hardcoded: LS length 23.31 (mergeOutputs.py:81), process names MYHLT/HLT (:178/:180), metBx/muonBx histos (:76), `PAGlist`, `datasets_for_corr` (Run-2 names), stream-classification string rules in aux.py.
12. CSV columns labelled "Rate (Hz)" hold raw counts in per-job outputs.
13. Leftovers: `Rates/#mergeOutputs.py#`, `aux.pyc`, `STEAM_maps/__pycache__`, `Old_Menu_HLT.py`, duplicate `ignore.py`, stale `filesInput*.py`, `list_cff.py` at the repo root, `Rates/list_cff_SkimD.py`, `make_files2.py`.
14. No L1 info in the ntuples → L1 rates need new ntuple content.
15. Opportunity: `HLTPathStatus::index()` in TriggerResults = last module run in a path. With the menu's per-path module list, this gives per-filter rates from the existing ntuples.

## Design direction (agreed principles, details open)
- One pipeline; data/MC differ only by (a) event selection (lumi JSON or none) and (b) a pluggable normalization (data: lumi × PS / (N_LS × t_LS); MC: σ × eff × L / N or Σw). MC is first-class: many samples, combine pT bins, weights.
- Input-reader layer separate from the counting core (current EDM TriggerResults via FWLite/uproot now, common L1+HLT ntuple later).
  - **The current `hlt_N.root` can be read without FWLite** (verified 2026-11-09, scratchpad `edm_uproot3.py`). uproot can't deserialize `edm::TriggerResults` (memberwise `vector<HLTPathStatus>`), but reading the branch as raw bytes (`AsJagged(AsDtype("u1"))`) works:
    - 24-byte header, with the path count as a big-endian `u4` at bytes 20–24;
    - then N big-endian `u2` status words: accept = `(s & 3) == 1`, last module index = `s >> 2`;
    - the 16-byte psetid is at bytes `[-26:-10]`.
  - All entries have the same size, so this is a numpy reshape. 5158 events × 1001 paths decode in 0.7 s, with 0 mismatches vs FWLite (accept + index, 1000 events).
  - Trigger names: psetid → `ParameterSets` (uproot fails on that tree too; use PyROOT once per distinct psetid/menu).
  - Timing (402360 ntuples, ~5k events/file, scratchpad `timing.py`/`timing_mp.py`). Both methods give identical counts.
    - Minimal FWLite loop (getByLabel + `accept(j)` for all paths): ~1.0k events/s, 5 s/file.
    - uproot: ~16–19k events/s, 0.3 s/file. That is 4 procs ≈ 64k ev/s and 8 procs ≈ 78k ev/s, so EOS reads are not the limit at this scale.
    - → Baseline 403663 (3.5k files, 13.5M events): about 12 min on 1 core, about 3–4 min with 4–8 procs, for plain counting.
  - Robust prototype: scratchpad `trdecode.py`.
    - Reads N from every entry's header and validates the header bytes.
    - Checks the streamer class versions (TriggerResults 11, HLTGlobalStatus 10, HLTPathStatus 10) and the memberwise flag.
    - Fails loudly on an unknown layout, mixed N, mixed psetids or non-empty `names_`.
    - Names come from `@trigger_paths` of the psetid PSet; strip `\x00` on both sides.
    - Tested vs FWLite (accept, index, names), all identical: GRunV51/V59/V62 (1001 paths each) and two HI menus (394 paths).
    - V51→V59 differ in 654 versioned names with the same count, so merge across menus on unversioned names. Keep FWLite as a validation/fallback reader.
  - `EventAuxiliary` (needed for run/LS/BX): uproot also fails, but the raw bytes have a fixed 109-byte layout.
    - run u4 @35, LS u4 @39, event u8 @43 (corrected; verified event by event vs FWLite in `steamrates/readers/edm.py`). LuminosityBlockAuxiliary: run @35, LS @39.
    - `bunchCrossing` = u4 at `[-12:-8]`, `orbitNumber` at `[-8:-4]`, `storeNumber` at `[-4:]`.
    - Validate the header and the 109-byte length the same way as for TriggerResults.
  - Trigger-pattern sparsity (402360 GRunV59, 40 files, 202k events): on average 3.5 fired paths/event (incl. the always-on `HLTriggerFirstPath`/`Status_OnCPU`), max 105.
    - Only **12.6k distinct fired-path patterns (6% of events)**.
    - → Storing "pattern → count (Σw, Σw²)" per chunk is compact, and ANY grouping/correlation/pure rate can then be computed at merge time without re-reading the ntuples.
  - Scaling decision (user, 2026-11-09): samples can be large, so keep batch support. Design: the work unit is a chunk of files → partial result (counts + (dataset, run, LS) set). The same function runs under a local process pool or condor, and the merge is executor-independent. Read large NanoAOD files in event chunks (`uproot.iterate`) to bound memory.
- A single config file → own work dir containing config copy, job files, partial results, final JSON with metadata.
- File lists are deduplicated and validated before job creation.

## New ntuples: HLTPhysics NanoAOD (preliminary "common format"; inspected 2026-11-09)
Files:
- 403663: `/eos/cms/store/data/Run2026D/HLTPhysics/NANOAOD/PromptReco-v1/000/403/663/00000/*.root`. 3 files, 1.0M events, 1.3 GB.
- 402360: `root://cms-xrd-global.cern.ch//store/data/Run2026B/HLTPhysics/NANOAOD/PromptReco-v1/000/402/360/00000/36a8e992-5ef5-4668-a01a-fb72197f4bed.root`. 346k events. Needs the grid proxy.
- uproot over xrootd needs `handler=uproot.source.xrootd.XRootDSource` (fsspec-xrootd isn't installed).

Structure: standard PromptReco NanoAOD.
- Trees: `Events`, `LuminosityBlocks` (run, LS, fill, crossingAngle, betaStar, energy; **no lumi value**), `Runs`, `MetaData`, `ParameterSets`.
- Trigger bits are flat `bool` branches, one per path:
  - 810 `HLT_*`, 26 `DST_*` (only the Scouting ones), 5 `Dataset_*Scouting*`, `HLTriggerFirst/FinalPath`;
  - 508 `L1_*` (+ `L1_FinalOR_BXmin1/2`, `L1_UnprefireableEvent*`);
  - 28 `Flag_*`;
  - plus L1 objects (`L1Mu/EG/Jet/Tau/EtSum`), `TrigObj_*`, and full physics objects.

Differences from the old STEAM EDM ntuples:
- **HLT bits are the ONLINE decisions** (process `HLT`, menu `/cdaq/physics/Run2026/2e34/v1.3.0/HLT/V3` for 403663), not a re-run of the menu under study (MYHLT).
  - Evidence: the 402360 nano lacks 38 paths that exist in the GRunV59 re-run.
  - → New-menu studies need re-HLT output in this format. Still to clarify with the colleagues.
- **Version suffix stripped**: `HLT_IsoMu24`, not `HLT_IsoMu24_v33`. Done by `PhysicsTools/NanoAOD/plugins/TriggerOutputBranches.cc`: `rfind("_v")` for HLT*/L1*/*Scouting*.
- **Only names starting with `HLT`, `L1`, `Flag` or containing `Scouting` are stored**. Dropped compared to the old ntuple (1001 paths): AlCa_ (13), Dataset_ (112 of 117), MC_ (25), DQM_ (3), Status_ (2), 2 non-scouting DST_.
- Paths missing from a run's menu are **back-filled with 0**. "Never fired" therefore can't be told apart from "not in menu" without the menu.
- If two processes store the same name, a `_p<process>` suffix is added (none seen).
- **L1 bits = `getAlgoDecisionFinal()`** (`L1TriggerResultsConverter.cc:143`), i.e. AFTER L1 prescale and mask.
  - 250 of 508 never fired (disabled or masked).
  - Unprescaled L1 rates need the L1 PS table (not in the file). Seeds with PS 0 give no information.
- LS are scattered across files (each file spans the whole run); no LS was found in two files.
  - → N_LS must be a **deduplicated set taken from the `LuminosityBlocks` tree** ∩ JSON. Summing N_LS per job (current code) would give 3 × N_LS here, with no compensating factor in hlt_ps.
  - Contrast: the current per-job N_LS sum works for EphemeralHLTPhysics 0–7 because the factor 8 (8 datasets × same LS) is put into hlt_ps on purpose (user confirmed). It breaks only if a (dataset, LS) pair is split across files or a file is listed twice. Cleaner design: count unique (dataset, run, LS) and let the normalization know the per-dataset PS and the number of datasets combined.
  - The LB tree has one LS more than the LS seen in events (an LS with 0 events).
- **Speed**: reading all ~1350 trigger bools for 372k events takes 14 s with uproot. A whole run is about 1 min on one core, so condor is likely not needed for plain counting.

**The `ParameterSets` tree embeds the full online HLT config.** Find the process PSet with `@trigger_paths` + `datasets` + `streams`; sub-PSets are referenced by `+Q(<hex hash>)`, and the blobs contain NUL bytes. It provides:
- `HLTConfigVersion.tableName` (menu name);
- `datasets` (dataset → versioned paths);
- `streams` (stream → datasets);
- `@trigger_paths`;
- per-path module lists;
- `hltL1s*` modules → L1 seed expression (783 HLT/DST paths have one);
- `PrescaleService` (15 columns, default `2p0E34`, 922 rows).

→ The stream/dataset maps (`streamMap`, `datasetMap`, `datasetStreamMap`) and the path → L1-seed maps can come **from the input file itself**, which replaces the ConfDB/cx_Oracle half of STEAM_maps (`hltConfigFromDB` + `hltDumpStream`).
  - It does NOT replace `groupMap`/`typeMap` (path → PAG/POG and type). These come from the ConfDB path *description* text ("group: …" lines, parsed in `hltDumpStream` ~L170) or from the hand-made `SteamDB.py`. They are curated by STEAM, are not part of the python config, and stay an external, hand-maintained input. Read it with PyROOT + `libFWCoreFWLite`: `std.pair("edm::Hash<1>","edm::ParameterSetBlob")`, and `hash.print(ostringstream)` for the hex key. Use `os._exit(0)` or `python -u`: PyROOT/XRootD can crash or hang at exit and lose buffered stdout.

Normalization facts (403663 menu):
- `HLT_Physics_v16` = `hltL1EventNumberL1Fat` (HLTL1NumberFilter, **period 107**, TCDS event number) + `hltPrePhysics` with PS 50 (25 in the Emergency column).
  - → effective PS **5350**, matching the "hidden 107" comment for run 403182.
  - "Hidden" because the HLTL1NumberFilter is not an `HLTPrescaler`: it isn't in the PS table and is only visible in the menu. It applies **only to HLTPhysics**, not to EphemeralHLTPhysics. Idea: derive the sample's effective PS automatically by walking the sample path's modules (HLTPrescaler × HLTL1NumberFilter period × dataset prescaler), with a config override.
- `HLT_EphemeralPhysics_v10` PS 220 (2p0E34) × `Dataset_EphemeralHLTPhysicsN` PS 8 = 1760 = baseline `hlt_ps`.
- HLTPhysics therefore has ~3× fewer events than EphemeralHLTPhysics 0-7 for the same LS.
- Sanity check, LS 400–1700 (1301 LS, 556,746 events), scale = 5350 / (1301 × 23.31 s) = 0.1764 Hz/count:
  - total L1 rate 98.2 kHz ✓
  - HLT_IsoMu24 377 Hz (2138 counts)
  - L1_SingleMu22 10.0 kHz
  - HLT_ZeroBias 148 Hz
- Still needed from outside the file: lumi per LS (lumi_in; OMS/brilcalc), the HLT/L1 PS column per LS, and the L1 prescale table.
- Per-filter rates: Nano keeps only path pass/fail (no `HLTPathStatus::index()`), so they're not possible from these files.

Inspection scripts (scratchpad, not in the repo): `inspect_nano.py`, `check_nano.py`, `compare_paths.py`, `psets2.py`, `prescales.py`, `sanity.py`.

## Answers from the user (2026-11-09)
- Common ntuple use cases: (a) recorded data with **re-run HLT** (+ possibly re-emulated L1), or (b) MC with emulated L1+HLT. Online-menu rates are not a use case.
  - MC: run/LS are meaningless (run = 1), and no LS-based normalization is needed.
- The user will ask the colleagues to include AlCa_/MC_/Dataset_ paths (needed).
- L1 bits: probably post-prescale only; pre-PS bits maybe later. Colleagues discussed storing L1 decisions for several PS columns. With the embedded menu (path → L1 seed expression) we could then also compute HLT rates under other L1 columns.
  - Caveat: with HLTPhysics data, only columns that tighten the online one are unbiased, because events that failed the online L1 are missing. MC and ZeroBias are fine.
  - Caveat: the HLT path result already contains the online L1 seed decision.
- Lumi/PS: the practice so far is to select LS with one PS column during lumi-levelling via the JSON, then use a constant hlt_ps and the average inst. lumi (from OMS) in `config_mergeOutputsData.py`. Keep that as the default.
  - Design the data normalization so per-LS lumi/PS tables (e.g. brilcalc; OMS needs an API key) can be plugged in later. **Don't build the per-LS source now.** MC-based Phase-2 is the main use.
- STEAM_maps: the PAG/POG group assignment is hand-made by STEAM; see the note in the NanoAOD section.

## Decisions (2026-11-09)
- Start the rewrite with the step-2 counting script, as a new package `steamrates/` next to the old code. The final directory layout gets decided at the end.
- Code style: functions/classes in separate modules, with thorough comments, especially in the byte decoders. All inputs come as CLI arguments (later wrapped in batch jobs). Output goes only to the explicit output path: no shared temp files, so parallel measurements are safe.
- **Combination ("pattern") table approach approved**:
  - A counting job stores the distinct fired-bit patterns with count/Σw/Σw²;
  - all rates, groupings, pure rates and overlaps are computed at merge time.
  - Fall back to fixed observables if it doesn't work out.
- **Partials are ROOT files** written by uproot (zstd), confirmed by the user. Final outputs will be JSON for the website.
- One **joint HLT+L1 table**.
- Menu reading = "option 2" (user): path names cached by TriggerResults psetid, plus menu metadata (name, datasets, streams, PS table) from the first file of each menu, stored in each partial. The per-file psetid is recorded.
- The STEAM PAG/POG group map stays a hand-maintained input; the stream/dataset maps can come from the embedded menu.

## steamrates package (counting step implemented 2026-11-09)
- `steamrates/` at the repo root; run `python3 -m steamrates.count|summary` from the repo root in cmsenv.
- The usage, module table, partial file format and validation are documented in `steamrates/README.md`.
- No PyROOT needed:
  - EDM TriggerResults, EventAuxiliary, LuminosityBlockAuxiliary and the ParameterSets tree are all decoded from raw bytes (`rootbytes.py`, `readers/edm.py`, `menu.py`).
  - ParameterSets entry: Hash<1> [0:29], blob headers [29:41], ROOT string at [41] (length byte; 255 → 4-byte length).
- Performance:
  - EDM: ~0.3 s/file (5k events), plus ~1.3 s for the first file of a menu (13.6 MB ParameterSets).
  - A full parse of the multi-MB process PSet is slow in Python, so top-level entries are located with anchored regexes (`ParameterSetRegistry.entry`).
  - NanoAOD: ~17k events/s.
  - MC: 241k events in 33 s, 1.3 GB RSS.
- Validation: `validation/compare_legacy_counts.py` and `validation/check_edm_reader_fwlite.py` (see the README).
  - **402360 GRunV59, 1136 files: total and pure counts of all 999 paths identical to the legacy nomaps output**, with 5,721,135 events.
  - Unique LS = 142 (legacy N_LS 1136 = 8 × 142, compensated by hlt_ps 480 = 60 × 8).
- Partial sizes: data ~0.4 MB per 300k events; MC 8.2 MB per 241k events.
- Next:
  - merge step: normalization (data: lumi/PS/N_LS per sample, extensible to per-LS tables; MC: σ/Σw), datasets/streams from the menu, groups from the STEAM map, pure rates, overlaps, prescales, JSON output;
  - job builder (local pool or condor);
  - plots;
  - regression vs the 403663 baseline once it's done.
- **Dropped: `metBx`/`muonBx` histograms** (BX distribution of events firing `HLT_PFMET*`/`HLT_IsoMu24*`, in `histos.N.root`). Nobody knows their purpose; the user is asking around. Re-add as an optional "BX distribution for selected paths" if someone needs them.

## Pattern-table measurements (2026-11-09; scratchpad `patterns_nano.py`, `table_size.py`)
Table = CSR-like sparse 0/1 matrix (rows = distinct patterns, cols = bits) plus per-row count/Σw/Σw².

| sample | events | bits | fired/event (HLT / L1 / joint) | distinct patterns HLT / L1 / joint | joint table JSON / ROOT-zstd |
|---|---|---|---|---|---|
| 403663 HLTPhysics NANOAOD (data) | 372k | 843+508 | 2.4 / 5.1 / 7.5 | 2.5% / 10.3% / 15.1% | 5.6 MB / 0.7 MB |
| VBFH→μμ Run3Winter26 NANOAODSIM (MC) | 100k–400k | 779+470 | 96 / 112 / 208 | 85% / 95.5% / 99.6% | 104 MB / 3.5 MB (100k ev) |

- MC menus are fully unprescaled (no PrescaleService in the MC menu), so every event is ~unique. The table then degenerates into a sparse per-event bit dump, still ≈1/70 of the input (3.5 MB vs ~250 MB).
- For MC, a joint table costs about the same as separate HLT and L1 tables. For data it is ~1.4×.
- Pattern counts grow sublinearly with N in data.
- Overlap matrix C = Xᵀ W X via `scipy.sparse`, cost ∝ Σ k² per pattern: 1.7 s for the data table (1351²), 13.6 s for 100k MC events.
  - **numpy in CMSSW_16_0_6_patch1 has no optimized BLAS** (dense float32 gram 2000×1351 takes 15 s). Never use dense matmul; use scipy.sparse or bincount.
- MC prescales must be applied at merge time anyway. With the table this is possible per pattern: P(path) = bit × 1/PS_HLT × P(L1 seed passes); OR of paths: 1 − Π(1 − p_i) (random-prescale approximation). The same works for any L1 PS column from unprescaled L1 bits, so multiple stored post-PS columns aren't needed.
- The MC sample has negative weights (`genWeight` ±4.18) → keep Σw and Σw² per pattern. The `Runs` tree has `genEventSumw/Sumw2/Count`.
- MC NANOAODSIM **embeds the HLT menu** in `ParameterSets` (process HLT, `/frozen/2025/2e34/v1.3/HLT/V4`, 927 paths, datasets; no PrescaleService) → the menu survives NanoAOD production (`menu_in_file.py`).

## Phase-2 trigger-ntuple preview (user-provided HTML doc of `step2_inNANOAODSIM.root`, 2026-11-09)
- NanoAOD-style flat ntuple. Events tree only documented: run/LS/event/BX/orbit and hlt* object collections (AK4 Puppi jets, electrons, photons, muons, taus, tracks, PF cands, PV, HT/MET).
- Trigger bits in the example: only `DST_PFScouting`, `HLTriggerFinalPath`, `L1TrackTrigger_step`, `L1simulation_step`. **No `HLT_*` paths and no L1 seed bits** (maybe a scouting-only menu).
  - NanoAOD keeps only names starting with HLT/L1/Flag or containing Scouting. Phase-2 GT algorithm paths (if named like `pSingleTkMuon22`; to verify) would be dropped.
- No `genWeight`/`Pileup_*` in Events. Runs-tree content (`genEventSumw`) and the `ParameterSets` tree are not covered by the doc; they are likely present if written by the standard NanoAOD output module (as in Run-3 NANOAODSIM). Needs a real file.
- The user will talk to the colleagues. Assume content similar to today's NanoAOD files for now; if the menu is missing, get it from ConfDB.

## Open questions for the user
- Format and timeline of the common L1+HLT ntuple (and whether Step 1 is dropped). Will path versions be kept?
- Website / JSON schema requirements.

## Status log
- 2026-11-09: Explored repo; wrote `docs/WORKFLOW.md` (8 Mermaid charts, validated with mermaid-cli) and these notes. No code changed yet.
- 2026-11-09: Inspected HLTPhysics NanoAOD for 403663/402360 (section above). Baseline 403663 (3534 files) is still running.
- 2026-11-09: Recorded the user's answers (section above). Verified uproot+numpy decoding of the old EDM TriggerResults, including the per-path last-module index.
- 2026-11-09: Design discussion → pattern-table approach, `steamrates/` package, metBx/muonBx dropped. Measured pattern tables on data and MC NanoAOD. Reviewed the Phase-2 ntuple doc.
- 2026-11-09: Slide figure `docs/figures/combination_table_slide.png` (script + data JSON alongside). **Local only:** `docs/figures/` is git-ignored and was removed from the history at the user's request (no binaries in the repo). Implemented and validated the counting step (`steamrates/`, `validation/`); see the section above. The user committed and pushed it to `phase2`.
