## Datathon Plan: Ultimate 2026 Ski Holiday

### Objectives
- **Primary**: Recommend the optimal 7‑day window in 2026 and the best ANZ ski resort for a winter getaway.
- **Constraints/considerations**: visitor volumes (crowding), climate/snow/weather suitability, price proxies, resort features/amenities, travel feasibility.
- **Outputs**: Ranked short‑list of resort × week options, final pick with rationale, and engaging visuals.

### Datasets
- **visitation.csv**: Weekly visitation counts by resort.
  - Columns: `Year`, `Week`, `Mt. Baw Baw`, `Mt. Stirling`, `Mt. Hotham`, `Falls Creek`, `Mt. Buller`, `Selwyn`, `Thredbo`, `Perisher`, `Charlotte Pass`.
  - Coverage: 2014–2023 (confirm), weeks ~1–15 (Austral winter).
  - Metric: **Visitor Days** (12‑hour blocks per person).
- **climate.csv**: Daily station observations (BOM schema).
  - Columns: `Bureau of Meteorology station number`, `Year`, `Month`, `Day`, `Maximum temperature (Degree C)`, `Minimum temperature (Degree C)`, `Rainfall amount (millimetres)`.
  - Notes: Missing values present; station mapping to resorts required; snow proxy derived from temperature + precipitation.
- **External (optional)**: Resort lift ticket price histories, accommodation indices, school holiday calendars (AU states + NZ), snowfall reports, terrain/open lift counts.
 - **events.csv / amenities.csv** (extension): curated inputs for youth/family targeting; event timing/intensity and static amenity scores per resort.

### Season anchors and week mapping
- **Season week starts (reference)**:
  - Week 1: 9‑Jun
  - Week 2: 16‑Jun
  - Week 3: 23‑Jun
  - Week 4: 30‑Jun
  - Week 5: 7‑Jul
  - Week 6: 14‑Jul
  - Week 7: 21‑Jul
  - Week 8: 28‑Jul
  - Week 9: 4‑Aug
  - Week 10: 11‑Aug
  - Week 11: 18‑Aug
  - Week 12: 25‑Aug
  - Week 13: 1‑Sep
  - Week 14: 8‑Sep
  - Week 15: 15‑Sep
  - Use these anchors to align daily climate data to `visitation.csv` week indices.

### School holidays and public holidays (2026)
- Victoria Term 2 school holidays: Saturday 27‑Jun – Sunday 12‑Jul 2026.
- New South Wales Term 2 school holidays: Saturday 4‑Jul – Sunday 19‑Jul 2026.
- King's Birthday long weekend: second Monday in June (opening weekend demand spike).
- Use overlap with these dates to flag high‑demand periods for crowd/price modeling.

### Assumptions (initial)
- 2026 season weeks align with the anchors above (weeks 1–15), adjusted to 2026 calendar dates.
- Climate station(s) selected per resort are suitably representative of resort conditions (with elevation adjustment if needed).
- Snow suitability proxy: cold enough and precipitating → higher likelihood of snow vs rain.

### Methodology
1. **Data audit & cleaning**
   - Load both CSVs; standardize column names; parse dates; handle missing data.
   - From `climate.csv`, construct a daily datetime and derive daily metrics: `tmax`, `tmin`, `rain_mm`.
   - Map stations → resorts using the following mapping:
     - 71032 → Thredbo AWS
     - 71075 → Perisher AWS
     - 72161 → Cabramurra SMHEA AWS (Selwyn)
     - 83024 → Mount Buller (also used for Mount Stirling)
     - 83084 → Falls Creek
     - 83085 → Mount Hotham
     - 85291 → Mount Baw Baw
     - Charlotte Pass → average of Thredbo (71032) and Perisher (71075)
     - Mount Stirling → shares Mount Buller (83024)
2. **Feature engineering (weekly)**
   - Aggregate daily climate to fixed season week windows (Week 1–15) using the provided anchors for each year; ensure alignment with `visitation.csv` indexing.
   - Derive snow suitability features per resort × week:
2b. **Rolling 7‑day climate windows (cross‑week sensitivity)**
   - Compute a rolling 7‑day window series for each resort across the entire season (stride = 1 day).
   - For each window, compute the same climate features as weekly (cold‑day count, cold‑precip intensity, thaw risk, comfort).
   - Identify top N climate‑optimal 7‑day windows per resort.
   - Map each 7‑day window to its overlapping `Week` indices; when scoring against weekly visitation/price, blend crowd/price signals proportionally to overlap (e.g., 4 days in Week 7, 3 days in Week 8).
   - Final recommendation operates on 7‑day windows; weekly outputs remain for comparability/visualization.
     - Cold‑day count: days with `tmax <= 2°C` OR `tmin <= -2°C`.
     - Freeze‑thaw risk: mean diurnal range, count of thaw days (`tmax > 3°C`).
     - Precipitation intensity: sum and 75th percentile of `rain_mm` (proxy snowfall when cold).
     - Snow‑likelihood score: f(precipitation on sub‑zero days).
   - Calendar features: school holidays/public holidays; long weekends.
3. **Visitation modeling**
   - Baseline (simple): per‑resort mean by `Week` index × school‑holiday uplift (learned from history) ± trend.
   - ML (per resort, Step 12‑ML): Ridge regression on `log1p(VisitorDays)` with features:
     - Week dummies (1–15), VIC/NSW holiday overlap, King’s Birthday flag, year trend.
     - Weekly climate features aggregated from daily: `cold_days`, `thaw_days`, `rain_sum`, `cold_precip_proxy` (via `rain_sum × cold_days/7`).
     - Validation uses leave‑one‑year‑out CV where the held‑out year’s climate features are replaced by climatology means (to mimic 2026 information).
   - 2026 forecast inputs: holiday overlaps + climate climatology (expected weekly climate features) → predict weekly visitors; invert log; form crowd curve.
   - Normalize within resort (z‑scores) to form the crowd score; for 7‑day windows, blend by day‑overlap weights.

4. **Window‑level refinements (Step 16)**
   - Price proxy at window‑level: `0.75 × (VIC_overlap_window + NSW_overlap_window) − 0.25 × discount_any(window)` from events.
   - Event‑driven crowd uplift: `crowd_adj = crowd_blended + 0.25 × event_attract_any(window)`.
5. **Scoring framework**
   - For each resort × week, compute standardized sub‑scores:
     - Crowd score (lower is better): inverse of predicted/typical visitation.
     - Snow score: based on cold‑day + cold‑precip signals.
     - Weather comfort: penalize extreme cold (< -10°C) or warm slush (> 5°C), wind proxy if available.
     - Price proxy: correlate historic visitation peaks with price surges; apply school holiday premiums.
     - Access/amenities bonus: categorical uplift for terrain size, beginner friendliness, nightlife (input via lookup table).
   - Combine via weighted sum. Default weights (tunable): `Snow 0.40`, `Crowd 0.30`, `Price 0.15`, `Comfort 0.10`, `Amenities 0.05`.
6. **2026 selection**
   - Use historical climate/visitation patterns to identify typical best weeks; overlay 2026 state school holiday calendars (VIC/NSW) and price proxies.
   - Present top 5 resort × week candidates with uncertainty bands; select final recommendation.

### Visuals (storytelling)
- Season curves: weekly visitation by resort with bands for school holidays.
- Climate ribbons: weekly cold‑day counts and cold‑precip bars per resort.
- Heatmap: resort × week composite scores.
- Calendar view: 2026 winter with recommended week highlighted.
- Resort profile cards: key features, pros/cons, accessibility map.
 - Visuals pack (Step 17): heatmaps for General/Youth/Family across resorts×start dates; weekly visitor bars with holiday highlights; climate mean + p20–p80 ribbon.

### Deliverables
- `notebook.ipynb` with reproducible analysis and visuals.
- Exported figures for presentation deck.
- One‑pager summary with final pick and reasoning.

### Risks & mitigations
- Missing/biased climate data → use multiple nearby stations or reweight; impute conservatively.
- Station elevation mismatch → adjust temperatures by lapse rate (~6.5°C/1,000m) if metadata available.
- Week alignment between datasets → validate against 2014–2023 examples; fix offsets early.

