# Prepare input data for CESM2

## Table of variables
| var | long name | units | dimensions | CESM output variable name | CMIP6 variable
|----------|-----------|-------|------------|--------------|-------|
|  u   | Zonal wind                          |  m/s	 | time lev lat lon | U   | ua
|  v   | Meridional wind                     |  m/s	 | time lev lat lon |  V  | va
|  t   | Temperature                         |   K   | time lev lat lon | T | ta
|  ts  | Surface temperature (radiative)     |   K   | time lat lon     | TS | ts 
|  rh  | Relative humidity                   |percent| time lev lat lon | RELHUM | hur
|  qa  | Specific humidity                   | kg/kg | time lev lat lon | Q | hus
|  slp |  Sea level pressure                 |  Pa	 | time lat lon     | PSL| psl |
|  cw |  Total (vertically integrated) precipitable water |  kg/m2	 | time lat lon     | TMQ| prw|


## CESM2 data processsing
```bash
#!/bin/bash

# Set the root directory, case name, and component
casename="cesm2.cntl.000"
root="/data0/jzhuo/CESM2/cesm2_flx_adj"

# Output directory and experiment ID
outexpid="r1i1p1f1"
outroot="/data0/jzhuo/CESM2_FLUXADJ_LE/control"

component="atm"
# Define start and end years
ys=1950
ye=2014

# List of variables to process
vars=(U V T TS RELHUM Q PSL TMQ)

# Loop through the years
for (( year=$ys; year<=$ye; year++ ))
do
    # Use a glob pattern to find matching files for each year
    for infile in ${root}/${casename}/${component}/hist/${casename}.cam.h0.${year}-??.nc
    do
        # Extract date info from infile path
        dateinfo=$(echo $infile | grep -oP '\d{4}-\d{2}.nc$')

        # Process each variable
        for var in "${vars[@]}"
        do
            outfile="${outroot}/${outexpid}/temp/${var}.${dateinfo}"
            echo "Processing $outfile"
            
            # Perform the ncremap operation
            # Note: You need to replace VAR with the actual variable in the ncremap command.
            # This might require a more complex substitution or a different approach in a real scenario.
            ncremap -v $var -d dat_dst.nc $infile $outfile
        done
    done
done

for var in "${vars[@]}"
do
    files="${outroot}/${outexpid}/temp/${var}.*.nc"
    combfile="${outroot}/${outexpid}/Amon/${var}.${ys}-${ye}.nc"
    cdo mergetime $files $combfile
done
rm -rf ${outroot}/${outexpid}/temp/*.nc
```

## CESM2 - AMIP data processsing
can be directly fetched via dods:
http://mary.ldeo.columbia.edu:81/CMIP6/.CMIP/.NCAR/.CESM2/.amip/.r1i1p1f1/.Amon/.psl/.gn/.v20190218/.psl_Amon_CESM2_amip_r1i1p1f1_gn_195001-201412.nc/.psl/dods![image](https://github.com/jingyizhuo/tc_downscaling/assets/141192552/6c1635b0-faa5-4928-8e76-851d25b7d600)

vars = ['', 'prw'
## Calculate TC related environmental variabels and TCGI:
Reference: https://github.com/YiXcite/TCGI/blob/main/cal_TCGI.ipynb
### [1] TC related environmental variabels x5
    "- Absolute Vorticity @ 850 hPa [s-1]\n",
    "- Vertical Wind Shear between 850- & 200-hPa [m s-1]\n",
    "- Column Relative Humidity\n",
    "- Saturation Deficit\n",
    "- Potential Intensity [m s-1]\n",
    "\n",
    "Just prepare a nc.file include avort, ws, PI, crh, sd, source data is u/v850/200, T3d, mslp, skin temp, total colum water",
    All the variables are very easy to derive, except for PI need to re-check

### [2] TCGI
$$
\mu_{\text{CRH}} = \exp\left(b + b_{\eta} \eta_{850} + b_{\text{rh}} \text{CRH} + b_{\text{PI}} \text{PI} + b_{\text{SHR}} \text{SHR}\right)
$$

$$
\mu_{\text{SD}} = \exp\left(b + b_{\eta} \eta_{850} + b_{\text{SD}} \text{SD} + b_{\text{PI}} \text{PI} + b_{\text{SHR}} \text{SHR}\right)
$$


mu_crh = np.exp(b_crh[0] + b_crh[1]*bc_now['avort'] + b_crh[2]*bc_now['crh'] + 
                            b_crh[3]*bc_now['PI'] + b_crh[4]*bc_now['ws']
