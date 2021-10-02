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

Programy využívající Spark budou programovány v jazyce Sparku nativním - 
Scale. Scala je víceparadigmatický jazyk se silným vlivem funkcionálního
programování běžící na JVM. Výhodou Scaly je vysoká expresivnost, obzvlášť
v porovnání s Javou. Nevýhodou Scaly je občasná nečitelnost, zbytečná
složitost, lehce zneužitelné vlastnosti (jako jsou třeba implicita) a
s nástupem Scaly 3 také zbytečné množství využitelné syntaxe.

#### Backend webové stránky

Backend bude implementovat REST API, na které se bude frontendová aplikace
dotazovat. REST API samozřejmě umožní uživatelům tvorbu vlastního klienta,
např. pokud by si uživatelé chtěli implementovat terminálového klienta 
pomocí Curses, ale také zajistí možnost tvorby např. mobilní aplikace,
bude-li to v budoucnosti žádoucí. K implementaci REST API poslouží
Kotlinní framework Ktor. Framework Ktor je moderní technologie vyvinutá
společností Jetbrains. Ktor slouží převážně k tvorbě asynchronních API.
Asynchronní zpracování dotazů umožní maximalizovat využití CPU a dosáhnout
vyšší efektivity, než pokud bychom implementovali vícevláknové zpracování,
jež může být neefektivní díky drahému context switchingu a zbytečnému
čekání během IO událostí.

#### Big Data

Při zpracování velkých dat bude samozřejmě použit framework Apache Hadoop.
Apache Hadoop je projekt sloužící právě za tímto účelem, nad Hadoopem je
poté postavená velká spousta užitečných technologií, jako je scheduler 
Yarn sloužící k alokaci zdrojů a vytváření exekutorů v Hadoop clusteru,
distribuovaný filesystem HDFS, apod.

K přijímání zpráv z přijímačů poslouží message broker Apache Kafka.
Kafka dokáže zpracovat řádově až desítky milionů zpráv za sekundu,
proto se jeví jako vhodná technologie k přijímání zpráv z ADS-B
přijímačů. Na základě modelu publisher-subscriber poté Kafka dokáže
předávat zprávy dál. Kafka garantuje, že bude každá zpráva doručena
alespoň jednou. Zprávy z Kafky lze zpracovat paralelně nebo
distribuovaně - pouze stačí využít sdílené `clientID` napříč
thready/exekutory.

K archivaci všech letových dat využijeme databázi Apache HBase. HBase
je distribuovaná NoSQL databáze ukládající data v HDFS. Výhodou HBase
je schopnost ukládat velmi velké množství dat právě za využití distribuce,
proto se hodí k ukládání historických dat, kterých bude velmi velké množství.

Původní Hadoop technologie MapReduce je dneska nahrazována převážně
modernějším Apache Sparkem, který bude využit i na backendu letového trackeru.
Spark poslouží nejen při zpracování dat z Kafky, ale také při zpracování
historických záznamů nebo při počítání statistik. Výhodou Sparku oproti
MapRedu je například extenzivnější a ergonomičtější API, vyšší flexibilita
při zpracování dat (např. volba mezi Scala lambdami nebo SQL dotazy), nebo
až desetinásobná rychlost v porovnání s MapRed, jíž jde dosáhnout právě díky
extenzivnějšímu API umožňujícímu chytřejší optimalizace.

#### Build systémy

C++ aplikace budou builděny systémem CMake. Závislosti C++ aplikací
budou instalovány package managerem - apt pro Debian v provozu, při vývoji
libovolný PM na libovolné distribuci.

Gradle poslouží jako build systém pro buildění JVM aplikací. Mezi velké
přednosti Gradlu lze zařadit podstatně vyšší čitelnost build konfigurace
využívající Groovy nebo Kotlin DSL, obzvlášť v porovnání s XML-based
project object modely systému Maven, které v případě složitějších
aplikací způsobují nežádoucí silné kolize hlavy programátora s nejbližší
stěnou. Gradle ovšem dokáže využít balíčky z Maven repozitářů, a proto
slouží jako dobrá moderní náhrada systému Maven.

#### Testování

K unit testování C++ aplikací lze využít knihovnu GoogleTest. JVM aplikace
můžou využít knihovnu JUnit.

