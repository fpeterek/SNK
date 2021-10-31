# Letecký tracker

---
Filip Peterek, PET0342

Online verze na Githubu: https://github.com/fpeterek/SNK/blob/master/p2/flight-tracker.md

Pro renderování MD lze využít i terminálový renderer `mdless` (v AUR balíček `ruby-mdless`).

---

Rozbor se týká projektu leteckého trackeru tvořeného v rámci předmětu Vývoj informačních
systémů (VIS).

Projekt je Open Source a je volně dostupný na Githubu: 
https://github.com/fpeterek/FlightLidar48

## Modularita

### Dodržení modularity

Jako příklad dodržení modularity lze uvést například fakt, že projekt leteckého trackeru
je rozdělen do více komponent (programů). Součástí projektu je například pipeline
sloužící ke zpracování vstupu. Jednotlivé kroky pipeline jsou na sobě ovšem nezávislé.
Spolu komunikují skrze message brokera (Kafku). Zprávy samozřejmě mají standardizovaný
formát, ovšem každý krok pipeline pouze přečte na vstupu zprávu v očekávaném formátu,
interně ji zpracuje, a na výstupu zase zapíše zprávu v cílovém formátu (případně data
zapíše do DB). Jednotlivé kroky pipeline ovšem nezajímá, jestli nějaké kroky probíhají
paralelně, kolik kroků danému kroku předchází, či které kroky následují. Díky tomu
je možné jednotlivé kroky rozdělit na více kroků, více kroků sloučit v jeden (možné,
ale pravděpodobně ne vhodné), jednotlivé kroky přepsat (klidně do jiných jazyků), apod.

