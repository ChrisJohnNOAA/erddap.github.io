# Blokkaaminen ERDDAP™ pyynnön sisältöön perustuvat pyynnöt, jotka eivät perustu IP:hen

Tämä sisältö perustuu a [Kirjoittanut Roy Mendelssohn ERDDAP™ Käyttäjäryhmä](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Ongelma

Jos olet kuin me, näet paljon botteja, jotka tekevät tietopyyntöjä sinulle. ERDDAP™ ja pyynnöt näyttävät usein siltä, että ne tehdään huonosti koodatuilla käsikirjoituksilla, olivatpa ne sitten LLM: llä. (Arvaukseni) tai ihmisillä. Yksi asia LLM: ssä on se, että he tekevät juuri sen, mitä kehotat heitä tekemään, joten jos et käske heitä käyttämään käsikirjoituksen palautuskoodia, eivätkä kerro, mitä tehdä eri tapauksissa, koodi ei yleensä tee mitään näistä asioista. Ja jos sanot, että jatkat yrittämistä, niin se tulee. Sitä me ainakin näemme.

Ne, jotka tekevät numeron meidän ERDDAP™ Tällaisia ovat:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Nämä lähettävät ERDDAP™ Koskaan, koska paras, jonka voin määrittää, että tämä URL ei vain pyydä koko tietoaineistoa, mikä on en tiedä kuinka monta teratavua, mutta haluan sen muuntavan parquet-tiedostoksi. Mitä pahempia pyyntöjä on tulossa liikkuvista IP: istä, joten IP: n estäminen on pahempaa kuin valaanpyynti, et koskaan saa niitä kaikkia estettyinä.

Kysymys kuuluukin, miten voit estää pyynnön sisällön? Ennen kuin menet pidemmälle, jos sinulla on kokemusta onnettomuuksista, se voi olla eri syistä, mitä sitten näemme - vietin paljon aikaa lokien läpi ja saan ilmoituksen, kun on vaarallinen korkea muistin käyttö, ja huomata, kun onnettomuus seurasi tätä ilmoitusta, ja myös huomata, mikä oli yhteinen nimittäjä näissä onnettomuuksissa. Tutki tätä ennen kuin ryhdyt mihinkään toimiin, mutta tämä voi antaa sinulle mahdollisuuden estää nämä pyynnöt.

Niin, että olen selvä, voit ajatella ERDDAP™ Pyyntö:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

Tapauksessamme onnettomuuksien yhteinen nimittäjä oli griddap- tai tabletop-pyyntö tietyillä tiedostotyypeillä, mutta ei rajoituksia. Kysymys on siitä, miten hakemukset voidaan estää ilman rajoitusta. Paitsi että se ei ole niin yksinkertainen, koska on olemassa useita tiedostotyyppejä, jotka eivät tarvitse rajoitusta ja ovat hyvin käyttäytyviä, joten emme halua estää niitä. Puhuttuani Chrisille, ja epäilemättä olen jättänyt jotain, tiedostotyypit, jotka toimivat hyvin ilman rajoituksia ovat:

- .croissant
- .iso19115_2
- .iso19139_2007
- .iso19115_3_2016
-  .nc CF Header
-  .nc CFMA Head
- .das
- .dds
- .html
- .grafiikka
- .subset
-  .nc Pääjohtaja
- .help
- .fgdc
- .iso19115
-  .nc OJson Head (Tämä on tulossa uuteen julkaisuun) .

## Ratkaisu

Ratkaisu "löysin" toimii Apache2:ssa, en tiedä nginxiä, mutta kuvittelen, että jotain vastaavaa on. Aluksi kokeilin mod_securitya, mutta se ei toiminut kovinkaan hyvin. Ratkaisu on mod_rewrite. Nyt olen kaikkea muuta kuin asiantuntija, joten kun määrittelin, mitä yritin saavuttaa, ratkaisu ja selitys johtuvat Claudesta. Ai. ChatGPT antaa sinulle periaatteessa saman vastauksen.

1. Varmista, että mod_rewrite on asennettu ja käytössä. Koska tämä vaihtelee OS: n kanssa, voit kysyä suosikki chatbotistasi.

Vaihe 2. Sopiva tiedosto, joka määrittää sl apache2 (joka taas vaihtelee) Esimerkiksi se voi olla jotain, kuten /etc/apache2/sites-yhteensopiva/sl.conf, lisätä seuraavan asianmukaisen VirtualHost määritelmän mukaisesti. (Huomaa, että jos kopioit tämän on vain neljä riviä, kolmas rivi voidaan kääriä, poista se.) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Vaihe 3. Tarkista, että kokoonpano on voimassa: `Sudo Apache2ctl konfigurtti` 

Askel 4. Käynnistä apache2: `Sudo Systemctl uudelleenkäynnistetty Apache2` 

Vaihe 5 Tarkista lokisi, että mitään ei ole estetty, ja että asianmukaiset pyynnöt ilman rajoitusta palauttaa 429 osumatta tomcat.

## Selitys

Miksi tämä toimii ja mitä se tekee - tässä on selitys Claude.ai:lle:

Linja 1 - `Uudelleenkirjoitus On On On` 
Käännä mod_rewrite-prosessointi tälle laajuudelle. Ilman sitä alla olevat RewriteCond/RewriteRule -direktiivit jätetään huomiotta.

Linja 2 - `RewriteCond % [QUERY_STRING]` 
Ehto, joka on totta ennen alla olevaa sääntöä, on voimassa. `Prosentti (QUERY)` Onko kaikki sen jälkeen? pyynnöstä URL. • on regex, joka tarkoittaa "jousi alkua, jota seuraa välittömästi merkkijonon päättyessä" eli tyhjää jonoa. Joten tämä ehto on totta vain silloin, kun kyselylomaketta ei ole lainkaan – ei alisäämistä tai rajoittavaa ilmaisua pyynnöstä.

Linja 3 - `RewriteCond % [REQUEST_URI] (Griddap |  tabledap ) [ ] (??&#33; (? :-)) $) [A-Za-z0-9_+]` 
Toinen ehto, tarkastettu `Prosentti (viittaukset)` - kirjaimellinen pyyntöpolku asiakkaan lähettämänä, aina täysi tie riippumatta siitä, missä konfiguraatiossa tämä sääntö elää. (tarkoituksellisesti valittu, kun RewriteRule-kuvio itsessään tekee sovituksen, koska kuvio-ottelu sisällä ` <Location> ` blokki voi käyttäytyä epäselvästi – katso alapuolelta) . Ryöstää regex:

 `^/erddap/` Pitäisi aloittaa /erddap
 (Griddap |  tabledap ) • seuraa yksi kahdesta ERDDAP™ Käyttötavat

 `[ ] +` - datasetID Yksi tai useampi henkilö, jotka eivät ole / tai

 `\\.` Kirjaimellinen piste

 ` (??&#33; (:croissant | 115_2 | ............ | ncoJson Header) $) ` kielteinen näkemys: "Niin kauan kuin seuraa, ei ole yksi näistä tarkoista tiedostoista. Tyypin nimet ovat tien päähän asti.” Nämä ovat tiedostotyypit ERDDAP™ voi laillisesti toimia ilman rajoituksia (Metadata, rakenne, muotosivut jne.) – Näkö on se, mikä sulkee heidät pois.

 `[A-Za-z0-9_+]` Todellinen tiedosto Tyypin laajennus (Kirjaimet, Digits, Underscore) Jouduttiin juoksemaan loppuun asti.
Tämä ehto on totta vain silloin, kun reitti on verho/ tabledap Pyydä tiedostoa Tyyppi, joka ei ole turvassa ilman rajoituksia.

Linja 4 - `Rewrite Rule - [R=429, L]` 
itse sääntö. Koska molemmat edellä mainitut olosuhteet ovat jo totta, että Apache voi jopa arvioida tämän linjan, kuvion ei tarvitse tarkistaa mitään muuta - ↑ vain vastaa "jousi alkua", joka on aina totta. tarkoittaa ”Älä kirjoita URL-osoitetta uudelleen mihinkään muuhun” (Emme ole uudelleenohjautumassa mihinkään, vain lyhytkestoinen pyyntö.) . Liput:

 `R = 429` vastata HTTP:n uudelleenohjaustoiminnolla, jolla on statuskoodi 429 ("Liikaa pyyntöjä") pyynnön sijaan.
L - "Last": Lopeta uusien sääntöjen käsittely, kun tämä tulee.
Yhdessä: jos kyselylomake on tyhjä, ja pyyntö on verkon/ tabledap tiedostotiedosto Tyyppi, joka ei ole turvallista rajoittamatonta luetteloa, palauttaa välittömästi 429 - ilman, että otat yhteyttä. ERDDAP Tomcat taustalla.

Miksi miksi miksi `Prosentti (viittaukset)` Sen sijaan, että RewriteRule-malli sopisi suoraan polkuun (Se on arvokasta, sillä se on ei-selvä osa.) Sisällä A ` <Location> ` blokki, mitä paljas RewriteRule-malli todella sopii yhteen, voi käyttäytyä epäjohdonmukaisesti Apache-version ja kontekstin mukaan. Täydellistä reittiä RewriteCondin kautta `Prosentti (viittaukset)` Se on aina kirjaimellinen, täydellinen pyyntöpolku, joten regex käyttäytyy kirjallisesti riippumatta siitä, missä sääntö on pestetty.


Olen käyttänyt tätä jo useita päiviä ja se näyttää toimivan hyvin, se estää sen, mitä yritän estää ja estää sen, mitä en halua estää. Ja meidän ERDDAP™ Siitä on tullut paljon vakaampi.
