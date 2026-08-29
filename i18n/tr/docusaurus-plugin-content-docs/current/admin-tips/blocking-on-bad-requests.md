# Bloking ERDDAP™ İstek içeriğine dayanarak, IP'ye bağlı değil

Bu içerik bir şeye dayanıyor [Roy Mendelssohn'dan gelen mesaj ERDDAP™ kullanıcılar grubu](https://groups.google.com/g/erddap/c/XcvPkoGtchg) .

## Sorun sorun

Eğer bizim gibiyseniz, veri taleplerinizi sizin için birçok bot görüyorsunuz ERDDAP™ Ve talepler genellikle kötü kodlanmış senaryolar tarafından yapılır gibi görünüyor, LLMs tarafından (Benim tahminim) Ya da insanlar tarafından. LLM'ler hakkında bir şey, onları tam olarak yapmak için ne yapmanız gerektiğidir, bu yüzden senaryo çek geri dönüş kodlarına sahip olmadıklarını ve farklı durumlarda ne yapacağını söylemiyorsanız, genellikle kod bu şeylerden herhangi biri yapmaz. Ve bunu denemeye devam etmesini söylerseniz, bu olacaktır. En azından gördüğümüz şey budur.

Ama bizim üzerinde bir sayı yapanlar ERDDAP™ Bunun gibi:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Bu gönderiler ERDDAP™ Hiçbir zaman toprak, çünkü, en iyi karar verebilirim, bu URL sadece tüm veri setini talep etmiyor, bu kaç tane terabay bilmiyorum, ancak bir parke dosyasına dönüştürülmek istiyor. İstekler IP'leri hareket etmekten daha kötü geliyor, bu yüzden IP'leri whack-a-mole'den daha kötüdür, hepsini asla engelleyemezsiniz.

Yani soru, istek içeriğine dayanarak nasıl engelleyebilirsiniz? Daha da ileri gitmeden önce, deneyim çökerse, farklı sebeplerden dolayı ne görüyoruz - sunucunuz için çok fazla zaman harcadım ve ayrıca tehlikeli yüksek hafıza kullanımı olduğunda haberdar oldum ve bir kazanın bu bildirimin ardından da yaygın bir denominatör olduğunu fark ettim. Bu yüzden herhangi bir eylem almadan önce bunu araştırma, ama bu size bu talepleri nasıl engelleyeceğinizi bir fikir verebilir.

Sadece bu yüzden açıkım, bir düşünebilirsiniz ERDDAP™ Talep:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

Bizim durumda, kazaların ortak denomi, belirli dosya türleri ile ızgara veya masa üstü talepleriydi, ancak kısıtlama yok. Peki soru, talepleri kısıtlama olmadan nasıl engelleyebilir? Bunun dışında o kadar basit değil, çünkü bir kısıtlamaya gerek olmayan bir dizi dosya türü var ve iyi niyetli olacağız, bu yüzden bunları engellemek istemiyoruz. Chris’le konuştuktan sonra ve hiç şüphem bir şey bıraktı, bir kısıtlama olmadan iyi davrandığı dosya türleri:

- .croissant
- .iso19115_2
- .iso19139_2007
- .iso19115_3_2016
-  .nc CFHeader
-  .nc CFMAHeader
- .das
- .dd
- .html
- .graph
- .subset
-  .nc Header
- .help
- .fgdc
- .iso19115
-  .nc OJsonHeader (Bu bir sonraki yeni sürümde olacak) .

## Çözüm çözümü

Çözüm "Ben bulundum" Apache2'de çalışıyor, nginx bilmiyorum ama benzer bir şey olduğunu hayal ediyorum. İlk başta mod_security denedim, ama bu çok karmaşıktı ve çok iyi çalışmadım. Çözüm mod_rewrite kullanmak. Şimdi bu konuda bir uzman değilim, bu yüzden ne yapmaya çalıştığımı tanımladığımda, çözüm, açıklamanın yanı sıra, Claude nedeniyle. ai. ChatGPT sizi temel olarak aynı cevapla tedarik edecektir.

1. Adım, Mode_rewrite'ın yüklenmesi ve etkinleştirilmesinden emin olun. Bunu nasıl yapılır OS ile değişir, bu en sevdiğiniz chatbot'u sorabileceğiniz bir şeydir.

2. Adım, apache2 için yapılandıran uygun dosyada (Hangisi yine OS tarafından değişir) Örneğin, uygun VirtualHost tanımı altında aşağıdakileri ekleyin //apache2/sites-tili/sl.conf gibi bir şey olabilir. (Bunu sadece 4 satır olduğunu kopyalasanız, üçüncü çizgi sarılı olabilir, unwrap it) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

3. Adım 3. yapılandırmanın geçerli olduğunu kontrol edin: `Sudo apache2ctl  configuretest` 

Adım Adım Adım Adım 4. Restart apache2: `Sudo sistemik yeniden başlar apache2` 

5. Adım 5. Girişlerinizi kontrol edin, hiçbir şey engellenmemelidir ve bir kısıtlama olmadan bu uygun istekler, tomcatcatla 429'u hiç vurmadan vurmadan 429'u geri döndürür.

## Açıklama

Neden bu iş ve bunu yapan şey - burada Claude.ai'den açıklama:

Line 1 – `Yeniden yazmaMühendis Onda` 
Mode_rewrite processing for this scope. Bu olmadan, Aşağıdaki RewriteCond/RewriteRule yönergeleri sadece göz ardı edilir.

Line 2 – `RewriteCond%&#123;QUERY_STRING&#125; ^$` 
Aşağıdaki kural uygulanmadan önce doğru olması gereken bir koşul. `%&#123;QUERY_STRING&#125;` Ne oldu? İstek URL'de. ^$ is a regex means "start of string immediately follow by end of string" - i.e., an empty string. Bu nedenle bu durum sadece sorgu dizesi olmadığı zaman doğrudur - istekte alt sıra / ifade yoktur.

Line 3 – `RewriteCond%&#123;REQUEST_URI&#125; ^/erddap / (network |  tabledap ) / [^/?]+\\. (?&#33; (?) $ $ $ $ $ $) [A-Za-z0-9_]+$` 
İkinci bir koşul, karşı kontrol `%&#123;REQUEST_URI&#125;` - istemcinin gönderdiği gibi gerçek istek yolu, her zaman bu kuralın ne olursa olsun tam yol (RewriteRule modelinin kendisini eşleştirme yapmasına izin vermek için kasıtlı olarak seçilir, çünkü desen içinde bir araya gelmek ` <Location> ` Blok belirsiz davranabilir - aşağıda not bakınız) . Regex'i yok edin:

 `^ /erddap /` - /erddap /
 (network |  tabledap ) / - ikisinden biri tarafından takip ERDDAP™ erişim yöntemleri

 `[^/?]+` - datasetID : Değil / veya olmayan bir veya daha fazla karakter?

 `\\.` - Bir gerçek bir dot

 ` (?&#33; (?:croissant | iso19115_2 | ... | ncoJsonHeader) $ $ $ $ $ $) ` - negatif bir bakış: “Bu kesin dosyadan biri olmadığı sürece Tip isimleri tüm yol dizenin sonuna kadar.” Bunlar dosyaTypes ERDDAP™ Yasal olarak kısıtlama olmaksızın hizmet edebilir (metadata, yapı, form sayfaları, vb.) - Bakahead onları bloke olmaktan dışlayan şeydir.

 `[A-Za-z0-9_]+$` - Gerçek dosya Type extension (mektuplar, sayılar,) Ancak dizenin sonuna kadar koşmak gerekir.
Yani bu durum sadece yol bir griddap / tabledap Bazı dosya için talep Güvenli olmayan listede olmayan Type that isn't on the safe- without-constraint list.

Line 4 – `RewriteRule ^ - [R=429,L]` 
Kuralın kendisi. Çünkü her iki koşul da Apache için bu çizgiyi bile değerlendirmek için gerçek olmalı, burada desen başka bir şeyi kontrol etmek zorunda değil - ^ "start of the string", which is always true. - " URL'yi farklı bir şeye yeniden yazma" anlamına gelir. (Her yere yönlendirmeyiz, sadece istek kısası) . Bayraklar:

 `R=429` — statüsü taşıyan HTTP yönlendirme sınıf eylemi ile yanıt 429 ("Too Many Requests") İsteke hizmet etmek yerine.
L – "Son": Bu yangınlar bir kez daha yeniden yazma kurallarını işlemeyi bırakın.
Birlikte koyun: sorgu dizesi boşsa ve istek bir griddap / tabledap Dosya dosyası Güvenli kısıtlanmış listede olmayan tip, hemen 429 geri döner - hiç temas olmadan ERDDAP / Tomcat backend.

Neden Neden Neden Neden Neden? `%&#123;REQUEST_URI&#125;` Yeniden yazmaRule desenine izin vermek yerine doğrudan yolu eşleştirin (meslektaşları için bir not olarak da değer, çünkü bu önemsiz kısım) : içeride ` <Location> ` Blok, çıplak RewriteRule deseni aslında Apache versiyonuna ve bağlamına bağlı olarak hareket edebilir. Explicitly the full road via RewriteCond `%&#123;REQUEST_URI&#125;` Tamamen belirsizliğe sahip olan yan adımlar – her zaman gerçek, tam istek yolu, bu yüzden regex tam olarak kuralın nested olduğuna bakılmaksızın yazılır.


Bunu birkaç gün boyunca kullanıyorum ve çok iyi çalışıyor gibi görünüyor, blok yapmaya çalıştığımı engelliyor ve blok yapmak istemediğimi engellemem. Ve bizim ERDDAP™ Çok daha istikrarlı hale geldi.
