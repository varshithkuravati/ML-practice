# Golden Evidence Manifest

**Every proposal artifact must trace to this manifest before inclusion in any presentation.**

---

## Verified Figures

| # | Figure | Scenario | Drive | Raw Files | Start Idx | End Idx | Distance (m) | S6 Error (m) | S6 Drift (%) | Script | Model | Reproducible | Proposal-Safe |
|:-:|:-------|:---------|:------|:----------|----------:|--------:|--------------:|-------------:|-------------:|:-------|:------|:-------------|:-------------|
| 1 | fig1_trajectory_1km_comparison.png | Vfa01_t70s_d60s | Vfa01 | S-Vfa01.csv + V-Vfa01.csv | 700 | 1300 | 1139.7 | 44.66 | 3.92 | generate_proposal_figures.py | inertial_odom.pt (SHA256: 1c4c7a76) | YES (seed=42) | YES |
| 2 | fig2_gnss_blackout_route_overview.png | Vfa01_t70s_d60s (on full trip) | Vfa01 | S-Vfa01.csv + V-Vfa01.csv | 700 | 1300 | 1139.7 | 44.66 | 3.92 | generate_proposal_figures.py | same | YES | YES |
| 3 | fig3_position_error_vs_time.png | Vfa01_t70s_d60s | Vfa01 | same | 700 | 1300 | 1139.7 | 44.66 | 3.92 | generate_proposal_figures.py | same | YES | YES |
| 4 | fig4_ai_speed_estimation_tracking.png | Vfa01_t70s_d60s | Vfa01 | same | 700 | 1300 | 1139.7 | N/A (speed) | N/A | generate_proposal_figures.py | same | YES | YES |
| 5 | fig5_architecture_drift_comparison.png | Vfa01_t70s_d60s | Vfa01 | same | 700 | 1300 | 1139.7 | 44.66 | 3.92 | generate_proposal_figures.py | same | YES | YES |

All 5 figures are generated from the same single scenario (Vfa01_t70s_d60s). They show different views (trajectory, error growth, speed, ablation) of one experiment.

---

## Verified Metrics (CSV Ground Truth: hardened_per_scenario_metrics.csv)

| Scenario | Distance (m) | S6 Final Error (m) | S6 Drift (%) | Passed <10% | Split |
|:---------|-------------:|--------------------:|-------------:|:------------|:------|
| Vfa01_t30s_d60s | 1162.5 | 30.02 | 2.58 | YES | VALIDATION |
| Vfa01_t45s_d60s | 1159.2 | 35.71 | 3.08 | YES | VALIDATION |
| Vfa01_t50s_d60s | 1149.1 | 44.35 | 3.86 | YES | VALIDATION |
| Vfa01_t70s_d60s | 1139.7 | 44.66 | 3.92 | YES | VALIDATION |
| Vfa01_t90s_d60s | 1157.5 | 65.68 | 5.67 | YES | VALIDATION |
| Vfa01_t220s_d60s | 1037.3 | 161.97 | 15.61 | NO | VALIDATION |

---

## Documents with Stale Metrics

| Document | Contains Stale Metrics | Status |
|:---------|:----------------------|:-------|
| proposal_evidence_matrix.csv | YES: claims t220s 20.36m/1.96% | MUST CORRECT |
| SIH_Proposal_Evidence_Audit.md | YES: claims 90.84m/7.97% and 20.36m/1.96% | MUST CORRECT |
| PPT_evidence_content.md | YES: claims 90.84m/7.97%, 210.32m/18.45%, 20.36m/1.96% | MUST CORRECT |
| MUST_HAVE_FOR_SCREENING.md | YES: claims 90.84m/7.97% and 20.36m/1.96% | MUST CORRECT |
| DO_NOT_CLAIM.md | YES: cites 7.97%/90.84m, 1.96%/20.36m as verified causal | MUST CORRECT |
| SHOULD_HAVE_FOR_CREDIBILITY.md | No stale drift metrics | OK |
