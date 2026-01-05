# GlobalWeather-ETL

**Projekt na Databázové Technológie – ELT proces v Snowflake**

**Autori:**
Matej Medvec
Juraj Pálenkáš

**Dátum:** Január 2026

---

## 1. Úvod a popis zdrojových dát

**Téma projektu:**
Analýza denných predpovedí počasia pre rôzne lokality po svete.

**Prečo sme si vybrali tento dataset:**
Dataset pochádza zo **Snowflake Marketplace** (Weather Source LLC: *Frostbyte – OnPoint ID Forecast Day*). Je voľne dostupný v sample verzii, obsahuje reálne hyper-lokálne meteorologické predpovede a je vhodný na precvičenie ELT procesov, dimenzionálneho modelovania a práce s databázou Snowflake.

**Biznis procesy, ktoré dáta podporujú:**

* Plánovanie predaja v retail a food sektore (vplyv počasia na dopyt)
* Logistika a doprava (riziká zrážok a sneženia)
* Energetika (predikcia spotreby energie podľa teploty)
* Poistenie (hodnotenie rizík extrémneho počasia)

**Typy údajov:**
Časové údaje (dátumy), geografické údaje (poštové kódy, mestá, krajiny) a numerické metriky (teploty v °F, zrážky a sneženie v palcoch, vlhkosť, vietor, oblačnosť).

**Účel analýzy:**
Cieľom je analyzovať trendy teplôt a zrážok a ich vzťahy naprieč lokalitami a časom. Dimenzionálny model umožňuje efektívne odpovedať na reportingové otázky, ako napríklad identifikáciu najteplejších miest alebo vplyv typu zrážok na teplotu.

**Zdrojová tabuľka:**
`WEATHER_SOURCE_LLC_FROSTBYTE.ONPOINT_ID.FORECAST_DAY` – denné predpovede počasia pre vybrané poštové kódy.

### ERD pôvodnej normalizovanej štruktúry dát (3NF)

![Normalizovaný ERD](/img/erd_normalized_3nf.png)

Pôvodná štruktúra bola denormalizovaná; pre účely návrhu ERD bola rozdelená do tretej normálnej formy (ENTITY: LOCATION, DATE, WEATHER_DAY).

---

## 2. Návrh dimenzionálneho modelu

Bol navrhnutý dimenzionálny model typu **Star Schema** pozostávajúci z jednej faktovej tabuľky a piatich dimenzií.

**Faktová tabuľka:** `DIMENSIONAL.FACT_WEATHER_DAY`

* Kompozitný kľúč: `DATE_KEY`, `LOCATION_KEY`
* Cudzie kľúče: `DATE_KEY`, `LOCATION_KEY`, `WEATHER_BAND`, `PRECIPITATION_TYPE`, `SOURCE_KEY`
* Hlavné metriky: `AVG_TEMP_F`, `PRECIPITATION_IN`, `SNOWFALL_IN`
* Odvodené metriky pomocou analytických funkcií:

  * `TEMP_DAY_DELTA` – medzidenná zmena teploty
  * `PRECIPITATION_7D_SUM` – sedemdňový kumulatívny úhrn zrážok

**Dimenzie:**

1. **DIM_DATE** – časová dimenzia (SCD typ 0)
2. **DIM_LOCATION** – lokalita (SCD typ 2)
3. **DIM_WEATHER_BAND** – teplotné pásma (SCD typ 1)
4. **DIM_PRECIPITATION_TYPE** – typ zrážok (SCD typ 1)
5. **DIM_SOURCE** – zdroj dát (SCD typ 0)

### Star Schema

![Star Schema](/img/star_schema_dimensional.png)

---

## 3. ELT proces v Snowflake

### 3.1 Vytvorenie databázy a schém

```sql
CREATE OR REPLACE DATABASE PEACOCK_GIRAFFE_PROJECT_DB;
USE DATABASE PEACOCK_GIRAFFE_PROJECT_DB;

CREATE OR REPLACE SCHEMA STAGING;
CREATE OR REPLACE SCHEMA DIMENSIONAL;
CREATE OR REPLACE SCHEMA NORMALIZED;
```

### 3.2 Extract – načítanie dát zo Snowflake Marketplace

```sql
CREATE OR REPLACE TABLE STAGING.STG_FORECAST_DAY AS
SELECT *
FROM WEATHER_SOURCE_LLC_FROSTBYTE.ONPOINT_ID.FORECAST_DAY;
```

### 3.3 Transform – dimenzie

#### DIM_DATE

