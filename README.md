# Vehicle Efficiency Explorer

I work on my own car, so I built this around a question I actually care about: is this model efficient for its class, or does it just look fine on a spec sheet?

**Live dashboard:** https://public.tableau.com/views/VehicleEfficiencyExplorer/VehicleEfficiencyExplorer

[![Vehicle Efficiency Explorer dashboard](docs/dashboard.png)](https://public.tableau.com/views/VehicleEfficiencyExplorer/VehicleEfficiencyExplorer)

## What it does

Pick a year, make, class, and model. The dropdowns only show combinations that exist in the EPA file, so you cannot end up with a 1984 Subaru CR-V.

From there you get:

- Combined, city, and highway MPG, plus the EPA's estimated annual fuel cost
- How far that vehicle sits above or below the median for its class in that year
- Every EPA-tested setup for that model (engine, transmission, drivetrain), not trims like LE or XSE
- The most efficient models in that class and year
- How the selected make's efficiency in that class has moved over time, split by powertrain

A 2023 Camry sits in the middle of midsize cars at 26 MPG. Gas midsize cars improved until about 2019, then flattened. The cars at the top of the class chart are EVs. Those numbers are MPGe, which is a different unit from regular MPG.

## Data source

U.S. EPA / DOE [FuelEconomy.gov](https://www.fueleconomy.gov/feg/download.shtml) vehicle file (`vehicles.csv`). Same ratings that go on a new car's window sticker. Public domain.

- Download: https://www.fueleconomy.gov/feg/epadata/vehicles.csv.zip
- Field list: https://www.fueleconomy.gov/feg/ws/index.shtml#vehicle

One row is one certified configuration (year, make, model, drive, transmission, engine, fuel). That is why one Camry model year shows up more than once.

## How I prepared the file

The EPA download has ~80 fields, including older test methods and second-fuel variants. I kept 20 columns the dashboard uses (ids, class, drivetrain, engine, city/highway/combined MPG, CO2, annual fuel cost) and added 3 helper fields. No rows were dropped: 50,242 in, 50,242 out. No imputation, no deduping. The grain is unchanged.

`powertrain_type` is the main derived field. EPA splits that signal across `atvType`, `fuelType`, `fuelType1`, and `phevBlended`, so a plug-in can look electric in one field and gasoline in another. The mapping is a mutually exclusive CASE with EV first so a PHEV cannot fall into EV or Hybrid:

```sql
CASE
  WHEN atvType LIKE '%EV%'
    OR (CONCAT(fuelType, ' ', fuelType1) LIKE '%electricity%'
        AND CONCAT(fuelType, ' ', fuelType1) NOT LIKE '%gas%')
    THEN 'EV'
  WHEN atvType LIKE '%plug-in%'
    OR phevBlended = 'true'
    OR CONCAT(fuelType, ' ', fuelType1) LIKE '%plug-in%'
    THEN 'PHEV'
  WHEN atvType LIKE '%hybrid%'
    OR CONCAT(fuelType, ' ', fuelType1) LIKE '%hybrid%'
    THEN 'Hybrid'
  WHEN CONCAT(fuelType, ' ', fuelType1) LIKE '%diesel%'
    THEN 'Diesel'
  WHEN CONCAT(fuelType, ' ', fuelType1) LIKE '%gasoline%'
    OR CONCAT(fuelType, ' ', fuelType1) LIKE '%regular%'
    OR CONCAT(fuelType, ' ', fuelType1) LIKE '%premium%'
    THEN 'ICE'
  ELSE 'Other'   -- CNG, bi-fuel, hydrogen, etc. (102 rows)
END AS powertrain_type
```

`make_model` and `vehicle_label` are concatenations for filters and tooltips (`make + model`, `year + make + model`). Those three fields could have been Tableau calculated fields; they live in `data/vehicles_v1.csv` so this repo is the same table the viz is built on.

### In Tableau

- **Cascading filters, Year → Make → Class → Model.** Putting every filter on "only relevant values" on the same sheet circularizes the class domain (every class stays selectable). Year and Make are context filters on a small helper sheet; the Vehicle Class card is "All values in context," so the list is actually valid combinations.
- **Class median is a LOD**, not a table calc, so it does not move when you pick a model:

```text
{ FIXED [VClass], [year] : MEDIAN([comb08]) }
```

  vs Class Median is `AVG([comb08]) / AVG([Class Year Median]) - 1`. The "most efficient in this class" bar chart uses the same median as a reference line. On that sheet, Year and class are in context and the view is Top 9 models by average combined MPG, so the ranking is "top in this class-year," not top in the whole file.
- **Trends** is median combined MPG by year, colored by `powertrain_type`, scoped to the selected make and class across all model years. EV values are MPGe.

Fuel cost on the dashboard is the EPA's own annual estimate (15,000 miles, 55% city, national average prices).

### What's in the file

| | |
| --- | --- |
| Rows | 50,242 |
| Columns | 23 (20 from EPA + 3 I added) |
| Model years | 1984 to 2027 |
| Makes | 146 |
| EPA vehicle classes | 34 |
| Powertrain mix | ICE 44,936 · Hybrid 1,873 · EV 1,574 · Diesel 1,310 · PHEV 447 · Other 102 |

2027 is in the file because the EPA posts certifications as manufacturers submit them. It is early ratings, not a forecast.

## Columns

| Column | What it is |
| --- | --- |
| `id` | EPA record ID |
| `year` | Model year |
| `make` | Manufacturer |
| `model` | Model name as certified |
| `VClass` | EPA size class (Midsize Cars, Small SUV 4WD, etc.) |
| `drive` | Front-wheel, rear-wheel, 4-wheel, etc. |
| `trany` | Transmission type and speed count |
| `fuelType` / `fuelType1` | Fuels it is certified for / primary fuel |
| `atvType` | Alternative-tech tag (Hybrid, Plug-in Hybrid, EV, Diesel, CNG). Blank for a normal gas car. |
| `cylinders` / `displ` | Engine size. Blank for EVs. |
| `city08` / `highway08` / `comb08` | City, highway, combined MPG. For EVs, combined is MPGe. |
| `co2TailpipeGpm` | Tailpipe CO2, grams per mile |
| `fuelCost08` | EPA annual fuel cost, dollars |
| `baseModel` | Model name with extra engine/trim text stripped |
| `eng_dscr` | EPA engine notes (TURBO, GUZZLER, etc.) |
| `phevBlended` | Whether a plug-in hybrid blends gas and electric while the battery is draining |
| `powertrain_type` | Added. EV / PHEV / Hybrid / Diesel / ICE / Other, using the CASE above |
| `make_model` | Added. Make + model |
| `vehicle_label` | Added. Year + make + model |

## Using the dashboard

- **MPG and MPGe are not the same unit.** MPGe converts electricity into a gasoline-equivalent number. Useful for ranking, not the same as burning gas.
- **Filters go Year → Make → Class → Model.** If a chart goes blank after you change year, an older make or model is still selected. Pick something that exists for that year.
- **To open the workbook,** use the Download button on the Tableau Public page. That file includes this same data.

## What's in this repo

| Path | What it is |
| --- | --- |
| `vehicles_v1.csv` | The analysis file described above (~11 MB). Opens in Excel or Tableau. |
| `dashboard.png` | Snapshot of the published dashboard |
