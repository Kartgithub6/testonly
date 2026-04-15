# testonly
# CoSES_Thermal_ProHMo_PHiL

**Thermo-Energetic Analysis of Multi-Zone Buildings using Digital Twin Power Hardware-in-the-Loop Environments**

Master's Thesis — Karthik Murugesan  
Chair of Renewable and Sustainable Energy Systems, Technical University of Munich  
Supervisor: Ulrich Ganslmeier M.Sc. | Advisor: Prof. Dr. rer. nat. Thomas Hamacher  
Submitted: March 31, 2026

---

## Overview

This repository contains the complete source code for a multi-zone residential building thermal digital twin developed for Power Hardware-in-the-Loop (PHiL) experiments at the [CoSES Smart Grid Laboratory](https://www.ei.tum.de/en/ens/research/coses/), TU Munich (Garching).

The core model, **ThreeZoneBuilding_PHiL**, represents a three-zone residential building (living area, cellar, roof office) connected to a shared hydronic heating circuit. It is validated across three simulation platforms — Dymola, NI VeriStand (real-time PHiL), and SimulationX — and exported as an FMI 2.0 Co-Simulation FMU for hardware deployment.

### Key Contributions

| # | Contribution |
|---|---|
| 1 | Multi-zone Modelica building model (`ThreeZoneBuilding_PHiL`) using IBPSA FourElements RC network |
| 2 | Zone-level hydronic heating with EN 442-2 radiators and PI temperature controllers |
| 3 | FMU export (FMI 2.0, Co-Simulation, 32-bit) and real-time deployment in NI VeriStand 2018 |
| 4 | Custom C wrapper DLL (`fmu_veristand_wrapper.c`) with SEH auto-recovery for VeriStand integration |
| 5 | Cross-platform validation against CoSES SimulationX SF1 reference model |
| 6 | Python/Flask automation framework (`CoSES Project Builder`) for generating Dymola simulation variants |

---

## Repository Structure

```
CoSES_Thermal_ProHMo_PHiL/
│
├── package.mo                        # Root Modelica package (load this in Dymola)
├── package.order                     # Package load order
│
├── BuildingSystem/                   # Core thermal zone and building models
│   ├── HeatedZone.mo                 # Single-zone IBPSA FourElements RC model
│   ├── ThreeZoneBuilding_optimized.mo
│   └── ThreeZoneBuilding_PHiL_optimized_withWeather.mo  # PHiL-ready model (FMU source)
│
├── HydronicSystem/                   # Hydronic heating sub-system
│   └── SystemWithZoneAndHydronics_Optimized.mo  # Radiator + valve + PI controller per zone
│
├── Examples/                         # Simulation test harnesses
│   ├── TestThreeZoneBuilding_withWeather.mo      # Scenario A (dynamic 24-hour)
│   └── TestThreeZoneBuilding_StaticInputs.mo     # Scenario B (VeriStand-aligned static)
│
├── BoundaryConditions/               # Weather and disturbance signal wrappers
├── WeatherData/                      # Weather file interfaces (IBPSA .mos format)
├── HeatPorts/                        # Thermal connector definitions
├── Interface/                        # FMU scalar I/O interface (PHiL wrapper layer)
├── Interfaces/                       # Modelica connector definitions
├── ThermoEnergeticAnalysis/          # Post-processing models and energy balance tools
│
├── Python Builder/                   # Flask-based automation framework
│   ├── CoSES_Thermal_ProHMo_PHiL.py  # Main Flask server (localhost:5000)
│   ├── CoSES_Complete_Workflow.py    # HTML UI template and configuration interface
│   └── perfect_generator.py         # PerfectDymolaVariantGenerator class
│
├── Data/                             # Simulation input data
│   └── SimulationData/building/inner_loads/zone_x.txt  # Occupancy/load profiles
│
├── Plots/                            # Output plot scripts and generated figures
├── Backup/                           # Model version backups
├── Storage.mo                        # Thermal storage component
├── Consumer.mo                       # Consumer-side interface
├── HeatGenerators.mo                 # Heat generator models
├── Scenarios.mo                      # Scenario configuration
├── WeatherData.mo                    # Weather data interface
├── Building_CHP_CB_PHiL.bak.mo       # Archived earlier model variant
├── ConvertFromCoSES_Thermal_ProHMo_PHiL.mos  # Dymola conversion script
│
├── blob/main/data/
│   └── veristand_model_export.pdf    # Step-by-step VeriStand FMU import guide
│
├── dsmodel.c                         # Auto-generated Dymola C model code
├── dsfinal.txt                       # Final simulation states (Dymola)
└── buildlog.txt                      # Dymola build log
```

---

## Model Description

### Building Zones

| Zone | Floor Area | Height | Volume | Setpoint | T_init |
|------|-----------|--------|--------|----------|--------|
| Living | 100 m² | 2.5 m | 250 m³ | 20 °C | 20 °C |
| Cellar | 80 m² | 2.2 m | 176 m³ | 18 °C | 18 °C |
| Roof Office | 60 m² | 2.3 m | 138 m³ | 19 °C | 17 °C |

Total floor area: **240 m²** | Specific heating load: ~26.1 W/m²

### Envelope Parameters (all zones, uniform)

| Parameter | Value |
|-----------|-------|
| Wall U-value | 0.20 W/(m²·K) |
| Window U-value | 1.0 W/(m²·K) |
| Window g-value | 0.60 |
| Interior convection α | 7.5 W/(m²·K) |
| Exterior convection α | 20.0 W/(m²·K) |
| Wall thickness | 0.36 m |
| Wall density | 1800 kg/m³ |
| Wall specific heat | 920 J/(kg·K) |

### Hydronic System (EN 442-2 Radiators)

| Parameter | Roof | Living | Cellar |
|-----------|------|--------|--------|
| Nominal power | 2000 W | 6000 W | 1500 W |
| Nominal supply temp | 70 °C | 70 °C | 70 °C |
| Nominal return temp | 50 °C | 50 °C | 50 °C |
| Nominal flow | 0.04 kg/s | 0.06 kg/s | 0.04 kg/s |
| Min valve opening | 2 % | 2 % | 1 % |
| Radiative fraction | 0.35 | 0.35 | 0.35 |

### PI Controller Parameters (all zones)

| kP | Ti | ymax |
|----|----|------|
| 0.3 | 500 s | 1.0 |

---

## FMU / VeriStand Integration

### FMU Export (must be repeated after every model change)

1. Open `package.mo` in **Dymola 2023x** (network license: `10.162.231.89`)
2. Translate the model: `F8`
3. Export FMU: `Simulation → Export → FMU`
   - FMI version: **2.0**
   - Type: **Co-Simulation**
   - Compiler flag: **`/arch:IA32`** ← critical for 32-bit VeriStand 2018
4. Unzip the exported `.fmu` file
5. Extract `binaries/win32/*.dll`
6. Rename to `tzb_fmu.dll`
7. Copy to: `D:\coses_main\Veristand\Models\HostPC\`

### VeriStand Deployment

8. Recompile the C wrapper: `fmu_veristand_wrapper.c`
9. Open `D:\coses_main\Veristand\CoSES_Smart_Grid.nivsproj` in VeriStand System Explorer
10. Reload the DLL under *Simulation Models*
11. Remap all **9 inports** and **11 outports**

> **Important:** Set the environment variable `DYMOLA_RUNTIME_LICENSE` before launching VeriStand if the FMU requires a runtime license.

### FMU Inports (9)

| Signal Name | Value (Scenario B) | Description |
|-------------|-------------------|-------------|
| `STM_HCVLaM_degC` | 50 °C | Supply temperature |
| `SFW_HCRLbM_l_per_min` | 12 l/min | Flow rate (reference input) |
| `T_ambient_degC` | 2 °C | Ambient temperature |
| `nPersons_living_in` | 3 | Living zone occupancy |
| `nPersons_roof_in` | 1 | Roof zone occupancy |
| `nPersons_cellar_in` | 1 | Cellar zone occupancy |
| `P_appliances_living_W_in` | 500 W | Living appliance load |
| `P_appliances_roof_W_in` | 200 W | Roof appliance load |
| `P_appliances_cellar_W_in` | 100 W | Cellar appliance load |

### FMU Outports (11)

| Signal Name | Steady-State Value | Description |
|-------------|-------------------|-------------|
| `T_roomIs_degC` | 21.0 °C | Living zone temperature |
| `T_roofIs_degC` | 18.1 °C | Roof zone temperature |
| `T_cellarIs_degC` | 17.8 °C | Cellar zone temperature |
| `STM_HCRL_Set_degC` | 48.4 °C | Return water temperature |
| `SFW_HCRLbM_Set_l_per_min` | 28.4 l/min | Reference volume flow |
| `Q_flow_living_W` | 323.2 W | Heat flow — living zone |
| `Q_flow_roof_W` | 323.2 W | Heat flow — roof zone |
| `Q_flow_cellar_W` | 323.2 W | Heat flow — cellar zone |
| `valve_living_opening` | 0.1 | Living valve position [0–1] |
| `valve_cellar_opening` | 1.0 | Cellar valve position [0–1] |
| `valve_roof_opening` | 1.0 | Roof valve position [0–1] |

### Critical Technical Notes

- **Solver**: Use **DASSL** (not CVODE). CVODE crashes at t = 0.01 s on the first `DoStep` call in VeriStand.
- **Energy dynamics**: Must be `FixedInitial` (not `SteadyStateInitial`) in the FMU-exported version to prevent 0 °C zone initialization.
- **Supply temperature init**: `STM_HCVLaM_degC(start=50)` — do not set to 60.
- **Zone init temperatures**: Set `TZoneInit` values 2–3 °C **below** setpoints to avoid cold-start transients that exceed the simulation window.
- **Flow rate gain**: `k = 0.000222 m³/s` in the three Gain blocks (≈ 12 l/min at typical valve positions). The `flowConvertOut` block uses `k = 60000`.
- **FMI function prefix**: Leave the prefix string **empty** (unprefixed names: `fmi2Instantiate`, `fmi2DoStep`, etc.) — this is a consequence of the `/arch:IA32` flag.
- **SEH wrapper**: The C wrapper includes `__try/__except` auto-recovery around `fmi2DoStep` to handle structured exceptions from the 32-bit FMU memory alignment issue.
- **Diagnostic log**: FMU execution is logged to `C:\temp\tzb.txt` for offline verification.

---

## Python Automation Framework (CoSES Project Builder)

The Flask-based tool automates Dymola project variant generation. It eliminates manual `.mo` file editing by applying regex substitution to inject user-specified parameters.

### Starting the Server

```bash
cd "Python Builder"
pip install flask scipy matplotlib numpy
python CoSES_Thermal_ProHMo_PHiL.py
# Open browser: http://localhost:5000
```

### Workflow

1. **Select building type** (9 presets from Small House 70 m² to Mall 500 m²) or use Custom
2. **Set project name**, simulation duration, and weather location
3. Click **Step 1: Generate Project** → creates timestamped Dymola project folder and launches Dymola
4. Press **F9** in Dymola to simulate → writes `.mat` result file
5. Click **Step 2: Generate 9 Plots** → reads `.mat` and renders all output figures in browser

### Building Presets

| Building Type | Living [m²] | Cellar [m²] | Roof [m²] | kP | Ti [s] |
|---------------|-------------|-------------|-----------|-----|--------|
| Small House | 70 | 50 | 40 | 0.35 | 400 |
| Medium House | 100 | 80 | 60 | 0.30 | 500 |
| Large House | 130 | 100 | 80 | 0.25 | 600 |
| Office | 150 | 100 | 80 | 0.25 | 600 |
| School | 200 | 120 | 100 | 0.20 | 700 |
| Factory | 250 | 150 | 120 | 0.20 | 700 |
| Hotel | 300 | 200 | 150 | 0.18 | 800 |
| Hospital | 400 | 150 | 200 | 0.18 | 800 |
| Mall | 500 | 300 | 250 | 0.15 | 900 |

### Key Known Issues Fixed

- `buildingData` parameter not updating: required a `redeclare` regex branch for `Examples/` files
- Dynamic package prefix extraction: the generator now reads the prefix from `package.mo` rather than hardcoding it

---

## Simulation Scenarios

### Scenario A — Dynamic 24-Hour (Dymola)

Uses CombiTimeTable input blocks with IBPSA Munich weather data (`DEU_Munich.108660_IWEC.mos`), occupancy schedules (peak 3 persons living room at 21:00), and appliance profiles (750 W living room 18:00–23:00).

- Solver: DASSL | Tolerance: 1e-6 | Step output: 60 s
- Supply temperature: 45–50 °C (morning boost 08:00–18:00)

### Scenario B — Static Inputs (VeriStand-aligned, Dymola)

All CombiTimeTable blocks replaced by `Modelica.Blocks.Sources.Constant` blocks at values matching the VeriStand workspace (Table 5.4 in thesis). Used for direct platform comparison.

---

## Key File Paths (CoSES Lab Machine)

| Resource | Path |
|----------|------|
| Modelica source | `C:\Users\CoSES-Guest\Documents\Dymola\CoSES_Thermal_ProHMo_PHiL\` |
| VeriStand project | `D:\coses_main\Veristand\CoSES_Smart_Grid.nivsproj` |
| Wrapper DLL + FMU DLL | `D:\coses_main\Veristand\Models\HostPC\` |
| FMU debug log | `C:\temp\tzb.txt` |
| Dymola license | `C:\Users\CoSES-Guest\AppData\Roaming\DassaultSystemes\Dymola\dymola.lic` |
| License server | `10.162.231.89` |

---

## Dependencies and Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Dymola | 2023x Refresh 1 | Primary simulation environment, FMU export |
| NI VeriStand | 2018 | Real-time PHiL deployment |
| SimulationX | 4.7 | Cross-platform reference (SF1 model) |
| IBPSA Modelica Library | latest | FourElements zone model, EN 442-2 radiator |
| CoSES_thermal_ProHMo | TUM internal | Reference building parameters |
| Python | 3.14.0 | Post-processing, automation framework |
| Flask | latest | Web UI for automation tool |
| SciPy | latest | MAT file reading, interpolation |
| Matplotlib / NumPy | latest | Plot generation |
| MSVC (32-bit) | — | C wrapper compilation (`/arch:IA32`) |

### Modelica Library Dependencies

The `package.mo` requires both libraries to be loaded before translating:

```
IBPSA          → https://github.com/ibpsa/modelica-ibpsa
CoSES_thermal_ProHMo → https://github.com/DZinsmeister/CoSES_thermal_ProHMo
```

---

## Cross-Platform Validation Results (Summary)

| Signal | VeriStand (PHiL) | Dymola (Scenario A) | SimulationX (SF1) |
|--------|-----------------|---------------------|-------------------|
| T_living | 21.0 °C | 19–21 °C | 21.0 °C |
| T_cellar | 17.8 °C | ~12 °C* | 17–19 °C |
| T_roof | 18.1 °C | 16–18 °C | 15–18 °C |
| T_return | 48.4 °C | 39–48 °C | 22–47 °C |
| Volume flow | 28.4 l/min** | 1–13 l/min | 1–7 l/min |
| FMU step time | 2–4 ms / 60 s step | — | — |

*Cellar underheating in Dymola is a controller tuning issue (valve at minimum opening), not a zone model defect.  
**VeriStand flow is a proportional monitoring estimate from PI outputs, not a physical hydraulic measurement.

Living zone agreement (±1 °C of 20 °C setpoint) across all three platforms represents the primary quantitative validation result.

---

## Known Limitations and Open Issues

- **Cellar underheating**: The cellar valve saturates at its minimum opening under 45–50 °C supply. Raising the minimum valve opening from 1 % to 5 % and implementing weather-compensated supply temperature scheduling is recommended.
- **Energy balance chart**: Values in `coses_energy_balance.py` appear ~300–500× too large due to a suspected Kelvin vs. Celsius unit mismatch in MAT file signal reading.
- **Parameter discrepancies** (thesis vs. model):
  - Living zone nominal power: 6000 W in model vs. 3500 W in Table 4.5
  - Cellar appliance load: 500 W (calibrated correct value) vs. 50 W in Table 4.4
  - Roof zone setpoint: 19 °C in model vs. 20 °C referenced in text
  - Scenario B supply temp: conflict between Table 5.4 (50 °C) and Figure A.2 annotation
- **No dynamic VeriStand validation**: Scenario A time-varying inputs were not replayed through VeriStand due to workspace constraints. A LabVIEW-timed CSV playback channel is the recommended path forward.
- **Simplified hydraulics**: Valve pressure-drop characteristics and pump curves are not explicitly modelled; the `qvRef` output is a linear PI-output estimate only.

---

## Future Work

- Dynamic PHiL validation with RMSE and cross-correlation metrics over a full 24-hour cycle
- Cellar controller retuning (min valve opening ↑, weather-compensated supply schedule)
- Heat source coupling (boiler / heat pump / NeoTower2 CHP) at the supply side
- Model Predictive Control (MPC) layer using weather-forecast and occupancy-forecast inputs
- Multi-building network via ProsNet library integration
- Uncertainty quantification on envelope parameters (U-value, thermal mass, infiltration rate)

---

## Repository

**GitHub**: https://github.com/Kartgithub6/CoSES_Thermal_ProHMo_PHiL

**VeriStand export guide**: `blob/main/data/veristand_model_export.pdf`

---

## Citation

If you use this model or framework, please cite:

> Murugesan, K. (2026). *Thermo-Energetic Analysis of Multi-Zone Buildings using Digital Twin Power Hardware-in-the-Loop Environments*. Master's Thesis, Chair of Renewable and Sustainable Energy Systems, Technical University of Munich.

---

## License

This repository is shared for research and instructional purposes at the Chair of Renewable and Sustainable Energy Systems, TUM, per the Declaration for Transfer of Thesis. Copyright and personal right of use remain with the author.
