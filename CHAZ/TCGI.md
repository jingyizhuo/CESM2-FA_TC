# Note on Calculating Tropical Cyclone Genesis Indices (TCGI)

### 📚 References: 
- [Camargo et al. 2014 JCL](https://journals.ametsoc.org/view/journals/clim/27/24/jcli-d-13-00505.1.xml)
- [Tippett et al. 2011 JCL](https://journals.ametsoc.org/view/journals/clim/24/9/2010jcli3811.1.xml)

### 👩‍🏫‍ Code Acknowledgment
The core components of the TCGI computation code in this repository are adapted from original scripts developed by [Yi Xia](mailto:yx2820@columbia.edu), a PhD student at Columbia University. Please contact her directly for access to the full scripts and ensure proper attribution.

### 🗝 My Scripts
Scripts used to compute TCGI for multiple ensemble members of CESM2 and CESM2-FA are available at:

- `/data0/jzhuo/tc_risk/CESM2/code_TCGI/main.sh`
- `/data0/jzhuo/tc_risk/CESM2/code_TCGI/main_fa.sh`

These scripts call `get_TCGI_predictor.py` to get the predictors of TCGI (vws; vmax, which is mpi in terms of wind speed; avort; crh; sd), and then call `cal_TCGI.py` calculate the TCGI of both CRH (Tippett et al. 2011 JCL) and SD (Camargo et al. 2014 JCL) versions.

✅ An **example** notebook for calculating TCGI—whether using ERA5, other reanalysis products, or climate model outputs—is available at: `/data0/jzhuo/tc_risk/code_TCGI/example_code/code/TCGI.ipynb`. 

### 📂 My data
- Path of TCGI predictors: `/data0/jzhuo/tc_risk/CESM2/data_TCGI/tcgi_predictor_cesm2_cesm2fa_full_forcing`
- Path of TCGI: `/data0/jzhuo/tc_risk/CESM2/data_TCGI/TCGI_CRH_PI` and `/data0/jzhuo/tc_risk/CESM2/data_TCGI/TCGI_SD_PI`
- Path of PI: `/data0/jzhuo/tc_risk/CESM2/data_PI`

### 🔒 Note:
TCGI is a weighted sum of several TC-related environmental variables, with weights derived from ERA5.
If you are applying these weights to datasets other than ERA5, bias correction is required. For example:

```python
clim = data_era5 # the 1981-2021 climatology of a predictor based on ERA5 
data = data_yours # your data that need bias correction
data_bc = data + (clim - data.groupby('time.month'))
```
The ERA5 climatology data are saved at: ```/data0/jzhuo/tc_risk/CESM2/data_TCGI/clim_ERA5```
