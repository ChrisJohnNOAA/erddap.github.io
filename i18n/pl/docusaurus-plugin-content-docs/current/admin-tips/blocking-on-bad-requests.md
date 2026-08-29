# Blokowanie ERDDAP™ wnioski oparte na treści wniosku, a nie na OD

Ta zawartość jest oparta na [wiadomość od Roy Mendelssohn do ERDDAP™ grupa użytkowników](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Problem

Jeśli jesteś jak my, widzisz wiele boty, które żądają danych do ERDDAP™ , a prośby często wydają się być wykonywane przez słabo zakodowane skrypty, czy przez LLM (Zgaduję.) lub przez ludzi. Jedną z rzeczy w LLM jest to, że będą robić dokładnie to, do czego ich nakłaniasz, więc jeśli nie powiesz im, żeby mieli kod zwrotny do sprawdzania skryptów i nie mów im, co mają robić w różnych przypadkach, to zwykle kod nie zrobi żadnej z tych rzeczy. I jeśli powiesz mu, żeby próbował, to będzie. Co przynajmniej widzimy.

Ale te, które robią numer na naszej ERDDAP™ są takie:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Te wysyłają ERDDAP™ do nigdy- nigdy nie wylądować, ze względu na najlepsze, że mogę określić, że ten URL nie tylko prosi o cały zestaw danych, co jest nie wiem, ile terabajtów, ale chce go przekształcić do pliku parkietu. Gorzej są prośby z ruchomych IP, więc blokowanie IP jest gorsze niż wack-a-mole, nigdy ich nie zablokują.

Pytanie brzmi, jak można blokować na podstawie treści wniosku? Przed pójściem dalej, jeśli jesteś doświadczenie awarii może być z różnych przyczyn, a następnie to, co widzimy - Spędziłem dużo czasu przeglądając dzienniki, a także dostaje powiadomienie, gdy istnieje niebezpieczne wysokie wykorzystanie pamięci, i zauważył, kiedy wypadek po tej notyfikacji, a także zauważył, co było wspólnym mianownikiem w tych awarii, co może nie być w przypadku serwera. Zbadaj to przed podjęciem jakichkolwiek działań, ale to może dać jakiś pomysł, jak zablokować te żądania.

Żeby było jasne, możesz pomyśleć o... ERDDAP™ wniosek:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

W naszym przypadku wspólnym mianownikiem wypadków były prośby griddap lub tabletop z niektórych typów plików, ale bez ograniczeń. Pytanie brzmi, jak blokować wnioski bez ograniczeń? Tylko, że to nie jest takie proste, ponieważ istnieje wiele typów plików, które nie potrzebują ograniczenia i będą dobrze zachowane, więc nie chcemy ich blokować. Po rozmowie z Chrisem, i bez wątpienia coś pominąłem, filetypy, które są dobrze zachowane bez ograniczeń są:

- .croissant
- .iso19115 _ 2
- .iso19139 _ 2007
- .iso19115 _ 3 _ 2016
-  .nc CFHeader
-  .nc CFMAHeader
- .das
- .dds
- .html
- .graph
- .subset
-  .nc Nagłówek
- .help
- .fgdc
- .iso19115
-  .nc oJsonHeader (Ten będzie w nadchodzącej nowej wersji) .

## Roztwór

Rozwiązanie "znalazłem" działa w Apache2, Nie znam nginx, ale wyobrażam sobie, że jest coś podobnego. Na początku próbowałem mod _ security, ale to się skomplikowało i nie wyszło za dobrze. Rozwiązaniem jest użycie mod _ pipe. Jestem ekspertem w tej dziedzinie, więc podczas gdy definiowałem to, co chciałem osiągnąć, rozwiązanie, a także wyjaśnienie, zawdzięczam Claude 'owi. ai. ChatGPT udzieli ci tej samej odpowiedzi.

Krok 1. Upewnij się, że mod _ pipe jest zainstalowany i włączony. Ponieważ jak to zrobić różni się od OS, jest to coś, o co możesz zapytać swój ulubiony czatbot.

Krok 2. W odpowiednim pliku, który konfiguruje ssl dla apache2 (który ponownie różni się według OS) , na przykład może to być coś w rodzaju / etc / apache2 / sites- enable / ssl.conf, dodaj pod odpowiednią definicją VirtualHost (Uwaga: jeśli to skopiujesz, są tylko 4 linie, trzecia linia może być zapakowana, rozpakuj ją) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Punkt 3. Sprawdź poprawność konfiguracji: `sudo apache2ctl confixtest` 

Krok 4. Przywróć apache2: `sudo systemctl restart apache2` 

Punkt 5. Sprawdź swoje dzienniki, że nic nie jest zablokowane, że nie powinno być, i że odpowiednie prośby bez ograniczenia zwrócić 429 bez nigdy nie uderzył kota

## Wyjaśnienie

Dlaczego to działa i co to robi - oto wyjaśnienie z Claude.ai:

Linia 1 - `RewriteEngine On` 
Włącza przetwarzanie mod _ pipe dla tego zakresu. Bez niej poniższe dyrektywy RewriteCond / RewriteRule są po prostu ignorowane.

Linia 2 - `RewriteCond% &#123;QUERY _ STRING&#125; ^ $` 
Warunek, który musi być spełniony przed zastosowaniem poniższej zasady. `&#123;QUERY _ STRING&#125;` Czy wszystko po? w URL wniosku. ^ $to regex oznaczający "początek łańcucha natychmiast po nim koniec łańcucha" - czyli pusty ciąg. Tak więc warunek ten jest prawdziwy tylko wtedy, gdy nie ma żadnego łańcucha zapytań - nie ma wyrażenia subsetting / ograniczenie na żądanie.

Linia 3 - `RewriteCond% &#123;REQUEST _ URI&#125; ^ / erddap / (griddap |  tabledap ) / [^ /?] +\\. (? (:...) $) [A- Za- z0- 9 _] + $` 
Drugi warunek, sprawdzony `% &#123;WNIOSEK _ URI&#125;` - dosłowna ścieżka żądania, jak to wysłał klient, zawsze pełna ścieżka niezależnie od tego, gdzie w konfigu ta reguła mieszka (celowo wybrany ponad pozwolić RewriteRule wzór sam zrobić dopasowanie, ponieważ wzór dopasowania wewnątrz ` <Location> ` blok może zachowywać się niejednoznacznie - patrz uwaga poniżej) . Rozbijając regex:

 `^ / erddap /` - należy rozpocząć od / erddap /
 (griddap |  tabledap ) / - a następnie jeden z dwóch ERDDAP™ metody dostępu

 `[^ /?] +` - datasetID : jeden lub więcej znaków, które nie są / lub?

 `\\.` - dosłownie kropka

 ` (? (?: croissant | iso19115 _ 2 | ... | ncoJsonHeader) $) ` - negatywne spojrzenie: "tak długo, jak to, co następuje nie jest jednym z tych dokładnych plików Wpisz nazwy aż do końca łańcucha". To są typy plików ERDDAP™ może legalnie służyć bez ograniczeń (metadane, struktura, formularze itp.) - to przez to nie mogą być zablokowane.

 `[A- Za- z0- 9 _] + $` - faktyczny plik Rozszerzenie typu (litery, cyfry, podkreślenie) , wymagane do uruchomienia do końca łańcucha.
Więc ten warunek jest prawdziwy tylko wtedy, gdy ścieżka jest griddap / tabledap prośba o jakiś plik Typ, którego nie ma na liście bez ograniczeń.

Linia 4 - `RewriteRule ^ - [R = 429, L]` 
Sama zasada. Ponieważ oba powyższe warunki muszą być już prawdziwe, aby Apache nawet oszacował tę linię, wzór tutaj nie musi sprawdzać niczego innego - ^ po prostu pasuje do "początku łańcucha", co jest zawsze prawdą. - oznacza "nie przepisywać URL na cokolwiek innego" (Nigdzie nie przekierowujemy, tylko zwarcie.) . Flagi:

 `R = 429` - odpowiadać za pomocą HTTP redirect- class action z kodem statusu 429 (Zbyt wiele żądań) zamiast służyć prośbie.
L - "Last": zaprzestać przetwarzania jakichkolwiek dalszych przepisanych zasad, gdy ten zostanie uruchomiony.
Złóż razem: jeśli łańcuch zapytań jest pusty, a żądanie dotyczy griddap / tabledap plik Typ, który nie jest na bezpiecznej liście, natychmiast zwrócić 429 - nigdy nie kontaktować się z ERDDAP / Tomcat backend.

Dlaczego? `% &#123;WNIOSEK _ URI&#125;` zamiast pozwalać RewriteRule wzór dopasować ścieżkę bezpośrednio (warte włączenia jako notatka dla kolegów, ponieważ jest to nieoczywista część) : wewnątrz ` <Location> ` block, co goły wzór RewriteRule rzeczywiście jest dopasowany do może zachowywać się niekonsekwentnie w zależności od wersji Apache i kontekstu. Wyraźnie ciągnąc pełną ścieżkę przez RewriteCond `% &#123;WNIOSEK _ URI&#125;` sidestes tej dwuznaczności całkowicie - zawsze jest to dosłowna, kompletna ścieżka żądania, więc regex zachowuje się dokładnie tak, jak napisano niezależnie od tego, gdzie reguła jest zagnieżdżona.


Używam tego od kilku dni i wydaje się działać bardzo dobrze, blokuje to, co próbuję zablokować, a nie blokuje to, czego nie chcę blokować. I nasze ERDDAP™ stała się bardziej stabilna.
