# Blocare ERDDAP™ cereri bazate pe conținutul cererii, nu pe PA

Acest conţinut se bazează pe [mesaj de la Roy Mendelssohn la ERDDAP™ grupul de utilizatori](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Problema

Dacă sunteți ca noi, vedeți o mulțime de roboți care fac cereri de date ERDDAP™ , și cererile par adesea ca acestea sunt realizate de scripturi slab codificate, dacă de LLMs (Bănuiala mea) sau de oameni. Un lucru despre LLMs este că ei vor face exact ceea ce îi vei îndemna să facă, așa că, dacă nu le spui să aibă codurile script check return, și nu le spune ce să facă în diferite cazuri, atunci, de obicei, codul nu va face oricare dintre aceste lucruri. Şi dacă îi spui să continue să încerce, o va face. Ceea ce vedem cel puţin.

Dar cei care fac un număr pe nostru ERDDAP™ sunt așa:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Acestea trimit ERDDAP™ în never-never land, due, to the best that I can determine, that this URL is not only requesting the whole settle, which is I don't know how many terabytes, but wants it converted to a parquet file. Mai rău cererile vin de la mutarea IP-uri, astfel încât blocarea IP-uri este mai rău decât whack-a-mole, nu le va primi toate blocat.

Deci, întrebarea este cum poți bloca pe baza conținutului cererii? Înainte de a merge mai departe, dacă sunteți accidente de experiență poate fi din cauze diferite, atunci ceea ce vedem - am petrecut o mulțime de timp merge prin busteni și, de asemenea, am fost notificat atunci când există o utilizare periculoasă de înaltă memorie, și a remarcat atunci când un accident a urmat acea notificare, și, de asemenea, a observat care a fost numitorul comun în aceste accidente, care nu poate fi cazul serverului. Cercetaţi acest lucru înainte de a lua orice acţiune, dar acest lucru vă poate da o idee cum să blocaţi aceste cereri.

Doar ca să fiu clar, te poţi gândi la un ERDDAP™ cerere ca:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

În cazul nostru, numitorul comun al accidentelor a fost griddap sau cereri tabletop cu anumite tipuri de fișiere, dar nici o constrângere. Întrebarea este cum să blocăm cererile fără nicio constrângere? Cu excepția faptului că nu este atât de simplu, pentru că există orice număr de fișiere care nu au nevoie de o constrângere și va fi bine comportat, așa că nu vrem să le blocheze. Dupa ce am vorbit cu Chris, si fara indoiala am omis ceva, fisierele care se comporta bine fara nici o obligatie sunt:

.croissant
.iso19115_2
.iso19139_2007
.iso19115_3_2016
 .nc CFHeader
 .nc CFMAheader
.das
.dds
.html
.graph
.subset
 .nc Antet
Ajutor.
.fgdc
.iso19115
 .nc OJsonHeader (Acesta va fi în viitoarea versiune) .

## Soluţie

Soluția "am găsit" funcționează în Apache2, nu știu nginx, dar îmi imaginez că există ceva similar. La început am încercat mod_securitate, dar care a devenit prea complicat și nu a funcționat foarte bine. Solutia este sa folosesti mod_rescrie. Acum sunt orice altceva decât un expert în acest sens, așa că în timp ce am definit ce încercam să realizez, soluția, precum și explicația, se datorează lui Claude. Ai. ChatGPT vă va furniza practic același răspuns.

Pasul 1. Asigurați-vă că modul_rescrie este instalat și activat. Deoarece modul de a face acest lucru variază cu OS, acest lucru este ceva ce puteți întreba chatbot preferat.

Pasul 2. În fișierul corespunzător care configurează ssl pentru apache2 (care din nou variază cu SG) , de exemplu, ar putea fi ceva de genul /etc/apache2/sites-enabled/ssl.conf, adăugați următoarele sub definiția VirtualHost corespunzătoare (Notă dacă copiați acest lucru există doar 4 linii, a treia linie poate fi înfășurat, despachetați-l) 

Rescrie motorul On
RescrieCond % &#123;QUERY_STRING&#125; ^$
RescrieCond % &#123;REQUEST_URI&#125; ^/erddap/ (griddap |  tabledap ) / [^/?]+\\. (&#33; (?:croissant | izo19115_2 | izo19139_2007 | izo19115_3_2016 | ncCFheader | ncCFMAheader | das | dds | html | grafic | subset | ncHeader | Ajutor | fgdc | izo19115 | ncoJsonHeader) $) [A-Za-z0-9_]+$
Rescrierea regulii ^ - [R=429,L]

Pasul 3. Verificați dacă configurația este validă: sudo apache2ctl configtest

Pas 4. Restart apache2: sudo systemctl repornire apache2

Pasul 5. Verificați jurnalele că nimic nu este blocat care nu ar trebui să fie, și că cererile adecvate fără o constrângere returna un 429 fără a lovi vreodată moca ta

## Explicație

De ce funcționează acest lucru și ce face acest lucru - aici este explicația de la Claude.ai:

Linia 1  On
Pornește procesul mod_rescrie pentru acest domeniu de aplicare. Fără aceasta, directivele privind regulamentul de rescriere/rescriere de mai jos sunt pur și simplu ignorate.

Rescrie linia 2 Cond % &#123;QUERY_STRING&#125; ^$
O condiție care trebuie să fie adevărată înainte de aplicarea regulii de mai jos. Totul e după? în URL-ul cererii. ^$ este un regex care înseamnă "start de șir imediat urmat de sfârșitul șir" . Adică, un șir gol. Deci, această condiție este adevărat numai atunci când nu există nici un șir de întrebări la toate 

Rescrie linia 3 Cond % [REQUEST_URI&#125; ^/erddap/ (griddap |  tabledap ) / [^/?]+\\. (&#33; (?:...) $) [A-Za-z0-9_]+$
O a doua condiție, verificată împotriva % &#123;REQUEST_URI&#125;  (ales în mod deliberat pentru a lăsa modelul RescrieRegula în sine face potrivire, deoarece model-potrivire în interiorul unui <Location> blocul se poate comporta ambiguu ) . Spargerea regex:

Trebuie să începem cu /erddap/
 (griddap |  tabledap ) /  ERDDAP™ metode de acces
[^/?]+ datasetID : unul sau mai multe personaje care nu sunt / sau ?
Un punct literal
 (&#33; (?:croissant | izo19115_2 | ... | ncoJsonHeader) $) "atâta timp cât ceea ce urmează nu este unul dintre aceste dosar exact Scrieţi nume până la capăt." Acestea sunt tipurile de fișiere ERDDAP™ poate servi în mod legitim fără nici o constrângere (metadate, structură, pagini de formă etc.) 
[A-Za-z0-9_]+$  Extensie tip (litere, cifre, subliniere) , necesar pentru a rula la sfârșitul șirului.
Deci, această condiție este adevărat numai atunci când calea este o griddap / tabledap cerere pentru un fișier Tip care nu este pe lista de siguranţă-fără-constrângere.

Linie 4 - [R=429,L]
Regula în sine. Deoarece ambele condiții de mai sus trebuie să fie deja adevărat pentru Apache pentru a evalua chiar și această linie, modelul aici nu are nevoie pentru a verifica nimic altceva  - înseamnă "nu rescrie URL-ul la nimic diferit" (Nu redirecţionăm nicăieri, doar scurtcircuităm cererea.) . Steagurile:

R=429  ("Prea multe cereri") în loc de a servi cererea.
L 
Pune împreună: în cazul în care șirul de interogare este gol, și cererea este pentru o griddap / tabledap fișier Tip care nu este pe lista neconstrânsă în condiții de siguranță, întoarce imediat ERDDAP /Tomcat suport.

De ce % &#123;REQUEST_URI&#125; în loc de a lăsa modelul de rescriere se potrivesc direct cu calea (în valoare de inclusiv ca o notă pentru colegi, deoarece este partea non-evident) : în interiorul <Location> block, ceea ce un model de rescriere gol se potriveste de fapt împotriva poate comporta inconsecvent în funcție de versiunea Apache și context. Tragerea explicită a căii complete prin RescriereCond % &#123;REQUEST_URI&#125; pași laterali care ambiguitate în întregime 


Am fost folosind acest timp de câteva zile acum și se pare că funcționează foarte bine, este blocarea ceea ce am încercat să blochez și nu blocarea ceea ce nu vreau să blochez. Şi a noastră ERDDAP™ a devenit mult mai stabil.
