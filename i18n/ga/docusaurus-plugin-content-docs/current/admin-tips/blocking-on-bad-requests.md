# Blocáil ERDDAP™ iarrataí bunaithe ar ábhar na hiarrata, nach bhfuil bunaithe ar an IP

Tá an t-ábhar seo bunaithe ar [teachtaireacht ó Roy Mendelssohn go dtí an ERDDAP™ web development](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## An fhadhb

Má tá tú cosúil linn, tá tú ag féachaint ar a lán de na bots a dhéanamh iarratais sonraí ar do ERDDAP™ , agus is cosúil na hiarrataí go minic cosúil go bhfuil siad déanta ag scripteanna droch códaithe, cibé acu ag LLMs (mo buille faoi thuairim) nó ag daoine. Rud amháin faoi LLMs go mbeidh siad a dhéanamh go díreach cad pras tú iad a dhéanamh, mar sin más rud é nach bhfuil tú ag insint dóibh go bhfuil na cóid ar ais seiceáil script, agus nach insint dóibh cad atá le déanamh i gcásanna éagsúla, ansin de ghnáth ní bheidh an cód a dhéanamh ceachtar de na rudaí. Agus má deir tú é a choinneáil ar ag iarraidh, beidh sé. Cad é an méid atá á fheiceáil againn ar a laghad.

Ach na cinn atá ag déanamh roinnt ar ár ERDDAP™ Tá siad mar seo:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Na sheoladh ERDDAP™ i riamh-never talamh, mar gheall ar, go dtí an chuid is fearr gur féidir liom a chinneadh, go bhfuil an URL ní amháin ag iarraidh an tacar sonraí ar fad, a bhfuil mé nach bhfuil a fhios cé mhéad terabytes, ach ba mhaith leis a thiontú go comhad parquet. Worse na hiarratais ag teacht i ó IPanna ag gluaiseacht, mar sin tá IPanna blocála níos measa ná whack-a-mole, ní bheidh tú iad a fháil go léir blocked.

Mar sin, is é an cheist conas is féidir leat bloc bunaithe ar ábhar na hiarrata? Sula ag dul a thuilleadh, má tá tú tuairtí taithí d'fhéadfadh sé a bheith ó cúiseanna éagsúla ansin cad tá muid ag féachaint - Chaith mé a lán ama ag dul trí logs agus freisin a fháil ar an eolas nuair a bhíonn úsáid cuimhne ard contúirteach, agus faoi deara nuair a lean tuairteála go fógra, agus freisin faoi deara cad a bhí an denominator coitianta sna tuairtí, d'fhéadfadh nach bhfuil an cás do do fhreastalaí. Mar sin, taighde seo roimh aon ghníomh, ach d'fhéadfadh sé seo a thabhairt duit roinnt smaoineamh conas a bloc na hiarratais.

Díreach mar sin tá mé soiléir, is féidir leat smaoineamh ar ERDDAP™ iarraidh mar:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

In ár gcás bhí an denominator coiteann de na tuairtí griddap nó iarratais boird le cineál comhaid áirithe ach gan srian. Mar sin, is é an cheist conas iarrataí a bhlocáil gan srian? Ach amháin nach bhfuil sé sin simplí, toisc go bhfuil aon líon de cineál comhaid nach gá srian agus beidh a bheith dea-behaved, mar sin ní féidir linn ag iarraidh a bloc siúd. Tar éis caint le Chris, agus aon amhras tá mé d'fhág amach rud éigin, tá na cineálacha comhaid atá behaved go maith gan srian:

taiseachas aeir: fliuch
Seirbhís do Chustaiméirí
.iso19139_2007
Tuilleadh roghanna...
 .nc CUMARSÁID
 .nc FM a chosaint
Seirbhís do Chustaiméirí
.dds
..
.graf
. subset
 .nc Ceannteideal
.help
.Fgdc
Seirbhís do Chustaiméirí
 .nc Seirbhís do Chustaiméirí (beidh an duine seo sa scaoileadh nua atá le teacht) .

## i gceannas par64, faoi stiúir lampa

Oibríonn an réiteach "Faigh mé" in Apache2, níl a fhios agam nginx ach shamhlú mé go bhfuil rud éigin den chineál céanna. Ar dtús rinne mé mod_security, ach a fuair ró-chasta agus ní raibh ag obair go han-mhaith. Is é an réiteach a úsáid mod_rewrite. Anois tá mé rud ar bith ach saineolaí ar seo, mar sin cé a shainmhínítear mé cad a bhí mé ag iarraidh a chur i gcrích, an réiteach, chomh maith leis an míniú, mar gheall ar Claude. ai. Cuirfidh ChatGPT an freagra céanna ar fáil duit go bunúsach.

Céim 1. Déan cinnte go bhfuil mod_rewrite suiteáilte agus ar chumas. Ós rud é conas é seo a dhéanamh athraíonn le OS, tá sé seo rud éigin is féidir leat a iarraidh ar do chatbot is fearr leat.

Céim 2. Sa chomhad cuí a chumrú ssl le haghaidh apache2 (a athraíonn arís ag OS) , mar shampla d'fhéadfadh sé a bheith rud éigin cosúil / srl / pache2/sites-chumasaithe / Ssl.conf, cuir an méid seo a leanas faoin sainmhíniú cuí VirtualHost (Tabhair faoi deara mura bhfuil ach 4 líne ann, d'fhéadfaí an tríú líne a fhillte, gan é a fhillte) 