```sql
CREATE OR REPLACE TABLE DIMENSIONAL.DIM_DATE AS
SELECT DISTINCT
    DATE_VALID_STD AS DATE_KEY,
    DATE_VALID_STD AS FULL_DATE,
    YEAR(DATE_VALID_STD) AS YEAR,
    MONTH(DATE_VALID_STD) AS MONTH,
    DAY(DATE_VALID_STD) AS DAY,
    DAYOFWEEKISO(DATE_VALID_STD) AS DAY_OF_WEEK,
    WEEKISO(DATE_VALID_STD) AS WEEK_OF_YEAR,
    DOY_STD AS DAY_OF_YEAR
FROM STAGING.STG_FORECAST_DAY;
```

#### DIM_LOCATION (SCD typ 2)

```sql
CREATE OR REPLACE TABLE DIMENSIONAL.DIM_LOCATION AS
SELECT
    ROW_NUMBER() OVER (ORDER BY POSTAL_CODE, CITY_NAME, COUNTRY) AS LOCATION_KEY,
    POSTAL_CODE,
    CITY_NAME,
    COUNTRY,
    CURRENT_DATE() AS VALID_FROM,
    NULL AS VALID_TO,
    TRUE AS IS_CURRENT
FROM (
    SELECT DISTINCT POSTAL_CODE, CITY_NAME, COUNTRY
    FROM STAGING.STG_FORECAST_DAY
);
```

#### DIM_WEATHER_BAND

```sql
CREATE OR REPLACE TABLE DIMENSIONAL.DIM_WEATHER_BAND AS
SELECT DISTINCT
    CASE
        WHEN AVG_TEMPERATURE_AIR_2M_F < 32 THEN 'Freezing'
        WHEN AVG_TEMPERATURE_AIR_2M_F BETWEEN 32 AND 50 THEN 'Cold'
        WHEN AVG_TEMPERATURE_AIR_2M_F BETWEEN 51 AND 70 THEN 'Mild'
        WHEN AVG_TEMPERATURE_AIR_2M_F BETWEEN 71 AND 85 THEN 'Warm'
        ELSE 'Hot'
    END AS WEATHER_BAND
FROM STAGING.STG_FORECAST_DAY;
```

#### DIM_PRECIPITATION_TYPE

```sql
CREATE OR REPLACE TABLE DIMENSIONAL.DIM_PRECIPITATION_TYPE AS
SELECT DISTINCT
    CASE
        WHEN TOT_SNOWFALL_IN > 0 THEN 'Snow'
        WHEN TOT_PRECIPITATION_IN > 0 THEN 'Rain'
        ELSE 'None'
    END AS PRECIPITATION_TYPE
FROM STAGING.STG_FORECAST_DAY;
```

#### DIM_SOURCE

```sql
CREATE OR REPLACE TABLE DIMENSIONAL.DIM_SOURCE AS
SELECT
    1 AS SOURCE_KEY,
    'Weather Source LLC' AS PROVIDER,
    'Frostbyte' AS DATASET_NAME,
    'Snowflake Marketplace' AS INGEST_METHOD;
```

### 3.4 Load – faktová tabuľka

```sql
CREATE OR REPLACE TABLE DIMENSIONAL.FACT_WEATHER_DAY AS
SELECT
    d.DATE_KEY,
    l.LOCATION_KEY,
    wb.WEATHER_BAND,
    pt.PRECIPITATION_TYPE,
    s.SOURCE_KEY,
    f.AVG_TEMPERATURE_AIR_2M_F AS AVG_TEMP_F,
    f.TOT_PRECIPITATION_IN AS PRECIPITATION_IN,
    f.TOT_SNOWFALL_IN AS SNOWFALL_IN,
    f.AVG_TEMPERATURE_AIR_2M_F
      - LAG(f.AVG_TEMPERATURE_AIR_2M_F)
        OVER (PARTITION BY l.LOCATION_KEY ORDER BY d.DATE_KEY)
      AS TEMP_DAY_DELTA,
    SUM(f.TOT_PRECIPITATION_IN)
        OVER (PARTITION BY l.LOCATION_KEY
              ORDER BY d.DATE_KEY
              ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
      AS PRECIPITATION_7D_SUM
FROM STAGING.STG_FORECAST_DAY f
JOIN DIMENSIONAL.DIM_DATE d
    ON f.DATE_VALID_STD = d.DATE_KEY
JOIN DIMENSIONAL.DIM_LOCATION l
    ON f.POSTAL_CODE = l.POSTAL_CODE AND l.IS_CURRENT = TRUE
JOIN DIMENSIONAL.DIM_WEATHER_BAND wb
    ON wb.WEATHER_BAND = CASE
        WHEN f.AVG_TEMPERATURE_AIR_2M_F < 32 THEN 'Freezing'
        WHEN f.AVG_TEMPERATURE_AIR_2M_F BETWEEN 32 AND 50 THEN 'Cold'
        WHEN f.AVG_TEMPERATURE_AIR_2M_F BETWEEN 51 AND 70 THEN 'Mild'
        WHEN f.AVG_TEMPERATURE_AIR_2M_F BETWEEN 71 AND 85 THEN 'Warm'
        ELSE 'Hot'
    END
JOIN DIMENSIONAL.DIM_PRECIPITATION_TYPE pt
    ON pt.PRECIPITATION_TYPE = CASE
        WHEN f.TOT_SNOWFALL_IN > 0 THEN 'Snow'
        WHEN f.TOT_PRECIPITATION_IN > 0 THEN 'Rain'
        ELSE 'None'
    END
JOIN DIMENSIONAL.DIM_SOURCE s
    ON s.SOURCE_KEY = 1;
```

