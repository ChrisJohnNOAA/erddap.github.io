# Blockierung ERDDAP™ Anträge auf Grundlage des Inhalts der Anfrage, nicht auf der Grundlage der IP

Dieser Inhalt basiert auf einer [Nachricht von Roy Mendelssohn an die ERDDAP™ Benutzergruppe](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Das Problem

Wenn Sie wie wir sind, sehen Sie viele Bots, die Datenanfragen an Ihre ERDDAP™ , und die Anfragen scheinen oft, als würden sie von schlecht codierten Skripten durchgeführt, ob von LLMs (Meine Vermutung) oder durch Menschen. Eine Sache an LLMs ist, dass sie genau das tun, was Sie sie zu tun veranlasst, so wenn Sie nicht sagen, dass sie die Skript-Check-Return-Codes haben, und nicht sagen, was Sie in verschiedenen Fällen tun, dann in der Regel wird der Code nicht von diesen Dingen tun. Und wenn du es sagst, um weiter zu versuchen, wird es das. Was wir zumindest sehen.

Aber die, die eine Nummer auf unserer ERDDAP™ sind wie diese:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Diese senden ERDDAP™ in nie-never Land, aufgrund der besten, die ich bestimmen kann, dass diese URL nicht nur den gesamten Datensatz anfordert, was ich nicht weiß, wie viele Terabytes, sondern will, dass sie in eine Parkettdatei umgewandelt wird. Worse, die Anfragen kommen aus bewegten IPs, so Sperren IPs ist schlechter als whack-a-mole, Sie werden nie alle blockiert.

Die Frage ist also, wie können Sie basierend auf dem Inhalt der Anfrage blockieren? Bevor Sie weiter gehen, wenn Sie Erfahrungen Crashes sind, kann es von verschiedenen Ursachen sein, dann, was wir sehen - Ich verbrachte eine Menge Zeit durch Protokolle und auch ich benachrichtigt, wenn es gefährlich hohe Speichernutzung, und bemerkte, wenn ein Crash folgte dieser Benachrichtigung, und auch bemerkte, was der gemeinsame Nenner in diesen Crashs, die möglicherweise nicht der Fall für Ihren Server. So forschen Sie dies, bevor Sie irgendwelche Maßnahmen ergreifen, aber dies kann Ihnen einige Vorstellung geben, wie Sie diese Anträge blockieren.

Nur damit ich klar bin, können Sie an eine ERDDAP™ Anfrage als:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

In unserem Fall war der gemeinsame Nenner der Crashs Gridap- oder Tabletop-Anfragen mit bestimmten Dateitypen, aber keine Einschränkung. Die Frage ist also, wie man Anfragen ohne Einschränkung blockieren kann? Außer es ist nicht so einfach, denn es gibt eine Reihe von Dateitypen, die keine Einschränkung brauchen und sich gut verhalten werden, so wollen wir diese nicht blockieren. Nach dem Reden mit Chris, und ohne Zweifel habe ich etwas ausgelassen, die Dateitypen, die gut verhalten sind ohne Einschränkung sind:

- .croissant
- .iso19115_2
- .iso19139_2007
- .iso19115_3_2016
-  .nc CFHeader
-  .nc CFMAHeader
- .das
- .ddd
- .html
- .graph
- .subset
-  .nc Kopf
- .help
- .fgdc
- .iso19115
-  .nc OJsonHeader (dieser wird in der kommenden neuen Veröffentlichung) .

## Lösung

Die Lösung "Ich fand" funktioniert in Apache2, ich weiß nicht Nginx, aber ich denke, es gibt etwas Ähnliches. Zuerst versuchte ich mod_security, aber das wurde zu kompliziert und funktionierte nicht sehr gut. Die Lösung besteht darin, mod_rewrite zu verwenden. Jetzt bin ich etwas anderes als ein Experte, also während ich definierte, was ich zu erreichen versuchte, ist die Lösung sowie die Erklärung auf Claude zurückzuführen. ai. ChatGPT wird Ihnen grundsätzlich die gleiche Antwort geben.

Schritt 1. Stellen Sie sicher, dass mod_rewrite installiert und aktiviert ist. Da, wie man dies mit OS tut, ist dies etwas, das Sie Ihren Lieblings-Chatbot fragen können.

Schritt 2. In der entsprechenden Datei, die ssl für apache2 konfiguriert (die nach Betriebssystemen wieder variiert) , zum Beispiel, es könnte sein, wie /etc/apache2/sites-enabled/ssl.conf, fügen Sie das folgende unter der entsprechenden VirtualHost Definition (Hinweis, wenn Sie dies kopieren, gibt es nur 4 Zeilen, die dritte Zeile kann gewickelt werden, unwrap it) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Schritt 3. Überprüfen Sie, ob die Konfiguration gültig ist: `sudo apache2ctl configtest` 

Schritt 4. Neustart apache2: `sudo systemctl restart apache2` 

Schritt 5. Prüfen Sie Ihre Protokolle, dass nichts blockiert wird, das nicht sein sollte, und dass entsprechende Anfragen ohne Einschränkung zurück einen 429, ohne jemals Ihren tomcat zu treffen

## Erläuterung

Warum funktioniert diese Arbeit und was tut dies - hier ist die Erklärung von Claude.ai:

Zeile 1 — `RewriteEngine Auf` 
Schaltet die mod_rewrite Verarbeitung für diesen Bereich ein. Ohne sie werden die untenstehenden RewriteCond/RewriteRule-Richtlinien einfach ignoriert.

Zeile 2 — `RewriteContent %&#123;QUERY_STRING&#125;` 
Eine Bedingung, die wahr sein muss, bevor die nachstehende Regel gilt. `%&#123;QUERY_STRING&#125;` Ist alles nach dem ? in der Anfrage-URL. ^$ ist ein Regex, der "Start des Strings unmittelbar gefolgt von Ende des Strings" bedeutet, d.h. ein leerer String. Diese Bedingung ist also nur dann wahr, wenn überhaupt keine Abfrage-Strings vorhanden sind – kein Subsetting/Constraint-Expression auf der Anfrage.

Zeile 3 — `RewriteCondat %&#123;REQUEST_URI&#125; ^/erdap/ (Netzteil |  tabledap ) /[^/?]+\\. (?&#33; (?:) $) [A-Za-z0-9_]+$` 
Eine zweite Bedingung, gegen `%&#123;REQUEST_URI)` — der buchstäbliche Anforderungspfad, wie der Client es gesendet hat, immer der volle Pfad, unabhängig davon, wo diese Regel in der config lebt (bewusst gewählt über das RewriteRule-Muster selbst die Anpassung tun, weil Musteranpassung in einem ` <Location> ` Block kann sich mehrdeutig verhalten — siehe Anmerkung unten) . Aufbrechen des Regex:

 `^/erdap/` — muss mit /erddap/ beginnen
 (Netzteil |  tabledap ) / — gefolgt von einem der beiden ERDDAP™ Zugriffsmethoden

 `[^/?]+` — die datasetID : ein oder mehrere Zeichen, die nicht / oder ?

 `\\.` — ein wörtlicher Punkt

 ` (?&#33; (? | Iso19115_2 | ... | NcoJsonHeader) $) ` — ein negativer Lookahead: "Solange das Folgende nicht eine dieser genauen Datei ist Geben Sie Namen bis zum Ende der Zeichenkette ein." Dies sind die DateiTypen ERDDAP™ kann rechtmäßig ohne Einschränkung dienen (Metadaten, Struktur, Formularseiten, etc.) — der Lookahead ist, was sie davon ausschließt, blockiert zu werden.

 `[A-Za-z0-9_]+$` — die tatsächliche Datei Typ Verlängerung (Buchstaben, Ziffern, Unterstrich) , benötigt, um zum Ende der Saite zu laufen.
Diese Bedingung ist also nur dann wahr, wenn der Pfad ein Gridap/ tabledap Anfrage für einige Datei Typ, der nicht auf der Safe-ohne-constraint-Liste ist.

Zeile 4 — `RewriteRule ^ - [R=429,L]` 
Die Regel selbst. Da beide oben genannten Bedingungen bereits für Apache gelten müssen, um diese Zeile sogar auszuwerten, braucht das Muster hier nichts anderes zu überprüfen — ^ passt einfach zu "Start der Saite", was immer wahr ist. - bedeutet "die URL nicht auf etwas anderes neu schreiben" (wir leiten nirgendwo um, kurzschließen die Anfrage) . Die Flaggen:

 `R = 429` — mit einer HTTP-Umleitungs-Klasse-Aktion mit Statuscode 429 reagieren ("Zu viele Anfragen") anstatt die Anfrage zu bedienen.
L — "Letzte": Stoppen Sie die Bearbeitung weiterer Nachschreiben Regeln, sobald dieser feuert.
Zusammenfügen: wenn der Abfragestring leer ist, UND die Anfrage ist für ein Raster/ tabledap Datei Typ, der nicht auf der Safe-unconstrained-Liste ist, sofort zurück 429 - ohne jemals Kontakt mit der ERDDAP /Tomcat Backend.

Warum? `%&#123;REQUEST_URI)` anstatt das RewriteRule-Muster direkt auf den Pfad passen zu lassen (als Hinweis für Kollegen, da es der nichtobvious Teil ist) : innerhalb eines ` <Location> ` block, was ein bloßes RewriteRule-Muster tatsächlich angepasst wird, kann sich unkonsistent je nach Apache-Version und Kontext verhalten. Explizit den vollen Weg über RewriteCond ziehen `%&#123;REQUEST_URI)` Nebenschritte, die Mehrdeutigkeit ganz — es ist immer der wörtliche, vollständige Anforderungspfad, so verhält sich der Regex genau so geschrieben, unabhängig davon, wo die Regel geschachtelt ist.


Ich habe das schon seit mehreren Tagen benutzt und es scheint sehr gut zu funktionieren, es blockiert, was ich versuche zu blockieren und nicht zu blockieren, was ich nicht blockieren will. Und unser ERDDAP™ ist viel stabiler geworden.
