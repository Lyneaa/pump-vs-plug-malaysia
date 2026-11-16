**Vehicle Running Cost Prediction and Fuelling Infrastructure Route Optimisation across ICE, HEV and EVs**

WQD7001 project: a data-driven framework

The framework has two goals. The first is to estimate what a vehicle costs to run in Malaysia (RM per 100 km and per year) for any powertrain: petrol or diesel (ICE), mild hybrid (MHEV), full hybrid (HEV), plug-in hybrid (PHEV) and battery-electric (BEV). The second is to supply the vehicle energy and charging parameters that a charging and refuelling route optimiser needs.

This repository covers the first two steps. Notebook 1 merges and cleans two public datasets into one analysis-ready table. Notebook 2 explores that table and tests eight research questions about energy use, running cost and charging\*\* with the flagged outliers removed\*\*.

**Data sources (cars)**

| **Source** | **File** | **Coverage** | **Contributes** | **Limitations** |
| :-: | :-: | :-: | :-: | :-: |
| fueleconomy.gov (US EPA / DOE) | vehicles.csv | 50,407 configurations, model years 1984-2027, every powertrain | EPA-tested city, highway and combined fuel and electricity use, CO₂, EV range, US annual fuel cost, engine and transmission specs | US units; sentinel codes (-1, 0); no battery or charging specs; no engine power |
| OpenEV Data v1.24.0 | open-ev-data-v1.24.0.csv | 1,189 EV trims, 2023-2025, 65 makes, global market | Battery capacity (net and gross) and chemistry, DC and AC charging power, connectors, power, torque, 0-100 km/h time, WLTP and EPA range | EVs only; no measured consumption; partial coverage (no Kia, GMC, Subaru and others); some inconsistent EPA and connector fields |


fueleconomy.gov is the backbone because it covers every powertrain under one standardised test procedure. OpenEV adds the battery and charging attributes that route optimisation needs, and it adds EVs that are not sold in the US (BYD, MG, Zeekr and others) but are sold in Malaysia.

**Assumptions and constants**

**Prices (MYR), set in Notebook 1 and saved to assumptions.json**

| **Item** | **Value** | **Notes** |
| :-: | :-: | :-: |
| RON95 | RM 3.80 / L | Unsubsidised market price, November 2026 |
| RON95 (BUDI95 subsidised) | RM 1.99 / L | Used in the subsidy comparisons in Notebook 2 |
| RON97 | RM 4.20 / L | Used for vehicles that require premium fuel |
| Diesel | RM 4.90 / L | Peninsular |
| Electricity, home | RM 0.45 / kWh | Residential effective tariff |
| Electricity, public DC | RM 1.30 / kWh | DC fast charging |
| Home-charging share (base case) | 70 % | The rest is public DC |
| Blended electricity price | RM 0.705 / kWh | 0.70 × 0.45 + 0.30 × 1.30 |
| Annual distance | 20,000 km | Typical Malaysian mileage |


US Regular and Midgrade fuel map to RON95, Malaysia's base grade. US Premium (AKI 91 and above) maps to RON97.

**Unit conversions and physical constants**

| **Quantity** | **Formula / value** |
| :-: | :-: |
| Fuel consumption | L/100 km = 235.215 / MPG  (100 × 3.785411784 L/gal ÷ 1.609344 km/mi) |
| Electricity consumption | kWh/100 km = kWh/100 mi ÷ 1.609344 |
| CO₂ | g/km = g/mi ÷ 1.609344 |
| Range | km = mi × 1.609344 |
| Petrol energy content | 8.904 kWh/L (EPA MPGe definition: 33.705 kWh per US gallon) |
| Diesel energy content | 10.06 kWh/L (1 US gal of diesel ≈ 1.13 gasoline-gallon-equivalent, US DOE AFDC) |
| EPA city/highway weighting | combined = 55 % city + 45 % highway (by distance) |
| EPA/WLTP range ratio | 0.773 (median of 269 OpenEV trims that report both; 95 % bootstrap CI 0.767 to 0.785) |


