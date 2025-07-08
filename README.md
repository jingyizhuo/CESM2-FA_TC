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

### Step 1: Prepare Input Data

Inputs may come from reanalysis or climate model outputs.

Variables required: T, PSL, TS, TMQ, U, V, Q and RELHUM.

#### 🐾 TC Genesis preprocessing: 

Compute **TCGI** (both **CRH** and **SD** versions are supported).  
Note that **GPI** is a predictor of TCGI thus will be computed during this step. We need to save the GPI for the later usage for generating TC intensity data.

> 🖥 **If you are an LDEO local machine user**, please refer to the following example notebooks:
- [`TCGI.ipynb`]( /data0/jzhuo/tc_risk/code_TCGI/example_code/code/TCGI.ipynb ) → *Calculate TCGI*  
- [`MPI.ipynb`]( /data0/jzhuo/tc_risk/code_TCGI/example_code/code/MPI.ipynb ) → *Calculate GPI*


#### 〰️ TC Track preprocessing
Calculate covariance matrix A for synthetic wind at 250/850 hPa and monthly-daily wind covariance. 
One can skip this preprocessing step if they only have montly data, instead, contact Chia-Ying to obtain appropriate wind covariance data for you.

#### 🌀 TC Intensity preprocessing
Prepare predictor fields and climatological statistics. 
Ensure correct paths are set for your saved model output fields:
  - `U`, `V`, `RELHUM`
  - Prepare a `.csv` file specifying paths to these inputs.  
  > 📄 *An example CSV format can be found at:*  
  `/path/to/example.csv`



### Step 2: Run CHAZ Preprocessing

Edit `Namelist.py`:

```bash
sed -i '/runCHAZ/c\runCHAZ=False' Namelist.py
sed -i '/runPreprocess/c\runPreprocess=True' Namelist.py
sed -i "/^ENS/c\ENS = '$imem'" Namelist.py

# Optional adjustments depending on your data:
sed -i 's/calWind = True/calWind = False/g' Namelist.py   >>>> jzhuo: I forthet why False for this. 
sed -i 's/calA = True/calA = False/g' Namelist.py
```

If not calculate A when only monthly data is avaliable, link pre-calculated A files from existing source:
```bash
path_cesm2_daily=/data0/clee/CMIP6/CESM2/r4i1p1f1/pre
ln -sf $path_cesm2_daily/A*.nc $path_pre/
```

Then run the preprocessing to get YYYY*r1i1p1f1.nc    >>>> jzhuo: I need to double check to ensure the CESM2 output are used
```bash
python CHAZ.py
```

### Step 2.1: Apply Bias Correction 
```bash
python getMeanStd.py
```

### Step 3: Run CHAZ Downscaling
Prepare the working directory:
```bash
ln -sf $path_pre/PI*.mat $path_wdir/
ln -sf $path_pre/TCGI*.mat $path_wdir/
ln -sf $path_pre/A*.nc $path_wdir/
ln -sf $path_pre/*_${imem}.nc $path_wdir/
ln -sf $path_pre/coefficient_meanstd.nc $path_wdir/
ln -sf $path_pre/trackPredictorsbt_obs.pik $path_wdir/
```

Edit Namelist.py for downscaling:
```bash
cd $path_work
sed -i '/runCHAZ/c\runCHAZ=True' Namelist.py
sed -i '/runPreprocess/c\runPreprocess=False' Namelist.py
sed -i "/^ENS/c\ENS = '$imem'" Namelist.py

```

Copy source code:
```bash
cp /home/jzhuo/chaz/src_nodaily/* $path_wdir
cp ./Namelist.py $path_wdir
```


Run downscaling:
```bash
cd $path_wdir
echo "Running downscaling..."
python CHAZ.py
```

### Step 3: Run CHAZ Downscaling
```bash
python rev.pik2netcdf_merge_2100.py
echo "Done. Check the NetCDF output."
```

#### Refs:
```bash
/data0/jzhuo/tc_risk/code_TCGI/example_code/code/TCGI.ipynb >> Calculate TCGI
/data0/jzhuo/tc_risk/code_TCGI/example_code/code/MPI.ipynb >>> Calculate GPI
/home/jzhuo/chaz/chaz/run_CESM2_FLX/main.sh >> Run CHAZ
/home/jzhuo/chaz/chaz/src_nodaily/CHAZ.py >> Source code
```