Přístup do databáze nebo Kafky je abstrahován 
(separátní knihovnou)[https://github.com/fpeterek/FlightLidar48/tree/master/database],
a tak by při změne message brokera nebo databázového systému stačilo upravit tuto
knihovnu a přebuildit ostatní komponenty.

### Nedodržení modularity

Knihovna `org.fpeterek.flightlidar48.database` je také zajímavá z hlediska modularity.
Interně je rozdělena do modulů `util`, `database`, `kafka` a `json`. 

Package `database` slouží ke komunikaci s databází. Obsahuje Gateway třídy a Java
recordy reprezentující data v databázi. 

Package `kafka` abstrahuje komunikaci s Kafkou. Díky této abstrakci by se například
interně mohla knihovna `KafkaStreams` zaměnit za `Flink` a nikdo by si nevšiml. 

Package `json` obsahuje logiku sloužící k parsování JSON zpráv zasílaných Kafkou. Package
`kafka` však na package `json` nezávisí (ani naopak, `kafka` se nezajímá o obsah zpráv,
`json` se nezajímá o logiku streamování). 

Package `util` poté obsahuje třídu `ConfigLoader` a record `GeoPoint`. `GeoPoint` slouží
k ukládání lat/lon párů, `ConfigLoader` poté k načítání konfigurace z env proměnných. 

Všechny tyto package se balí jako jeden .jar archiv. V takovém případě můžeme název
`database` pro naši knihonu považovat za přinejmenším nešťastný. Zároveň je třeba zmínit,
že ačkoliv jsou jednotlivé package knihovny nezávislé, a můžeme tak mluvit o nulové
provázanosti (coupling), současně však lze mluvit o nulové souvislosti v rámci packagů
knihovny, neboť každý package knihovny řeší jinou funkcionalitu. Samozřejmě by bylo
vhodné jednotlivé package oddělit do vlastních knihoven, package `util` poté rozdělit
na dvě samostatné knihovny, autor na to v době před odevzdáním projektu však neměl čas,
a tak utrpěla soudržnost a správnost.

Jako další příklad špatné modularity lze uvést například metodu 
(`writeHealthcheck()`)[https://github.com/fpeterek/FlightLidar48/blob/126880dfb5d4ea768c05ca39b8bc9bea21238c91/database/database/src/main/java/org/fpeterek/flightlidar48/kafka/MessageWriter.java#L44]
třídy `MessageWriter`. Třída `MessageWriter` slouží k zápisu zpráv do message brokera.
Ke čtení zpráv poté slouží třída `MessageStream`. Problém s metodou `writeHealhcheck()`
spočívá v tom, že zapisuje 'natvrdo' zadefinovanou zprávu ve formátu JSON.
`MessageWriter` si tak metodou `writeHealthcheck()` zbytečně vynucuje způsob zpracování
healthchecku. Samozřejmě lze ke zpracování healthchecku využít také metodu `write`, které
lze dodat vlastní zprávu. Metoda `writeHealthcheck()` tak sice nezpůsobuje příliš velké
problémy a není ji třeba použít, programátorovi neznalému vnitřnímu fungování dané třídy
ovšem může evokovat nutnost využití dané metody. Veškeré problémy tak zde způsobuje
nešťastně navržený interface.

Healthcheck slouží k ověření funkčnosti čtení z Kafky. Každou minutu se do Kafky
zapíše Healthcheck zpráva. Program poté musí periodicky přečíst alespoň jednu zprávu
z Kafky za určitou dobu, jinak je spojení s Kafkou považováno za přerušené a program
musí být restartován. Přečtená zpráva nemusí být nutně Healthcheck, může to být
libovolná zpráva. Healthcheck slouží především k ověření funkčnosti v případě, 
kdy do Kafky nepřichází zprávy odjinud (např. pokud by spadla předchozí komponenta
pipeline, nemusí se nutně restartovat celá pipeline).

```json
{
    "healthcheck": "healthcheck"
}
```

```java
public void writeHealthcheck() {
  producer.send(healthcheckRecord);
}
```

```java
public void write(String data) {
  producer.send(createRecord(data));
}
```

Dále by se dalo třídě 
(`ReceiverDatabase`)[https://github.com/fpeterek/FlightLidar48/blob/master/validator/src/main/java/org/fpeterek/flightlidar48/validator/ReceiverDatabase.java]
vytknout, že slouží pouze jako thread-safe wrapper nad HashMapou, a tak může být
generická, nebo třídě
(`MailClient`)[https://github.com/fpeterek/FlightLidar48/blob/master/validator/src/main/java/org/fpeterek/flightlidar48/validator/MailClient.java],
že posílá předdefinovaný email. Obojí zbytečně znemožňuje znovupoužití kódu.

## Soudržnost

Jako příklad soudržnosti v projektu nám poslouží
(validátor dat)[https://github.com/fpeterek/FlightLidar48/tree/master/validator]

Příkladem vysoké soudržnosti může být package 
`org.fpeterek.flightlidar48.validator.validators.messagevalidators`.
Tento package obsahuje pouze třídy související s validací jedné konkrétní zprávy.
Při validaci zprávy je použit návrhový vzor Chain of Command. Jednotlivé validátory
postupně dostávají vstup v podobě zprávy, zprávu zvalidují, a pokud je validní, předají
zprávu dalšímu validátoru (pokud není, zkratují a vrátí `false`). Můžeme zde mluvit
o funkcionální soudržnosti, neboť všechny třídy v packagi slouží právě jedné funkci -
validaci zprávy. Předávají si navzájem zprávy, pracují všechy s tím stejným vstupem,
volají se postupně za sebou. Jedna třída poté slouží k vytvoření Chain of Command
a následnému zavolání vytvořeného řetězce.

Naopak package `org.fpeterek.flightlidar48.validator` se již tak velkou soudržností
nevyznačuje. Tento package obsahuje např. třídu `ReceiverDatabase`, což je thread-safe
abstrakce nad `java.util.HashMap`, třídu `Main`, třídu `Config`, apod. Jistě je zřejmé,
že třída `Config` a třída `Main` slouží k jinému účelu. Zde nepochybně došlo k obětování
akademické správnosti ve prospěch pragmatičnosti. Třídy, které spolu souvisí temporálně
nebo procedurálně byly sdruženy do jednoho package, aby se nevytvářelo více packagů,
z nichž každý by obsahoval pouze jednu třídu. Přesto však o optimální soudržnosti mluvit
nelze.

Dále soudržnost utrpěla například v knihovně `database`, jak již bylo dříve popsáno.

## Provázání

Jak již bylo předestřeno výše, provázání na úrovni komponent je nízké. Komponenty v rámci
pipeline zpracovávající vstup z příjímačů (v případě tohoto projektu simulátoru
přijímačů) si pouze skrze message brokera předávají JSON zprávy, jinak jsou na sobě 
nezávislé.

Projekt obsahuje také REST API a dva různé klienty - webového klienta využívajícího API
Mapy.cz a desktop klienta. REST API také snižuje provázanost komponent. Klienti jsou
nezávislí na sobě, ale také na backendu API. Pouze posílají HTTP requesty, ve kterých si
vyžádají data, v odpovědi dostávají JSON, který poté zpracovávají vlastním způsobem
(například vykreslením do map Mapy.cz nebo Swing okna).

Jako příklad možná nadbytečného provázání však lze uvést interní fungování 
(REST API)[https://github.com/fpeterek/FlightLidar48/tree/master/flightlidar48].
Třída `DataProxy` slouží jako wrapper nad Gatewayi a databází. Přesto vrací recordy
z `database` knihovny. To může způsobit problémy, změní-li se některý z recordů
z knihovny `database`. Jakákoliv změna by způsobila, že bude třeba upravit všechna místa
v kódu, kde se s danými recordy pracuje. Pokud bychom místo toho record převedli
na třídu (nebo klidně zase record) vlastní přímo kódu REST API, mohlo by v některých
případech postačit upravit pouze tento převod mezi strukturami. Zbytek kódu by tak mohl
zůstat nedotčen.

## Použití rozhraní

Bavíme-li se o rozhraních na úrovni jednotlivých komponent, lze znovu uvést jako příklad
komunikaci skrze message brokera (v tomto případě Apache Kafku) nebo REST API. Komunikace
za využití Kafky je definována formátem zprávy. (Simulované) přijímače zapíšou data,
ta jsou zvalidována a následně zapsána. Každá komponenta má pevně definovaný formát
zprávy, kterou na svém vstupu přijímá. Podobně má REST API přesně definované endpointy.
Pokud bychom chtěli jednu z komponent přepsat, musíme toto rozhraní zachovat, abychom
mohli novou komponentu zapojit do systému.

Jako příklad použití rozhraní v rámci jedné komponenty (tedy jednotka=třída,
ne komponenta) poté lze zmínit například interface `MessageHandler` využívaný třídou
`MessageStream`.

```java
public interface MessageHandler {
  void handleMessage(String key, String value);
}
```

Interface `MessageHandler` definuje rozhraní objektu, jež poslouží ke zpracování
zpráv streamovaných z message brokera. Jak můžeme vidět, `MessageHandler` zpracovává
každou zprávu zvlášť. Streamovací logice žádný feedback nevrací. Streamovací logika
se stará pouze o doručení zprávy, její zpracování ji už nijak nezajímá.

```java
KStream<String, String> stream = builder.stream(inputTopic);
stream.foreach(handler::handleMessage);
```

V některých případech jsou poté rozhraní definována abstraktní třídou. Jako příklad stojí
za to zmínit validátory zpráv, jež všechny dědí z abstraktní třídy `MessageValidator`.

```java
public abstract class MessageValidator {

  MessageValidator next = null;

  protected abstract boolean validateMessage(KafkaMessage msg);

  public void setNext(MessageValidator validator) {
    next = validator;
  }

  public boolean validate(KafkaMessage msg) {
    return validateMessage(msg) && (next == null || next.validate(msg));
  }

}
```

Třída `MessageValidator` zároveň definuje rozhraní (metoda `validate(KafkaMessage)`),
a zároveň implementuje návrhový vzor Chain of Command. Zde bychom opět mohli mluvit
o nečistém řešení. Rozhraní validátoru a logiku vzoru Chain of Command by bylo vhodné
spíše rozdělit do dvou různých rozhraní nebo tříd. Jedná se tak o návrh rozhraní,
současně je však tento návrh špatný a nevhodný.

Jako příklad, kdy definice rozhraní úplně chybí, můžeme uvést třídu
(`MessageStream`)[https://github.com/fpeterek/FlightLidar48/blob/master/database/database/src/main/java/org/fpeterek/flightlidar48/kafka/MessageStream.java].
Tato třída abstrahuje knihovnu KafkaStreams a umožňuje přístup ke streamu zpráv z Kafky.
Teoreticky by se ale mohlo stát, že budeme chtít umožnit využití více message brokerů
nebo knihoven (například pokud by se prokázalo, že knihovna KafkaStreams je lepší pro
vývoj, ale v ostrém provozu chceme z nějakých důvodů používat třeba Flink). Pro tento
příklad by tak určitě bylo vhodné, aby bylo rozhraní třídy `MessageStream` definováno
interfacem.

## Zapouzdření

Zapouzdření je využíváno, kde to jen jde. Jako příklad lze uvést například
třídu (`DataProxy`)[https://github.com/fpeterek/FlightLidar48/blob/master/writer/src/main/java/org/fpeterek/flightlidar48/writer/DataProxy.java],
jež se v určitých podobách objevuje ve více komponentách a abstrahuje práci s Gateway
objekty. Třída `Dataproxy` sice v parametrech konstruktoru přijímá Gateway objekty,
ty si však následně drží pouze interně a ven je nijak nevystavuje. Veškerá práce
s Gateway objekty je tak schována za metodami (např. metoda `fetchFlights()`).

Příkladem horšího zapouzdření budiž record 
(`Airline`)[https://github.com/fpeterek/FlightLidar48/blob/master/database/database/src/main/java/org/fpeterek/flightlidar48/database/records/Airline.java]

```java
public record Airline(
  String designator,
  String name,
  List<Aircraft> fleet
) {
    public void addToFleet(Aircraft aircraft) { ... }
}
```

Java recordy mají všechny atributy implicitně konstantní (final). Problém s final
atributy je ovšem ten, že jako final je označen pouze pointer na daný objekt, ne objekt
samotný. Například v případě Listu `fleet` to znamená, že by kdokoliv mohl přidat letadlo
do flotily metodou `List::add`, aniž by letadlo bylo validováno (viz metoda
`addToFleet()`). Z hlediska zapouzdření zde existují dvě možnosti. První možnost je
nevracet pointer na list držený recordem, ale celý list zkopírovat, což je ovšem
nežádoucí z hlediska výkonu. Druhým možným řešením je immutable interface nad Listem.
Modernější jazyky běžící na JVM, jakými jsou třeba Kotlin nebo Scala, rozdělují rozhraní
kolekcí na mutable a immutable rozhraní. Mutable kolekci poté lze předat pomocí svého
immutable rozhraní, což nám umožní zabránit kopírování kolekce i nechtěné mutaci. Naopak
to ovšem nejde -> pokud bychom immutable kolekci chtěli převést na mutable kolekci,
museli bychom ji zkopírovat. Bohužel, projekt byl psaný v Javě a ne v Kotlinu, což
neumožnilo jednoduše využít druhé, efektivní, řešení (ovšem pravděpodobně bude existovat
open source řešení, které by pro danou aplikaci bylo využitelné, což však v době psaní
projektu nebyla priorita).

## Zákon Demeter

Dodržování zákona Demeter je zde, podobně jako dříve, určitým způsobem vneseno rozdělením
systému do více komponent. Jednotlivé komponenty neví skoro nic o zbytku systému.

Příkladem dodržení zákona Demeter budiž práce se streamem zpráv z message brokera.
Knihovna KafkaStreams je abstrahována třídou `MessageStream`. Třída `MessageStream`
využívá dependency injection, vnitřně pracuje pouze s knihovnou KafkaStreams, jež
inicializuje a streamuje díky ní zprávy. Zprávy poté pouze předá `MessageHandler`u
předanému za využití dependendency injection. O zpracování nebo obsahu zprávy však
nic vědět nemusí. Podobně třídy implementující rozhraní `MessageHandler` nemusí vůbec
nic vědět o Kafce, o inicializaci streamu, atp. `MessageStream` a `MessageHandler`
objekty pak můžou být vytvářeny jiným objektem, který je zodpovědný za inicializaci
programu.

Zákon Demeter úzce souvisí s provázáním systému, proto jako jeho nedodržení lze považovat
např. violaci nízkého provázání způsobené třídami `DataProxy`, která vrací recordy
z knihovny `database`. Tyto recordy souvisí s databází, proto je možné říct, že třídy,
které pracují s výsledky volání metod tříd `DataProxy`, pracují naprosto zbytečně
s detaily knihovny `database`.

```
    .--.  
   |o_o |  
   |:_/ |  
  //   \ \  
 (|     | )  
/'\_   _/`\  
\___)=(___/  
```