Athscríobh Ar an tsráid
Athscríobh % &#123;QUERY_STRING&#125; ^$
Athscríobh % &#123;REQUEST_URI&#125; (cineál gas: in airde |  tabledap ) /[^/?] +\\. (?&#33; (:Cruthú | An bhfuil a fhios agat? | Iso1939 | Tuilleadh roghanna... | ncCFHeader | ncCFMAHeder | taiseachas aeir: fliuch | taiseachas aeir: fliuch | html | grafa grater | fotha RSS | ncHeader | cabhrú le | Seirbhís do Chustaiméirí | Seirbhís do Chustaiméirí | cliceáil grianghraf a mhéadú) $ CAIBIDIL $) [A-Za-z0-9_] + $
Athscríobh ^ - [R = 429,L]

Céim 3. Seiceáil go bhfuil an chumraíocht bailí: sudo apache2ctl configtest

Céim na Céime 4. Atosú a dhéanamh ar apache2: atosú córasach atosú

Céim 5. Seiceáil do logs go bhfuil aon rud á blocked nár chóir a bheith, agus go iarrataí cuí gan srian ar ais 429 gan riamh bualadh do tomcat

## An tSraith Shinsearach

Cén fáth a dhéanann an obair seo agus cad a dhéanann sé seo - is é seo an míniú ó Claude.ai:

Líne 1 - RewriteEngine Ar an tsráid
Cas ar phróiseáil mod_rewrite don raon feidhme seo. Gan é, na treoracha RewriteCond / RewriteRule thíos neamhaird go simplí.

Líne 2 - Athscríobh Déan teagmháil linn % &#123;QUERY_STRING&#125; ^$
Coinníoll nach mór a bheith fíor roimh an riail thíos. % &#123;QUERY_STRING&#125; Tá gach rud tar éis an ? sa URL iarratais. Is ^ $ a chiallaíonn regex "tú teaghrán ina dhiaidh sin láithreach ag deireadh teaghrán" - ie, teaghrán folamh. Mar sin, tá an coinníoll fíor ach amháin nuair nach bhfuil aon teaghrán cheist ar chor ar bith - aon fo-thacarála / léiriú constraint ar an iarratas.

Líne 3 — Athscríobh Déan teagmháil linn % &#123;REQUEST_URI&#125; ^/níos airde/ (cineál gas: in airde |  tabledap ) /[^/?] +\\. (?&#33; (?:...) $ CAIBIDIL $) [A-Za-z0-9_] + $
A dara coinníoll, a sheiceáil i gcoinne % &#123;REQUEST_URI&#125; - an cosán a iarraidh litriúil mar a chuir an cliant é, i gcónaí ar an cosán iomlán beag beann ar áit sa config saol riail seo (d'aon ghnó a roghnaíodh thar ligean an patrún RewriteRule féin a dhéanamh ar an meaitseáil, mar gheall ar patrún-oiriúnú taobh istigh <Location> Is féidir bloc féin a iompar débhríoch - féach nóta thíos) . Briseadh síos an regex:

