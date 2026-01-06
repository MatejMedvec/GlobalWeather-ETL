# GlobalWeather-ETL

Tento repozitár predstavuje ukážkovú implementáciu **ELT procesu v prostredí Snowflake** a návrh **dátového skladu s hviezdicovou schémou (Star Schema)** na základe meteorologických dát. Projekt pracuje s datasetom **Weather Source LLC – Frostbyte: OnPoint ID Forecast Day**, ktorý je dostupný prostredníctvom **Snowflake Marketplace** a obsahuje denné hyper-lokálne predpovede počasia pre vybrané geografické lokality.

Cieľom projektu je analyzovať **teplotné a zrážkové trendy v čase a priestore** a demonštrovať návrh **normalizovaného modelu (3NF)** aj **dimenzionálneho modelu** vhodného pre analytické dotazy a reporting.

Výsledný dátový model umožňuje **multidimenzionálnu analýzu** meteorologických údajov, ako je porovnanie priemerných teplôt medzi lokalitami, analýza výskytu zrážok alebo sledovanie vývoja počasia v čase.

Projekt slúži ako **referenčná ukážka správnej dokumentácie, implementácie ELT procesu a tvorby vizualizácií** pre záverečný projekt z predmetu *Databázové technológie*.

---

## 1. Úvod a popis zdrojových dát

V tomto projekte analyzujeme dáta o **denných predpovediach počasia** pre rôzne geografické lokality po celom svete.  
Cieľom analýzy je porozumieť:

- vývoju teplôt v čase,
- výskytu a intenzite zrážok,
- rozdielom v počasí medzi lokalitami,
- vzťahom medzi typom zrážok a teplotou.

Zdrojové dáta pochádzajú zo **Snowflake Marketplace** a sú poskytované spoločnosťou **Weather Source LLC** v rámci datasetu *Frostbyte – OnPoint ID Forecast Day*, dostupného na nasledujúcom odkaze:  
https://app.snowflake.com/marketplace/listing/GZTSZAS2KF/weather-source-llc-frostbyte

Dataset obsahuje tri hlavné tabuľky v rámci normalizovaného modelu (3NF):

- **DATE** – časové údaje,
- **LOCATION** – geografické údaje,
- **WEATHER_DAY** – denné meteorologické merania.

Účelom ELT procesu bolo tieto dáta pripraviť, transformovať a sprístupniť pre **viacdimenzionálnu analýzu** pomocou dimenzionálneho modelu typu **Star Schema**.

---

### 1.1 Dátová architektúra

Surové zdrojové dáta sú usporiadané v relačnom modeli v tretej normálnej forme (3NF), ktorý je znázornený na **entitno-relačnom diagrame (ERD)**:

<p align="center">
  <img src="/img/erd_normalized_3nf.png" alt="Normalizovaný ERD" width="700">
</p>

<p align="center">
  <em>Obrázok 1 Entitno-relačná schéma normalizovaného modelu (3NF)</em>
</p>

---

## 2. Návrh dimenzionálneho modelu

Na základe normalizovanej štruktúry zdrojových dát bol navrhnutý **dimenzionálny model typu Star Schema**, ktorý pozostáva z **jednej faktovej tabuľky** a **piatich dimenzií**. Model je optimalizovaný pre analytické dotazy a reporting v prostredí dátového skladu.

<p align="center">
  <img src="/img/star_schema_dimensional.png" alt="Star Schema" width="800">
</p>

<p align="center">
  <em>Obrázok 2: Dimenzionálny model typu Star Schema</em>
</p>

### Faktová tabuľka

**DIMENSIONAL.FACT_WEATHER_DAY**

- **Kompozitný kľúč:** `DATE_KEY`, `LOCATION_KEY`
- **Cudzie kľúče:**  
  `DATE_KEY`, `LOCATION_KEY`, `WEATHER_BAND`, `PRECIPITATION_TYPE`, `SOURCE_KEY`
- **Hlavné metriky:**  
  `AVG_TEMP_F`, `PRECIPITATION_IN`, `SNOWFALL_IN`
- **Odvodené metriky (analytické funkcie):**
  - `TEMP_DAY_DELTA` – medzidenná zmena priemernej teploty,
  - `PRECIPITATION_7D_SUM` – sedemdňový kumulatívny úhrn zrážok.

Faktová tabuľka uchováva denné meteorologické merania pre jednotlivé lokality a slúži ako centrálna analytická entita dimenzionálneho modelu.

