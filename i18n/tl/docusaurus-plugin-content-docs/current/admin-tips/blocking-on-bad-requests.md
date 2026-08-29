# Pagharang ERDDAP™ mga kahilingan batay sa nilalaman ng kahilingan, hindi batay sa IP

Ang nilalamang ito ay batay sa isang [mensahe mula kay Roy Mendelssohn hanggang sa ERDDAP™ grupong gumagamit](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Ang problema

Kung katulad mo kami, nakikita mo ang maraming bot na humihiling ng impormasyon sa iyong mga anak ERDDAP™ , at ang mga kahilingan ay kadalasang parang ginagawa ng mga script na hindi nai - code, ito man ay sa pamamagitan ng mga LLM (hula ko) o ng mga tao. Ang isang bagay tungkol sa mga LLM ay gagawin nila kung ano ang ipinagagawa mo sa kanila, kaya kung sasabihin mo sa kanila na ipa - check return codes, at sasabihin sa kanila ng doncilt kung ano ang gagawin sa iba't ibang kaso, kung gayon karaniwan nang hindi gagawin ng kodigo ang alinman sa mga bagay na iyon. At kung sasabihin mo ito na patuloy na magsikap, iyon ay mangyayari. Iyan ang nakikita natin sa paano man.

Ngunit ang mga gumagawa ng marami sa atin ERDDAP™ ay gaya nito:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Ang mga ito ay nagpapadala ERDDAP™ sa hindi kailanman lupain, dahil sa, sa pinakamahusay na kaya kong matiyak, na ang URL na ito ay hindi lamang humihiling ng buong dataset, na siyang doniket ko kung ilan ang mga terabyte, kundi nais ko na ito ay gawing parquet file. Mas masahol pa ang mga kahilingan mula sa paglipat ng mga IP, kaya ang pagharang sa mga IP ay mas masahol pa sa wlack-a-mole, hindi mo kailanman mabarahan ang lahat ng ito.

Kaya ang tanong ay paano mo mapagtatakpan ang nilalaman ng kahilingan? Bago ka magpatuloy, kung ikaw ay nakararanas ng mga banggaan ito ay maaaring mula sa iba't ibang sanhi kung ano ang aming nakikita - ako ay gumugol ng maraming panahon sa pagdaan sa mga troso at nalalaman ko rin kung kailan may mapanganib na gamit sa memorya, at napansin mo kapag ang pagbagsak ay kasunod ng notasyong iyon, at napansin din kung ano ang karaniwang nangyayari sa mga banggaang ito, na maaaring hindi siyang kalagayan ng iyong server. Kaya saliksikin ito bago kumilos, subalit ito'y maaaring magbigay sa iyo ng ideya kung paano hahadlangan ang mga kahilingang ito.

Kung paanong ako'y maliwanag, maaari kang mag - isip ng isang bagay ERDDAP™ humiling ng:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

Sa aming kaso ang karaniwang hangganan ng mga banggaan ay ang mga kahilingan sa griddap o tabletop na may ilang filetype subalit walang pumipigil. Kaya ang tanong ay kung paano hahadlangan ang mga kahilingan nang walang hadlang? Maliban sa hindi ganiyang kapayakan, dahil may anumang bilang ng mga filetype na nangangailangan ng doniket at magiging well-behaved, kaya't nais nating pigilan ang mga iyon. Pagkatapos makipag - usap kay Chris, at walang alinlangang may iniwan ako, ang mga filetype na pinangangasiwaan nang walang hadlang ay:

- .croissant
- .iso19115_2
- .iso19139_2007
- .iso19115_3_2016
-  .nc CFHeader
-  .nc CFMAHeader
- .das
- .ds
- .html
- .grap
- .subset
-  .nc Ulo
- .help
- .fgdc
- .iso19115
-  .nc OJson Header (ito ay sa darating na bagong release) .

## Lunas

Ang solusyong "Nakasumpong ako" ng mga gawa sa Apache2, hindi ko alam ang nginx ngunit sa palagay ko'y may pagkakatulad. Noong una ay sinubukan ko ang paggamit ng mod_security, subalit iyan ay naging napakasalimuot at hindi mabisa. Ang solusyon ay gumamit ng mod_rewrite. Ngayon ay hindi na ako eksperto rito, kaya habang ipinaliliwanag ko kung ano ang sinisikap kong gawin, ang lunas, gayundin ang paliwanag, ay dahil kay Claude. ai. Ang ChatGPT ay magbibigay sa iyo ng gayunding sagot.

Hakbang 1. Tiyakin na ang mod_rewrite ay naka-install at kaya. Yamang ang paraan ng paggawa nito ay iba - iba sa OS, ito ay isang bagay na maaari mong itanong sa iyong paboritong chatbot.

Hakbang 2. Sa angkop na talaksan na nag-aayos ng ssl para sa apache2 (na muling nagkakaiba - iba sa pamamagitan ng OS) , halimbawa ito ay maaaring isang bagay na katulad ng /etc/apache2/sites-enabled/ssl.conf, idagdag ang mga sumusunod sa ilalim ng angkop na totalHost na depinisyon (Pansinin kung kinopya mo ito nang 4 na linya lamang, ang ikatlong linya ay maaaring nakabalot, nang walang takip) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Hakbang 3. Tiyakin na ang pagsasaayos ay may bisa: `sudo apache2ctl Decreetest` 

Hakbang 4. Restarm apache2: `sudo systemctl restart apache2` 

Hakbang 5. Suriin ang iyong mga troso na walang anumang bagay na nababarahan, at na ang angkop na mga kahilingan nang hindi pinagbabawalang ibalik ang 429 nang hindi kailanman tinatamaan ang iyong tomcat

## Paliwanag

Bakit ito ginagawa at ang ginagawa nito ay paliwanag mula kay Claude.ai:

Line 1 — `Muling Pagsulat Patuloy` 
Buksan ang mod_rewrite processing para sa saklaw na ito. Kung wala ito, ang mga instruksiyon ng RewriteCond/RewriteRule sa ibaba ay basta hindi pinapansin.

Line 2 — `Isulat muli ang %ićQURY_STRINGivić ^$` 
Isang kalagayan na kailangang maging totoo bago ikapit ang tuntunin sa ibaba. `%°EQURY_STRINGiON` ang lahat ng bagay pagkatapos ng ? sa kahilingan ng URL. Ang ^$ ay isang regex na nangangahulugang "bituin ng kuwerdas na kaagad na sinusundan ng dulo ng kuwerdas" — i.e., isang walang laman na strando. Kaya ang kondisyong ito ay totoo lamang kapag walang query string — walang subsetting/constraint expression sa kahilingan.

Line 3 — `Isulat ang % EXTREQUESST_URI&#125; ^/erddap/ (" griddap " |  tabledap ) /[^/?]+\\. (?&#33; (?:...) $) [A-Za-z0-9_]+$` 
Ikalawang kondisyon, laban sa `%°TQUEST_URI&#125;` — ang literal na landas ng paghiling habang ipinadadala ito ng kliyente, laging ang ganap na landas saanman nakatira ang tuntuning ito (Sadyang pinili sa pagpapaubaya sa rewriteRule pattern na gawin mismo ang pagtutugma, dahil ang pattern-match sa loob ng isang a ` <Location> ` Ang block ay maaaring kumilos nang malabo — tingnan ang nota sa ibaba) . Pagpapahinto sa regex:

 `^/erddap/` — dapat magsimula sa /erddap/
 (" griddap " |  tabledap ) / — sinundan ng isa sa dalawa ERDDAP™ paraan ng pag-akses

 `[^/?]+` — ang datasetID : isa o higit pang karakter na hindi / o ?

 `\\.` — isang literal na tuldok

 ` (?&#33; (?: Croissant | iso19115_2 | ... | NcoJson Header) $) ` — negatibong anyo: "Habang ang sumusunod ay hindi isa sa eksaktong talaksang ito Pangalan ng tipo hanggang dulo ng kuwerdas." Ito ang mga fileType ERDDAP™ ay maaaring maglingkod nang walang hadlang (metadata, istruktura, mga anyong pahina, atbp.) — ang ulo ng tingin ang dahilan kung bakit hindi sila nahahadlangan.

 `[A-Za-z0-9_]+$` — ang aktuwal na talaksan Karagdagang uri (, numero, diin) , kailangan tumakbo sa dulo ng kuwerdas.
Kaya ang kondisyong ito ay totoo lamang kapag ang landas ay isang griddap/ tabledap Humingi ng talaksan Type na hindi nasa ligtas-walang-constraint na talaan.

Linya 4 — `RewriteRule ^ - [R=429,L]` 
Ang tuntunin mismo. Dahil ang dalawang kondisyong ito sa itaas ay dapat nang totoo para sa Apache na suriin pa ang linyang ito, ang padron dito ay hindi na kailangang suriin ang anumang bagay — ^ lamang ang mga posporong "bituin ng strando," na laging totoo. - nangangahulugang "huwag isulat muli ang URL sa anumang kakaibang bagay" (Hindi kami nagreredirect kahit saan, pero sandali lang.) . Ang mga bandila:

 `R=429` — tumutugon sa pamamagitan ng isang HTTP redirect-class aksyon na nagdadala ng status code 429 ("Ato Maraming Kahilingan") sa halip na sundin ang kahilingan.
L — "Huli": itigil na ang pagpoproseso ng anumang karagdagang mga tuntunin sa muling pagsulat minsang ang isang ito ay magliyab.
Pinagsama-sama: kung walang laman ang query string, AT ang hiling ay para sa isang griddap/ tabledap talaksan Uri na hindi nasa ligtas na-unconstrained list, agad bumalik 429 — nang hindi man lamang nakikipag-ugnayan sa ERDDAP /Tomcat backend.

Bakit `%°TQUEST_URI&#125;` Sa halip na hayaan ang rewriteRule pattern ay tuwirang tumutugma sa landas (halaga bilang isang nota para sa mga kasamahan, dahil ito ang hindi-obvious na bahagi) : sa loob ng a ` <Location> ` block, anong walang palamang RewriteRule pattern ay aktuwal na nagtutugma laban ay maaaring mag-aasal na pabagu-bago depende sa Apache bersiyon at konteksto. May kasanayang hinihila ang buong landas sa pamamagitan ng RewriteCond `%°TQUEST_URI&#125;` Ang di - tiyak na mga hakbang na iyon — ito ang laging literal, kumpletong paraan ng paghiling, kaya ang regex ay kumikilos na kagayang - kagaya ng nasusulat saanmang dako ginagawa ang tuntunin.


Ginagamit ko ito sa loob ng ilang araw na ngayon at wari bang ito'y gumagana nang mahusay, hinahadlangan nito ang sinisikap kong hadlangan at hindi hinahadlangan ang gusto kong hadlangan. At ang aming ERDDAP™ ay naging mas matatag.