Integrační testy poté lze realizovat za využití technologie Docker, jež
nám umožní v kontejneru spustit požadovanou technologii nebo komponentu,
s níž chceme testovat spojení.

#### Databáze

Jak již bylo zmíněno, k uchování všech historických dat bude využita
databáze Apache HBase. V HBase budou ukládána například kompletní letová
data všech letů za poslední rok.

K uchování dat, kterých bude rozumné množství, a mezi kterými budou
definovány jasné vztahy (relace), bude využita relační databáze
PostgreSQL. V Postgres můžou být uchovávány například informace 
o aerolinkách a jejich flotilách, podrobné informace o letadlech,
případně aktuální letová data bez kompletní historie. Postgres
tak umožní rychlejší a jednodušší vyhledávání než by v takovémto
případě umožnila HBase.

Poslední využitou databází bude velmi známá in-memory databáze
Redis. Redis poslouží, nepřekvapivě, ke cachování. Mapu světa lze rozdělit na 
určitý počet dílků. Data z databáze můžeme poté vytahovat ne podle
přesných hranic mapy ve webovém prohlížeči, ale podle dílů, do kterých
mapa zapadá. Ačkoliv se tak může stát, že budeme vracet více dat,
než je nutně potřeba, můžeme takto konkrétní díly mapy cachovat,
a pokud záznam v cachi není starší než jedna sekunda, můžeme data 
vydat z Redisu namísto Postgresu. Důvodem cachování je, že určité
body na mapě budou cílem podstatně vyššího zájmu než zbytek mapy.
Například můžeme předpokládat, že frekventovaná mezinárodní letiště budou
sledovanější než libovolné místo na Antarktidě - na Antarktidu půjde minimum
requestů, proto nám nevadí dané dotazy odpovědět výsledkem z Postgresu. Naopak
velké letiště jako LHR, JFK, FRA, důležité letecké koridory jakým je třeba
Arabský záliv, případně letecké události typu PAS, vzbudí vysoký zájem
a velké množství dotazů za sekundu, proto je možné i vhodné je odbavovat
převážně z Redisu, pouze s pravidelným obnovením cache dle aktuálních dat, čímž
se ulehčí také Postgresu, který poté dokáže rychleji odbavovat také dotazy
na nezajímavé místa s minimem leteckého provozu, jako jsou třeba Antarktida
nebo Letiště Leoše Janáčka (OSR), díky čemuž dokážeme obsloužit právě
až požadovaných 1000 současných uživatelů.

Při práci s databázemi lze použít kromě oficiálních klientů 
(např. `org.postgresql:postgresql` pro Postgres) také knihovnu Hibernate.

#### Virtualizace

Některé backendové komponenty lze virtualizovat pomocí technologie Docker.
Docker umožní jednoduchou správu závislostí, zabrání možným kolizím
balíčků, apod. Dále Docker zafunguje jako sandbox, v případě problémů,
padů, bugů apod. pak stačí zabít Docker kontejner, který nezpůsobí
větší škody např. na celém stroji, na kterém běží.

Docker kontejnery budou spouštěny v orchestračním nástroji Kubernetes.
Kubernetes slouží jako orchestrátor kontejnerů, dokáže spravovat a přiřazovat
zdroje clusteru, schedulovat a spouštět programy, apod. Dále nám
Kubernetes umožní škálování v případě vyššího trafficu, kdy, pokud
by backendové komponenty nestíhaly odpovídat na dotazy, může
Kubernetes dynamicky přiřadit komponentám více zdrojů a vytvořit
více podů a tak zvýšit propustnost.

Samozřejmě ne vše je vhodné virtualizovat. Hadoop, HDFS a HBase poběží
na separántím clusteru, podobně svůj cluster dostanou Kafka a Postgres.
Některé Spark joby bude lepší spouštět v Hadoop clusteru. Redis, REST 
API nebo jednodušší Spark joby lze ovšem spouštět v Kubernetu.

#### ADS-B Receivery