### Dimenzie

- **DIM_DATE** – časová dimenzia (SCD typ 0),
- **DIM_LOCATION** – dimenzia lokality s historizáciou zmien (SCD typ 2),
- **DIM_WEATHER_BAND** – klasifikácia teplotných pásiem (SCD typ 1),
- **DIM_PRECIPITATION_TYPE** – typ zrážok (SCD typ 1),
- **DIM_SOURCE** – zdroj meteorologických dát (SCD typ 0).

Navrhnutý dimenzionálny model umožňuje **viacdimenzionálnu analýzu** vývoja počasia v čase, porovnávanie lokalít a efektívne vytváranie analytických pohľadov a vizualizácií.

---

## 3. ELT proces v Snowflake

ELT (Extract–Load–Transform) je prístup spracovania dát, pri ktorom sú zdrojové dáta najskôr sprístupnené zo zdrojového systému, následne uložené do databázy a transformácie sú vykonávané priamo v databázovom prostredí. Tento prístup je typický pre cloudové dátové sklady, ako je Snowflake.

---

### 3.1 Vytvorenie databázy a schém

V tomto kroku je vytvorená databáza projektu a jednotlivé schémy reprezentujúce vrstvy ELT architektúry: **STAGING**, **NORMALIZED** a **DIMENSIONAL**

```sql
CREATE OR REPLACE DATABASE PEACOCK_GIRAFFE_PROJECT_DB;
USE DATABASE PEACOCK_GIRAFFE_PROJECT_DB;

CREATE OR REPLACE SCHEMA STAGING;
CREATE OR REPLACE SCHEMA DIMENSIONAL;
CREATE OR REPLACE SCHEMA NORMALIZED;
```
---

### 3.2 Extract & Load – načítanie dát zo Snowflake Marketplace do STAGING

Keďže zdrojový dataset pochádza priamo zo Snowflake Marketplace, fázy Extract a Load sú realizované v jednom kroku.
Príkaz SELECT zabezpečuje prístup k zdrojovým dátam (Extract), zatiaľ čo CREATE TABLE AS SELECT zabezpečuje ich fyzické uloženie do staging vrstvy (Load).

```sql
CREATE OR REPLACE TABLE STAGING.STG_FORECAST_DAY AS
SELECT *
FROM WEATHER_SOURCE_LLC_FROSTBYTE.ONPOINT_ID.FORECAST_DAY;
```
---

### 3.3 Transform – tvorba dimenzií

V transformačnej fáze sú zo staging vrstvy vytvorené jednotlivé dimenzie dimenzionálneho modelu. Transformácie zahŕňajú výber relevantných atribútov, odvodenie časových charakteristík a klasifikáciu meteorologických údajov do analyticky využiteľných kategórií.

#### DIM_DATE
Časová dimenzia obsahuje odvodené atribúty z dátumu predpovede a je implementovaná ako SCD typ 0, keďže historické hodnoty sa nemenia.

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

#### DIM_LOCATION
Dimenzia lokality uchováva informácie o geografickej polohe a je navrhnutá ako pomaly sa meniaca dimenzia typu 2 (SCD 2), čo umožňuje historizáciu zmien atribútov lokality v čase.

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
Dimenzia teplotných pásiem klasifikuje priemerné denné teploty do logických kategórií. Je implementovaná ako SCD typ 1, pri ktorom sa zmeny hodnôt prepíšu bez uchovania histórie, keďže historický stav tejto klasifikácie nie je analyticky relevantný.

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
Dimenzia typu zrážok rozdeľuje dni podľa výskytu dažďa, sneženia alebo absencie zrážok. Rovnako ako pri teplotných pásmach je použitý SCD typ 1, keďže ide o odvodenú klasifikáciu bez potreby historizácie.

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
Dimenzia zdroja dát obsahuje základné informácie o pôvode datasetu. Keďže tieto údaje sú nemenné, dimenzia je implementovaná ako SCD typ 0.

```sql
CREATE OR REPLACE TABLE DIMENSIONAL.DIM_SOURCE AS
SELECT
    1 AS SOURCE_KEY,
    'Weather Source LLC' AS PROVIDER,
    'Frostbyte' AS DATASET_NAME,
    'Snowflake Marketplace' AS INGEST_METHOD;
```
---

### 3.4 Load – vytvorenie faktovej tabuľky

