# Letecký tracker

Zatím nepojmenovaný letecký tracker bude sloužit k trackování letů
na mapě. Tracker bude uživatelům prezentovat informace o momentálně
probíhajících letech, umožní vykreslení aktuálního leteckého provozu
na mapu, umožní vyhledávat letiště, lety, letadla, apod.

## Technologie

### Backend

#### Programovací jazyky

Backend webové stránky bude naprogramován v programovacím jazyce Kotlin.
K volbě tohoto jazyka vede například možnost využití JVM ekosystému,
ergonomičnost jazyka Kotlin v porovnání s Javou, ale také statické typování,
které lze považovat za výhodu Kotlinu při srovnání s Pythonem.

Programy využívající Spark budou programovány v jazyce Sparku nativním -- 
Scale. Scala je víceparadigmatický jazyk se silným vlivem funkcionálního
programování běžící na JVM. Výhodou Scaly je vysoká expresivnost, obzvlášť
v porovnání s Javou. Nevýhodou Scaly je občasná nečitelnost, zbytečná
složitost, lehce zneužitelné vlastnosti (implicita) a s nástupem Scaly 3
také zbytečné množství využitelné syntaxe.

#### Backend webové stránky

Backend bude implementovat REST API, na které se bude frontendová aplikace
dotazovat. REST API samozřejmě umožní uživatelům tvorbu vlastního klienta,
např. pokud by si uživatelé chtěli implementovat terminálového klienta 
pomocí Curses, ale také zajistí možnost tvorby např. mobilní aplikace
v budoucnosti. K implementaci REST API poslouží Kotlinní framework Ktor.
Framework Ktor je moderní technologie vyvinutá společností Jetbrains.
Ktor slouží převážně k tvorbě asynchronních API. Asynchronní zpracování
dotazů umožní maximalizovat využití CPU a dosáhnout vyšší efektivity
než pokud bychom implementovali vícevláknové zpracování, jež může být
neefektivní díky drahému context switchingu a zbytečnému čekání během
IO událostí.

#### Big Data

Při zpracování velkých dat bude samozřejmě použit framework Apache Hadoop.
Apache Hadoop je projekt sloužící právě za tímto účelem, nad Hadoopem je
poté postavená velká spousta užitečných technologií, jako je scheduler 
Yarn sloužící k alokaci zdrojů a vytváření exekutorů v Hadoop clusteru,
distribuovaný filesystem HDFS, apod.

K přijímání zpráv z přijímačů poslouží message broker Apache Kafka.
Kafka dokáže zpracovat řádově až desítky milionů zpráv za sekundu,
proto se jeví jako vhodná technologie k přijímání zpráv z ADS-B
přijímačů. Modelem publisher-subscriber poté Kafka dokáže předávat
zprávy dál. Kafka garantuje, že bude každá zpráva doručena alespoň
jednou. Zprávy z Kafky lze zpracovat paralelně nebo distribuovaně --
pouze stačí využít sdílené clientID napříč thready/exekutory.

K archivaci všech letových dat využijeme databázi Apache HBase. HBase
je distribuovaná NoSQL databáze ukládající data v HDFS. Výhodou HBase
je schopnost ukládat velmi velké množství dat právě za využití distribuce,
proto se hodí k ukládání historických dat, kterých bude velmi velké množství.

Původní Hadoop technologie MapReduce je dneska nahrazována převážně
modernějším Apache Sparkem, který bude využit i na backendu letového trackeru.
Spark poslouží nejen při zpracování dat z Kafky, ale také při zpracování
historických záznamů nebo při počítání statistik. Výhodou Sparku oproti
MapRedu je například extenzivnější a ergonomičtější API, vyšší flexibilita
při zpracování dat (např. Scala lambdy x SQL dotazy), nebo až desetinásobná
rychlost v porovnání s MapRed plynoucí právě díky extenzivnějšímu API
umožňujícímu chytřejší optimalizace.

#### Build systémy

C++ aplikace budou builděny systémem CMake. Závislosti C++ aplikací
budou instalovány package managerem -- apt pro Debian v provozu, při vývoji
libovolný pm na libovolné distribuci.

Gradle poslouží jako build systém JVM aplikací. Mezi velké přednosti Gradlu
lze zařadit podstatně vyšší čitelnost build konfigurace v porovnání XML-based
project object modely systému Maven, které v případě složitějších aplikací
způsobují nežádoucí silné kolize hlavy programátora s nejbližší stěnou. Gradle
ovšem dokáže využít balíčky z Maven repozitářů, a proto slouží jako dobrá
moderní náhrada systému Maven.

#### Testování

K unit testování C++ aplikací lze využít knihovnu GoogleTest. JVM aplikace
můžou využít knihovnu JUnit.

Integrační testy poté lze realizovat za využití technologie Docker, jež
nám umožní v kontejneru spustit požadovanou technologii nebo komponentu,
s níž chceme testovat spojení.

#### Databáze

#### Virtualizace

#### ADS-B Receivery

ADS-B receivery budou obyčejné zařízení Raspberry Pi, ke kterým bude
připojena anténa. Na těchto zařízeních poběží OS Raspbian. Software
přijímače bude poté naprogramován v programovacím jazyce C++.

### Frontend

Frontend bude implementován za využití knihovny React, API Mapy.cz
a knihovny JAK. Knihovna JAK (JAvascriptová Knihovna), je knihovna
vytvořená firmou Seznam.cz, a je využívána v implementaci API
Mapy.cz, proto se jí při využití daného API nelze vyhnout.
React je v současné době nejvyužívanější Javascriptovou knihovnou
určenou k tvorbě UI/webových stránek.

## Key Issues in Software Design

```
    .--.  
   |o_o |  
   |:_/ |  
  //   \ \  
 (|     | )  
/'\_   _/`\  
\___)=(___/  
```