### Next steps
1) Create the notebook scaffold and data‑loading cells.
2) Build weekly climate aggregation and station mapping.
3) Engineer snow/comfort features and compute visitation predictions.
4) Implement scoring and produce ranked candidates + visuals.



### Validation plan
- **Visitor forecasting (weekly, per resort)**
  - Protocol: Leave‑one‑year‑out CV (train on all but one season, test on the held‑out season).
  - ML protocol: for the held‑out year, use climate climatology features (not realized weather) to mimic 2026 information.
  - Metrics: RMSE on log(visitors+1), MAPE, Pearson r by week.
  - Baselines: compare Ridge vs week‑mean × holiday uplift; per resort choose the better; fallback to baseline if ML underperforms.
  - Uncertainty (optional): Monte‑Carlo over climate features sampled from historical distributions to produce visitor prediction bands.
- **Climate (rolling 7‑day probabilistic)**
  - Coverage checks: for each held‑out year, actual 7‑day climate scores should fall within historical 50%/80% bands at expected rates.
  - Sensitivity: vary thresholds (cold day, thaw) and feature weights; expect top‑k windows to be stable.
  - Station QA: missingness, continuity, and Charlotte Pass averaging sanity checks.
- **Composite decision (resort × 7‑day window)**
  - Weight sensitivity: perturb weights ±10–20% and measure rank stability (Kendall tau; top‑k overlap).
  - Scenario tests: with/without holiday uplift; with/without trend; verify final recommendation is robust.
  - Explainability: component breakdown (climate mean + band, crowd z, holiday overlap, comfort) for finalists.

### Audience‑focused extension (Youth vs Family)
- Goal: Provide tailored recommendations for two audiences and show how the top picks change vs the general recommendation.
- Additional datasets to curate:
  - `events.csv`: `resort`, `event_name`, `start_date`, `end_date`, `audience_tags` (pipe‑separated: youth|family), `impact_type` (attracts|discount), `impact_strength` (0–1).
  - `amenities.csv`: `resort`, `beginner_friendly` (0–1), `ski_school_quality` (0–1), `nightlife` (0–1), `terrain_parks` (0–1), `family_services` (0–1), `access_ease` (0–1).
- Feature engineering for each resort × 7‑day window:
  - Event overlap: sum impact strengths for overlapping windows by tag → `event_attract_youth`, `event_attract_family`, `discount_youth`, `discount_family`.
  - Amenities: join static 0–1 attributes per resort.
- Audience fit scores:
  - Youth_fit = 0.35×nightlife + 0.25×terrain_parks + 0.20×event_attract_youth + 0.20×discount_youth.
  - Family_fit = 0.30×beginner_friendly + 0.25×ski_school_quality + 0.20×family_services + 0.15×access_ease + 0.10×discount_family.
- Audience composite (weights are tunable):
  - Youth: 0.35×Climate + 0.20×(−Crowd) + 0.10×(−Price) + 0.35×Youth_fit (all standardized within resort).
  - Family: 0.35×Climate + 0.35×(−Crowd) + 0.10×(−Price) + 0.20×Family_fit.
- Optional event uplift: if `impact_type=attracts`, add small crowd uplift during event windows; if `discount`, reduce price proxy. Cap effects.
- Notebook: Add Step 15 to load events/amenities, compute overlaps, audience fits, and audience‑specific top‑N; include a toggle to show general vs audience picks side‑by‑side.
- Validation: scenario on/off events; ±20% weight sensitivity; top‑k overlap changes per audience to demonstrate progression.

### Notebook steps (expanded)
- Step 12‑ML: Per‑resort Ridge visitor forecasting with climate features; LOYO CV using climatology for held‑out; produce 2026 ML forecast.
- Step 15: Audience extension — load `events.csv`/`amenities.csv`, compute event overlaps, build Youth/Family fits, output audience top‑N.
- Step 16: Refine price/crowd using window‑level holiday overlap and event‑driven uplift; recompute General/Youth/Family composites; print side‑by‑side top‑10 tables and comparison plots.
- Step 17: Visuals pack — heatmaps, weekly visitor bars with holiday highlights, climate uncertainty ribbon; export PNGs to `figures/`.
