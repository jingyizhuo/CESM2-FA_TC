# 🌎 👉 🌀 TC downscaling from CESM2 and CESM2-FA by CHAZ 

## 1️⃣ CHAZ source code
### ⚠️ Do not modify the original source code directly.
*These src odes have been modified to fit for (1) python version3 and for (2) the input data format of CESM2/CESM2-FA (i.e., make sure the PI and TCGI's x axis, y axis are 'lon', 'lat', and the 2D 2deg variable shape is (91, 180).*
- `/data0/jzhuo/tc_risk/chaz_src/src` — for runs **with daily** U and V wind data  
- `/data0/jzhuo/tc_risk/chaz_src/src_nodaily` — for runs **with only monthly** U and V wind data  
  
## 2️⃣ Input data preparation and a create working folder：
1. Prepare a `.csv` file specifying the paths to all CHAZ input files:  
   `/data0/jzhuo/tc_risk/CESM2/CHAZ/input_data/cesm2_cesm2fa_chaz_data.csv`

2. Create symbolic links for PI and TCGI input files:
   > This is one inelegant aspect of CHAZ — it relies on symbolic links to organize input files, which can make the setup messy.  
   > However, this step is necessary unless you plan to modify the CHAZ source code yourself.

Example code, which has been integrated into the `main.sh`:
```bash
for file in $path_PI/PI_${model}_${forcing}_${imem}_*.mat; do
    year=$(basename $file | sed -n 's/.*_\([0-9]\{4\}\).mat/\1/p')
    ln -sf $file $path_pre/PI_$year.mat
done
```



## 3️⃣ Run CHAZ Preprocessing： 
```
bash main.sh
```

```bash
#!/bin/bash
path_work=/home/jzhuo/chaz/run_CESM2_FLX
cd $path_work

####################################################
# Basic settings                                ####
####################################################
imem=r1i2p1f1
model=CESM2_FLX
forcing=HIST
reso=Amon
TCGI=TCGI_CRH_PI

# Create folders:
path_pre=/data0/jzhuo/tc_risk/chaz/$model/$imem/pre
path_wdir=/data0/jzhuo/tc_risk/chaz/$model/$imem/wdir
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

# 1. Calculate PI & TCGI, link to the pre folder:
#     python cal_PI_TCGI.py  >> Done on Perlmutter, and save in the format of ##.mat, then move data from Perlmutter to taroko
path_PI=/data0/jzhuo/tc_risk/PI/$model/$forcing/$imem

#    link PI to path_pre, and rename the PI_*.mat files from PI_CESM2_FLX_HIST_r1i1p1f1_YYYY.mat to PI_YYYY.mat
for file in $path_PI/PI_${model}_${forcing}_${imem}_*.mat; do
    year=$(basename $file | sed -n 's/.*_\([0-9]\{4\}\).mat/\1/p')
    ln -sf $file $path_pre/PI_$year.mat
done

# 2. Calculate TCGI
#     python cal_PI_TCGI.py  >> Done on Perlmutter, and save in the format of ##.mat, then move data from Perlmutter to taroko
path_TCGI=/data0/jzhuo/tc_risk/$TCGI/$model/$forcing/$imem
#    link TCGI to path_pre, and rename the TCGI_*.mat files from TCGI_CRH_PI_CESM2_FLX_HIST_r1i1p1f1_YYYY.mat to TCGI_CRH_PI_YYYY.mat
for file in $path_TCGI/${TCGI}_${model}_${forcing}_${imem}_*.mat; do
    year=$(basename $file | sed -n 's/.*_\([0-9]\{4\}\).mat/\1/p')
    ln -sf $file $path_pre/${TCGI}_$year.mat
done

# 3. Run CHAZ.py to get A_*YYYYMM.nc & YYYY*r1i1p1f1.nc:
#     Set runCHAZ=False, runPreprocess=True 
sed -i '/runCHAZ/c\runCHAZ=False' Namelist.py
sed -i '/runPreprocess/c\runPreprocess=True' Namelist.py
sed -i "/^ENS/c\ENS = '$imem'" Namelist.py

#     Get A_*YYYYMM.nc: covariance matrix
if [ "$reso" == "Amon" ]; then
    sed -i 's/calWind = True/calWind = False/g' Namelist.py
    sed -i 's/calA = True/calA = False/g' Namelist.py
    # Link A from other locations:
    path_cesm2_daily=/data0/clee/CMIP6/CESM2/r4i1p1f1/pre
    ln -sf $path_cesm2_daily/A*.nc $path_pre/
fi

#     Get YYYY*r1i1p1f1.nc >> True calculation:
#      Update the Namelist in the pre folder and run CHAZ.py to get YYYY*r1i1p1f1.nc
cp /home/jzhuo/chaz/src_nodaily/* $path_pre
#     Updata Namelist
cp ./Namelist.py $path_pre

cd $path_pre
echo Wait .... now is calculating YYYY.$imem.nc
python CHAZ.py

#    Run bias correction:
echo Wait .... now is running get_MeanStd.py
python getMeanStd.py

####################################################
# CHAZ downscaling                              ####
# CHAZ downscaling                              ####
####################################################
# 1. Link PI.mat, TCGI.mat & A.mat data to the wdir path
ln -sf $path_pre/PI*.mat $path_wdir/
ln -sf $path_pre/TCGI*.mat $path_wdir/
ln -sf $path_pre/A*.nc $path_wdir/
ln -sf $path_pre/*_${imem}.nc $path_wdir/
ln -sf $path_pre/coefficient_meanstd.nc $path_wdir/
ln -sf $path_pre/trackPredictorsbt_obs.pik $path_wdir/

# 2. Go to wdir to run downscaling;  Modify Namelist.py to get ready for CHAZ downscaling 
cd $path_work
sed -i '/runCHAZ/c\runCHAZ=True' Namelist.py
sed -i '/runPreprocess/c\runPreprocess=False' Namelist.py
sed -i "/^ENS/c\ENS = '$imem'" Namelist.py

cp /home/jzhuo/chaz/src_nodaily/* $path_wdir
#     Updata Namelist
cp ./Namelist.py $path_wdir

# 3. Downscaling with CHAZ:
cd $path_wdir
echo Wait .... now is running downscaling
python CHAZ.py

# 4. Data modification:
python rev.pik2netcdf_merge_2100.py
echo Done, check the necfile output
```
