# Nanopore-alapú eukarióta DNS-metilációs elemző Nextflow pipeline

**Projekt:** MSc Bioinformatika projekt\
**Hallgató:** Szűcs Krisztina\
**Mentor:** Dr. Freytag Csongor

## 1. A projekt célja és hatóköre

A projekt célja egy reprodukálható, moduláris Nextflow DSL2 pipeline és
egy saját, objektumorientált Python-analitikai csomag fejlesztése
nanopore-alapú eukarióta DNS-metilációs adatok feldolgozására.

A pipeline elsődleges bemenete a már bázishívott, módosított bázisokat
leíró BAM-tagságokat (MM/ML) tartalmazó **modBAM**. Opcionálisan
támogatható a POD5 bemenetből induló, CPU-n vagy GPU-n végzett Dorado
basecalling is.

A fő elemzési célok:

-   bemeneti fájlok és mintaleírók validálása;
-   technikai minőségellenőrzés;
-   metilációs hívások összesítése CpG-helyekre és genomrégiókra;
-   promoter-, CpG-sziget- és gén-test szintű aggregáció;
-   metagénprofilok előállítása;
-   differenciális metilációs régiók (DMR-ek) keresése;
-   összefoglaló HTML-jelentés és MultiQC-riport készítése.

A biológiai elemzés elsődlegesen CpG-helyekre fókuszál. Az 5mC és 5hmC
elkülönített elemzése csak akkor vállalható, ha a bemeneti adatok és a
használt basecaller/modifikációs modell ezt ténylegesen támogatja. A
modifikációtípusokat az elemzés előtt ellenőrizni kell; nem szabad
előzetesen feltételezni, hogy minden adatkészlet 5hmC-hívásokat
tartalmaz.

## 2. Fő kutatási kérdés

Mennyire reprodukálhatóan és megbízhatóan lehet nanopore
modBAM-adatokból metilációs profilokat, genomrégió-szintű összesítéseket
és differenciális metilációs régiókat előállítani egy moduláris
Nextflow-munkafolyamattal, és hogyan viszonyulnak az eredmények egy
publikált workshop-elemzéshez?

A projekt három különálló validációs szintet alkalmaz:

1.  **Technikai validáció:** a workflow végigfutása, fájlformátumok,
    kimenetek és futási metaadatok ellenőrzése.
2.  **Szintetikus validáció:** ismert, mesterségesen beültetett
    metilációs különbségeket tartalmazó adatokkal az érzékenység és a
    hamis pozitív találatok arányának mérése.
3.  **Módszertani összehasonlítás:** azonos vagy összehangolt
    beállításokkal kapott eredmények összevetése egy publikált
    workshop-folyamattal. Ez összehasonlítás, nem automatikus
    bizonyítéka a biológiai igazságnak.

## 3. Tervezett munkafolyamat

``` mermaid
flowchart TD
    A[Mintaleíró és bemenetek] --> B[Bemenetvalidálás]
    B --> C{Bemenet típusa}
    C -->|POD5| D[Dorado basecalling opcionálisan]
    C -->|modBAM| E[modBAM ellenőrzés]
    D --> F[Illesztés és előfeldolgozás]
    F --> G[Minőségellenőrzés]
    E --> G
    G --> H[modkit pileup / bedMethyl]
    H --> I[Metilációs helyek összesítése]
    I --> J[Promoter, CpG-sziget, gén-test aggregáció]
    I --> K[Metagénprofil]
    I --> L[DMR-elemzés]
    G --> M[QC és MultiQC]
    J --> N[HTML-jelentés]
    K --> N
    L --> N
    M --> N
```

### 3.1. Bemenet és előfeldolgozás

A pipeline a mintaleíró fájl alapján ellenőrzi a bemeneti fájlok
meglétét, olvashatóságát, a szükséges mezőket és a referencia-genomhoz
való illeszkedést. A modBAM esetében ellenőrizni kell a fejlécet, a
referenciát és az MM/ML tagek jelenlétét, ahol azok elvártak.

POD5-ből induló futás esetén a Dorado basecalling külön, opcionális ág.
A CPU/GPU módot paraméterezhetővé kell tenni, és rögzíteni kell a
basecaller, a modell és a szoftververzió adatait.

### 3.2. Minőségellenőrzés és metilációs hívások

A minőségellenőrzés összegzi többek között a bemeneti olvasatok és
illesztések jellemzőit, a lefedettséget, a metilációs hívások számát és
a hiányzó vagy szokatlan értékeket. A modkit segítségével bedMethyl
formátumú, helyszintű összesítés készülhet.

### 3.3. Genomrégió-szintű elemzések

