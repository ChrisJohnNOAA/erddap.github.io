# Blokování ERDDAP™ žádosti založené na obsahu žádosti, které nejsou založeny na OŠ

Tento obsah je založen na [Zpráva od Roye Mendelssohna ERDDAP™ skupina uživatelů](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Problém

Pokud jste jako my, vidíte hodně robotů, kteří požadují vaše údaje ERDDAP™ , A požadavky často vypadají, že jsou prováděny špatně kódované skripty, zda LLM (Můj odhad) nebo lidmi. Jedna věc o LLMs je, že udělají přesně to, k čemu je přimějete, takže pokud jim neřeknete, aby měli skriptové kódy, a neřeknete jim, co mají dělat v různých případech, pak obvykle kód neudělá ani jednu z těchto věcí. A když mu řekneš, aby to dál zkoušel, tak ano. Což je alespoň to, co vidíme.

Ale ti, kteří dělají číslo na našich ERDDAP™ jsou takhle:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Tyto odesílají ERDDAP™ do země nikdy, kvůli tomu nejlepšímu, co mohu zjistit, že tato URL nejenže požaduje celý datový soubor, což je nevím, kolik terabytů, ale chce převést na parketový soubor. Horší jsou požadavky přichází z pohybujících se IP, takže blokování IP je horší než cvaknutí-a-mole, nikdy je všechny blokovat.

Otázkou tedy je, jak můžete blokovat na základě obsahu žádosti? Předtím, než půjdete dále, pokud jste zkušenosti havárie to může být z různých příčin pak to, co vidíme - strávil jsem hodně času procházet záznamy a také jsem byl upozorněn, když je nebezpečné vysoké užívání paměti, a poznamenal, když havárie následovala toto oznámení, a také si všimli, co byl společný jmenovatel v těchto zkratech, což nemusí být případ pro váš server. Prozkoumejte to, než začnete něco podniknout, ale to vám může dát představu, jak tyto požadavky zablokovat.

Jen aby bylo jasno, můžete si představit ERDDAP™ žádost jako:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

V našem případě byly společným jmenovatelem zkratů roštové nebo tabletové požadavky s určitými typy souborů, ale bez omezení. Takže otázkou je, jak zablokovat žádosti bez omezení? Až na to, že to není tak jednoduché, protože existuje několik typů souborů, které nepotřebují omezení a budou dobře vychované, takže je nechceme blokovat. Po rozhovoru s Chrisem a bezpochyby jsem něco vynechal, soubory, které se bez omezení chovají dobře, jsou:

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
-  .nc Hlavička
- .help
- .fgdc
- .iso19115
-  .nc oJsonHeader (Tento bude v nadcházející nové verzi) .

## Roztok

Řešení "jsem našel" funguje v Apache2, nevím, nangx, ale myslím, že tam je něco podobného. Zpočátku jsem se snažil mod_security, ale to se příliš zkomplikovalo a nefungovalo velmi dobře. Řešením je použít mod_rewrite. Jsem na to expert, takže zatímco jsem definoval, čeho jsem se snažil dosáhnout, řešení, stejně jako vysvětlení, je způsobeno Claudem. Ai. ChatGPT vám dodá v podstatě stejnou odpověď.

Krok 1. Ujistěte se, že mod_rewrite je nainstalován a povolen. Vzhledem k tomu, jak to udělat se liší s OS, to je něco, co můžete požádat své oblíbené chatbot.

Krok 2. V příslušném souboru, který konfiguruje ssl pro apache2 (který se opět liší podle OS) , například to může být něco jako /etc/apache2/sites-enabled/ssl.conf, přidat následující pod příslušnou definici VirtualHost (Pokud to zkopírujete, jsou jen 4 řádky, třetí řádek může být zabalen, rozbalit) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Krok 3. Zkontrolujte, zda je konfigurace platná: `sudo apache2ctl configtest` 

Krok 4. Restartovat apache2: `sudo systemctl restart apache2` 

Krok 5 Zkontrolujte své záznamy, že nic není blokováno, že by nemělo být, a že odpovídající požadavky bez omezení vrátit 429 bez nikdy trefit vaše kočka

## Vysvětlení

Proč to funguje a co to dělá - zde je vysvětlení od Claude.ai:

Čára 1 a 2 `PřepsatMotor Na` 
Zapne zpracování mod_rewrite pro tento rozsah. Bez ní jsou směrnice RewriteCond/RewriteRule jednoduše ignorovány.

Čára 2 a 3 `PřepsatCond % &#123;QUERY_STRING&#125; ^$` 
Podmínka, která musí být pravdivá před níže uvedeným pravidlem. `% &#123;QUERY_ STRING&#125;` Je všechno po tom? v URL požadavku. ^$ je regex, což znamená "začátek řetězce okamžitě následuje konec řetězce," tj. prázdný řetězec. Takže tato podmínka platí pouze tehdy, když není žádný řetězec dotazů vůbec žádný subsetting/constrict výraz na žádost.

Čára 3 a 3 `PřepsatCond % &#123;REQUEST_URI&#125; ^/erddap/ (griddap |  tabledap ) /[^/?] +\\. (?&#33; (?:...) $) [A-Za-z0-9_]+$` 
Druhá podmínka, ověřena proti `% &#123;REQUEST_URI&#125;` Doslovná cesta žádosti, jak ji klient poslal, vždy plná cesta bez ohledu na to, kde v konfiguraci toto pravidlo žije (Úmyslně zvolena nad tím, aby RewriteRule vzor sám dělat odpovídající, protože vzor-shoduje uvnitř ` <Location> ` blok se může chovat dvojznačně, viz poznámka níže) . Zlomení regexu:

 `^/erddap/` Musí začít s / erddap/
 (griddap |  tabledap ) Následuje jeden ze dvou. ERDDAP™ přístupové metody

 `[^/?] +` ? datasetID : jeden nebo více znaků, které nejsou / nebo ?

 `\\.` Doslova tečka

 ` (?&#33; (?:croissant | iso19115_2 | ... | ncoJsonHeader) $) ` "Pokud to, co následuje, není jedním z těchto přesných souborů: Napište jména až na konec řetězce." Toto jsou souboryTypes ERDDAP™ může legálně sloužit bez omezení (metadata, struktura, stránky formuláře atd.) Podívejte se dopředu, to vylučuje, aby byly blokovány.

 `[A-Za-z0-9_]+$` ? Typ rozšíření (písmena, číslice, podtržení) , povinen běžet na konec řetězce.
Takže tato podmínka platí pouze tehdy, je-li cesta mřížkou/ tabledap žádost o nějaký soubor Typ, který není na seznamu bez omezení.

Čára 4 a 4 `Přepsat pravidlo ^ - [R=429,L]` 
Samotné pravidlo. Vzhledem k tomu, že obě výše uvedené podmínky musí být pravdivé, aby Apache dokonce vyhodnotit tuto přímku, vzor zde nemusí kontrolovat nic jiného ? ^ jen odpovídá "start řetězce," což je vždy pravda. - znamená "nepřepisujte URL na nic jiného" (Nikde nepřesměrujeme, jen zkratujeme požadavek.) . Vlajky:

 `R=429` Odpovědět pomocí HTTP přesměrování třídy akce nesoucí status kód 429 ("Moc žádostí") místo podání žádosti.
"Poslední": přestaňte zpracovávat další pravidla přepsání, jakmile tento vystřelí.
Dejte dohromady: je-li řetězec dotazů prázdný, a požadavek je pro mřížku/ tabledap soubor Typ, který není na bezpečném seznamu bez omezení, okamžitě vrátit 429  ERDDAP Tomcat backend.

Proč? `% &#123;REQUEST_URI&#125;` místo toho, aby se vzor Přepisu shoduje přímo s cestou (stojí za to zahrnout jako poznámku pro kolegy, protože je to ne-zjevná část) : uvnitř ` <Location> ` block, co bare RewriteRule vzor skutečně dostane zápas proti může chovat nekonzistentní v závislosti na Apache verzi a kontext. Explicitně táhne celou cestu přes PřepsatCond `% &#123;REQUEST_URI&#125;` Je to vždy doslovná, úplná cesta žádosti, takže regex se chová přesně tak psané bez ohledu na to, kde je pravidlo hnízděno.


Používám to už několik dní a zdá se, že to funguje velmi dobře, blokuje to, co se snažím zablokovat a neblokovat to, co nechci blokovat. A naše ERDDAP™ je mnohem stabilnější.
