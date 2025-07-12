# TC Downscaling Using CHAZ

This repository provides a notebook for running the Columbia TC hazard model CHAZ, a statistical-dynamical downscaling tool. CHAZ uses large-scale atmospheric dynamic and thermodynamic fields from reanalysis or climate model outputs to simulate the genesis, tracks, and intensities of synthetic tropical cyclones (TCs).

For further model details, please refer to [Lee et al., 2018](https://doi.org/10.1002/2017MS001186).

---

## Three basic elements of the CHAZ model

### Genesis

CHAZ offers two formulations of genesis potential:

$$
\mu_{\text{CRH}} = \exp\left(b + b_{\eta} \eta_{850} + b_{\text{rh}} \text{CRH} + b_{\text{PI}} \text{PI} + b_{\text{SHR}} \text{SHR}\right)
$$

$$
\mu_{\text{SD}} = \exp\left(b + b_{\eta} \eta_{850} + b_{\text{SD}} \text{SD} + b_{\text{PI}} \text{PI} + b_{\text{SHR}} \text{SHR}\right)
$$

### Track

$$
V_{track} = \alpha V_{850} + (1 - \alpha)V_{250} + V_{\beta}
$$

$$
\quad \alpha = 0.8, \quad V_{\beta} \text{  is the beta drift vector which is a function of latitude}
$$


### Intensity

$$
v_{t+\Delta t} - v_t = MLR(X_t, X_{t+\Delta t}, v_t, v_{t-\Delta t}) + \varepsilon_t
$$

Predictors include monthly wind (vorticity, shear, steering flow), temperature and moisture (PI, humidity, and/or saturation deficit). The stochastic term \(\varepsilon_t\) accounts for internal storm dynamics not explicitly tied to the environment.

---

## 🔁 Workflow Overview

![flow_chart](https://user-images.githubusercontent.com/46905677/126709479-ad3eab03-a4bd-4ea5-a85b-79f1a83bed83.png)

## 🔧 Step-by-Step Instructions

### Step 1: Data preparation/pre-processing:

Inputs may come from reanalysis or climate model outputs.

Variables required: T, PSL, TS, TMQ, U, V, Q and RELHUM.

#### 🐾 TC Genesis preprocessing, also see [TCGI.md](https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/TCGI.md): 

Compute **TCGI** (both **CRH** and **SD** versions are supported). 
Note that **PI** is a predictor of TCGI thus will be computed during this step. We need to save the PI for the later usage for generating TC intensity data.

> 🖥 **If you are an LDEO local machine user**, please refer to the following example notebooks:
- `/data0/jzhuo/tc_risk/code_TCGI/example_code/code/TCGI.ipynb`
- My scripts that run parallelly to get each ensemble member's TCGI and PI are saved at `/data0/jzhuo/tc_risk/CESM2/code_TCGI/main*.sh`
- My data of CESM2 and CESM2-FA TCGI and PI are saved at `/data0/jzhuo/tc_risk/CESM2/data_TCGI/` and `/data0/jzhuo/tc_risk/CESM2/data_PI/`

#### 〰️ TC Track preprocessing
Calculate covariance matrix A for synthetic wind at 250/850 hPa and monthly-daily wind covariance. 
One can skip this preprocessing step if they only have montly data, instead, contact Chia-Ying to obtain appropriate wind covariance data for you.

#### 🌀 TC Intensity preprocessing
Prepare predictor fields and climatological statistics. 
Ensure correct paths are set for your saved model output fields:
  - `U`, `V`, `RELHUM`
  - Prepare a `.csv` file specifying paths to these inputs.  
  > 📄 An example CSV format file denoting the input data path can be found at:  
  `/data0/jzhuo/tc_risk/CESM2/CHAZ/input_data/cesm2_cesm2fa_chaz_data.csv`

---

### Step 2: Run CHAZ, see [CHAZ.md](https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/CHAZ.md): 