V tomto kroku sú dáta načítané do centrálnej faktovej tabuľky FACT_WEATHER_DAY, ktorá spája jednotlivé dimenzie a obsahuje denné meteorologické merania.
V súlade s ELT prístupom sú transformácie a výpočty odvodených metrík realizované priamo počas načítania dát do cieľovej tabuľky.

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
---

### 3.5 Normalizovaný model (3NF)

Popri dimenzionálnom modeli je vytvorený aj normalizovaný model v tretej normálnej forme (3NF), ktorý reprezentuje pôvodnú relačnú štruktúru zdrojových dát a slúži najmä na dokumentačné účely.

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
---

### 3.6 Validácia dát

Záverečný krok ELT procesu sa zameriava na základnú validáciu dát, konkrétne na kontrolu integrity cudzích kľúčov a overenie rozsahov hodnôt hlavných metrík vo faktovej tabuľke.

```sql
SELECT COUNT(*)
FROM DIMENSIONAL.FACT_WEATHER_DAY
WHERE LOCATION_KEY IS NULL OR DATE_KEY IS NULL;

SELECT MIN(AVG_TEMP_F), MAX(AVG_TEMP_F)
FROM DIMENSIONAL.FACT_WEATHER_DAY;
```

---

## 4. Vizualizácia dát
V tejto časti sú prezentované vybrané analytické dotazy nad dimenzionálnym modelom, ktoré demonštrujú možnosti viacdimenzionálnej analýzy meteorologických dát a slúžia ako podklad pre následné vizualizácie.

<p align="center">
  <img src="/img/Dashboard.png" alt="Dashboard vizualizácií" width="900">
</p>

<p align="center">
  <em>Obrázok 3: Dashboard vizualizácií meteorologických dát</em>
</p>

---

### 4.1 Priemerná predpovedaná teplota v čase

Dotaz analyzuje vývoj priemernej predpovedanej teploty v čase naprieč všetkými lokalitami. Výsledok umožňuje identifikovať globálne teplotné trendy a sezónne výkyvy.

```sql
SELECT d.FULL_DATE, AVG(f.AVG_TEMP_F) AS AVG_TEMP_F
FROM DIMENSIONAL.FACT_WEATHER_DAY f
JOIN DIMENSIONAL.DIM_DATE d ON f.DATE_KEY = d.DATE_KEY
GROUP BY d.FULL_DATE
ORDER BY d.FULL_DATE;
```

---

### 4.2 Top 15 miest podľa priemernej predpovedanej teploty

Dotaz identifikuje lokality s najvyššou priemernou predpovedanou teplotou za celé sledované obdobie. Analýza je vhodná na porovnanie klimatických podmienok medzi mestami.

```sql
SELECT l.CITY_NAME, AVG(f.AVG_TEMP_F) AS AVG_TEMP_F
FROM DIMENSIONAL.FACT_WEATHER_DAY f
JOIN DIMENSIONAL.DIM_LOCATION l ON f.LOCATION_KEY = l.LOCATION_KEY
GROUP BY l.CITY_NAME
ORDER BY AVG_TEMP_F DESC
LIMIT 15;
```

---

### 4.3 Priemerná teplota podľa typu zrážok

Cieľom dotazu je preskúmať vzťah medzi typom zrážok a priemernou teplotou. Výsledok ukazuje, ako sa teplota líši v dňoch bez zrážok, s dažďom alebo so snežením.

```sql
SELECT PRECIPITATION_TYPE, AVG(AVG_TEMP_F) AS AVG_TEMP_F
FROM DIMENSIONAL.FACT_WEATHER_DAY
GROUP BY PRECIPITATION_TYPE
ORDER BY AVG_TEMP_F DESC;
```

---

### 4.4 Rozdelenie predpovedí podľa teplotného pásma

Dotaz zobrazuje počet dní spadajúcich do jednotlivých teplotných pásiem. Slúži na analýzu frekvencie výskytu rôznych teplotných podmienok v sledovanom období.

```sql
SELECT WEATHER_BAND, COUNT(*) AS DAYS_COUNT
FROM DIMENSIONAL.FACT_WEATHER_DAY
GROUP BY WEATHER_BAND
ORDER BY DAYS_COUNT DESC;
```

---

### 4.5 Percento lokalít s očakávaným dažďom v čase

Dotaz vypočítava percentuálny podiel lokalít, v ktorých sa v daný deň očakával výskyt zrážok. Analýza umožňuje sledovať časový vývoj pravdepodobnosti dažďa na úrovni celého sledovaného územia.

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
