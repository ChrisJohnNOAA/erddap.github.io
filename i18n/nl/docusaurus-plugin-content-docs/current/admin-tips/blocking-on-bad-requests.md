# Blokkeren ERDDAP™ verzoeken op basis van de inhoud van het verzoek, niet gebaseerd op het OT

Deze inhoud is gebaseerd op een [bericht van Roy Mendelssohn aan de ERDDAP™ gebruikersgroep](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Het probleem

Als u net als wij bent, ziet u veel bots het maken van data verzoeken om uw ERDDAP™ , en de verzoeken lijken vaak gedaan door slecht gecodeerde scripts, of door LLM's (Mijn gok) of door mensen. Een ding over LLMs is dat ze precies zullen doen wat je hen vraagt om te doen, dus als je niet vertellen hen om het script controle retour codes, en vertel ze niet wat te doen in verschillende gevallen, dan meestal de code zal niet doen een van die dingen. En als je zegt dat het moet blijven proberen, zal het dat ook doen. Dat zien we tenminste.

Maar degenen die een aantal doen op onze ERDDAP™ zijn als volgt:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Deze sturen ERDDAP™ in nooit-nooit land, vanwege, naar het beste dat ik kan bepalen, dat deze URL is niet alleen het vragen van de hele dataset, dat is ik weet niet hoeveel terabytes, maar wil het omgezet naar een parket bestand. Erger nog, de verzoeken komen van bewegende IP's, dus het blokkeren van IP's is erger dan whack-a-mole, je zult ze nooit allemaal blokkeren.

Dus de vraag is hoe je kunt blokkeren op basis van de inhoud van het verzoek? Alvorens verder te gaan, als je ervaring crasht kan het zijn van verschillende oorzaken dan wat we zien - Ik bracht veel tijd door met het doorzoeken van logs en ook word ik geïnformeerd wanneer er gevaarlijk hoog geheugengebruik, en opgemerkt wanneer een crash volgde die kennisgeving, en ook merkte wat de gemeenschappelijke noemer in deze crashes, wat misschien niet het geval is voor uw server. Onderzoek dit voordat je actie onderneemt, maar dit kan je een idee geven hoe je deze verzoeken kunt blokkeren.

Even voor de duidelijkheid, je kunt denken aan een ERDDAP™ verzoek als:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

In ons geval waren de gemene deler van de crashes griddap of tabletop verzoeken met bepaalde bestandstypen maar geen beperking. De vraag is dus hoe verzoeken zonder beperking te blokkeren? Behalve het is niet zo eenvoudig, want er zijn een aantal bestandstypen die geen beperking nodig hebben en goed gedragen zullen zijn, dus we willen die niet blokkeren. Na het gesprek met Chris, en ik heb ongetwijfeld iets weggelaten, zijn de bestandstypen die goed gedragen zijn zonder beperking:

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
-  .nc Kop
- hulp
- .fgdc
- Iso19115
-  .nc oJsonHeader (deze komt in de nieuwe release) .

## Oplossing

De oplossing "Ik vond" werkt in Apache2, Ik weet niet nginx maar ik denk dat er iets vergelijkbaars is. Eerst probeerde ik mod_security, maar dat werd te ingewikkeld en werkte niet erg goed. De oplossing is om mod_rewrite te gebruiken. Nu ben ik allesbehalve een expert in dit, dus terwijl ik bepaalde wat ik probeerde te bereiken, is de oplossing, evenals de uitleg, te wijten aan Claude. ai. ChatGPT zal u in principe hetzelfde antwoord geven.

Stap 1. Zorg ervoor dat mod_rewrite is geïnstalleerd en ingeschakeld. Aangezien hoe dit te doen varieert met OS, dit is iets wat u kunt vragen uw favoriete chatbot.

Stap 2. In het juiste bestand dat ssl voor apache2 configureert (die weer varieert naar besturingssysteem) , bijvoorbeeld kan het iets zijn als /etc/apache2/sites-enabled/ssl.conf, voeg het volgende toe onder de toepasselijke VirtualHost definitie (let op als je dit kopieert er zijn slechts 4 regels, de derde regel kan worden verpakt, uitpakken) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Stap 3. Controleer of de configuratie geldig is: `sudo apache2ctl configtest` 

Stap 4. Herstart apache2: `sudo systemctl herstart apache2` 

Stap 5. Controleer uw logs dat niets wordt geblokkeerd dat niet zou moeten zijn, en dat passende verzoeken zonder een beperking retour een 429 zonder ooit uw kater raken

## Toelichting

Waarom doet dit werk en wat dit doet - hier is de uitleg van Claude.ai:

Regel 1 `Engine herschrijven Aan` 
Schakel mod_rewrite verwerking in voor deze scope. Zonder het, worden de RewriteCond / RewriteRule richtlijnen hieronder gewoon genegeerd.

Regel 2 `RewriteCond %&#123;RELATY_STRING&#125; ^$` 
Een voorwaarde die moet gelden voordat de onderstaande regel van toepassing is. `%&#123;ONTREDY_STRING&#125;` Is alles na de? in de aanvraag-URL. ^$ is een regex die betekent "start van de tekenreeks onmiddellijk gevolgd door einde van de tekenreeks," d.w.z. een lege tekenreeks. Deze voorwaarde is dus alleen waar als er helemaal geen query string is, geen subsetting/contraint expressie op het verzoek.

Regel 3 `Herschrijfcond %&#123;REQUEST_URI&#125; ^/erdap/ (griddap |  tabledap ) /[^/?]+\\. (?&#33; (?: ...) $) [A-Za-z0-9_]+$` 
Een tweede voorwaarde, gecontroleerd tegen `%&#123;REQUEST_URI&#125;` Het letterlijke verzoek pad zoals de client het stuurde, altijd het volledige pad ongeacht waar in de configuratie deze regel leeft (bewust gekozen over het laten van de RewriteRule patroon zelf doen de matching, omdat patroon-matching binnen een ` <Location> ` blok kan zich dubbelzinnig gedragen ) . De regex afbreken:

 `^/erdap/` Moet beginnen met /erdap/
 (griddap |  tabledap ) En daarna één van de twee. ERDDAP™ toegangsmethoden

 `[^/?]+` De datasetID : een of meer tekens die niet / of ?

 `\\.` Een letterlijke stip.

 ` (?&#33; (?:croissant | iso19115_2 | ... | ncoJsonHeader) $) ` Een negatieve blik vooruit: "Zolang wat volgt is niet een van deze exacte bestand Typ namen tot aan het einde van de string." Dit zijn de bestandstypen ERDDAP™ zonder beperkingen te kunnen dienen (metadata, structuur, formulieren enz.) De blik vooruit is wat hen uitsluit te worden geblokkeerd.

 `[A-Za-z0-9_]+$` Het werkelijke bestand Type uitbreiding (letters, cijfers, underscore) , vereist om te draaien naar het einde van de string.
Dus deze voorwaarde is alleen waar als het pad een griddap is/ tabledap verzoek om een bestand Type dat niet op de kluis zonder beperking lijst staat.

Lijn 4 `Regel herschrijven ^ - [R=429,L]` 
De regel zelf. Omdat beide bovenstaande voorwaarden al waar moeten zijn voor Apache om deze regel zelfs te evalueren, hoeft het patroon hier niet om iets anders te controleren. ^ komt alleen overeen met "start van de string," wat altijd waar is. - betekent "niet herschrijven van de URL naar iets anders" (We gaan nergens heen, alleen het verzoek kortsluiten.) . De vlaggen:

 `R=429` Antwoord met een HTTP redirect-class actie met statuscode 429 ("Teveel verzoeken") In plaats van het verzoek in te dienen.
L "Laatst": stop met het verwerken van verdere herschrijfregels zodra deze brandt.
Samenvoegen: als de query string leeg is, EN het verzoek is voor een griddap/ tabledap bestand Type dat niet op de veilige-ongeremde lijst, onmiddellijk terug te keren en zonder ooit contact met de ERDDAP Tomcat backend.

Waarom? `%&#123;REQUEST_URI&#125;` in plaats van het patroon van de regels te laten overeenkomen met het pad (de moeite waard als notitie voor collega's, omdat het niet-duidelijk deel) : binnen een ` <Location> ` blok, wat een kale RewriteRegel patroon daadwerkelijk wordt afgestemd tegen kan zich inconsistent gedragen, afhankelijk van Apache versie en context. Expliciet het volledige pad trekken via RewriteCond `%&#123;REQUEST_URI&#125;` zijstappen die ambiguïteit volledig 


Ik gebruik dit al enkele dagen en het lijkt heel goed te werken, het blokkeert wat ik probeer te blokkeren en niet te blokkeren wat ik niet wil blokkeren. En onze ERDDAP™ is veel stabieler geworden.
