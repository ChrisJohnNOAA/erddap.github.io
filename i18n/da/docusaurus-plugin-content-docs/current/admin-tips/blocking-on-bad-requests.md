# Blokering ERDDAP™ anmodninger baseret på indholdet af anmodningen, ikke baseret på IP

Dette indhold er baseret på en [besked fra Roy Mendelssohn til te ERDDAP™ Brugere gruppe](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Problemet

Hvis du er som os, ser du en masse bots, der gør data anmodninger til dine ønsker ERDDAP™ , og forespørgslerne synes ofte, som de er udført af dårligt kodede scripts, om LLMs (mit gæt) eller af mennesker. En ting om LLMs er de vil gøre præcis, hvad du beder dem om at gøre, så hvis du ikke fortæller dem at have script check returkoder, og ikke fortælle dem, hvad du skal gøre i forskellige tilfælde, så vil koden normalt ikke gøre nogen af disse ting. Og hvis du fortæller det at holde på at prøve, vil det. Hvad ser vi mindst ud.

Men dem, der gør et nummer på vores ERDDAP™ er ligesom dette:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Disse sender ERDDAP™ i aldrig-never land, på grund af det bedste, at jeg kan afgøre, at denne URL ikke kun anmoder om hele datasættet, som jeg ikke ved, hvor mange terabytes, men ønsker det konverteret til en parket fil. Forbedring af anmodninger kommer i fra bevægelige IP'er, så blokering af IP'er er værre end whack-a-mole, vil du aldrig få dem alle blokeret.

Så spørgsmålet er, hvordan kan du blokere baseret på indholdet af anmodningen? Før du går videre, hvis du oplever nedbrud, kan det være fra forskellige årsager, så hvad vi ser - jeg brugte en masse tid på at gå gennem logs og også jeg får besked, når der er farlig høj hukommelse brug, og bemærkede, hvornår et nedbrud fulgte denne meddelelse, og også se, hvad der var den fælles denominator i disse nedbrud, som måske ikke er tilfældet for din server. Så undersøge dette, før du tager nogen handling, men dette kan give dig nogle idé om, hvordan du blokerer disse anmodninger.

Bare så jeg er klar, kan du tænke på en ERDDAP™ anmodning som:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

I vores tilfælde blev denominator af nedbrud gitteretap eller tabletop anmodninger med visse filtyper, men ingen begrænsninger. Så spørgsmålet er, hvordan man blokerer anmodninger uden begrænsning? Bortset fra det er ikke så simpelt, fordi der er nogen række filtyper, der ikke behøver en begrænsning, og vil være velbevaret, så vi ikke ønsker at blokere dem. Efter at have talt til Chris, og ingen tvivl har jeg efterladt noget, de filtyper, der er godt opført uden begrænsning, er:

.croissant
.iso19115_2
.iso19139_2007
.iso19115_3_2016
 .nc CFHeader
 .nc CFMAHeader
.das
.dds
.html
.ografi
.subset
 .nc Sidehoved
.help
.fgdc
.iso19115
 .nc I nærheden af oJsonHeader (Dette vil være i den kommende nye udgivelse) .

## Løsning af løsning

Løsningen "Jeg fandt" virker i Apache2, jeg ved ikke nginx, men jeg forestiller mig der er noget lignende. Først forsøgte jeg mod_security, men det fik for kompliceret og ikke fungere meget godt. Løsningen er at bruge mod_rewriting. Nu er jeg noget, men en ekspert på dette, så mens jeg definerede, hvad jeg forsøgte at opnå, løsningen og forklaringen, skyldes Claude. ai. ChatGPT vil give dig dybest set samme svar.

Trin 1. Sørg for, at mod_rewriting er installeret og aktiveret. Da hvordan man gør dette varierer med OS, er dette noget, du kan spørge din favorit chatbot.

Trin 2. I den passende fil, der konfigurerer ssl til apache2 (som igen varierer fra OS) , for eksempel kan det være noget som /etc /apache2/sites-aktiveret /sl.conf, tilføje følgende under den relevante VirtualHost definition (Bemærk, hvis du kopierer dette, er der kun 4 linjer, den tredje linje kan blive pakket, upakket den) 

RewritingMotor På On On On
RewritingCond %&#123;QUERY_STRING&#125; ^$
RewritingCond %&#123;REQUEST_URI&#125; ^ (gitteretap |  tabledap ) /[redigér | redigér wikikode] (?&#33;&#33; (?:croissant | I nærheden af iso19115_2 | I nærheden af iso19139_2007 | I nærheden af iso19115_3_2016 | ncCFHeader | ncCFMAHeader | Billeder af das | dds | html | graf graf | subset | ncHeader | hjælper med at hjælpe | fgdc | iso19115 | I nærheden af ncoJsonHeader) $ $ $ $ $) [A-Za-z0-9_]+$
RewritingRule ^ - [R=429,L]

Trin 3. Kontroller, at konfigurationen er gyldig: sudo apache2ctl configtest

Trin Trin Trin Trin Trin Trin Trin 4. Genstart apache2: sudo systemctl genstart apache2

Trin 5. Tjek dine logfiler, at intet blokeres, der ikke bør være, og at passende anmodninger uden begrænsning returnerer en 429 uden nogensinde at ramme din tomcat

## Eksplanation

Hvorfor gør dette arbejde og hvad gør dette - her er forklaringen fra Claude.ai:

Linje 1 — RewritingEngine På On On On
Drejer på mod_rewriting behandling for dette område. Uden det ignoreres RewritingCond/RewritingRule-direktiverne nedenfor.

Linje 2 — Rewriting Cond Supplerende oplysninger om %&#123;QUERY_STRING&#125;
En betingelse, der skal være sandt, før reglen nedenfor gælder. %&#123;QUERY_STRING&#125; er alt efter ? på anmodningssiden. ^$ er en regex, der betyder "start af strenge umiddelbart efterfulgt af tråden" — dvs. en tom streng. Så denne betingelse er kun gyldig, når der ikke er nogen forespørgselsstreng overhovedet — ingen underindstilling/konstraint udtryk på anmodning.

Linje 3 — Rewriting Cond %&#123;REQUEST_URI&#125; ^ (gitteretap |  tabledap ) /[redigér | redigér wikikode] (?&#33;&#33; (?:...) $ $ $ $ $) [A-Za-z0-9_]+$
En anden betingelse, kontrolleret mod %&#123;REQUEST_URI&#125; — den bogstavlige anmodning sti som klienten sendte den, altid den fulde vej, uanset hvor i config denne regel bor (bevidst valgt at lade RewritingRule mønsteret selv gøre matchende, fordi mønstermatching inde i et <Location> blok kan opføre sig utvetydigt — se note nedenfor) . Breaking down te regex:

^/erddap/ — skal starte med /erddap/
 (gitteretap |  tabledap ) / — efterfulgt af en af de to ERDDAP™ adgangsmetoder
[Flere oplysninger] datasetID : en eller flere tegn, der ikke er / eller ?
\\. — en bogstavelig dot
 (?&#33;&#33; (?:croissant | I nærheden af iso19115_2 | ...... | I nærheden af ncoJsonHeader) $ $ $ $ $) — en negativ lookahead: "så længe, hvad der følger, er ikke en af disse nøjagtige fil Typenavne hele vejen til enden af strengen." Disse er filtypeerne ERDDAP™ kan lovligt tjene uden begrænsninger (metadata, struktur, formularsider osv.) — lookahead er, hvad der udelukker dem fra at blive blokeret.
[A-Za-z0-9_]+$ — den egentlige fil Type udvidelse (bogstaver, cifre, understregning) , skal køre til slutningen af strengen.
Så denne betingelse er sand kun, når stien er en gitteretap/ tabledap anmodning om nogle fil Type, der ikke er på sikker liste.

Linjelinje 4 — RewritingRule ^ - [R=429,L]
Selvstyren. Fordi begge betingelser ovenfor skal allerede være sande for Apache til selv at vurdere denne linje, behøver mønsteret her ikke at kontrollere noget andet – ^ bare matcher "start af strengen", som altid er sandt. - betyder "ikke at skrive URL til noget andet" (Vi omdirigerer ikke hvor som helst, bare kortslutning af anmodningen) . Flagene:

R=429 — svare på en HTTP-registreringshandling med statuskode 429 ("Too Mange anmodninger") I stedet for at tjene anmodningen.
L — "Sideste": stoppe med at behandle yderligere omskrive regler, når denne brand.
Sæt sammen: Hvis forespørgselsstrengen er tomt, og anmodningen er for en gitteretap/ tabledap filfil Type, der ikke er på den sikre liste, straks returnere 429 — uden nogensinde at kontakte den ERDDAP /Tomcat backend.

Hvorfor %&#123;REQUEST_URI&#125; i stedet for at lade RewritingRule mønster matche stien direkte (værd at inkludere som en note for kolleger, da det er den manglende del) : indeni en <Location> blok, hvad et bare RewritingRule mønster faktisk bliver matchet mod kan opføre sig uforeneligt afhængigt af Apache version og kontekst. Forsøgt at trække den fulde vej via RewritingCond %&#123;REQUEST_URI&#125; sidetrin, der ambiguity helt — det er altid den bogstavelige, fuldstændige anmodningssti, så regex opfører sig nøjagtigt som skrevet, uanset hvor reglen er redegjort.


Jeg har brugt dette i flere dage nu, og det ser ud til at arbejde meget godt, det blokerer hvad jeg forsøger at blokere og ikke blokere, hvad jeg ikke ønsker at blokere. Og vores ERDDAP™ er blevet meget mere stabil.
