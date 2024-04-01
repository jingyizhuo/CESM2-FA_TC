# Check the CHAZ input data in CESM2

| notation | variable name | long name | units | dimensions | avaliability | note |
|----------|-----------|-------|------------|--------------|----------|------|
|  U   | U      | Zonal wind                          |  m/s	| time lev lat lon | yes | to get SHEAR & vorticity| 
|  V   | V      | Meridional wind                     |  m/s	| time lev lat lon | yes | to get V200 V850 | 
|  T   |  T     | Temperature                         |   K   | time lev lat lon | yes | for PI 
|  Ts  | TS     | Surface temperature (radiative)     |   K   | time lat lon     | yes |
|  RH  | RELHUM | Relative humidity                   |percent| time lev lat lon | yes | to get mean RH
|  qa  | Q      | Specific humidity                   | kg/kg | time lev lat lon | yes | use T to get qs then can get SD = qs - qa
| PSL  | PSL    | Sea level pressure                  |  Pa	  | time lat lon     | yes |
|  P   to get PI
| TMQ  |  TMQ   | Total (vertically integrated) precipitable water|kg/m2| time lat lon | yes |