A helyszintű hívásokból az alábbi régiótípusokra készül összesítés:

-   promóterek;
-   CpG-szigetek;
-   gén-testek;
-   szükség esetén egyéb, felhasználó által megadott BED-régiók.

A metagénprofil a gének normalizált testét és a kiválasztott környező
régiókat egységes bin-ekre osztva jeleníti meg. A régiódefiníciók és a
referencia-annotáció verzióját minden futásnál rögzíteni kell.

### 3.4. Differenciális metiláció

A DMR-elemzéshez statisztikai módszer vagy külön, tesztelt backend
használható, például DSS vagy dmrseq, az adatstruktúrához és a
mintaszámhoz igazítva. A pipeline-nak ellenőriznie kell, hogy a
kísérleti terv tartalmaz-e értelmezhető csoportokat és biológiai
ismétléseket.

A differenciális elemzés nem futtatható megalapozottan pusztán azért,
mert több BAM áll rendelkezésre: a minták biológiai kapcsolatait, az
ismétléseket, a csoportokat és az esetleges párosítást a forrásadatok
alapján kell tisztázni.

## 4. Szoftverarchitektúra

### 4.1. Nextflow DSL2

A Nextflow felel a lépések függőségeinek, párhuzamosításának,
újraindíthatóságának és futási metaadatainak kezeléséért. A folyamat
külön modulokra bontandó, hogy az egyes lépések önállóan is tesztelhetők
legyenek.

Tervezett komponensek:

-   bemenetvalidálás és mintaleíró feldolgozása;
-   opcionális basecalling;
-   illesztés és BAM-előfeldolgozás;
-   QC;
-   modkit pileup;
-   régiószintű aggregáció;
-   metagénprofil;
-   DMR-elemzés;
-   riportkészítés és MultiQC.

Ahol megfelelő, bevált nf-core modulok használhatók, például minimap2,
samtools, modkit, FastQC és MultiQC körül. Az nf-core/methylseq
biszulfit-szekvenálásra épülő célja miatt nem tekintendő közvetlen
helyettesítőnek a nanopore modBAM-alapú metilációs feldolgozáshoz. Az
nf-core/methylong használható kiindulópontként vagy összehasonlítási
alapként, de az illeszkedést és a verzióspecifikus bemeneti elvárásokat
ellenőrizni kell.

### 4.2. Saját Python-csomag

A saját, objektumorientált Python-csomag a workflow által előállított
táblázatos eredményeket dolgozza fel, és az elemzési logikát a
Nextflow-futtatástól elkülönítve tesztelhetővé teszi.

Javasolt osztályok:

-   `SampleSheet`: mintaleíró betöltése és validálása;
-   `BedMethylTable`: bedMethyl-adatok beolvasása és ellenőrzése;
-   `QCSummary`: minőségellenőrzési mutatók összesítése;
-   `AggregationStrategy`: genomrégiók szerinti aggregáció;
-   `MetageneProfiler`: metagénprofil előállítása;
-   `DifferentialAnalyzer`: differenciális elemzés interfésze;
-   `ReportBuilder`: ábrák és összefoglaló jelentés készítése.

Javasolt parancssori felület:

-   `validate`
-   `qc`
-   `aggregate`
-   `metagene`
-   `dmr`
-   `report`

A konkrét osztály- és parancsnevek a fejlesztés során változhatnak; a
lényeg az egyértelmű felelősségi kör, a dokumentált interfész és az
egységtesztelhetőség.

## 5. Bemenetek és paraméterezés

A saját pipeline mintaleírója például az alábbi mezőket tartalmazhatja:

  Mező           Jelentés
  -------------- ----------------------------------------
  `sample`       Egyedi mintanév
  `group`        Biológiai csoport
  `replicate`    Biológiai ismétlés azonosítója
  `input_type`   Például `modbam` vagy `pod5`
  `path`         Bemeneti fájl vagy fájlok elérési útja

A mintaleíró formátumát nem szabad automatikusan azonosnak tekinteni egy
meglévő workflow saját formátumával. Például az nf-core/methylong által
elvárt oszlopok és értékek eltérhetnek; adaptert vagy külön mintaleírót
kell használni a választott verzió dokumentációja szerint.

Fontos paraméterek:

-   referencia-genom és annotáció;
-   bemeneti fájltípus;
-   basecaller és modifikációs modell, ha szükséges;
-   minimális lefedettség;
-   vizsgált modifikációtípusok;
-   régiódefiníciók;
-   DMR-módszer és statisztikai küszöbök;
-   futási erőforrások és konténerbeállítások.

