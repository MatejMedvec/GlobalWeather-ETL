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
Dataset pochádza zo **Snowflake Marketplace** (Weather Source LLC: *Frostbyte – OnPoint ID Forecast Day*). Je voľne dostupný v sample verzii, obsahuje reálne hyper-lokálne meteorologické predpovede a je ideálny na precvičenie ELT procesov, dimenzionálneho modelovania a práce s pokročilými funkcionalitami Snowflake.

**Biznis procesy, ktoré dáta podporujú:**

* Plánovanie predaja v retail a food sektore (vplyv počasia na dopyt)
* Logistika a doprava (riziká zrážok a sneženia)
* Energetika (predikcia spotreby energie podľa teploty)
* Poistenie (hodnotenie rizík extrémneho počasia)

**Typy údajov:**
Časové údaje (dátumy), geografické údaje (poštové kódy, mestá, krajiny) a numerické metriky (teploty v °F, zrážky a sneženie v palcoch, vlhkosť, vietor, oblačnosť a pod.).

**Účel analýzy:**
Cieľom je pochopiť trendy teplôt a zrážok a ich vzťahy naprieč lokalitami a časom. Dimenzionálny model umožňuje rýchlo odpovedať na reportingové otázky, ako napríklad priemerné teploty, najteplejšie mestá alebo vplyv typu zrážok na teplotu.

**Zdrojová tabuľka:**
`WEATHER_SOURCE_LLC_FROSTBYTE.ONPOINT_ID.FORECAST_DAY` – denné predpovede počasia pre vybrané poštové kódy.

### ERD pôvodnej normalizovanej štruktúry dát (3NF)

![Normalizovaný ERD](/img/erd_normalized_3nf.png)

*Pôvodná štruktúra bola denormalizovaná; pre potreby ERD sme ju rozdelili do 3NF (LOCATION, DATE, WEATHER_DAY).*

---

## 2. Návrh dimenzionálneho modelu

Navrhli sme **hviezdicovú schému (Star Schema)** pozostávajúcu z jednej faktovej tabuľky a piatich dimenzií.

**Faktová tabuľka:** `DIMENSIONAL.FACT_WEATHER_DAY`

* Kompozitný kľúč: `DATE_KEY` + `LOCATION_KEY`
* Cudzie kľúče: `DATE_KEY`, `LOCATION_KEY`, `WEATHER_BAND`, `PRECIPITATION_TYPE`, `SOURCE_KEY`
* Hlavné metriky: `AVG_TEMP_F`, `PRECIPITATION_IN`, `SNOWFALL_IN`
* **Window functions:**

  * `TEMP_DAY_DELTA` – medzidenná zmena teploty (LAG)
  * `PRECIPITATION_7D_SUM` – 7-dňový kumulatívny úhrn zrážok (SUM OVER)

**Dimenzie:**

1. **DIM_DATE** – časová dimenzia (SCD typ 0)
2. **DIM_LOCATION** – lokalita (SCD typ 2 – surrogate key, `VALID_FROM`, `VALID_TO`, `IS_CURRENT`)
3. **DIM_WEATHER_BAND** – teplotné pásma (SCD typ 1)
4. **DIM_PRECIPITATION_TYPE** – typ zrážok (SCD typ 1)
5. **DIM_SOURCE** – zdroj dát (SCD typ 0)

### Star Schema (hviezdicová schéma)

![Star Schema](/img/star_schema_dimensional.png)

---

## 3. ELT proces v Snowflake

### 📥 Extract

Zdrojom dát je Snowflake Marketplace.

```sql
CREATE OR REPLACE DATABASE PEACOCK_GIRAFFE_PROJECT_DB;
USE DATABASE PEACOCK_GIRAFFE_PROJECT_DB;

CREATE OR REPLACE SCHEMA STAGING;

CREATE OR REPLACE TABLE STAGING.STG_FORECAST_DAY AS
SELECT *
FROM WEATHER_SOURCE_LLC_FROSTBYTE.ONPOINT_ID.FORECAST_DAY;
```

### Transform & Load

V tejto fáze sme vytvorili dimenzionálne tabuľky a faktovú tabuľku vrátane výpočtov pomocou window functions.
Podrobné SQL skripty sa nachádzajú v priečinku `/sql/`.

### Validácia dát

```sql
-- Kontrola chýbajúcich kľúčov
SELECT COUNT(*)
FROM DIMENSIONAL.FACT_WEATHER_DAY
WHERE LOCATION_KEY IS NULL
   OR DATE_KEY IS NULL;  -- očakávaný výsledok: 0

-- Rozsah teplôt
SELECT MIN(AVG_TEMP_F), MAX(AVG_TEMP_F)
FROM DIMENSIONAL.FACT_WEATHER_DAY;
```

---

## 4. Vizualizácia dát
![Dashboard vizualizácií](/img/Dashboard.png)
### 1. Priemerná predpovedaná teplota v čase

```sql
SELECT d.FULL_DATE,
       AVG(f.AVG_TEMP_F) AS AVG_TEMP_F
FROM DIMENSIONAL.FACT_WEATHER_DAY f
JOIN DIMENSIONAL.DIM_DATE d
  ON f.DATE_KEY = d.DATE_KEY
GROUP BY d.FULL_DATE
ORDER BY d.FULL_DATE;
```

**Interpretácia:**
Graf ukazuje výrazné sezónne výkyvy priemernej teploty naprieč všetkými lokalitami.

---

### 2. Top 15 miest podľa priemernej teploty

```sql
SELECT l.CITY_NAME,
       AVG(f.AVG_TEMP_F) AS AVG_TEMP_F
FROM DIMENSIONAL.FACT_WEATHER_DAY f
JOIN DIMENSIONAL.DIM_LOCATION l
  ON f.LOCATION_KEY = l.LOCATION_KEY
GROUP BY l.CITY_NAME
ORDER BY AVG_TEMP_F DESC
LIMIT 15;
```

**Interpretácia:**
Identifikuje najteplejšie lokality v datasete.

---

### 3. Priemerná teplota podľa typu zrážok

```sql
SELECT PRECIPITATION_TYPE,
       AVG(AVG_TEMP_F) AS AVG_TEMP_F
FROM DIMENSIONAL.FACT_WEATHER_DAY
GROUP BY PRECIPITATION_TYPE
ORDER BY AVG_TEMP_F DESC;
```

**Interpretácia:**
Dni bez zrážok majú najvyššiu priemernú teplotu, zatiaľ čo dni so snehom najnižšiu.

---

### 4. Rozdelenie dní podľa teplotného pásma

```sql
SELECT WEATHER_BAND,
       COUNT(*) AS DAYS_COUNT
FROM DIMENSIONAL.FACT_WEATHER_DAY
GROUP BY WEATHER_BAND
ORDER BY DAYS_COUNT DESC;
```

**Interpretácia:**
Väčšina dní spadá do miernych až teplých teplotných pásiem.

---

### 5. Percento lokalít s očakávaným dažďom v čase

```sql
SELECT DATE_KEY,
       COUNT_IF(PRECIPITATION_IN > 0) * 100.0 / COUNT(*) AS PCT_LOCATIONS_WITH_RAIN
FROM DIMENSIONAL.FACT_WEATHER_DAY
GROUP BY DATE_KEY
ORDER BY DATE_KEY;
```

**Interpretácia:**
Ukazuje variabilitu podielu lokalít s očakávaným dažďom v jednotlivých dňoch.

---

## Autori projektu

Matej Medvec
Juraj Pálenkáš
