# TC downscaling based on CHAZ
This is a notebook for running The Columbia TC hazard model CHAZ - a statistical-dynamical downscaling model which uses large-scale conditions representing the atmospheric dynamic and thermodynamic environment from a global model to predict the genesis, tracks and intensities of synthetic TCs. (Lee et al. 2018, JAMES)

## Three elements of the CHAZ model
### Genesis
$$
\mu_{\text{CRH}} = \exp\left(b + b_{\eta} \eta_{850} + b_{\text{rh}} \text{CRH} + b_{\text{PI}} \text{PI} + b_{\text{SHR}} \text{SHR}\right)
$$

$$
\mu_{\text{SD}} = \exp\left(b + b_{\eta} \eta_{850} + b_{\text{SD}} \text{SD} + b_{\text{PI}} \text{PI} + b_{\text{SHR}} \text{SHR}\right)
$$

### Track
$$
V_{track} = \alpha V_{850} + (1 - \alpha)V_{250} + V_{\beta}, \quad \alpha = 0.8, \quad V_{\beta} \text{ is a function of latitude}
$$

### Intensity
$$
v_{t+\Delta t} - v_t = MLR(X_t, X_{t+\Delta t}, v_t, v_{t-\Delta t}) + \varepsilon_t
$$

Predictors: MONTHLY wind (vorticity, shear, steering flow); temperature & moisture (PI, humidity or/and saturation deficit)
## Environment Set Up


## Input Data

### ERA5

### Climate model simulations

### Formating Input

## Running the Model

## Model Output