A futás minden lényeges paraméterét és szoftververzióját naplózni kell.

## 6. Adatok és esettanulmány

### 6.1. Publikált workshop-adatkészlet

Tervezett esettanulmány a NanoMethViz Bioc2024 workshop
Zenodo-adatkészlete:

-   Adatkészlet: https://zenodo.org/records/12747551
-   Workshop-munkafolyamat:
    https://shians.github.io/differential_methylation_workshop/articles/workflow.html
-   Kapcsolódó publikáció:
    https://pmc.ncbi.nlm.nih.gov/articles/PMC11907187/

A leírás alapján az adatkészlet hat, haplotípus szerint szétválasztott
egér neurális őssejtes BAM-ot tartalmaz: `nsc_1_mat`, `nsc_1_pat`,
`nsc_2_mat`, `nsc_2_pat`, `nsc_3_mat`, `nsc_3_pat`. Az adatok mm10
referenciához illesztettek; a kezdeti próbaelemzés korlátozható a 7-es
kromoszómára.

A referencia-genomot külön le kell tölteni, ha az archívum nem
tartalmazza. A fájlok integritását a közzétett ellenőrzőösszeggel kell
ellenőrizni; a leírásban szereplő MD5:
`e45e140d507d2686e62213796511106a`.

**Fontos értelmezési korlát:** a `mat` és `pat` jelölések anyai és apai
haplotípusra utalnak, nem automatikusan kezelés- és kontrollcsoportra. A
statisztikai összehasonlítás előtt tisztázni kell a biológiai ismétlések
szerkezetét, a párosítást és a publikált kísérleti tervet. Az elemzési
csoportokat nem szabad pusztán a fájlnevekből kikövetkeztetni.

A modBAM/MM/ML tagek és a ténylegesen mért modifikációtípusok
ellenőrzése kötelező. A modBAM-alapú bemenetnek a kiválasztott
methylong-verzióval való kompatibilitását, beleértve az esetleges
`--reset` opció szükségességét, a verzió dokumentációja alapján kell
eldönteni.

### 6.2. Szintetikus adatok

A szintetikus tesztadatok ismert metilációs mintázatokat és
mesterségesen beültetett különbségeket tartalmaznak. Ezekkel
ellenőrizhető, hogy a pipeline megtalálja-e az előre meghatározott
különbségeket, és milyen gyakran jelez tévesen eltérést.

A szintetikus tesztek legalább az alábbi eseteket fedjék le:

-   nincs beültetett különbség;
-   néhány ismert differenciális régió;
-   alacsony és magas lefedettség;
-   hiányzó értékek és szélsőséges értékek;
-   eltérő mintaszám és régióméret.

## 7. Validációs terv

  -----------------------------------------------------------------------
  Szint                   Ellenőrzés              Elvárt eredmény
  ----------------------- ----------------------- -----------------------
  Workflow smoke test     A tesztprofil           Minden szükséges
                          végigfutása             folyamat sikeres

  Bemenetvalidálás        Hibás vagy hiányzó      Érthető hibaüzenet,
                          fájlok                  korai leállás

  Egységtesztek           Aggregáció, QC,         Ismert bemenethez
                          táblázatkezelés         ismert eredmény

  Szintetikus adatok      Ismert differenciális   Érzékenység és hamis
                          régiók                  pozitív arány mérhető

  Esettanulmány           Workshop-adatok         Reprodukálható
                          feldolgozása            táblázatok és ábrák

  Módszertani összevetés  Publikált               Az eltérések
                          workshop-eredmények     dokumentált,
                                                  beállításokhoz kötött
                                                  értelmezése
  -----------------------------------------------------------------------

A korábbi próba során az nf-core/methylong 2.0.0 `test` profilja
sikeresen lefutott. A rögzített futás körülbelül 4 perc 24--26
másodpercig tartott; egy kis, 22-es kromoszómára korlátozott régióból
1508 bedMethyl-hely készült, az átlagos lefedettség megközelítőleg 5,1×
volt, és 1020 hely érte el az 5× lefedettséget.

Ezek az értékek **technikai próbaeredmények**, nem biológiai
következtetések és nem a teljes esettanulmány végleges eredményei. A
későbbi futásokat külön kell dokumentálni a paraméterekkel, verziókkal
és bemeneti adatokkal együtt.

A publikált workshop-eredményekkel való összevetés csak akkor
értelmezhető, ha a referencia, a régiók, a lefedettségi szűrők, a
csoportdefiníciók és a statisztikai módszerek kellően összehangoltak. A
`modkit`-tel végzett független feldolgozás opcionális módszertani
keresztellenőrzés lehet, de önmagában nem tekinthető biológiai
igazságnak.