**1. Notebook 1: data integration and cleaning**

**1.1 Cleaning pipeline and row lineage**

| **Step** | **Rows** | **What and why** |
| :-: | :-: | :-: |
| Raw fueleconomy.gov | 50,407 |  |
| Keep model years 2015-2026 | 15,455 | Pre-2015 vehicles do not represent today's fleet, and 2027 is a partial model year |
| Drop hydrogen (FCV), CNG and LPG | 15,409 | 46 vehicles with no comparable refuelling network in Malaysia |
| Drop specification duplicates | 15,277 | 132 records identical on every kept attribute, differing only in a free-text note |
| Domain and plausibility checks | 15,277 | No violations (see below) |
| Remove plug-in hybrids from OpenEV | 15,277 | 57 OpenEV trims (1,189 to 1,132); fueleconomy.gov rows are unaffected (section 5.4) |
| Left-join OpenEV attributes | 15,277 | Row count asserted unchanged |
| Append OpenEV-only EV trims | 15,609 | 332 non-US EVs (section 5.5) |


The other cleaning steps:

- Sentinel values. fueleconomy.gov uses -1 for "unknown" (co2, feScore, ghgScore) and 0 for "not applicable" (the electricity and range fields of non-EVs, and charge240). Both became NaN. Values that really are zero, such as BEV tailpipe CO₂, are set explicitly from the powertrain.

- Types and flags. Code letters became booleans: tCharger="T" became turbo, sCharger="S" became supercharger, and guzzler and startStop (Y/N) were converted the same way. rangeA, stored as text, was parsed to a number.

- Category clean-up:

  - Drivetrain went from 7 labels to FWD, RWD, AWD, 4WD and 2WD.

  - Vehicle class went from 22 EPA VClass labels to 14 size\_class values and 5 vehicle\_class values (Car, SUV, Pickup, Van, Special Purpose).

  - EPA transmission codes such as S8, AM7, AV-S6 and A1 became transmission\_type (Automatic, Automated manual/DCT, CVT, Manual, Single-speed) plus n\_gears.

  - Make spellings were unified (Vinfast became VinFast).

- Domain rules. A record is dropped if it breaks a physical limit: 2 to 40 L/100 km, 8 to 70 kWh/100 km, city and highway figures present, and BEV range present. Extreme vehicles that are still possible (Bugatti, Hummer EV) are kept and flagged (section 5.7).

**1.2 Powertrain taxonomy**

| **Powertrain** | **Rule** | **Records** |
| :-: | :-: | :-: |
| ICE | conventional petrol, diesel or flex-fuel (atvType empty, Diesel or FFV) | 12,042 |
| MHEV | atvType = Hybrid and either eng\_dscr or the model name contains Mild, eAssist or eTorque, or the battery parsed from evMotor is below 100 V | 785 |
| HEV | every other Hybrid | 653 |
| PHEV | atvType = Plug-in Hybrid | 424 |
| BEV | atvType = EV, plus OpenEV-only EVs | 1,705 |


About half of the EPA's "hybrids" are 48 V mild hybrids, and they behave very differently from full hybrids. The median full HEV gets 39 MPG in the city against 20 MPG for an MHEV, with a battery of about 259 V against 48 V.

**1.3 Derived energy, emissions and cost variables**

