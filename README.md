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