ADS-B receivery budou obyčejné zařízení Raspberry Pi, ke kterým bude
připojena anténa. Na těchto zařízeních poběží OS Raspbian. Software
přijímače bude poté naprogramován v programovacím jazyce C++.
Síť přijímačů bude globální a každý přijímač bude jednou za sekundu
zapisovat jednu zprávu za každé detekované letadlo. Dat tak bude
velmi velké množství - a proto budou zprávy zapisovány do Kafky,
která dokáže i tak velké množství dat zpracovat.

### Frontend

Frontend bude implementován za využití knihovny React, API Mapy.cz
a knihovny JAK. Knihovna JAK (JAvascriptová Knihovna) je knihovna
vytvořená firmou Seznam.cz a je využívána v implementaci API
Mapy.cz, proto se jí při využití daného API nelze vyhnout.
React je v současné době nejvyužívanější Javascriptovou knihovnou
určenou k tvorbě UI/webových stránek.

## Key Issues in Software Design

### Concurrency

Souběžné zpracování Velkých Dat bude zajištěno Sparkem, který
dokáže zajistit správnou distribuci datasetu i výpočtu.

Dotazy na veřejné REST API budou zpracovány asynchronně a všechny
na sobě budou nezávislé. V případě velkého množství dotazů poté
budou naškálovány pody v Kubernetu, kontejnery ovšem na sobě
budou navzájem nezávislé, budou zapisovat pouze data konkrétního
uživatele, převážně však budou číst.

Jediný možný problém s paralelním přístupem se může objevit
k datům v databázích. Zde by ale mělo bohatě stačit nastavit
správnou úroveň zamykání. Globální data budou zapisována
pouze interně z jedné Sparkové komponenty. Navíc tolik nevadí
ani případné drobné zpoždění - uživateli je jedno, jestli je 
letadlo na mapě o sekundu opožděné, pravděpodobně si toho ani
nevšimne. Uživatelé budou zapisovat pouze vlastní uživatelská
data (např. uložené filtry apod.), v takovém případě ovšem
neočekáváme paralelní dotazy, a tak je nad rámec zamykání
na straně DB neřešíme.

### Control and handling of events

Spark nám umožní streamovat zprávy z Kafky a zpracovat každou zprávu
nezávisle. Sparku takto nastavíme sekvenci transformací (specifkované
pomocí Spark SQL nebo Scala lambda funkcí), které se nad každou
zprávou mají provést, poté spustíme streaming, Spark zareaguje
na každou zprávu a naše transformace provede.

V REST API budou díky asynchronní komunikaci často využívány callbacky -
asynchronním funkcím se předají funktory, které dokážou reagovat na
události, jako jsou dokončený přenos, chyby, apod.

Data se do databází dostanou skrze pipelajny naprogramové pomocí
frameworku Spark.

### Data Persistence

Všechna příchozí letová data budou okamžitě ukládána v HBase, kde
budou uložena po dobu následujícího roku.

Informace o aktuálních letech, stejně jako informace o letištích,
aerolinkách, letadlech, případně uživatelské informace, tedy data,
ve kterých potřebujeme často vyhledávat a mezi kterými existují jasně
definované relace, se kterými zároveň potřebujeme pracovat, budou
ukládána v PostgreSQL.

Redis bude použit pouze ke cachování. 

Apache HBase je postavená nad HDFS, PostgreSQL poběží na vlastním
clusteru. Tím zajistíme replikaci dat a zabráníme ztrátě dat
v případě např. problémů s hardwarem.

### Distribution of Components

Backend bude rozdělen do několika clusterů. Kromě Postgres a Kafka
clusteru zde bude také Hadoop cluster a Kubernetes cluster.
Alokaci zdrojů a spouštění komponent nad danými clustery zajistí
schedulery pro danou technologii - Yarn pro Hadoop a Kubernetes
scheduler v Kubernetu.

Při komunikaci s Kafkou budou využity oficiální Kafka knihovny
implementující protokol pro interakci s API Kafky.

Při komunikaci s databázemi budou využity oficiální JVM knihovny.

Komunikace mezi vlastními komponentami (minimálně mezi FE a BE,
počet komponent však může s časem růst) poté bude probíhat přes
protokol HTTP, všechny komponenty budou implementovat REST API.

### Error and Exception Handling and Fault Tolerance

Za provozu se, stejně jako všude jinde, můžou objevit, a určitě
také objeví, očekávané i neočekávané chyby a problémy.