^/aerddap/ — ní mór tús a chur leis / a tharraingt/
 (cineál gas: in airde |  tabledap ) / — agus ceann amháin den dá ERDDAP™ bealaí rochtana
[^/?]+ — datasetID : ceann amháin nó níos mó carachtair nach bhfuil / nó?
\\. — ponc litriúil
 (?&#33; (:Cruthú | An bhfuil a fhios agat? | ... | cliceáil grianghraf a mhéadú) $ CAIBIDIL $) - lookahead diúltach: "fad nach bhfuil an méid seo a leanas ar cheann de na comhad cruinn Cineál ainmneacha go léir ar an mbealach go dtí deireadh an teaghrán. " Is iad seo an fileTypes ERDDAP™ Is féidir freastal go dlisteanach gan aon srian (meiteashonraí, struchtúr, leathanaigh fhoirm, etc.) - is é an lookahead cad a eisiamh ó bheith bac.
[A-Za-z0-9_] + $ - an comhad iarbhír Cineál síneadh (litreacha, digití, underscore) , ag teastáil a reáchtáil go dtí deireadh an teaghrán.
Mar sin, tá an coinníoll fíor ach amháin nuair a bhíonn an cosán a griddap / tabledap iarraidh ar roinnt comhad Cineál nach bhfuil ar an liosta sábháilte-gan-constraint.

Líne Líne Líne Líne 4 - RewriteRule ^ - [R = 429,L]
An riail féin. Toisc nach mór an dá coinníollacha thuas a bheith fíor cheana féin le haghaidh Apache chun meastóireacht a dhéanamh fiú an líne seo, ní gá an patrún anseo rud ar bith eile a sheiceáil - ^ oireann ach "tús an teaghrán," atá i gcónaí fíor. - ciallaíonn "ná athscríobh an URL le rud ar bith difriúil" (nach bhfuil muid ag atreorú áit ar bith, ach gearr-chuaird an t-iarratas) . Na bratacha:

R = 429 — freagra a thabhairt le HTTP atreorú-aicme gníomh ag iompar cód stádais 429 ("Iarratais Too go leor") in ionad freastal ar an iarraidh.
L - "Last": stop a phróiseáil aon rialacha athscríobh breise uair amháin an tine amháin.
Cuir le chéile: má tá an teaghrán cheist folamh, AGUS is é an t-iarratas le haghaidh greille / tabledap comhad comhad Cineál nach bhfuil ar an liosta sábháilte gan srian, láithreach ar ais 429 - gan dul i dteagmháil riamh leis an ERDDAP / Tomcat siar.

Cén fáth % &#123;REQUEST_URI&#125; in ionad ligean ar an patrún RewriteRule mheaitseáil leis an cosán go díreach (fiú lena n-áirítear mar nóta do chomhghleacaithe, ós rud é go bhfuil sé an chuid neamh-obvious) : taobh istigh de <Location> bloc, cad a fhaigheann patrún RewriteRule lom iarbhír mheaitseáil i gcoinne féidir iad féin a iompar go neamhréireach ag brath ar leagan Apache agus comhthéacs. Explicitly ag tarraingt an cosán iomlán trí RewriteCond % &#123;REQUEST_URI&#125; sidesteps go ambiguity go hiomlán - tá sé i gcónaí ar an litriúil, cosán a iarraidh iomlán, mar sin an regex behaves go díreach mar atá scríofa beag beann ar áit a bhfuil an riail neadaithe.


Tá mé ag baint úsáide as seo ar feadh roinnt laethanta anois agus is cosúil go bhfuil sé ag obair go han-mhaith, tá sé blocála cad tá mé ag iarraidh a bloc agus gan blocáil cad nach bhfuil mé ag iarraidh a bloc. Agus ár ERDDAP™ tar éis éirí i bhfad níos cobhsaí.
