# 🌎 👉 🌀 TC Downscaling Using CHAZ

#### Update Notes — 2025.08.08
CHAZ was successfully installed on the Princeton Tiger3 cluster. This tutorial has now been updated to support all platforms, so you can run CHAZ on your local machines.

To solve issue like "ImportError: cannot import name '_lazywhere' from 'scipy._lib._util", Downgrade scipy to a version that still has _lazywhere:
```bash
conda activate #your_conda_env
conda install scipy=1.13.0
```


#### Update Notes — 2025.07.04
Created this GitHub page to demonstrate running CHAZ with my own simulation data — fully coupleds run using CESM2 and CESM2-FA (Zhuo et al. 2025 JCL).


## 1️⃣ Install CHAZ — Fetch the Source Code and Dependent Data to Your PC

The CHAZ package (source code, dependencies, and model parameters) is stored at:
```bash
/data0/jzhuo/tc_risk/chaz_src/
```
Copy this folder to your local machine. It contains:

`src/` — Code for runs with daily data

`src_nodaily/` — Code for runs with monthly-only data

`pyclee/` — CHAZ’s dependent Python modules (Remember to export the codes under this path!!)

`chaz_data/` — CHAZ’s dependent data, for example: landmask.nc, best-track data

⚠️ Do not modify the original source code directly.
The version provided here has been adapted for:
1. **Python 3 compatibility**  
   > Many changes were made to ensure CHAZ runs under Python 3. Some were also necessary for script execution, but the exact list of modifications is no longer fully documented.

2. **CESM2/CESM2-FA or other model/ERA input format support**
   > PI and TCGI .mat files must use 'lon' and 'lat' as axes, with shape (91, 180). One can use any *.nc file saved at `/data0/jzhuo/tc_risk/chaz_src/egdata/regrid_target.nc` as target grid for regridding.


### 📚 CHAZ Tutorials
For a deeper understanding of CHAZ’s source code, see the Jupyter notebook tutorials (runnable directly in your browser via Google Colab):

- [CHAZ Preprocessing Tutorial](https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/Tutorial_chaz_pre.ipynb)  
- [CHAZ Downscaling Tutorial](https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/Turtorial_chaz_downscaling.ipynb)

These tutorials aim to help users understand and reproduce the CHAZ workflow.
However, **running CHAZ on LDEO-Taroko or other local systems still requires following the step-by-step instructions below.**

## 2️⃣ Be Ready! — Prepare Your Data and Set Up the CHAZ Working Directory
**1. Prepare the input list**
Create a .csv file specifying the paths to all CHAZ input files.
Example file:
   `/data0/jzhuo/tc_risk/CESM2/CHAZ/input_data/cesm2_cesm2fa_chaz_data.csv`

⚠️ The current version of CHAZ does **not** include TCGI and PI calculations — you will need to generate these datasets separately (see [TCGI.md](https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/TCGI.md)) and link them into the CHAZ working directory using `ln -sf` (this step is already handled in the `main.sh` script below).

⚠️ If you only have monthly U, V, then you will must need to get the wind coveriance data from other sources. Here I use the wind coveriance derived from CESM2-CMIP6 daily U, V. (data saved at `/data0/jzhuo/tc_risk/CESM2/CMIP6_CESM2/windcov/`)

**2. Create a working directory**
For each case, make a dedicated working directory under:
   `/data0/jzhuo/tc_risk/CESM2/CHAZ/work/`


## 3️⃣ Run CHAZ! — Set Parameters and Execute the Analysis via a bash script `main.sh`

📝 CHAZ is controlled via `Namelist.py`, which determines whether to run the **preprocessing** or **downscaling** steps and defines the paths to required input data.  
Please review the file at `/data0/jzhuo/tc_risk/chaz_src/src_nodaily/Namelist.py` to get a sense of how the script is configured.

✅ The following `main.sh` script implements a complete, reproducible workflow for running CHAZ from preprocessing to downscaling. Copy and paste the script into the directory where you want to place `main.sh`, then run:

```bash
bash main.sh
```


