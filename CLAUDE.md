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
- A single config file → own work dir containing config copy, job files, partial results, final JSON with metadata.
- File lists are deduplicated and validated before job creation.

## Open questions for the user
- Format and timeline of the common L1+HLT ntuple (and whether Step 1 is dropped).
- Where L1 prescales/decisions come from for L1 rates.
- Website / JSON schema requirements.

## Status log
- 2026-11-09: Explored repo; wrote `docs/WORKFLOW.md` (8 Mermaid charts, validated with mermaid-cli) and these notes. No code changed yet.