### 3.5 Normalizovaný model (3NF)

```sql
CREATE OR REPLACE TABLE NORMALIZED.LOCATION AS
SELECT DISTINCT POSTAL_CODE, CITY_NAME, COUNTRY
FROM STAGING.STG_FORECAST_DAY;

CREATE OR REPLACE TABLE NORMALIZED.DATE AS
SELECT DISTINCT
    DATE_VALID_STD,
    YEAR(DATE_VALID_STD) AS YEAR,
    MONTH(DATE_VALID_STD) AS MONTH,
    DAY(DATE_VALID_STD) AS DAY,
    DOY_STD AS DAY_OF_YEAR,
    WEEKISO(DATE_VALID_STD) AS WEEK_OF_YEAR,
    DAYOFWEEKISO(DATE_VALID_STD) AS DAY_OF_WEEK
FROM STAGING.STG_FORECAST_DAY;

CREATE OR REPLACE TABLE NORMALIZED.WEATHER_DAY AS
SELECT
    DATE_VALID_STD,
    POSTAL_CODE,
    AVG_TEMPERATURE_AIR_2M_F,
    MIN_TEMPERATURE_AIR_2M_F,
    MAX_TEMPERATURE_AIR_2M_F,
    AVG_HUMIDITY_RELATIVE_2M_PCT,
    AVG_PRESSURE_2M_MB,
    AVG_WIND_SPEED_10M_MPH,
    AVG_CLOUD_COVER_TOT_PCT,
    TOT_PRECIPITATION_IN,
    TOT_SNOWFALL_IN,
    PROBABILITY_OF_PRECIPITATION_PCT,
    PROBABILITY_OF_SNOW_PCT
FROM STAGING.STG_FORECAST_DAY;
```

### 3.6 Validácia dát

```sql
SELECT COUNT(*)
FROM DIMENSIONAL.FACT_WEATHER_DAY
WHERE LOCATION_KEY IS NULL OR DATE_KEY IS NULL;

SELECT MIN(AVG_TEMP_F), MAX(AVG_TEMP_F)
FROM DIMENSIONAL.FACT_WEATHER_DAY;
```

---

## 4. Vizualizácia dát
![Dashboard vizualizácií](/img/Dashboard.png)


### 1. Priemerná predpovedaná teplota v čase

```sql
SELECT d.FULL_DATE, AVG(f.AVG_TEMP_F) AS AVG_TEMP_F
FROM DIMENSIONAL.FACT_WEATHER_DAY f
JOIN DIMENSIONAL.DIM_DATE d ON f.DATE_KEY = d.DATE_KEY
GROUP BY d.FULL_DATE
ORDER BY d.FULL_DATE;
```

### 2. Top 15 miest podľa priemernej predpovedanej teploty

```sql
SELECT l.CITY_NAME, AVG(f.AVG_TEMP_F) AS AVG_TEMP_F
FROM DIMENSIONAL.FACT_WEATHER_DAY f
JOIN DIMENSIONAL.DIM_LOCATION l ON f.LOCATION_KEY = l.LOCATION_KEY
GROUP BY l.CITY_NAME
ORDER BY AVG_TEMP_F DESC
LIMIT 15;
```

### 3. Priemerná teplota podľa typu zrážok

```sql
SELECT PRECIPITATION_TYPE, AVG(AVG_TEMP_F) AS AVG_TEMP_F
FROM DIMENSIONAL.FACT_WEATHER_DAY
GROUP BY PRECIPITATION_TYPE
ORDER BY AVG_TEMP_F DESC;
```

### 4. Rozdelenie predpovedí podľa teplotného pásma

```sql
SELECT WEATHER_BAND, COUNT(*) AS DAYS_COUNT
FROM DIMENSIONAL.FACT_WEATHER_DAY
GROUP BY WEATHER_BAND
ORDER BY DAYS_COUNT DESC;
```

### 5. Percento lokalít s očakávaným dažďom v čase

```sql
SELECT DATE_KEY,
       COUNT_IF(PRECIPITATION_IN > 0) * 100.0 / COUNT(*) AS PCT_LOCATIONS_WITH_RAIN
FROM DIMENSIONAL.FACT_WEATHER_DAY
GROUP BY DATE_KEY
ORDER BY DATE_KEY;
```

---

## Autori projektu

Matej Medvec
Juraj Pálenkáš
