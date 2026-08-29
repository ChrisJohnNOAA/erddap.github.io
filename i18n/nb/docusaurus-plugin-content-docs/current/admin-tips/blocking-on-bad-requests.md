# Blokkering ERDDAP™ forespørsler basert på innholdet i forespørselen, ikke basert på IP

Dette innholdet er basert på en [melding fra Roy Mendelssohn til ERDDAP™ brukergruppe](https://groups.google.com/g/erddap/c/XcvPkoGtchg) ..

## Problemet

Hvis du er som oss, ser du mange roboter som gjør dataforespørsler til din ERDDAP™ , og forespørselene virker ofte som de er gjort av dårlige kodede skript, enten av LLMs (Jeg gjetter) eller av mennesker. En ting med LLMs er at de vil gjøre akkurat det du ber dem om å gjøre, så hvis du ikke forteller dem å ha manuskontrollen returkoder, og ikke fortelle dem hva de skal gjøre i forskjellige tilfeller, så vanligvis vil koden ikke gjøre noe av disse tingene. Hvis du forteller det å fortsette å prøve, vil det. Det er det vi ser i det minste.

Men de som gjør et nummer på vår ERDDAP™ er slik:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Disse sender ERDDAP™ i aldri-aldri land, på grunn av det beste som jeg kan bestemme, at denne URL-en ikke bare ber om hele datasettet, som er jeg vet ikke hvor mange terabytes, men vil at den konverteres til en parkettfil. Verre forespørsler kommer inn fra å flytte IPs, så blokkering av IPs er verre enn puff-a-mole, vil du aldri få dem alle blokkert.

Så spørsmålet er hvordan kan du blokkere basert på innholdet i forespørselen? Før du går videre, hvis du opplever krasj kan det være fra forskjellige årsaker så hva vi ser - jeg brukte mye tid på å gå gjennom logger og også jeg får varslet når det er farlig høy minnebruk, og bemerket når en krasj fulgte det varselet, og også merket hva som var den felles nevneren i disse krasjene, som kanskje ikke er tilfelle for din server. Så forsking dette før du gjør noe, men dette kan gi deg noen ide om hvordan du blokkerer disse forespørsler.

Bare så jeg er klar, kan du tenke på en ERDDAP™ Forespørsel som

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

I våre tilfeller var fellesnevneren av krasjene netdap eller tabletop-forespørsler med visse filtyper, men ingen begrensning. Så spørsmålet er hvordan å blokkere forespørsler uten begrensning? Bortsett fra det er det ikke så enkelt, fordi det er et antall filtyper som ikke trenger en begrensning og vil bli velholdt, så vi vil ikke blokkere dem. Etter å ha snakket med Chris, og ingen tvil om at jeg har utelatt noe, filtypene som er godt oppført uten begrensning er:

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
-  .nc Topptekst
- .hjelp
- .fgdc
- .iso19115
-  .nc oJsonHeader (Dette vil være i den kommende nye utgivelsen) ..

## Løsning

Løsningen " jeg fant" virker i Apache2, jeg vet ikke nginx, men jeg tror det er noe lignende. Først prøvde jeg mod_security, men det ble for komplisert og fungerte ikke så bra. Løsningen er å bruke mod_rewrite. Nå er jeg alt annet enn en ekspert på dette, så mens jeg definerte det jeg prøvde å oppnå, er løsningen, så vel som forklaringen, på grunn av Claude. ai. ChatGPT vil gi deg det samme svaret.

Trinn 1. Sørg for at mod_rewrite er installert og aktivert. Siden hvordan du gjør dette varierer med OS, er dette noe du kan spørre din favoritt chatbot.

Trinn 2. I riktig fil som konfigurerer ssl for apache2 (som igjen varierer fra OS) , for eksempel kan det være noe som /etc/apache2/sites-aktivert/ssl.conf, legger til følgende under riktig VirtualHost definisjon (Merk at hvis du kopierer dette er det bare 4 linjer, den tredje linjen kan pakkes inn og pakkes ut.) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Trinn 3. Kontroller at konfigurasjonen er gyldig: `sudo apache2ctl konfigurasjon` 

Trinn 4. Start om apache2: `sudo systemctl omstart apache2` 

Trinn 5. Sjekk loggene dine at ingenting er blokkert som ikke bør være, og at passende forespørsler uten begrensning returnerer en 429 uten noensinne å treffe din tomcat

## Forklaring

Hvorfor fungerer dette og hva gjør dette - her er forklaringen fra Claude.ai:

Linje 1 — `OmskrivingEngine På` 
Slår på mod_rewrite-behandling for dette omfanget. Uten det blir RewriteCond/RewriteRule-direktivene nedenfor rett og slett ignorert.

Linje 2 — `OmskrivingCond %&#123;QUERY_STRING&#125; ^$` 
En betingelse som må være sant før regelen nedenfor gjelder. `% &#123;QUERY_STRING&#125;` Er alt etter det? i forespørselsadressen. ^$ er en regulær betydning " start av streng umiddelbart fulgt av slutten av strengen" dvs. en tom streng. Så denne betingelsen gjelder bare når det ikke er noen spørringsstreng i det hele tatt - ingen underinnstilling/begrenset uttrykk på forespørselen.

Linje 3 — `OmskrivingCond %&#123;REQUEST_URI&#125; ^/erddap/ (netdap |  tabledap ) /[^/?]+\\. (?&#33; (??...) $) [A-Za-z0-9_]+$` 
En annen tilstand, kontrollert mot `% &#123;REQUEST_URI&#125;` — den bokstavelige forespørselsbanen som klienten sendte den, alltid hele veien uansett hvor i konfigurasjonen denne regelen bor (Med vilje valgt over å la rewriteRule mønsteret selv gjøre matching, fordi mønster-matching inne i en ` <Location> ` blokk kan oppføre seg tvetydig - se notat nedenfor) .. Nedbryt regulært nivå:

 `^/erddap/` - må begynne med /erddap /
 (netdap |  tabledap ) Følgt av en av de to ERDDAP™ tilgangsmetoder

 `[^/?]+` — den datasetID En eller flere tegn som ikke er / eller ?

 ` \\.` — en bokstavelig prikk

 ` (?&#33; (?:croissant | iso19115_2 | ... | ncoJsonHeader) $) ` — et negativt utseende: " så lenge det følgende ikke er en av disse nøyaktige filene Skriv navn hele veien til slutten av strengen." Dette er filtypene ERDDAP™ kan legitimt tjene uten begrensning (metadata, struktur, skjemasider, etc.) — utseendet er det som utelukker dem fra å bli blokkert.

 `[A-Za-z0-9_]+$` — den faktiske filen Type- forlengelse (bokstaver, siffer, understrek) , kreves å kjøre til slutten av strengen.
Så denne tilstanden gjelder bare når banen er en gittedap/ tabledap forespørsel om noen fil Type som ikke er på sikker-uten-begrenset liste.

Linje 4 — `Omskrivingsregel ^ - [R=429,L]` 
Selve regelen. Fordi begge betingelsene ovenfor allerede må være sant for Apache å selv vurdere denne linjen, trenger mønsteret her ikke å sjekke noe annet - ^ bare matches - start på strengen, - som alltid er sant. - betyr " ikke omskriv URL til noe annet" (Vi omdirigerer ingen steder, bare kortslutning forespørselen) .. Flaggene:

 `R=429` — responder med en HTTP omdirigeringsklasse med statuskode 429 ("For mange forespørsler") I stedet for å betjene forespørselen.
L - - Siste - slutte å behandle ytterligere omskrivingsregler når denne brann.
Sett sammen: Hvis spørringsstrengen er tom, og forespørselen er for et rutenett/ tabledap fil Type som ikke er på den sikre, ubegrensede listen, umiddelbart returnere 429 - uten å kontakte ERDDAP /Tomcat backend.

Hvorfor `% &#123;REQUEST_URI&#125;` i stedet for å la omskrivingsregelmønsteret matche banen direkte (verdt å inkludere som et notat for kolleger, siden det er den ikke-uunngåelige delen) Inne i en ` <Location> ` blokk, hvilket bare omskrivingsregelmønster faktisk blir matchet mot kan oppføre seg inkonsekvent avhengig av Apache versjon og kontekst. Eksplisitt trekker hele banen via RewriteCond `% &#123;REQUEST_URI&#125;` sidetrinn som tvetydighet helt og holdent - det er alltid den bokstavelige, komplette forespørselsstien, så regulært oppfører seg nøyaktig som skrevet uansett hvor regelen er hekket.


Jeg har brukt dette i flere dager nå, og det ser ut til å fungere veldig bra, det blokkerer det jeg prøver å blokkere og ikke blokkere det jeg ikke vil blokkere. Og vår ERDDAP™ er blitt mye mer stabilt.
