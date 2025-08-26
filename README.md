# 🦖 How Tropical Pacific SST Trend Bias Shapes TC Trends? 

### 🌀 Understanding TCs from Coarse Climate Simulations

Understanding TC activity and associated risks in the past, present, and future are of paramount importance for climate mitigation and adaptation. But running multiple ensembles of storm-resolving climate simulations to assess TC trends remains impractical. Consequently, the "state-of-the-sciece" approach is **a hierarchical modeling framework**: coarse-resolution climate simulations (e.g., CMIP6 experiments) are conducted first, followed by TC downscaling models or high-resolution atmospheric-only GCMs to study TC activity and associated risks. 

Here, we explore how biases in forced trends of the tropical Pacific SST that persist in climate models impact the forced trends in TCs. To address this question, we use climate simulations from **CESM2** (biased) and **CESM2-FA** (flux-adjusted and less biased) at coarse resolution (~2° atmosphere, ~1° ocean), together with two TC downscaling models (CHAZ and Jonathan Lin’s model) and two high-resolution atmospheric models (GFDL HIRAM and AM2.5C360).


### 🌍 TC downscaling models and high-resolution atmospheric models:
- 💾 [**CESH2-FA**] ([https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/CHAZ.md](https://github.com/jingyizhuo/CESM2-FA)) - Constraining the mean-state bias of CESM2 via surface flux corrections, producing a more realistic historical forced SST trend.
- 📈 [**CHAZ**](https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/CHAZ.md) — TC downscaling using [CHAZ model](https://github.com/jingyizhuo/CESM2-FA_TC/tree/main/CHAZ)
- 📊 [**MIT**](https://github.com/jingyizhuo/CESM2-FA_TC/tree/main/MIT_model) — TC downscaling using [Jonathanlin's model](https://github.com/linjonathan/tropical_cyclone_risk)
- 🌊 [**GFDL**](https://github.com/jingyizhuo/CESM2-FA_TC/tree/main/GFDL_model) — sea surface temperature and sea ice forced high-resolution atmospheric models: HiRAM (50 km) and AM2.5C360 (25 km).
