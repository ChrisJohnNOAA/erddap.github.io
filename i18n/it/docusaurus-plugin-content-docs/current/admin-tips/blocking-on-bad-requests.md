# Blocco ERDDAP™ richieste basate sul contenuto della richiesta, non basate sull'IP

Questo contenuto si basa su un [messaggio da Roy Mendelssohn a ERDDAP™ utenti di gruppo](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Il problema

Se siete come noi, state vedendo un sacco di bot che fanno richieste di dati al vostro ERDDAP™ , e le richieste sembrano spesso come sono fatte da script scarsamente codificati, sia da LLMs (La mia ipotesi) o da esseri umani. Una cosa su LLMs è che faranno esattamente ciò che li spingete a fare, quindi se non dite loro di avere i codici di ritorno del controllo script, e non dite loro cosa fare in casi diversi, allora di solito il codice non farà nessuna di queste cose. E se lo dici di continuare a provare, lo fara'. E' quello che vediamo almeno.

Ma quelli che stanno facendo un numero sul nostro ERDDAP™ sono così:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Questi mandati ERDDAP™ in terra mai-never, a causa, al meglio che posso determinare, che questo URL non è solo la richiesta dell'intero set di dati, che è non so quanti terabyte, ma vuole che si converta in un file di parquet. Peggiore che le richieste vengano da IP in movimento, in modo da bloccare gli IP è peggio di whack-a-mole, non si otterrà mai tutti bloccati.

Quindi la domanda è come si può bloccare in base al contenuto della richiesta? Prima di andare oltre, se si sono crash di esperienza può essere da cause diverse poi quello che stiamo vedendo - Ho speso un sacco di tempo passando attraverso i registri e anche mi viene notificato quando c'è pericoloso uso di memoria alta, e annotato quando un crash ha seguito quella notifica, e anche notato che cosa era il denominatore comune in questi crash, che potrebbe non essere il caso per il vostro server. Quindi ricerca questo prima di intraprendere qualsiasi azione, ma questo può darvi qualche idea su come bloccare queste richieste.

Solo per essere chiaro, puoi pensare a un ERDDAP™ richiesta come:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

Nel nostro caso il denominatore comune dei crash erano richieste di griddap o tabletop con alcuni tipi di file, ma nessun vincolo. Quindi la domanda è come bloccare le richieste senza un vincolo? Tranne che non è così semplice, perché ci sono un certo numero di tipi di file che non hanno bisogno di un vincolo e saranno ben comportati, quindi non vogliamo bloccarli. Dopo aver parlato con Chris, e senza dubbio ho lasciato fuori qualcosa, i filetipi che sono ben comportati senza un vincolo sono:

- .croissant
- .iso191152
- .iso19139_2007
- .iso19115_3_2016
-  .nc CFHeader
-  .nc CFMAHeader
- .das
- .dds
- .html
- .
- .sottoset
-  .nc Intestazione
- .aiuto
- .
- .
-  .nc OJsonHeader (questo sarà nel prossimo nuovo rilascio) .

## Soluzione

La soluzione "ho trovato" funziona a Apache2, non conosco nginx ma immagino che ci sia qualcosa di simile. All'inizio ho provato mod_security, ma questo è diventato troppo complicato e non ha funzionato molto bene. La soluzione è quella di utilizzare mod_rewrite. Ora sono tutt'altro che un esperto su questo, così mentre ho definito quello che stavo cercando di realizzare, la soluzione, così come la spiegazione, è dovuta a Claude. Ai. ChatGPT ti fornirà fondamentalmente la stessa risposta.

Passo 1. Assicurarsi che mod_rewrite sia installato e abilitato. Dal momento che come fare questo varia con il sistema operativo, questo è qualcosa che si può chiedere il vostro chatbot preferito.

Passo 2. Nel file appropriato che configura ssl per apache2 (che varia di nuovo da OS) , per esempio potrebbe essere qualcosa come /etc/apache2/sites-enabled/sssl.conf, aggiungere il seguente sotto la definizione VirtualHost appropriata (nota se si copia questo ci sono solo 4 linee, la terza linea può essere avvolta, unwrap esso) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Passo 3. Controllare che la configurazione sia valida: `sudo apache2ctl configtest` 

Passo 4. Riavviare apache2: `sudo systemctl restart apache2` 

Passo 5. Controlla i tuoi registri che nulla è bloccato che non dovrebbe essere, e che le richieste appropriate senza un vincolo restituiscono un 429 senza mai colpire il tuo tomcat

## Spiegazione

Perché questo lavoro e cosa fa questo - ecco la spiegazione di Claude.ai:

Linea 1 — `RewriteEngine Su` 
Attiva l'elaborazione mod_rewrite per questo scopo. Senza di essa, le direttive RewriteCond/RewriteRule qui sotto sono semplicemente ignorate.

Linea 2 — `RewriteCond %&#123;QUERY_STRING&#125; ^$` 
Una condizione che deve essere vera prima della regola qui sotto si applica. `?` E' tutto dopo? nella richiesta URL. ^$ è un regex che significa "inizio di stringa immediatamente seguito da fine di stringa" — cioè una stringa vuota. Quindi questa condizione è vera solo quando non c'è nessuna stringa di query affatto — nessuna espressione subsetting/constraint su richiesta.

Linea 3 — `RewriteCond %&#123;REQUEST_URI&#125; ^/erddap/ (Grida |  tabledap ) [^/?]+\\. (? (?) &#33;) [A-Za-z0-9_]+$` 
Una seconda condizione, verificata contro `?` — il percorso di richiesta letterale come il client lo ha inviato, sempre il percorso completo indipendentemente da dove nella configurazione di questa regola vive (volutamente scelto sopra lasciando il modello RewriteRule stesso fare la corrispondenza, perché pattern-matching all'interno di un ` <Location> ` blocco può comportarsi ambiguamente — vedi nota sotto) . Rompere il regex:

 `^/erddap/` — deve iniziare con /erddap/
 (Grida |  tabledap ) / — seguito da uno dei due ERDDAP™ metodi di accesso

 `[^]+` — il datasetID : uno o più caratteri che non sono / o ?

 `\\.` — un punto letterale

 ` (? (? | iso19115_2 | ... | ncoJsonHeader) &#33;) ` — una fronte negativa: "finché ciò che segue non è uno di questi file esatti Digitare i nomi fino alla fine della stringa." Questi sono il fileTypes ERDDAP™ può servire legittimamente senza vincoli (metadati, struttura, pagine di forma, ecc.) — la fronte è ciò che li esclude dall'essere bloccati.

 `[A-Za-z0-9_]+$` — il file effettivo Tipo di estensione (lettere, cifre, underscore) , necessario eseguire fino alla fine della stringa.
Quindi questa condizione è vera solo quando il percorso è una grigliata/ tabledap richiesta di file Tipo che non e' nella lista sicura senza vincoli.

Linea 4 — `Traduzione:` 
La regola stessa. Poiché entrambe le condizioni di cui sopra devono già essere vere per Apache per valutare anche questa linea, il modello qui non ha bisogno di controllare nient'altro — ^ solo corrisponde "start of the string", che è sempre vero. - significa "non riscrivere l'URL a qualcosa di diverso" (Non stiamo reindirizzando da nessuna parte, solo cortocircuitando la richiesta) . Le bandiere:

 `R=429` — rispondere con un codice di stato di redirect-class HTTP 429 ("Molte richieste") invece di servire la richiesta.
L — "Ultima": interrompere l'elaborazione di ulteriori regole di riscrittura una volta che questo fuoco.
Mettere insieme: se la stringa di query è vuota, E la richiesta è di un grigliata/ tabledap file Tipo che non è nella lista sicura-non-constrained, restituire immediatamente 429 — senza mai contattare il ERDDAP /Tomcat backend.

Perché? `?` invece di lasciare che il modello RewriteRule corrisponda direttamente al percorso (vale anche come nota per i colleghi, dal momento che è la parte non ovvia) : ` <Location> ` blocco, quello che un nudo modello RewriteRule viene effettivamente abbinato contro può comportarsi in modo inconsistente a seconda della versione e del contesto Apache. Esplicitamente tirando il percorso completo tramite RewriteCond `?` sidesteps che ambiguità completamente — è sempre il percorso letterale e completo di richiesta, così il regex si comporta esattamente come scritto indipendentemente da dove la regola è nidificata.


Sto usando questo per diversi giorni ora e sembra funzionare molto bene, sta bloccando ciò che sto cercando di bloccare e non bloccare ciò che non voglio bloccare. E il nostro ERDDAP™ è diventato molto più stabile.
