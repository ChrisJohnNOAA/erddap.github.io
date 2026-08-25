# Blocking ERDDAP™ a kérelem tartalmán alapuló kérelmek, nem az IP alapján

Ez a tartalom egy [Roy Mendelssohn üzenete a ERDDAP™ felhasználók csoport](https://groups.google.com/g/erddap/c/XcvPkoGtchg) ...

## A probléma

Ha olyanok vagytok, mint mi, sok botot láttok, hogy adatkéréseket készítsenek az Ön számára ERDDAP™ , és a kérések gyakran úgy tűnnek, mintha rosszul kódolt szövegek teszik, függetlenül attól, hogy az LLM-ek (Szerintem) vagy az emberek. Egy dolog az LLM-ekről, pontosan azt fogják tenni, amit arra ösztönöznek, hogy tegyenek, így ha nem mondod el nekik, hogy a forgatókönyv-ellenőrzési kódok legyenek, és ne mondd el nekik, mit kell tenniük a különböző esetekben, akkor általában a kód nem fog megtenni ezeket a dolgokat. És ha azt mondod, hogy folytassa a próbálkozást, akkor ez meg fog. Amit legalább látunk.

De azok, akik számot tesznek a miénkről ERDDAP™ Olyanok, mint ez:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Ezek az üzenetek ERDDAP™ soha nem-so földre, mivel a legjobb, amit meghatározhatok, hogy ez az URL nem csak az egész adatkészletet kéri, ami nem tudom, hány terabytát, de azt akarja, hogy egy bankett fájlba konvertáljon. Rosszabb, hogy a kérések az IP-k mozgatásából érkeznek, így az IP-k blokkolása rosszabb, mint a hack-a-mole, soha nem fogja őket blokkolni.

Tehát a kérdés az, hogyan blokkolhatja a kérelem tartalmát? Mielőtt tovább menne, ha tapasztalsz összeomlik, akkor különböző okokból lehet, amit látunk - sok időt töltöttem a naplókon, és értesülök, amikor veszélyes magas memóriahasználat van, és megjegyeztem, amikor egy összeomlás követte ezt az értesítést, és észrevette, hogy mi volt a közös nevező ezekben a balesetekben, ami lehet, hogy nem a szerver esetében. Tehát kutassa ezt, mielőtt bármilyen akciót, de ez adhat némi ötletet, hogyan blokkolja ezeket a kéréseket.

Csak így vagyok világos, gondolhatsz egy ERDDAP™ kérés:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

Abban az esetben, ha a balesetek közös nevezője griddap vagy asztali kérelmek voltak bizonyos fájltípusokkal, de nem korlátozódtak. Tehát a kérdés az, hogyan lehet korlátozni a kérelmeket korlátozás nélkül? Kivéve, hogy ez nem olyan egyszerű, mert vannak olyan fájltípusok, amelyek nem igényelnek korlátozást, és jól viselkednek, ezért nem akarjuk ezeket blokkolni. Miután beszéltem Chris-vel, és kétségtelenül elhagytam valamit, a kontraint nélkül jól viselkedő fájltípusok:

.croissant
.iso19115_2
.iso19139_2007
.iso19115_3_2016
 .nc CFHeader
 .nc CFMAHeader
.das
.dds
.html
.gráf
.subset
 .nc Fejlesztő
.help
Fgdc
.iso19115
 .nc JsonHeader (ez lesz a közelgő új kiadásban) ...

## Megoldás

A "megtaláltam" megoldás az Apache2-ben működik, nem tudom, hogy nginx, de elképzelem, hogy van valami hasonló. Eleinte megpróbáltam a mod_biztonságot, de ez túl bonyolult volt, és nem működött nagyon jól. A megoldás a mod_rewrite használata. Most már bármi vagyok, csak egy szakértő vagyok ezen, így miközben meghatároztam, mit próbálok elérni, a megoldás, valamint a magyarázat, Claude miatt van. ai. A ChatGPT alapvetően ugyanazt a választ nyújtja.

1. lépés Bizonyosodjon meg arról, hogy a mod_rewrite telepített és engedélyezett. Mivel hogyan kell ezt megtenni az operációs rendszerrel, ez olyasmi, amit a kedvenc chatbotja kérhet.

2. lépés A megfelelő fájlban, amely konfigurálja az ssl-t az apache2 számára (amely ismét változik az OS) Például lehet valami, mint /etc/apache2/sites-enabled/ssl.conf, hozzáadja a következőt a megfelelő VirtualHost meghatározás szerint (jegyezze meg, hogy ha ezt másolja, csak 4 vonal van, a harmadik vonalat be lehet csomagolni, becsomagolni) 

RewriteEngine Tovább
RewriteCond %&#123;QUERY_STRING&#125; ^ $$
RewriteCond %&#123;REQUEST_URI&#125; ^/erddap/ (griddap |  tabledap ) /[^/?]+\\. (?&#33; (?:croissant | Io19115_2 | Iso19139_2007 | Io19115_3_2016 | ncCFHeader | ncCFMAHeader | Dávid | dds | html | gráf | aljzat | ncHeader | Segítségnyújtás | fgdc | Iso19115 | ncoJsonHeader) $ € $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $) [A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]

3. lépés Ellenőrizze, hogy a konfiguráció érvényes: sudo apache2ctl konfiguráció

Lépés 4. Indítsa újra az apache2-t: sudo rendszerctl újraindítás apache2

5. lépés Ellenőrizze a naplókat, hogy semmi sem akadályozható, ami nem lehet, és ez a megfelelő kérés korlátozott visszatérés nélkül 429 anélkül, hogy valaha elérné a tomcatot

## Magyarázat

Miért működik ez és mit csinál ez - itt van a magyarázat Claude.ai:

1. sor - RewriteEngine Tovább
Fordítja a mod_rewrite feldolgozását erre a körre. Enélkül a RewriteCond/RewriteRule irányelveket egyszerűen figyelmen kívül hagyják.

2. sor - Írás Megjegyzés %&#123;QUERY_STRING&#125; ^$
Az a feltétel, amely igaz, mielőtt az alábbi szabály érvényes. %&#123;QUERY_STRING&#125; minden a ? a kérelem URL. ↑ $ egy regex jelentés "a sztring kezdete azonnal követi a sztring végét" - azaz üres sztring. Tehát ez a feltétel csak akkor igaz, ha egyáltalán nincs lekérdezési sztring - nincs albeállítás / korlátozott kifejezés a kérésre.

3. sor - Írás Megjegyzés %&#123;REQUEST_URI&#125; ^/erddap/ (griddap |  tabledap ) /[^/?]+\\. (?&#33; (?:...) $ € $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $) [A-Za-z0-9_]+$
Második feltétel, ellenőrzött %&#123;REQUEST_URI&#125; - a szó szerinti kérési út, ahogy az ügyfél küldte, mindig a teljes út függetlenül attól, hogy hol él ez a szabály. (szándékosan választott, hogy a RewriteRule minta magát a megfelelő, mert a minta-olvadás belül egy <Location> A blokk kétértelműen viselkedhet - lásd a jegyzetet alább) ... A regex letörése:

↑/erddap/ - kezdődjön /erddap/
 (griddap |  tabledap ) / - követte a kettő egyikét ERDDAP™ hozzáférési módszerek
[^/?]+ — a datasetID Egy vagy több karakter, amelyek nem / vagy?
- egy szó szerinti pont
 (?&#33; (?:croissant | Io19115_2 | ... | ncoJsonHeader) $ € $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $ $) - negatív nézőpont: "Amíg az következik, nem az egyik ilyen pontos fájl Típusú nevek az út végére a sztring.” Ezek a fájlTypes ERDDAP™ jogosan szolgálhat korlátozás nélkül (metadata, szerkezet, forma oldalak stb.) A nézőfej az, ami kizárja őket, hogy blokkolják őket.
[A-Za-z0-9_]+$ - a tényleges fájl Type kiterjesztés (levelek, számjegyek, alulértékek) a húr végére kellett futni.
Tehát ez az állapot csak akkor igaz, ha az út egy griddap/ tabledap kérjen néhány fájlt Típus, amely nem a biztonságos korlátozás nélküli listán található.

Line 4 - RewriteRule ^ - [R=429,L]
A szabály maga. Mivel a fenti feltételeknek már igaznak kell lenniük az Apache számára, hogy értékelje ezt a vonalat, a minta itt nem kell mást ellenőrizni - ↑ csak egyezik a "start of the string" -val, ami mindig igaz. - azt jelenti, hogy "nem írja újra az URL-t bármi másra" (Nem átirányítunk sehol, csak rövid keringési kérelem) ... A zászlók:

R=429 – válasz egy HTTP átirányítási osztályú fellépésre, amely a 429-es státuszkódot hordozza ("Túl sok kérés") ahelyett, hogy szolgálná a kérést.
L - "Last": hagyja abba a további újraírási szabályok feldolgozását, ha ez egy tűz.
Tegyük fel: ha a lekérdezés üres, és a kérés egy griddap/ tabledap fájl Típus, amely nem a biztonságos nem konstruált listán van, azonnal visszatér 429 - anélkül, hogy valaha kapcsolatba lépne a ERDDAP /Tomcat backend.

Miért % &#123;REQUEST_URI&#125; ahelyett, hogy hagyná, hogy a RewriteRule minta megfeleljen az utat közvetlenül (érdemes, beleértve a kollégák jegyzetét is, mivel ez a nem nagy rész) : belül egy <Location> blokk, micsoda RewriteRule minta ténylegesen megegyezik az ellen, következetlenül viselkedhet az Apache verziótól és kontextustól függően. Következésképpen húzza a teljes utat a RewriteCond %&#123;REQUEST_URI&#125; oldalakon, amelyek teljesen kétértelműek - ez mindig a szó szerinti, teljes kérési út, így a regex pontosan olyan írásban viselkedik, függetlenül attól, hogy hol van a szabály.


Már több napja használom ezt, és úgy tűnik, nagyon jól működik, blokkolja azt, amit megpróbálok blokkolni, és nem blokkolni, amit nem akarok blokkolni. A mi ERDDAP™ sokkal stabilabbá vált.