| **Variable** | **Calculation** |
| :-: | :-: |
| fuel\_l\_100km\_\{city,hwy,comb\} | 235.215 / MPG (ICE, MHEV, HEV; PHEV in charge-sustaining mode) |
| elec\_kwh\_100km\_\{city,hwy,comb\} | EPA kWh/100 mi ÷ 1.609344, measured at the wall socket so it includes charging losses (BEV; PHEV in charge-depleting mode) |
| energy\_kwh\_100km | One energy scale for all powertrains: • ICE/MHEV/HEV: fuel\_l\_100km × kWh\_per\_L (8.904 petrol, 10.06 diesel) • BEV: elec\_kwh\_100km • PHEV: UF × elec + (1 − UF) × fuel\_energy, where UF is the EPA utility factor combinedUF |
| co2\_g\_km | tailpipe g/mi ÷ 1.609344; 0 for BEV |
| range\_km | BEV EPA range × 1.609344 (OpenEV-only EVs: EPA range, or WLTP × 0.773) |
| power\_kw | OpenEV system power; otherwise the sum of motor ratings parsed from FE evMotor (an upper bound) |
| cost\_myr\_100km | see the formula below |
| annual\_cost\_myr | cost\_myr\_100km × 20,000 / 100 |
| usable\_range\_km | 0.70 × range (the 10 % to 80 % state-of-charge window) |
| dc\_c\_rate | peak DC kW ÷ net battery kWh |
| dc\_10\_80\_min\_est | 0.70 × net kWh ÷ (0.70 × peak DC kW) × 60, assuming average session power is about 70 % of peak |
| km\_per\_charge\_min | usable\_range\_km / dc\_10\_80\_min\_est |


Running cost per 100 km is calculated as:

fuel\_cost = fuel\_l\_100km × price(fuel\_grade)            \# RON95 / RON97 / Diesel  
elec\_cost = elec\_kwh\_100km × (h × elec\_home + (1 − h) × elec\_dc)   \# h = home-charging share  
cost      = fuel\_cost                                    (ICE, MHEV, HEV)  
= elec\_cost                                    (BEV)  
= UF × elec\_cost + (1 − UF) × fuel\_cost        (PHEV)

*For ICE cars, energy\_kwh\_100km is converted from MPG using the fuel's energy content; nothing measures it directly. It is the chemical energy of the fuel that goes into the tank. For example, 25 MPG is 9.41 L/100 km, and 9.41 × 8.904 = 83.8 kWh/100 km. Refining and power-station losses are left out, so this is not a lifecycle (well-to-wheel) comparison. The variable is only used to compare powertrains on one scale. Costs always come from litres or kWh multiplied by the matching price.*

**1.4 Entity resolution: matching EPA EVs to OpenEV**

The two datasets share no key and name cars differently. FE calls a car Model 3 Long Range AWD where OpenEV has model Model 3 and trim Long Range. A plain fuzzy match produced false positives, for example matching Model S to Model 3 at 92 % similarity, so the matcher works in two stages with explicit rules.

OpenEV also lists plug-in hybrids and range extenders, such as the Range Rover PHEV models, Ferrari SF90 and 296 GTB, McLaren Artura, Karma GS-6 and Chery Tansuo 06. Left in, they would be appended as BEVs with consumption estimates of 30 to 46 kWh/100 km. Before matching, Notebook 1 removes 57 such trims: those whose model name contains PHEV, plug-in or hybrid, plus a documented list of known plug-in models.

1. Normalise names. Names are ASCII-folded and lower-cased, the make is stripped from the model name ("Polestar 2" becomes "2"), and noise tokens such as drivetrain words, wheel sizes and "with" are removed. An alias table covers documented renames: Volvo XC40/C40 Recharge to EX40/EC40, Toyota bZ (MY2026) to bZ4X, and Mercedes CLA250/350 with EQ Tech to CLA EV.

2. Match the model, strictly. An OpenEV model is a candidate only if all of its tokens appear in the FE model name. The most specific candidate wins, and the model year must be within ±1.

3. Score the trims. Each trim gets +20 for matching drivetrain, up to 40 for EPA-range closeness (40 at 0 % difference, falling to 0 at 20 %), and up to 30 for fuzzy trim-name similarity.

4. Assign confidence from the range difference, which is independent physical evidence: high if it is within 10 %, medium within 25 %, low beyond 25 %. Low-confidence matches have their trim-specific fields (battery, power and so on) set to null but keep model-level fields such as connector and chemistry.

Results:

- 981 of the 1,146 eligible EPA EVs matched (85.6 %): 705 with high confidence, 205 medium and 71 low. The unmatched makes (Kia, GMC, Subaru, Dodge and others) are simply absent from OpenEV.

- 888 pairs have a measured EPA range in both sources. On those, Lin's concordance correlation is 0.843, the median absolute difference is 5.1 %, and the Bland-Altman mean bias is +2.1 km (95 % limits −108 to +113 km).

- The wide limits come from wheel-size variants and from some wrong OpenEV EPA values. One Lexus RZ 450e, for example, is listed at 220 km against the EPA's 354 km.

**1.5 Integrating non-US EVs (test-cycle harmonisation)**

OpenEV models that matched no EPA record in any year were appended, which avoids double counting. They have no measured consumption. The figure that can be derived (net battery ÷ WLTP range) is battery-to-wheel on the WLTP cycle, whereas the EPA figure is wall-to-wheel. So the WLTP figure was calibrated to the EPA figure using matched cars where both are known:

kWh/100km (EPA, wall) = 2.80 + 1.266 × kWh/100km (WLTP, battery)

- Under 5-fold GroupKFold by car model (607 records, 54 models), the calibration scored a cross-validated R² of 0.60 and an MAE of 1.73 kWh/100 km (7.9 %). Using the WLTP number unchanged gives an MAE of 6.94.

- Adding body class, drivetrain or power improved the MAE by only about 0.1, so the one-variable model was kept.

- Of the 644 remaining OpenEV-only trims, 332 were appended. The other 312 report neither WLTP nor EPA range; most are NIO, XPeng, Leapmotor and GAC Aion.

- Every estimate carries consumption\_estimated = True, so these rows can be kept out of any analysis that needs measured consumption.

**1.6 Final dataset**

The cleaned table has 15,609 rows and 65 columns, covering 79 makes and model years 2015-2026. Battery data is available for 1,242 BEVs (73 %). data/processed/data\_dictionary.csv describes every column.

Most missing values are structural: petrol cars have no battery and BEVs have no engine displacement. These are left as NaN rather than imputed. The only imputation is OpenEV gross battery capacity for 4 trims, filled as net × 1.055 (the median gross/net ratio).

**1.7 Outliers**

Outliers were flagged with a robust z-score computed within each powertrain, z = 0.6745 × (x − median) / MAD, with |z| \> 3.5 as the cut-off (Iglewicz and Hoaglin). That flags 109 records: 80 ICE (0.7 %), 8 HEV (1.2 %) and 21 BEV (1.2 %), including Bugatti, Lamborghini, the Koenigsegg Regera, the Hummer EV and the Rimac Nevera. They stay in the data with the is\_energy\_outlier flag, since removing them changes none of the conclusions (section 6.4).

# **2. Notebook 2: EDA and statistical analysis**

**2.1 Approach**

- Each row is a vehicle configuration (model × year × engine), not a sold car, so the results describe what is on offer rather than what is on the road.

- Tests of consumption use the 15,277 EPA-measured rows only. The 332 estimated rows would make those tests circular.

- The data are non-normal and heteroscedastic, so the tests are rank-based, each reported with an effect size: Kruskal-Wallis with ε², Mann-Whitney with Cliff's δ, Wilcoxon with the rank-biserial r, and χ² with Cramér's V.

- Multiple comparisons use the Holm correction, and regression standard errors are clustered by car model.

**2.2 Key statistics**

| **Powertrain** | **n** | **Median energy (kWh/100 km)** | **Median fuel (L/100 km)** | **Median cost (RM/100 km)\*** | **vs ICE** |
| :-: | :-: | :-: | :-: | :-: | :-: |
| ICE | 11,962 | 95.2 | 10.7 | 42.95 |  |
| ICE (subsidised) | 11,962 | 95.2 | 10.7 | 21.29 | −50% |
| MHEV | 785 | 91.1 | 10.2 | 42.95 | 0 % |
| HEV | 645 | 55.1 | 6.2 | 23.52 | −45 % |
| HEV (subsidised) | 645 | 55.1 | 6.2 | 12.34 | −71 % |
| PHEV | 424 | 55.4 | 9.1 (CS) | 30.16 | −30 % |
| BEV | 1,684 | 22.2 |  | 15.61 | −64 % |