## 8. Konténerek és reprodukálhatóság

A futtatási környezetet lehetőség szerint konténerekben kell rögzíteni,
és a Nextflow-paramétereket konfigurációs fájlban tárolni. Minden
végrehajtásnál megőrzendő:

-   Nextflow- és pipeline-verzió;
-   konténerképek és szoftververziók;
-   referencia és annotáció verziója;
-   teljes paraméterlista;
-   bemeneti fájlok ellenőrzőösszegei;
-   futási jelentések és erőforrás-összegzések.

A tesztprofil és a kisebb kromoszómarészlet gyors fejlesztési
visszajelzést adhat; a teljes esettanulmány külön futásként kezelendő.

## 9. Kimenetek

Javasolt kimeneti könyvtárstruktúra:

``` text
results/
├── qc/
├── methylation/
├── aggregation/
├── metagene/
├── dmr/
├── report/
├── multiqc/
└── pipeline_info/
```

A kimenetek között szerepeljenek:

-   mintánkénti QC-táblák és ábrák;
-   bedMethyl és normalizált metilációs táblázatok;
-   régiószintű összesítések;
-   metagénprofilok;
-   DMR-táblázatok, ahol az adatok és a kísérleti terv ezt lehetővé
    teszik;
-   HTML-jelentés;
-   MultiQC-jelentés;
-   pipeline- és futási metaadatok.

A jelentés különítse el a sikeres technikai futást, a minőségellenőrzés
eredményeit és a biológiai következtetéseket.

## 10. Kockázatok és enyhítésük

  -----------------------------------------------------------------------
  Kockázat                            Enyhítés
  ----------------------------------- -----------------------------------
  A modBAM formátum vagy a            Korai validálás, verziódokumentáció
  modifikációs tagek nem megfelelőek  és kis próbaminta

  Hiányzik a referencia vagy az       Külön letöltés, verziórögzítés és
  annotáció                           ellenőrzőösszeg

  Nem tisztázott a kísérleti terv     A publikáció és metaadatok
                                      ellenőrzése a DMR-elemzés előtt

  Kevés biológiai ismétlés            A statisztikai korlátok
                                      dokumentálása; túlzó
                                      következtetések kerülése

  Nagy futási erőforrásigény          Kromoszómarészleten végzett
                                      fejlesztői futások és
                                      paraméterezhető erőforrások

  Eltérő eszközverziók                Konténerek, verziórögzítés és
                                      reprodukálható konfiguráció
  -----------------------------------------------------------------------

## 11. Ütemterv

  ------------------------------------------------------------------------
  Szakasz                 Feladat                  Eredmény
  ----------------------- ------------------------ -----------------------
  1\.                     Irodalom és adatkészlet  Rögzített adat- és
                          ellenőrzése              módszertani terv

  2\.                     Bemenetvalidálás és QC   Működő első
                                                   workflow-szakasz

  3\.                     modkit/bedMethyl és      Hely- és régiószintű
                          aggregáció               táblázatok

  4\.                     Metagénprofil és riport  Ábrák és HTML-jelentés

  5\.                     DMR-elemzés és           Validációs eredmények
                          szintetikus tesztek      

  6\.                     Workshop-eredményekkel   Dokumentált módszertani
                          való összevetés          összehasonlítás

  7\.                     Tesztek, dokumentáció és Reprodukálható projekt
                          véglegesítés             és szakdolgozati
                                                   beszámoló
  ------------------------------------------------------------------------

Az ütemezést a rendelkezésre álló félévhez és a mentorral egyeztetett
mérföldkövekhez kell igazítani.

## 12. Tervezett végtermékek

1.  Verziózott Nextflow DSL2 workflow.
2.  Saját, dokumentált és tesztelt Python-analitikai csomag.
3.  Konfigurációs fájlok és mintaleíró-séma.
4.  Szintetikus tesztadatok és automatizált tesztek.
5.  A workshop-adatkészleten végzett reprodukálható esettanulmány.
6.  QC- és HTML-jelentések, valamint az elemzéshez szükséges táblázatok
    és ábrák.
7.  Dokumentáció a telepítésről, futtatásról, paraméterezésről és az
    ismert korlátokról.

## 13. Hivatkozások

-   NanoMethViz Bioc2024 workshop-adatkészlet:
    https://zenodo.org/records/12747551
-   Differenciális metilációs workshop munkafolyamata:
    https://shians.github.io/differential_methylation_workshop/articles/workflow.html
-   Kapcsolódó publikáció:
    https://pmc.ncbi.nlm.nih.gov/articles/PMC11907187/
