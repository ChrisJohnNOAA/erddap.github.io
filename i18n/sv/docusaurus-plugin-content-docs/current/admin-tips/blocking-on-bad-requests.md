# Blockering ERDDAP™ Förfrågningar baserade på innehållet i begäran, inte baserat på IP

Detta innehåll är baserat på en [från Roy Mendelssohn till ERDDAP™ användare grupp](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Problemet

Om du är som oss ser du många bots som gör dataförfrågningar till din ERDDAP™ , och förfrågningarna verkar ofta som om de görs av dåligt kodad skript, oavsett om det är av LLMs (Min gissning) eller av människor. En sak om LLMs är att de kommer att göra exakt vad du uppmanar dem att göra, så om du inte berättar för dem att ha skriptet check return codes, och inte berätta vad man ska göra i olika fall, då brukar koden inte göra något av dessa saker. Och om du säger det för att fortsätta försöka, det kommer. Vilket är vad vi ser åtminstone.

Men de som gör ett nummer på våra ERDDAP™ är som denna:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Dessa skickar ERDDAP™ till aldrig någonsin land, på grund av det bästa jag kan avgöra, att denna URL inte bara begär hela datamängden, vilket är jag inte vet hur många terabyte, men vill att den konverteras till en parkettfil. Värre att förfrågningarna kommer in från att flytta IPs, så blockera IPs är värre än whack-a-mole, du kommer aldrig att få dem alla blockerade.

Så frågan är hur kan du blockera baserat på innehållet i begäran? Innan du går vidare, om du upplever kraschar kan det vara från olika orsaker då vad vi ser - jag spenderade mycket tid på att gå igenom loggar och även jag får meddelas när det finns farligt hög minne användning, och noteras när en krasch följt den meddelandet, och även märkte vad som var den gemensamma nämnaren i dessa kraschar, som kanske inte är fallet för din server. Så forska detta innan du vidtar några åtgärder, men detta kan ge dig en uppfattning om hur man blockerar dessa förfrågningar.

Så jag är tydlig, du kan tänka på en ERDDAP™ Förfrågan som:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

I vårt fall var den gemensamma nämnaren av krascherna griddap eller bordsförfrågningar med vissa filtyper men inga begränsningar. Så frågan är hur man blockerar förfrågningar utan ett hinder? Förutom att det inte är så enkelt, eftersom det finns ett antal filtyper som inte behöver ett hinder och kommer att bete sig väl, så vi vill inte blockera dem. Efter att ha pratat med Chris, och utan tvekan har jag lämnat ut något, är de filtyper som är väl uppförda utan ett hinder:

- .croissant
- .iso19115_2
- .iso19139_2007
- .iso19115_3_2016
-  .nc CFHeader
-  .nc CFMAHeader
- .das
- .dds
- .html
- .graph
- .subset
-  .nc Header
- .help
- Fgdc
- .iso19115
-  .nc OJson Header (Den här kommer att vara i den kommande nya releasen) .

## Lös lösning

Lösningen "Jag hittade" fungerar i Apache2, jag vet inte nginx men jag tror att det finns något liknande. Först försökte jag mod_security, men det blev för komplicerat och fungerade inte så bra. Lösningen är att använda mod_rewrite. Nu är jag allt annat än en expert på detta, så medan jag definierade vad jag försökte åstadkomma, är lösningen och förklaringen på grund av Claude. ai. ChatGPT kommer att ge dig samma svar.

Steg 1. Se till att mod_rewrite är installerat och aktiverat. Eftersom hur man gör detta varierar med OS, är detta något du kan fråga din favorit chatbot.

Steg 2. I lämplig fil som konfigurerar ssl för apache2 (som återigen varierar med OS) Till exempel kan det vara något som /etc/apache2/sites-enabled/ssl.conf, lägg till följande enligt lämplig VirtualHost definition (Observera om du kopierar detta finns det bara 4 rader, den tredje raden kan lindas, omforma den) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Steg 3. Kontrollera att konfigurationen är giltig: `sudo apache2ctl konfigtest` 

Steg Steg Restart apache2: `Sudo Systemctl omstart apache2` 

Steg 5. Kontrollera dina loggar att ingenting blockeras som inte ska vara, och att lämpliga förfrågningar utan ett hinder returnerar en 429 utan att någonsin träffa din tomcat.

## Förklaring

Varför fungerar detta och vad gör detta - här är förklaringen från Claude.ai:

Linje 1 - `RewriteEngine På` 
Vänds på mod_rewrite bearbetning för denna omfattning. Utan det ignoreras RewriteCond/RewriteRule-direktiven.

Linje 2 - `RewriteCond %&#123;QUERY_STRING&#125; ^$` 
Ett villkor som måste vara sant innan regeln nedan gäller. `% &#123;QUERY_STRING&#125;` Är allt efter ? på begäran URL. ^$ är en regex som betyder "start av sträng omedelbart följt av strängens ände" - dvs en tom sträng. Så detta villkor är sant endast när det inte finns någon fråga sträng alls - ingen subsetting / begränsning uttryck på begäran.

Linje 3 - `RewriteCond %&#123;REQUEST_URI&#125; ^/erddap/ (griddap |  tabledap ) [^/?]+. (?&#33; (?:...) $$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$) [A-Za-z0-9_]+$` 
Ett andra tillstånd, kontrollerat mot `% &#123;REQUEST_URI&#125;` Den bokstavliga sökvägen som kunden skickade den, alltid hela vägen oavsett var i konfigen denna regel lever. (avsiktligt valt över att låta RewriteRule-mönsteret själv göra matchningen, eftersom mönstermatchning inuti en ` <Location> ` block kan bete sig tvetydigt - se not nedan) . Att bryta ner regex:

 `^/erddap/` måste börja med /erddap/
 (griddap |  tabledap ) följt av en av de två ERDDAP™ Tillgångsmetoder

 `[^/?]+` - den datasetID En eller flera tecken som inte är / eller?

 `\\.` En bokstavlig punkt

 ` (?&#33; (Croissant | iso19115_2 | ...... | NcoJson Header) $$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$$) ` en negativ lookahead: "Så länge det som följer är inte en av dessa exakta filen Skriv namn hela vägen till slutet av strängen.” Dessa är filenTypes ERDDAP™ kan legitimt tjäna utan hinder (metadata, struktur, formsidor etc.) - lookahead är vad som utesluter dem från att blockeras.

 `[A-Za-z0-9_]+$` Den faktiska filen Typ förlängning (brev, siffror, underscore) krävs för att springa till slutet av strängen.
Så detta tillstånd är sant endast när vägen är en rutnät tabledap Förfrågan om någon fil Typ som inte finns på den säkra listan utan begränsningar.

Linje 4 - `RewriteRule ^ - [R=429,L]` 
Regeln själv. Eftersom båda villkoren ovan måste redan vara sant för Apache att ens utvärdera denna linje, behöver mönstret här inte kontrollera något annat - ^ bara matcher "start av strängen", vilket alltid är sant. - betyder "skriv inte om webbadressen till något annat" (Vi omdirigerar inte någonstans, bara kortslutning av begäran) . Flaggarna:

 `R=429` svara med en HTTP redirect-class-åtgärd som bär statuskod 429 (För många förfrågningar) istället för att betjäna begäran.
L - "Last": sluta bearbeta ytterligare omskrivningsregler när den här branden.
Sätt ihop: om frågan strängen är tom, och begäran är för en griddap/ tabledap fil Typ som inte finns på den säkra listan, returnera omedelbart 429 - utan att någonsin kontakta ERDDAP Tomcat backend.

Varför varför varför `% &#123;REQUEST_URI&#125;` istället för att låta RewriteRule-mönstret matcha vägen direkt (värt att inkludera som en anteckning för kollegor, eftersom det är den icke-uppenbara delen) Inuti en ` <Location> ` block, vad en bar RewriteRule mönster faktiskt blir matchad mot kan bete sig inkonsekvent beroende på Apache version och sammanhang. Uttryckligen drar hela vägen via RewriteCond `% &#123;REQUEST_URI&#125;` sidorna som tvetydighet helt - det är alltid den bokstavliga, fullständiga förfrågan väg, så regex beter sig exakt som skrivet oavsett var regeln är kapslad.


Jag har använt detta i flera dagar nu och det verkar fungera mycket bra, det blockerar vad jag försöker blockera och inte blockera vad jag inte vill blockera. och vår ERDDAP™ har blivit mycket stabilare.