```bash
#!/bin/bash
# Author: Jingyi zhuo (jzhuo@princeton.edu)

export PYTHONPATH=/scratch/gpfs/GEOCLIM/jzhuo/tc_risk/chaz_src/pyclee !!! ⚠️ Remember to import some dependent CHAZ codes

# Set up working path
root_work=/data0/jzhuo/tc_risk/CESM2/CHAZ/work # Modify to your own path !!! ⚠️ 
root_PI=/data0/jzhuo/tc_risk/CESM2/data_PI # Modify to your own path !!! ⚠️ 
root_TCGI=/data0/jzhuo/tc_risk/CESM2/data_TCGI # Modify to your own path !!! ⚠️ 
pth_chaz_src=/data0/jzhuo/tc_risk/chaz_src # Modify to your own path !!! ⚠️
path_windcov=/data0/jzhuo/tc_risk/CESM2/CMIP6_CESM2/windcov/ # Modify to your own path !!! ⚠️
cd $root_work

####################################################
# Basic settings                                ####
####################################################
imem=r1i1
model=cesm2
forcing=hist
reso=Amon
TCGI=TCGI_CRH_PI

if [ "$forcing" == "hist" ]; then
    year1=1965 # Modify to your own setting
    year2=2014
elif [ "$forcing" == "ssps585" ]; then
    year1=2015
    year2=2049
fi

# Create sub-folders:
path_pre=$root_work/$model/$imem/pre
path_wdir=$root_work/$model/$imem/wdir
mkdir -p $path_pre
mkdir -p $path_wdir
echo Start ... $model $imem

####################################################
# CHAZ preprocessing                            ####
# To get:
#  coefficient_meanstd.nc: for bias corr.       ####
#  Cov*YYYY.nc: monthly-daily wind covariance   ####
#  YYYY*r1i1p1f1.nc: intensity model predictors ####
#  A_*YYYYMM.nc: covariance matrix              ####
####################################################

# 1. Calculate PI & TCGI, and link data to the pre folder:
# >> This step needs to be done by yourself separately, since CHAZ doesn't integrate the TC genesis calculation part and use TCGI data from Suzana. SMH...
cp "$pth_chaz_src/src_nodaily"/* "$path_pre"
cd "$path_pre"
rm -rf "$path_pre/__pycache__"

path_PI=$root_PI/$model/$forcing/$imem
path_TCGI=$root_TCGI/$TCGI/$model/$forcing/$imem

#   [1] link PI to path_pre, and rename the PI_*.mat files from PI_$model_$forcing_$imem_YYYY.mat to PI_YYYY.mat
for file in $path_PI/PI_${model}_${forcing}_${imem}_*.mat; do
    year=$(basename $file | sed -n 's/.*_\([0-9]\{4\}\).mat/\1/p')
    ln -sf $file $path_pre/PI_$year.mat
done

#    [2] link TCGI to path_pre, and rename the TCGI_*.mat files from TCGI_CRH_$model_$forcing_$imem_YYYY.mat to TCGI_CRH_PI_YYYY.mat
for file in $path_TCGI/${TCGI}_${model}_${forcing}_${imem}_*.mat; do
    year=$(basename $file | sed -n 's/.*_\([0-9]\{4\}\).mat/\1/p')
    ln -sf $file $path_pre/${TCGI}_$year.mat
done

# 2. Run CHAZ.py to get A_*YYYYMM.nc & YYYY*r1i1p1f1.nc:
# Modify Namelist.py by using linux command sed-i:
# [1] Modify execution flags: set runCHAZ=False, runPreprocess=True 
sed -i '/runCHAZ/c\runCHAZ=False' $path_pre/Namelist.py
sed -i '/runPreprocess/c\runPreprocess=True' $path_pre/Namelist.py
# [2] Set ensemble member ID
sed -i "/^ENS/c\ENS = '$imem'" $path_pre/Namelist.py 
# [3] Set simulation period based on forcing type
sed -i "/^Year1/c\Year1 = $year1" $path_pre/Namelist.py
sed -i "/^Year2/c\Year2 = $year2" $path_pre/Namelist.py
# [4] Set model name and input file label
sed -i "/^Model/c\Model = '$model'" $path_pre/Namelist.py
sed -i "/^TCGIinput/c\TCGIinput = 'TCGI_CRH_PI'" $path_pre/Namelist.py  # or adapt if variable


#     Link A_*YYYYMM.nc: covariance matrix
# For users who don't have daily U, V data, you need to chat with Chia-Ying what data your can use. Here, I used existing wind covarience data (named A_*YYYYMM.nc) calculated using CMIP6-CESM2 !!! ⚠️ 

if [ "$reso" == "Amon" ]; then
    sed -i 's/calWind = True/calWind = False/g' Namelist.py
    sed -i 's/calA = True/calA = False/g' Namelist.py
    # Link A from other locations:
    ln -sf $path_windcov/A*.nc $path_pre/ # Modify to your own path !!! ⚠️ 
fi

#     Get YYYY*r1i1p1f1.nc >> True calculation:
#      Update the Namelist in the pre folder and run CHAZ.py to get YYYY*r1i1p1f1.nc

pwd

echo Wait .... now is calculating YYYY.$imem.nc
python CHAZ.py

#    Run bias correction:
echo Wait .... now is running get_MeanStd.py
python getMeanStd.py

####################################################
# CHAZ downscaling                              ####
# to get tc nc data                             ####
####################################################
# 1. Link PI.mat, TCGI.mat & A.mat data to the wdir path
ln -sf $path_pre/PI*.mat $path_wdir/
ln -sf $path_pre/TCGI*.mat $path_wdir/
ln -sf $path_pre/A*.nc $path_wdir/
ln -sf $path_pre/*_${imem}.nc $path_wdir/
ln -sf $path_pre/coefficient_meanstd.nc $path_wdir/
ln -sf $path_pre/trackPredictorsbt_obs.pik $path_wdir/

# 2. Go to wdir to run downscaling;  Modify Namelist.py to get ready for CHAZ downscaling 
cp "$pth_chaz_src/src_nodaily"/* "$path_wdir"
cd "$path_wdir"
rm -rf "$path_wdir/__pycache__"

# Modify Namelist.py by using linux command sed-i:
# [1] Modify execution flags: set runCHAZ=True, runPreprocess=False 
sed -i '/runCHAZ/c\runCHAZ=True' $path_wdir/Namelist.py
sed -i '/runPreprocess/c\runPreprocess=False' $path_wdir/Namelist.py
# [2] Set ensemble member ID
sed -i "/^ENS/c\ENS = '$imem'" $path_wdir/Namelist.py 
# [3] Set simulation period based on forcing type
sed -i "/^Year1/c\Year1 = $year1" $path_wdir/Namelist.py
sed -i "/^Year2/c\Year2 = $year2" $path_wdir/Namelist.py

# [4] Set model name and input file label
sed -i "/^Model/c\Model = '$model'" $path_wdir/Namelist.py
sed -i "/^TCGIinput/c\TCGIinput = 'TCGI_CRH_PI'" $path_wdir/Namelist.py  # or adapt if variable

# 3. Downscaling with CHAZ:
pwd
echo Wait .... now is running downscaling
python CHAZ.py

# 4. Data modification:
python rev.pik2netcdf_merge_2100.py
echo Done, check the necfile output
```