\*Base case: market fuel prices and 70 % home charging.

At 20,000 km a year, the median BEV saves about RM 5,500 and the median HEV about RM 3,900 against the median ICE. (NOTE: This is non subsidised)

**2.3 Findings by research question**

| **Question** | **Test** | **Result** |
| :-: | :-: | :-: |
| Is energy use normally distributed? | Shapiro-Wilk, D'Agostino K² | No, for every group (right skew of 0.4 to 2.0). A log transform removes most of it (ICE skew falls from 0.78 to 0.13), so log(consumption) is the better scale for regression. |
| Is efficiency improving? | Mann-Kendall and Theil-Sen on yearly geometric means | Raw BEV energy rises 1.7 % a year (τ = 0.58, p = 0.009). This is a composition effect: BEVs went from 100 % cars in 2015 to 59 % SUVs and 12 % pickups in 2026. After adjusting for class and drivetrain, the year effect is slightly negative. ICE's share of configurations fell from 94 % (2015) to 50 % (2026), and the BEV share rose from 1.5 % to 27.5 %. |
| Do powertrains differ in energy use? | Kruskal-Wallis; Welch ANOVA; pairwise Mann-Whitney with Holm correction | Yes: H = 5,389 and ε² = 0.35, a large effect. 8 of 10 pairs differ with a large Cliff's δ. The two exceptions are ICE vs MHEV (δ = 0.09) and HEV vs PHEV (δ = 0.08). Every BEV uses less energy than every ICE (δ = 1.00). The result holds when collapsed to 823 car models (ε² = 0.53). |
| Do they still differ after adjusting for body class, drivetrain and year? | OLS on log energy with cluster-robust SEs (R² = 0.86) | Relative to ICE: MHEV −7 % \[−11, −3\], HEV −35 % \[−38, −31\], PHEV −44 % \[−49, −39\], BEV −77 % \[−78, −76\]. Partial η² is 0.83 for powertrain, 0.30 for drivetrain, 0.10 for class and 0.003 for year. The hybrid advantage is largest for cars (−47 %) and smallest for pickups (−17 %). Body-type mix differs only weakly between powertrains (χ², Cramér's V = 0.07). |
| How much does a hybrid save on the same car? | Wilcoxon signed-rank on matched twins (same make, model, year and drive) | The HEV saves 28.7 % \[27.1, 29.9\] across 335 pairs and 55 models, and every pair improves (r = 1.00). The MHEV saves 7.1 % \[4.3, 11.1\] across 162 pairs. |
| City or highway? | Wilcoxon signed-rank per powertrain | City consumption divided by highway consumption is 1.36 for ICE, 1.32 for MHEV, 1.00 for HEV (p = 0.48, no difference) and 0.89 for BEV, which is 11 % more efficient in the city. Congested urban traffic favours electrified powertrains. |
| What drives consumption? | Spearman correlation; OLS with VIF | For combustion vehicles (R² = 0.88): displacement elasticity is about 0.31; each cylinder adds 3 %; a turbo adds 10 % at the same displacement; start-stop saves 4 %; diesel uses 18 % fewer litres; HEV saves 25 % and MHEV 9 % net of engine size; SUVs use 11 % more and AWD 9 % more; CVTs use 7 % less; model year has no effect. For BEVs (R² = 0.47): AWD adds 9 % and pickups or vans 15 %, while power and battery size are collinear. CO₂ and the EPA cost are near-deterministic functions of consumption (ρ ≥ 0.91), so they would leak the target into any consumption model. |
| What are EVs' charging capabilities, and can they cover the main corridors? | χ²; Spearman; Kruskal-Wallis | Connector standards are market-specific (Cramér's V = 0.54), with CCS2 on 97 % of non-US trims. Bigger batteries get faster chargers (ρ = 0.75). LFP batteries charge more slowly than NMC (median 1.56C against 2.17C, Cliff's δ = −0.57). A typical 10 % to 80 % stop takes about 30 minutes and adds about 9 km of range per minute. Corridor results are below. |
| Is the running-cost measure valid, and how sensitive is it? | Kruskal-Wallis; Spearman; scenario sweep | Cost differs strongly by powertrain (ε² = 0.33). The MYR cost ranks vehicles the same way as the EPA's independent US cost (ρ = 0.98). Charging behaviour and the subsidy can flip the ranking (below). |


Share of EPA-tested BEVs that can cover each corridor without charging, departing at 100 % and arriving with at least 10 %, using highway range:

| **Corridor (km)** | **EPA highway** | **+20 % (110 km/h, A/C)** | **With ≤ 1 DC stop (+20 %)** |
| :-: | :-: | :-: | :-: |
| KL to Ipoh (205) | 94 % | 94 % | 97 % |
| KL to Kuantan (250) | 94 % | 86 % | 96 % |
| KL to Johor Bahru (330) | 76 % | 44 % | 94 % |
| KL to Penang (355) | 66 % | 29 % | 94 % |
| KL to Kota Bharu (460) | 16 % | 4 % | 83 % |


Cost sensitivity for the median BEV (22.6 kWh/100 km):

| **Comparison** | **Comparator cost (RM/100 km)** | **Home-charging share at which the BEV becomes cheaper** |
| :-: | :-: | :-: |
| ICE, RON95 market price (RM 3.80) | 38.9 | 0 %: even on 100 % public DC (RM 29.4) the BEV is cheaper |
| ICE, BUDI95 (RM 1.99) | 20.4 | 47 % or more |
| HEV, RON95 market price | 22.3 | 37 % or more (the 70 % base case gives the BEV RM 15.9) |
| HEV, BUDI95 | 11.7 | 92 % or more |


The median BEV costs RM 29.4 per 100 km on 100 % public DC, RM 15.9 in the base case and RM 10.2 on 100 % home charging. At market prices, a BEV is the cheapest powertrain for anyone who does some charging at home, and it still beats a petrol car on public DC alone. For BUDI95-eligible drivers without a home charger, a full hybrid is cheapest.

**2.4 Sensitivity to outliers**

Notebook 2 without the 109 flagged outliers and compares the headline statistic of each research question. All removed vehicles are high-consumption cases with EPA-measured figures.

| **Statistic** | **With outliers** | **Outliers removed** |
| :-: | :-: | :-: |
| Skew / excess kurtosis, ICE | 0.78 / 1.19 | 0.47 / −0.26 |
| Skew / excess kurtosis, HEV | 1.99 / 6.75 | 0.99 / 0.26 |
| Skew / excess kurtosis, BEV | 1.01 / 2.20 | 0.32 / −0.54 |
| Kruskal-Wallis ε² (energy) | 0.353 | 0.354 |
| Adjusted effect vs ICE: MHEV / HEV / PHEV / BEV | −7.4 / −34.6 / −44.1 / −77.0 % | −6.9 / −35.2 / −43.8 / −77.1 % |
| BEV trend (%/yr, p) | +1.66 (0.009) | +1.48 (0.009) |
| HEV twin saving | 28.7 % | 28.7 % |
| Combustion regression R² / residual excess kurtosis | 0.876 / 1.96 | 0.879 / 0.65 |
| BEV home-charging share to beat a market-price HEV | 36.6 % | 36.1 % |


Removing the outliers changes the shape of the distributions: skew and kurtosis drop sharply, and standard deviations shrink by 5 to 16 %.

(In the notebook: The pipeline removes the outliers. The table above is just for reference if I did the pipeline without removing outliers)

