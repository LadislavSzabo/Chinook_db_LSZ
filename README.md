# **ETL proces datasetu Chinook database**

Databáza Chinook reprezentuje digitálny obchod s hudbou a obsahuje informácie o zákazníkoch, zamestnancoch, interpretoch, albumoch, skladbách, objednávkach a ďalších súvisiacich údajoch. Chinook je alternatívou k databáze Northwind, ktorá sa tiež často používa na podobné účely. Výsledný dátový model umožňuje multidimenzionálnu analýzu a vizualizácie dôležitých udajob, napríklad: vizualizácia žánrov s najvyšším predajom alebo albumov s najvyšším predajom. Pre analýzu dát tento repozitár obsahuje implementáciu ETL procesu v Snowflake.

---
## **1. Úvod a popis zdrojových dát**
Cieľom semestrálneho projektu je analyzovať údaje o zamestnancoch hudobného obchodu, zákazníkoch, skladbách, albumoch a príjmoch.

Zdrojové dáta pochádzajú z Kaggle datasetu dostupného [tu](https://www.kaggle.com/datasets/jacopoferretti/chinook-music-database/data). Dataset obsahuje 10 tabuliek:
- `Artist`
- `Album`
- `Track`
- `Playlist`
- `MediaType`
- `Genre`
- `InvoiceLine`
- `Invoice`
- `Customer`
- `Employee`

Účelom ETL procesu bolo tieto dáta pripraviť, transformovať a sprístupniť pre viacdimenzionálnu analýzu.

---
### **1.1 Dátová architektúra**

### **ERD diagram**
Surové dáta sú usporiadané v relačnom modeli, ktorý je znázornený na **entitno-relačnom diagrame (ERD)**:

<p align="center">
  <img src="https://github.com/LadislavSzabo/Chinook_db_LSZ/blob/main/Chinook_db_Erd_diagram.png" alt="ERD Schema">
  <br>
  <em>Obrázok 1 Entitno-relačná schéma Chinook dataset</em>
</p>

---
## **2 Dimenzionálny model**

Navrhnutý bol **hviezdicový model (star schema)**, pre efektívnu analýzu kde centrálny bod predstavuje faktová tabuľka **`fact_invoice_line`**, ktorá je prepojená s nasledujúcimi dimenziami:
- **`dim_track`**: Obsahuje podrobné informácie o skladbách (názov skladby, Dĺžka skladby v milisekundách, Veľkosť skladby v bajtoch,Predajná cena za skladbu)
- **`dim_invoice`**: Obsahuje údaje o faktúry, ako sú v Dátum vystavenia faktúry, Adresa pre fakturáciu, Mesto pre fakturáciu,Štát,Krajina,PSČ a Celková suma faktúry. 
- **`dim_customer`**: Obsahuje podrobné informácie o zákazníkoch ( Meno, priezvisko, adresa, Mesto, Štát,Krajina,PSČ,Telefónne číslo,Email).
- **`dim_date`**: Zahrňuje informácie o dátumoch hodnotení (deň, mesiac, rok, Deň v týždni = číslo) .
- **`dim_time`**: Obsahuje podrobné časové údaje (hodina, AM/PM).

Štruktúra hviezdicového modelu je znázornená na diagrame nižšie. Diagram ukazuje prepojenia medzi faktovou tabuľkou a dimenziami, čo zjednodušuje pochopenie a implementáciu modelu.

<p align="center">
  <img src="https://github.com/LadislavSzabo/Chinook_db_LSZ/blob/main/Chinook_db_star_schema.png" alt="Star Schema">
  <br>
  <em>Obrázok 2 Schéma hviezdy pre Chinook dataset</em>
</p>

---
## **3. ETL proces v Snowflake**
ETL proces pozostával z troch hlavných fáz: `extrahovanie` (Extract), `transformácia` (Transform) a `načítanie` (Load). Tento proces bol implementovaný v Snowflake s cieľom pripraviť zdrojové dáta zo staging vrstvy do viacdimenzionálneho modelu vhodného na analýzu a vizualizáciu.

---
### **3.1 Extract (Extrahovanie dát)**
Dáta zo zdrojového datasetu (formát `.csv`) boli najprv nahraté do Snowflake prostredníctvom interného stage úložiska s názvom `DRAGON_CHINOOK_STAGE`. Stage v Snowflake slúži ako dočasné úložisko na import alebo export dát. Vytvorenie stage bolo zabezpečené príkazom:

#### Príklad kódu:
```sql
CREATE OR REPLACE STAGE DRAGON_CHINOOK_STAGE;
```
Do stagingového prostredia boli vložené dátové súbory týkajúce sa chinok dataset udaje. Následne boli tieto súbory pomocou príkazu COPY INTO načítané do stagingových tabuliek. Každá tabuľka mala priradený špecifický príkaz na import dát.

```sql
COPY INTO ALBUM
FROM @DRAGON_CHINOOK_STAGE/Album.csv
FILE_FORMAT = (TYPE = 'CSV' FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1)
ON_ERROR = 'CONTINUE';
```
<p align="center">
  <img src="https://github.com/LadislavSzabo/Chinook_db_LSZ/blob/main/Copy.png" alt="Copy into">
  <br>
  <em>Obrázok 3 Priklad COPY INTO</em>
</p>
V prípade nekonzistentných záznamov bol použitý parameter `ON_ERROR = 'CONTINUE'`, ktorý zabezpečil pokračovanie procesu bez prerušenia pri chybách.

---
### **3.1 Transfor (Transformácia dát)**

V tejto fáze boli dáta zo staging tabuliek vyčistené, transformované a obohatené. Hlavným cieľom bolo pripraviť dimenzie a faktovú tabuľku, ktoré umožnia jednoduchú a efektívnu analýzu.

Dimenzie boli navrhnuté na poskytovanie kontextu pre faktovú tabuľku. `Dim_track` obsahuje informácie o jednotlivých skladbách.
```sql
CREATE OR REPLACE TABLE dim_track AS
SELECT 
  DISTINCT
	TRACKID,
	NAME,
	COMPOSER,
	MILLISECONDS,
	BYTES,
	UNITPRICE,
FROM TRACK;
```

`Dim_customers` obsahuje informácie o zákazníkoch.
```sql
CREATE OR REPLACE TABLE dim_customer AS
SELECT 
  DISTINCT
	CUSTOMERID,
	FIRSTNAME,
	LASTNAME,
	COMPANY,
	ADDRESS,
	CITY,
	STATE,
	COUNTRY,
	POSTALCODE,
	PHONE,
	FAX,
	EMAIL,
	SUPPORTREPID
FROM CUSTOMER;
```

`Dim_invoice` obsahuje informácie o fakturáciu.
```sql
CREATE OR REPLACE TABLE dim_invoice AS
SELECT 
  DISTINCT
	CUSTOMERID,
	FIRSTNAME,
	LASTNAME,
	COMPANY,
	ADDRESS,
	CITY,
	STATE,
	COUNTRY,
	POSTALCODE,
	PHONE,
	FAX,
	EMAIL,
	SUPPORTREPID
FROM CUSTOMER;
```

Rekurzívne CTE pre `dim_time` a `dim_date`:
Klauza WITH RECURSIVE inicializuje rekurzívne CTE s názvom time_generator.
Počiatočná iterácia CTE (UNION ALL) vytvorí základný záznam s hour=0 a ampm='AM'.
Nasledujúce iterácie (SELECT) zvýšia hodnotu hour o 1 a aktualizujú ampm na základe podmienok:
Ak hour dosiahne 12, prepne ampm medzi 'AM' a 'PM'.
Inak ponechá ampm z predchádzajúcej iterácie.
Rekurzia pokračuje až kým hour nedosiahne 23.


```sql
CREATE OR REPLACE TABLE dim_time AS
WITH RECURSIVE time_generator AS (
    SELECT 0 AS hour, 'AM' AS ampm
    UNION ALL
    SELECT 
        hour + 1 AS hour,
        CASE WHEN hour + 1 = 12 THEN 
            CASE ampm WHEN 'AM' THEN 'PM' ELSE 'AM' END
        ELSE ampm END AS ampm
    FROM time_generator
    WHERE hour < 23
)
SELECT
    ROW_NUMBER() OVER (ORDER BY hour) AS time_id,
    hour,
    ampm
FROM time_generator;


CREATE OR REPLACE TABLE dim_date AS
WITH RECURSIVE date_generator AS (
    SELECT DATE '2000-01-01' AS dt 
    UNION ALL
    SELECT dt + INTERVAL '1 day'
    FROM date_generator
    WHERE dt < DATE '2024-12-31' 
)
SELECT
    TO_CHAR(dt, 'YYYYMMDD')::INT AS DateKey,
    dt AS Date,
    EXTRACT(YEAR FROM dt) AS Year,
    EXTRACT(MONTH FROM dt) AS Month,
    EXTRACT(DAY FROM dt) AS Day,
    EXTRACT(DAYOFWEEK FROM dt) AS DayOfWeek,
FROM date_generator;
```

Faktová tabuľka `fact_ratings` Táto faktová tabuľka obsahuje kľúčové údaje o predaji a spája sa s inými dimenznými tabuľkami.
```sql
CREATE OR REPLACE TABLE Fact_invoice_line AS
SELECT 
    ROW_NUMBER() OVER (ORDER BY i.InvoiceDate, t.Name) AS Fact_invoice_lineId,
    il.Quantity,
    il.UnitPrice,
    il.Quantity * il.UnitPrice AS Total,
    i.invoiceId,
    t.trackId,
    c.CustomerId,
    d.Datekey,
    ti.time_Id
FROM 
    InvoiceLine AS il 
JOIN
    Dim_invoice i ON il.InvoiceId = i.InvoiceId
JOIN
    Dim_track t ON il.TrackId = t.TrackId
JOIN
    Dim_customer c ON i.CustomerId = c.CustomerId
JOIN
    Dim_date d ON DATE(i.InvoiceDate) = d.Date
JOIN
    Dim_time ti ON 
        EXTRACT(HOUR FROM i.InvoiceDate) = ti.hour 
        AND 
        CASE 
            WHEN EXTRACT(HOUR FROM i.InvoiceDate) >= 12 THEN 'PM' 
            ELSE 'AM' 
        END = ti.ampm;
```

---
### **3.3 Load (Načítanie dát)**

Po úspešnom vytvorení dimenzií a faktovej tabuľky boli dáta nahraté do finálnej štruktúry. Na záver boli staging tabuľky odstránené, aby sa optimalizovalo využitie úložiska.
ETL proces v Snowflake umožnil spracovanie pôvodných dát z `.csv` formátu do viacdimenzionálneho modelu typu hviezda. Tento proces zahŕňal čistenie, obohacovanie a reorganizáciu údajov. Výsledný model umožňuje analýzu čitateľských preferencií a správania používateľov, pričom poskytuje základ pre vizualizácie a reporty.

---
## **4 Vizualizácia dát**

Tento dashboard poskytuje prehľad o predajných trendoch hudobných žánrov v rôznych krajinách. Vizualizácie umožňujú identifikovať najpredávanejšie žánre, zistiť, ktoré krajiny majú najväčší záujem o určitý typ hudby a získať cenné informácie o preferenciách zákazníkov na globálnej úrovni.

<p align="center">
  <img src="https://github.com/LadislavSzabo/Chinook_db_LSZ/blob/main/Chinook_visualization.png" alt="ERD Schema">
  <br>
  <em>Obrázok 4 Dashboard Chinook music datasetu</em>
</p>

---
### **Graf 1:Celkové tržby v čase**
Zobrazuje celkové tržby generované počas určitého časového obdobia. Os x predstavuje časové obdobie a os y predstavuje celkovú sumu tržieb.
```sql
SELECT 
    d.Date AS SalesDate, 
    SUM(f.Total) AS TotalSales
FROM 
    Fact_invoice_line f
JOIN 
    Dim_date d ON f.DateKey = d.DateKey
GROUP BY 
    d.Date
ORDER BY 
    d.Date;
```
---
### **Graf 2: Výkonnosť zamestnancov**
Zobrazuje predajný výkon rôznych zamestnancov. 

```sql
SELECT 
    CONCAT(e.FirstName, ' ', e.LastName) AS EmployeeName, 
    SUM(f.Total) AS TotalRevenue
FROM 
    Fact_invoice_line f
JOIN 
    Dim_customer c ON f.CustomerId = c.CustomerId
JOIN 
    Employee e ON c.SupportRepId = e.EmployeeId
GROUP BY 
    e.FirstName, e.LastName
ORDER BY 
    TotalRevenue DESC;
```
---
### **Graf 3: Najpredávanejšie žánre**
Táto vizualizácia zobrazuje najpopulárnejšie hudobné žánre na základe predajného objemu.

```sql
SELECT 
    g.Name AS Genre, 
    SUM(f.Quantity) AS TotalSold
FROM 
    Fact_invoice_line f
JOIN 
    Dim_track t ON f.TrackId = t.TrackId
JOIN 
    Genre g ON t.GenreId = g.GenreId
GROUP BY 
    g.Name
ORDER BY 
    TotalSold DESC;
```
---
### **Graf 4: Krajiny zoradené podľa tržieb**
ento graf zobrazuje celkové tržby generované z rôznych krajín. Os x predstavuje krajiny a os y predstavuje celkové tržby.
 
```sql
SELECT 
    c.Country AS CustomerCountry, 
    SUM(f.Total) AS TotalRevenue
FROM 
    Fact_invoice_line f
JOIN 
    Dim_customer c ON f.CustomerId = c.CustomerId
GROUP BY 
    c.Country
ORDER BY 
    TotalRevenue DESC;
```
---
### **Graf 5: Top 10 zákazníkov za celý život**
Táto vizualizácia zobrazuje 10 najlepších zákazníkov na základe ich celkového výdavku

```sql
SELECT 
    CONCAT(c.FirstName, ' ', c.LastName) AS CustomerName, 
    SUM(f.Total) AS LifetimeValue
FROM 
    Fact_invoice_line f
JOIN 
    Dim_customer c ON f.CustomerId = c.CustomerId
GROUP BY 
    c.FirstName, c.LastName
ORDER BY 
    LifetimeValue DESC
LIMIT 10;
```
---



Dashboard poskytuje komplexný pohľad na dáta, pričom zodpovedá dôležité otázky týkajúce sa čitateľských preferencií a správania používateľov. Vizualizácie umožňujú jednoduchú interpretáciu dát a môžu byť využité na optimalizáciu odporúčacích systémov, marketingových stratégií a knižničných služieb.

---

**Autor:** Ladislav Szabo
