# 🌎 👉 🌀 TC downscaling from CESM2 and CESM2-FA by CHAZ 

## 1️⃣ CHAZ source code
### ⚠️ Do not modify the original source code directly.
- `/data0/jzhuo/tc_risk/chaz_src/src` — for runs **with daily** U and V wind data  
- `/data0/jzhuo/tc_risk/chaz_src/src_nodaily` — for runs **with only monthly** U and V wind data  
  
## 2️⃣ Input data preparation
1. Prepare a `.csv` file specifying the paths to all CHAZ input files:  
   `/data0/jzhuo/tc_risk/CESM2/CHAZ/input_data/cesm2_cesm2fa_chaz_data.csv`

2. Create symbolic links for PI and TCGI input files:  
   > This is one inelegant aspect of CHAZ — it relies on symbolic links to organize input files, which can make the setup messy.  
   > However, this step is necessary unless you plan to modify the CHAZ source code yourself.
