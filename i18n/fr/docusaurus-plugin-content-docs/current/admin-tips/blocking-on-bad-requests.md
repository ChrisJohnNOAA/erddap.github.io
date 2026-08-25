# Blocage ERDDAP™ requêtes fondées sur le contenu de la requête, non sur la PI

Ce contenu est basé sur [message de Roy Mendelssohn au ERDDAP™ groupe d'utilisateurs](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Le problème

Si vous êtes comme nous, vous voyez beaucoup de robots faire des demandes de données à votre ERDDAP™ , et les requêtes semblent souvent être faites par des scripts mal codés, que ce soit par LLMs (à mon avis) ou par des humains. Une chose à propos des LLMs est qu'ils feront exactement ce que vous les invitez à faire, donc si vous ne leur dites pas d'avoir le script vérifier les codes de retour, et ne leur dites pas quoi faire dans différents cas, alors généralement le code ne fera aucune de ces choses. Et si tu lui dis de continuer à essayer, ça le fera. C'est ce que nous voyons au moins.

Mais ceux qui font un nombre sur notre ERDDAP™ sont comme ceci:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Ils envoient ERDDAP™ dans jamais-jamais atterrir, en raison, au mieux que je peux déterminer, que cette URL ne demande pas seulement l'ensemble de données, qui est je ne sais pas combien de téraoctets, mais veut qu'il converti en un fichier parquet. Pire que les requêtes viennent du mouvement des IP, donc le blocage des IP est pire que le whack-a-mole, vous ne les aurez jamais toutes bloquées.

Donc la question est de savoir comment bloquer en fonction du contenu de la requête ? Avant d'aller plus loin, si vous êtes accidentés il peut être de différentes causes alors ce que nous voyons - J'ai passé beaucoup de temps à passer par les journaux et aussi je suis averti quand il ya une utilisation dangereuse de la mémoire élevée, et noté quand un accident a suivi cette notification, et a également remarqué ce qui était le dénominateur commun dans ces accidents, ce qui peut ne pas être le cas pour votre serveur. Donc, étudier ceci avant d'agir, mais cela peut vous donner une idée de comment bloquer ces demandes.

Juste pour que je sois clair, vous pouvez penser à un ERDDAP™ demander que:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

Dans notre cas, le dénominateur commun des crashes était quadrildap ou requêtes tabletop avec certains types de fichiers mais aucune contrainte. Donc la question est de savoir comment bloquer les requêtes sans contrainte ? Sauf que ce n'est pas si simple, parce qu'il y a un certain nombre de types de fichiers qui n'ont pas besoin d'une contrainte et seront bien tenus, donc nous ne voulons pas les bloquer. Après avoir parlé à Chris, et sans doute j'ai laissé de côté quelque chose, les types de fichiers qui se comportent bien sans contrainte sont:

.croissant
.iso19115_2
.iso19139_2007
.iso19115_3_2016
 .nc Chef de mission
 .nc Chef
.das
.dds
.html
Graphique
.sous-ensemble
 .nc En-tête
.aide
.fgdc
.iso19115
 .nc oJsonHeader (celui-ci sera dans la prochaine nouvelle version) .

## Solution

La solution "J'ai trouvé" fonctionne dans Apache2, je ne connais pas le nginx mais j'imagine qu'il y a quelque chose de similaire. Au début, j'ai essayé mod_security, mais ça a été trop compliqué et ça n'a pas très bien fonctionné. La solution est d'utiliser mod_rewrite. Maintenant, je ne suis qu'un expert sur ce sujet, alors pendant que je définissais ce que j'essayais d'accomplir, la solution, ainsi que l'explication, est due à Claude. Oui. ChatGPT vous fournira essentiellement la même réponse.

Étape 1. Assurez-vous que mod_rewrite est installé et activé. Puisque comment faire cela varie avec OS, c'est quelque chose que vous pouvez demander à votre chatbot préféré.

Étape 2. Dans le fichier approprié qui configure ssl pour apache2 (qui varie à nouveau selon le système d'exploitation) , par exemple, il pourrait être quelque chose comme /etc/apache2/sites-enabled/ssl.conf, ajouter ce qui suit sous la définition VirtualHost appropriée (notez que si vous copiez ceci il n'y a que 4 lignes, la troisième ligne peut être enveloppée, déballer) 

RéécrireEngine À
RéécritureCond %&#123;QUERY_STRING&#125; ^$
RéécritureCond %&#123;REQUEST_URI&#125; ^/erddap/ (quadrillé |  tabledap ) [^/?]+\\. (Qu'est-ce qu'il y a ? (?:croissant | iso19115_2 | iso19139_2007 | iso19115_3_2016 | ncCFHeader | ncCFMAHeader | das | dds | html | graphique | sous-ensemble | ncEn-tête | Aide | fgdc | iso19115 | NcoJsonHeader) $) [A-Za-z0-9_]+$
Règle de réécriture ^ - [R=429,L]

Étape 3. Vérifiez que la configuration est valide : sudo apache2ctl configtest

Étape 4. Redémarrer apache2: sudo systemctl redémarrer apache2

Étape 5. Vérifiez vos journaux que rien n'est bloqué qui ne devrait pas être, et que les requêtes appropriées sans contrainte retourner un 429 sans jamais frapper votre tomcat

## Explication

Pourquoi ce travail et ce qu'il fait - voici l'explication de Claude.ai:

Ligne 1 — Réécrire le moteur À
Active le traitement mod_rewrite pour cette portée. Sans cela, les directives RewriteCond/RewriteRule ci-dessous sont simplement ignorées.

Ligne 2 — Réécrire Cond %&#123;QUERY_STRING&#125; ^$
Une condition qui doit être vraie avant l'application de la règle ci-dessous. %&#123;QUERY_STRING&#125; est tout après le ? dans l'URL de la requête. ^$ est un regex qui signifie "démarrage de la chaîne immédiatement suivi par fin de chaîne" — c'est-à-dire une chaîne vide. Cette condition n'est donc vraie que lorsqu'il n'y a pas de chaîne de requête du tout — aucune expression de subsetting/contrainte sur la requête.

Ligne 3 — Réécrire Cond %&#123;REQUEST_URI&#125; ^/erddap/ (quadrillé |  tabledap ) [^/?]+\\. (Qu'est-ce qu'il y a ? (?:...) $) [A-Za-z0-9_]+$
Une seconde condition, cochée avec %&#123;REQUEST_URI&#125; — le chemin littéral de requête comme le client l'a envoyé, toujours le chemin complet quel que soit l'endroit où se trouve cette règle dans la configuration (délibérément choisi au lieu de laisser le motif RewriteRule lui-même faire la correspondance, parce que l'appariement de motif à l'intérieur d'un <Location> bloc peut se comporter ambiguement — voir note ci-dessous) . Découper le régex :

^/erddap/ — doit commencer par /erddap/
 (quadrillé |  tabledap ) / — suivie d'un des deux ERDDAP™ méthodes d'accès
[^/?]+ — datasetID : un ou plusieurs personnages qui ne sont pas / ou ?
\\. — un point littéral
 (Qu'est-ce qu'il y a ? (?:croissant | iso19115_2 | ... | NcoJsonHeader) $) — un regard négatif: "aussi longtemps que ce qui suit n'est pas un de ces fichiers exacts Tapez des noms jusqu'à la fin de la chaîne." Ce sont les types de fichiers ERDDAP™ peut servir légitimement sans contrainte (métadonnées, structure, pages de formulaire, etc.) — le regard est ce qui les exclut d'être bloqués.
[A-Za-z0-9_]+$ — le fichier réel Extension du type (lettres, chiffres, soulignement) , requis pour courir jusqu'à la fin de la chaîne.
Cette condition est donc vraie seulement lorsque le chemin est un quadrillé/ tabledap requête pour un certain fichier Type qui n'est pas sur la liste de sécurité sans contrainte.

Ligne 4 — Réécrire la règle ^ - [R=429,L]
La règle elle-même. Parce que les deux conditions ci-dessus doivent déjà être vraies pour qu'Apache puisse même évaluer cette ligne, le modèle ici n'a pas besoin de vérifier autre chose — ^ correspond simplement au "début de la chaîne", ce qui est toujours vrai. - signifie "ne pas réécrire l'URL à quelque chose de différent" (nous ne redirigeons nulle part, juste court-circuiter la demande) . Les drapeaux :

R=429 — répondez avec une action de classe de redirection HTTP portant le code d'état 429 (Trop de demandes) au lieu de répondre à la demande.
L — "Dernier": arrêter de traiter d'autres règles de réécriture une fois que celui-ci tire.
Mise en place : si la chaîne de requête est vide, ET la requête est pour un griddap/ tabledap fichier Type qui n'est pas sur la liste de sécurité-sans contrainte, retourner immédiatement 429 — sans jamais contacter le ERDDAP Le moteur Tomcat.

Pourquoi %&#123;REQUEST_URI&#125; au lieu de laisser le motif RewriteRule correspondre directement au chemin (mérite d'être inclus comme note pour les collègues, car c'est la partie non évidente) : à l'intérieur <Location> block, ce qu'un motif nu RewriteRule est en fait adapté contre peut se comporter de façon incohérente selon la version et le contexte d'Apache. Tirer explicitement sur le chemin complet via RewriteCod %&#123;REQUEST_URI&#125; fait complètement abstraction de cette ambiguïté — c'est toujours le chemin littéral et complet de la requête, de sorte que le regex se comporte exactement comme écrit indépendamment de l'endroit où la règle est nichée.


Je l'utilise depuis plusieurs jours maintenant et il semble fonctionner très bien, il bloque ce que j'essaie de bloquer et ne bloque pas ce que je ne veux pas bloquer. Et notre ERDDAP™ est devenu beaucoup plus stable.
