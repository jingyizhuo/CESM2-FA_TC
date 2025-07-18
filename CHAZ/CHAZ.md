# 🌎 👉 🌀 TC Downscaling from CESM2/CESM2-FA Using CHAZ

## 1️⃣ CHAZ Source Code

- `/data0/jzhuo/tc_risk/chaz_src/src` — for runs **with daily** data  
- `/data0/jzhuo/tc_risk/chaz_src/src_nodaily` — for runs **with only monthly** data

⚠️ Please **do not modify** the original source code directly.
The source code provided here has been adapted for:
1. Compatibility with **Python 3** (*Quick note: Many modifications have been made to ensure CHAZ running wiht Python v3, many other modificaitons are also made to ensure scripts running, but it has been a while, I don't remember exactly what modifications I've made*)
2. Support for **CESM2/CESM2-FA input formats**, ensuring that PI and TCGI .mat files use `'lon'` and `'lat'` as axes and have a shape of `(91, 180)`

### 📚 CHAZ Tutorials
I have also created two CHAZ tutorials by converting Chia-Ying's source code into Jupyter notebooks, which can be run directly in your browser via Google Colab. 

- [CHAZ Preprocessing Tutorial](https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/Tutorial_chaz_pre.ipynb)  
- [CHAZ Downscaling Tutorial](https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/Turtorial_chaz_downscaling.ipynb)
- You can also find the same jupyter notebooks that are uploaded to [CHAZ github](https://github.com/cl3225/CHAZ/tree/main)

These tutorials are designed to help users understand and reproduce the CHAZ workflow. **However, running CHAZ on LDEO-Taroko or other local machines will still require following my step-by-step instructions below.**

## 2️⃣ Input Data Preparation & Working Directory Setup
1. Prepare a `.csv` file specifying the paths to all CHAZ input files:  
   `/data0/jzhuo/tc_risk/CESM2/CHAZ/input_data/cesm2_cesm2fa_chaz_data.csv`
   
2. Create a working directory for each case under:  
   `/data0/jzhuo/tc_risk/CESM2/CHAZ/work/`
  
## 3️⃣ Running CHAZ

📝 CHAZ is controlled via `Namelist.py`, which determines whether to run the **preprocessing** or **downscaling** steps and defines the paths to required input data.  
Please review the file at `/data0/jzhuo/tc_risk/chaz_src/src_nodaily/Namelist.py` to get a sense of how the script is configured.

⚠️ The current version of CHAZ does **not** include TCGI and PI calculations — you will need to generate these datasets separately (see [TCGI.md](https://github.com/jingyizhuo/CESM2-FA_TC/blob/main/CHAZ/TCGI.md)) and link them into the CHAZ working directory using `ln -sf` (this step is already handled in the `main.sh` script below).

✅ The following `main.sh` script implements a complete, reproducible workflow for running CHAZ from preprocessing to downscaling. Copy and paste the script into the directory where you want to place `main.sh`, then run:

```bash
bash main.sh
```


```bash
#!/bin/bash
# Author: Jingyi zhuo (jz4351@princeton.edu)

# Set up working path
root_work=/data0/jzhuo/tc_risk/CESM2/CHAZ/work 
root_PI=/data0/jzhuo/tc_risk/CESM2/data_PI
root_TCGI=/data0/jzhuo/tc_risk/CESM2/data_TCGI
pth_chaz_src=/data0/jzhuo/tc_risk/chaz_src
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
    year1=1965
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
# bash /data0/jzhuo/tc_risk/CESM2/code_TCGI/main*.sh 
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
##     Note here we use the A from the existing data calculated using CMIP6-CESM2 for not saving daily u, v at the beginning
if [ "$reso" == "Amon" ]; then
    sed -i 's/calWind = True/calWind = False/g' Namelist.py
    sed -i 's/calA = True/calA = False/g' Namelist.py
    # Link A from other locations:
    ln -sf /data0/clee/CMIP6/CESM2/r4i1p1f1/pre/A*.nc $path_pre/
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
