# Letecký tracker

---
Filip Peterek, PET0342

Online verze na Githubu: https://github.com/fpeterek/SNK/blob/master/p2/flight-tracker.md

Pro renderování MD lze využít i terminálový renderer `mdless` (v AUR balíček `ruby-mdless`).

---

## Publish-Subscribe

Na principu architektonického stylu Publish-Subscribe funguje message broker Apacha Kafka.
Publisher (Kafka využívá název 'Producer') posílá zprávy, které se ukládají do konkrétního
topicu. Subscriber (v Kafce 'Consumer') poté odebírá konkrétní topic. Kafka zajistí doručení
každé zprávy minimálně jednou každému Consumeru, který odebírá daný topic. Jedná se tedy
o topic-based systém.

Jak bylo zmíněno dřívě, v projektu leteckého trackeru bude Kafka hojně využívána - v Kafce
budou ukládána např. data zaznamenaná ADS-B přijímači. Proto se počítá, že bude styl
Publish-Subscribe důležitou součástí návrhu systému.

Částečně je tento architektonický styl vynucen Kafkou - potřebujeme message brokera, 
s velmi vysokým throughputem. Zároveň je Kafka open source projekt (viz Apache). Apache Kafka
tak splňuje naše potřeby.

Současně je ovšem tento styl pro danou aplikaci vhodný. Přijímače na vstupu z antény získají
aktuální letová data, která poté zapíšou do Kafky. Data následně můžou být zpracována pomocí
naprogramované pipeline ([viz sekce Pipes and Filters](#pipes-and-filters)). V rámci pipeline
může více komponent vyžadovat přístup k těm samým datům (např. po očištění vstupních dat
můžou dvě nezávislé komponenty odlévat data do PostgreSQL a HBase). Model Publish-Subscribe
nám také umožní jednoduché přidávání dalších komponent do pipeline. Např. budeme-li chtít
implementovat novou komponentu, která bude využívat již existující data, nemusíme danou
komponentu explicitně volat po síti, jako by tomu bylo například pokud bychom (pro tuto konkrétní
aplikaci) využili styl Client-Server. Nové komponentě pouze stačí zaregistrovat se jako Consumer
konkrétního topicu. Model Publish-Subscribe také dobře funguje v kombinaci s modelem Pipes
and Filters.

## Pipes and Filters

Architektonický styl Pipes and Filters lze použít například k paralelizaci déle trvajících
částí pipeline, nebo také ke znovupoužití kódu. Obojí lze dobře využít také v případě letového 
trackeru. Složitější a déle trvající tasky můžou běžet na více podech v Kubernetu, při čtení
z Kafky poté stačí používat jedno Consumer ID napříč všemi pody, čímž dosáhneme paralelizace
části pipeline. 

Jako příklad znovupoužití kódu lze uvést například zpracování dat z více zdrojů.  Teoreticky
bychom mohli chtít využít více zdrojů letových dat. Kromě vlastní sítě ADS-B přijímačů bychom
tak mohli chtít využít také externí zdroje (např. satelitní přijímače). V takovém případě
by nezávisle na sobě mohlo běžet více komponent pipeline - každá by zpracovávala data z jiného
zdroje. Nakonec by ovšem všechny komponenty zapsaly data ve stejném formátu do stejného Kafka
topicu. Zbytek pipeline už poté může být proveden na datech nezávisle na jejich zdroji.

## Client-Server

Styl Client-Server je v dnešním světě skoro nevyhnutelný. Bezpochyby budeme mít server
implementující REST API schopný komunikace (nejen) s browserem. Současně bude komunikace
s PostgreSQL nebo Redisem fungovat na principu Client-Server. Při zpracování dat ovšem nemusí
být model (explicitní) Client-Server vždy vhodný. Např. pro implementaci pipeline byla dříve
nastíněna implementace pomocí modelu Publisher-Subscriber za využití Kafky. Kafka bezpochyby
interně funguje také jako server, jednotliví Produceři a Consumeři poté jako klienti.
Důležité zde ovšem je právě skrytí tohoto modelu za model Publisher-Subscriber. Explicitní
Client-Server architektura by si vyžádala explicitní obvolávání všech komponent, vynutila si
implementaci automatické registrace Subscriberů, nebo manuální konfigurování při každém
přidání nové komponenty sdílející data s jinou komponentou. Tak bychom ve výsledku došli
buď k nečitelnému kódu, nebo k vlastní implementaci Publisher-Subscriber modelu. Podobně
můžeme zmínit také zpracování Big Data. Při zpracování Big Data můžeme chtít pomocí jednoho
programu zapsat data na HDFS, pomocí jiného je poté přečíst a zpracovat. Jelikož se může
jednat o terabyty dat, je vhodné je uložit (alespoň dočasně) na disk, není vhodné je posílat
na server, který by s nimi mohl chtít ihned pracovat. Jelikož je však HDFS distribuovaný 
filesystem, interně také funguje na principu Client-Server. Navenek se však chová jako obyčejný
filesystem. 

Můžeme tak tvrdit, že styl Client-Server je bezpochyby nevyhnutelný, obzvlášť s ohledem na moderní
trendy směřující k tvorbě více menších komponent v neprospěch monolitů, často jej však chceme
abstrahovat, nechceme s ním pracovat v jeho explicitní podobě.

## Repository

Vzor Repository slouží k abstrahování přístupu k databázím. V projektu leteckého trackeru bude
nejen nutné přistupovat k databázi z více komponent, současně bude také nutné přistupovat k více
databázím zároveň (např. Redis a Postgres). Proto je určitě důležité přístup k databázím abstrahovat.
Nejen, že tímto ulehčíme případný budoucí přechod na novější verze PostgreSQL (případně i úplně jiný
databázový systém), zjednoduššíme případné úpravy DB schématu, ale také tím získáme konzistentní
rozhraní pro perzistentní i in-memory cachovací databázi. Ve výsledku tak budeme moct využít
ten samý kód pro přístup jak k PostgreSQL, tak k Redisu. Vzor Repository se tedy nabízí jako
elegantní řešení přístupu k databázím.

## Peer to Peer

Architektonický styl Peer to Peer se pro uvažovanou aplikaci leteckého trackeru naopak příliš
nehodí. Backend bude složený převážně z pipeline zpracovávajících data a serveru zajišťujícího
komunikaci s frontendem. Tento server však bude komunikovat pouze s databázemi. Jednotlivé
komponenty pipeline si poté budou mezi sebou předávat data za využití message brokera,
a to z důvodů nastíněných výše. Jako poslední příležitost implementace P2P komunikace se můžou
nabízet už jen ADS-B přijímače, jež teoreticky budou všechny na stejné úrovni a budou si rovny.
Mezi přijímači ovšem nechceme komunikovat. Na přijímačích není důvod implementovat složitou
logiku - naopak by to bylo na škodu, navíc přijatá data stejně potřebujeme validovat. Proto
pro model Peer to Peer nevidím využití.

```
    .--.  
   |o_o |  
   |:_/ |  
  //   \ \  
 (|     | )  
/'\_   _/`\  
\___)=(___/  
```

