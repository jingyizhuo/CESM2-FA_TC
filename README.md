# TC downscaling based on CHAZ
This is a notebook for running The Columbia TC hazard model CHAZ - a statistical-dynamical downscaling model which uses large-scale conditions representing the atmospheric dynamic and thermodynamic environment from a global model to predict the genesis, tracks and intensities of synthetic TCs. For more details of the CHAZ model please refer to [Lee et al. 2018](https://doi.org/10.1002/2017MS001186). 

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
V_{track} = \alpha V_{850} + (1 - \alpha)V_{250} + V_{\beta}
$$

$$
\quad \alpha = 0.8, \quad V_{\beta} \text{  is the beta drift vector which is a function of latitude}
$$

### Intensity
$$
v_{t+\Delta t} - v_t = MLR(X_t, X_{t+\Delta t}, v_t, v_{t-\Delta t}) + \varepsilon_t
$$

Predictors: MONTHLY wind (vorticity, shear, steering flow); temperature & moisture (PI, humidity or/and saturation deficit); \varepsilon_t is stochastic forcing component accounts, in a statistically representative sense, for the internal storm dynamics or other physical processes that do not depend explicitly on the environment.

## Workflow of the CHAZ model
![flow_chart](https://user-images.githubusercontent.com/46905677/126709479-ad3eab03-a4bd-4ea5-a85b-79f1a83bed83.png)


## Quick Start: Run CHAZ
Here is a quick start command list to get the model running (TBD)

## Environment Set Up


## Input Data

### ERA5

### Climate model simulations

### Formating Input

## Running the Model

## Model Output