Jako jeden z možných problémů lze zmínit invalidní zprávu v Kafce.
Taková zpráva může například obsahovat data v invalidním formátu, 
nesmyslná data (referovat na neexistující let, obsahovat
timestamp z budoucnosti, apod.). Takové zprávy by měly být
odfiltrovány již při čtení z Kafky.

Dále se může stát, že databáze neodpoví backendu, nebo že
backend neodpoví frontendu. V takovém případě by bylo vhodné
aproximovat pohyb letadel na obloze na základě jejich aktuální
pozice, směru pohybu a rychlosti vůči zemi. Tím sice nemusíme dostat
přesnou polohu letadla, protože letadla můžou zpomalovat, zrychlovat
či zatáčet, zabráníme tím však alespoň problikávání, pokud by například
došlo k situaci, že databáze odpoví jen co druhou sekundu, a tak
se na mapě periodicky budou střídat dva stavy - reálný stav letového
provozu a prázdná obloha.

Samozřejmě vždy existuje vysoká šance invalidního requestu ze strany
uživatele. V takovém případě je vhodné vrátit HTTP status 404, a to
i v případě, že cesta zadaná uživatelem skutečně existuje, uživatel
k ní ovšem nemá přístup.

Jako ochrana proti chybám nám také slouží virtualizace a Kubernetes.
V případě chyby a případného pádu programu, nebo dokonce celého
podu, bude ovlivněn pouze jeden pod, Kubernetes jej tak okamžitě
může nahradit podem novým. Zároveň můžeme podů vytvořit víc a tím
zabránit výpadku služby. V případě závažné chyby v softwaru dále
můžeme pouze upravit specifikaci Kubernetes nasazení, aby se využíval
starší Docker image obsahující funkční zdrojový kód.

Spoustě chyb však dokáže zabránit null-safety Kotlinu. Kotlin je,
narozdíl od jazyka Java, null safe, a dokáže tak bezpečně zabránit
NPE, pokud si je tvrdohlavý programátor nevyvolá sám.

Dále lze spoustu chyb zachytit unit testováním.

### Interaction and Presentation

Framework Ktor je relativně drobný framework sloužící čistě k tvorbě
asynchronních REST API. Podobně jako třeba Python framework Flask
tak neobsahuje ORM, neimplementuje modely MVC nebo MVVM. To však
ani není potřeba - API bude relativně jednoduché, převážně bude číst
letová data (ovšem dovolí i zápis uživatelských dat), frontendu ovšem
bude všechna data posílat ve formátu JSON (samozřejmě s výjimkou
statických souborů, jako jsou grafika, HTML, skripty). Na straně BE
tak stačí data serializovat do formátu JSON těsně přes odesláním, a
to například pomocí Kotlin knihovny 
`org.jetbrains.kotlinx:kotlinx-serialization` nebo Java knihovny
`org.json:json`.

Veškeré renderování a interakce s uživatelem poté bude probíhat
pouze na frontendu. V případě webové aplikace bude renderování
zajišťovat knihovna React, díky využití REST API ovšem nebude náročné
implementovat také jiné klienty.

### Security

V dnešním světě je standardem schovávat provozní komponenty za Nginx
nebo Apache server. Nginx nebo Apache nám tímto způsobem zajistí
šifrování trafficu pomocí SSL.

Kubernetes i Hadoop jsou široce rozšířené technologie, a proto také
obsahují řadu vlastních i third party security řešení. Hadoop například
pro autentifikaci využívá technologii Kerberos.

Kubernetes dále může zvýšit bezpečnost nejen tím, že jednotlivé programy
běží v kontejneru, ale také nastavením práv, zákazu přístupu do venkovní
sítě, apod.

Kontejnerizace dále slouží jako ochrana proti výpadku služby, a to tak,
že můžeme spustit více podů pro každou službu, dané pody monitorovat, 
a v případě problémů zabít problematický pod a nastartovat nový.
Kubernetes a Docker samozřejmě nejsou spásou světa, ovšem `rm -rf /`
v Docker kontejneru má méně destruktivní následky než `rm -rf /`
na fyzickém stroji.

```
    .--.  
   |o_o |  
   |:_/ |  
  //   \ \  
 (|     | )  
/'\_   _/`\  
\___)=(___/  
```

