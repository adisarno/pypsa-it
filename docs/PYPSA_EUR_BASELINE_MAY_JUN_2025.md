# PyPSA-Eur reduced Italian market baseline - May-June 2025

Run: reduced Italy-plus-neighbours hourly baseline, May-June 2025

## Scope and configuration

The run is an hourly, electricity-only operational backtest for 2025-05-01 00:00 through 2025-06-30 23:00 (1,464 snapshots). It keeps Italy's seven bidding zones and uses one bus each for France, Switzerland, Austria, and Slovenia. France is set to administrative level 0 so Corsica (`FR15AC`) and mainland France share the single French market-price node. The resulting clustered network has 11 buses.

The overlay is `config/it_market_backtest_may_jun_2025.yaml`. It retains PyPSA-Eur's default technology and operating-cost inputs, fixes generation and transmission capacities for the historical dispatch, disables non-electricity sectors, and uses HiGHS simplex for reliable LP dual prices. No GME offers, awards, or prices enter the solve.

Historical electricity demand is built from the archived ENTSO-E/OPSD/NESO inputs. The synthetic load archive ends in 2023, so `load.supplement_synthetic` is disabled. ENTSO-E data cover the requested 2025 hours for all selected countries; three Slovenian missing hours are filled by PyPSA-Eur's standard gap interpolation. The 2025 ERA5 cutout was retrieved (6.7 GB).

## Compatibility fixes

The first demand build failed because the workflow attempted `.loc` on 2025 timestamps in `load_synthetic_raw.csv`, which only runs through 2023. The overlay now limits the countries to the agreed Italy-plus-border scope and disables that out-of-range supplement.

The first solve failed in `scripts/solve_network.py`: with pandas 3.0.1, empty `n.stores.index` is integer-typed, so `.index.str` raises an exception. String matching in the optional storage/CHP helper functions now casts component indexes to strings first. This changes no model constraints or assumptions.

The first HiGHS IPM attempt finished with status `Unknown` because IPM had crossover disabled and its postsolve objective residual exceeded the configured check. Re-solving the identical network with the existing PyPSA-Eur `highs-simplex` profile yielded `Optimal` for both the planning-stage fixed-capacity solve and the operational solve. The pooled mean absolute price difference from the IPM output is 0.0024 EUR/MWh, with a largest single zone-hour difference of 1.67 EUR/MWh; the pooled RMSE is unchanged to the displayed precision. Use the simplex result below.

## Model inputs actually used for marginal costs

`conventional.dynamic_fuel_price` is `false`; the model therefore does not use the World Bank monthly oil/gas/coal series. It uses the 2025 static technology-data table at `resources/it-border-backtest-may-jun-2025-v3/costs_2025_processed.csv`. For CCGT, the used row has fuel 42.9003 EUR/MWh-thermal, efficiency 0.57, VOM 5.744 EUR/MWh-electric, and a derived marginal cost of 81.0077 EUR/MWh-electric before carbon.

`costs.emission_prices.enable` is `false`, so CO2 cost is zero in the solve. Thus this baseline is PyPSA-Eur's default techno-economic dispatch, not an Italian 2025 actual-commodity SRMC reconstruction. Official documentation: [technology cost assumptions](https://pypsa-eur.readthedocs.io/en/latest/costs/) and [cost and price workflow](https://pypsa-eur.readthedocs.io/en/latest/rules/costs/).

## MGP price backtest

The evaluator-only target is the observed hourly GME zonal price series for May-June 2025. It is joined only after solving; it is not a dispatch input. Coverage is complete for all seven zones: 1,464 hours x 7 zones = 10,248 zone-hours.

The GME data were joined only after the PyPSA solves, using UTC delivery-start timestamps. Mapping: `ITCAAC-CALA`, `ITCNAC-CNOR`, `ITCSAC-CSUD`, `ITN1AC-NORD`, `ITSAAC-SARD`, `ITSIAC-SICI`, `ITS1AC-SUD`.

| Metric | PyPSA-Eur vs GME actual |
|---|---:|
| Zone-hours | 10,248 |
| Mean model price | 73.62 EUR/MWh |
| Mean actual price | 102.83 EUR/MWh |
| Bias (model - actual) | -29.21 EUR/MWh |
| MAE | 42.09 EUR/MWh |
| RMSE | 50.12 EUR/MWh |
| Correlation | 0.093 |

| Zone | Actual mean | Model mean | Bias | MAE | RMSE | Correlation |
|---|---:|---:|---:|---:|---:|---:|
| CALA | 103.44 | 78.75 | -24.69 | 39.01 | 44.44 | 0.461 |
| CNOR | 102.91 | 83.26 | -19.65 | 33.50 | 38.73 | 0.555 |
| CSUD | 103.57 | 80.56 | -23.01 | 37.63 | 43.34 | 0.429 |
| NORD | 101.93 | 84.55 | -17.38 | 31.39 | 36.20 | 0.573 |
| SARD | 100.79 | 27.25 | -73.54 | 77.18 | 84.25 | 0.352 |
| SICI | 103.83 | 80.47 | -23.36 | 38.13 | 43.93 | 0.466 |
| SUD | 103.39 | 80.54 | -22.85 | 37.79 | 43.49 | 0.406 |

The model price trace is almost flat within zones while actual MGP prices show large hourly movement, including frequent low-price hours and evening peaks. Average prices and correlation therefore show that the untouched PyPSA-Eur cost-stack is a poor price reproducer for this Italian backtest. These scores do not establish that the physical fleet is wrong: the run has static generic fuel costs, no CO2 cost, and no market offer/commitment effects.

### SARD diagnostic

The model maps `ITSAAC` to the Sardinian bus (coordinates 9.06 E, 40.48 N). It has mean load of about 503 MW. The built network includes a 1,000 MW DC link from SARD to CSUD; it flows at its 1,000 MW maximum in all 1,464 hours. A second 300 MW DC link to France reaches its limit in 249 hours. The local 1,230 MW coal generator averages 1,220 MW dispatch at PyPSA-Eur marginal cost 26.09 EUR/MWh. This cheap coal and uncongested/favourable export stack explain why the SARD dual stays near 27 EUR/MWh in the model. It is a high-priority stack/network review item, not evidence by itself that the GME-to-bus mapping is wrong.

## Artifacts and reproduction

Generated charts, tables, and network files are reproducible run outputs under the corresponding `results/` directory and are ignored by Git.

The evaluator price comparison CSV and PNG are in `diagnostics_simplex`. The source GME data remain separate from the model inputs; no source dataset was modified. Do not use target GME prices as dispatch inputs in future forecasts.

Re-run from the repository with:

```bash
pixi run --frozen snakemake solve_operations_networks \
  --cores 4 --resources mem_mb=24000 \
  --set-resources solve_network:mem_mb=22000 solve_operations_network:mem_mb=22000 \
  --configfile=config/it_market_backtest_may_jun_2025.yaml
```

The repository changes for this run are limited to the overlay configuration, the empty-index compatibility fix, and this report. No test suite was run.
