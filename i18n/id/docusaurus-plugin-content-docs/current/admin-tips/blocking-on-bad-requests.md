# Login ERDDAP™ permintaan berdasarkan konten permintaan, tidak didasarkan pada IP

Konten ini didasarkan pada [Pesan dari Mendelssohn ke ERDDAP™ grup pengguna](https://groups.google.com/g/erddap/c/XcvPkoGtchg) Sitemap

## Masalah

Jika Anda seperti kami, Anda melihat banyak bot yang membuat permintaan data Anda ERDDAP™ , dan permintaan sering tampak seperti mereka dilakukan oleh skrip berkode miskin, apakah oleh LLMs (Sitemap) atau oleh manusia. Satu hal tentang LLMs adalah mereka akan melakukan persis apa yang Anda minta mereka untuk melakukannya, jadi jika Anda tidak memberi tahu mereka untuk memiliki script memeriksa kode pengembalian, dan tidak memberitahu mereka apa yang harus dilakukan dalam kasus yang berbeda, maka biasanya kode tidak akan melakukan hal-hal tersebut. Dan jika Anda memberitahunya untuk terus mencoba, itu akan. Meme it Apa yang kita lihat setidaknya. Meme it

Tapi orang-orang yang melakukan sejumlah di kami Meme it ERDDAP™ seperti ini:

 https://coastwatch.pfeg.noaa.gov/erddap/jplMURSST41.parquetWMeta
 

Sitemap ERDDAP™ ke tanah tidak pernah, karena, yang terbaik yang bisa saya tentukan, bahwa URL ini tidak hanya meminta seluruh dataset, yang saya tidak tahu berapa banyak terabyte, tetapi ingin dikonversi ke file parket. Worse permintaan datang dari IP bergerak, sehingga memblokir IP lebih buruk daripada whack-a-mole, Anda tidak akan pernah mendapatkan mereka semua diblokir.

Jadi pertanyaan adalah bagaimana Anda dapat memblokir berdasarkan konten permintaan? Sebelum lebih lanjut, jika Anda mengalami kecelakaan itu mungkin dari penyebab yang berbeda maka apa yang kita lihat - Saya menghabiskan banyak waktu yang melalui log dan juga saya mendapatkan notifikasi ketika ada penggunaan memori tinggi berbahaya, dan dicatat ketika kecelakaan mengikuti pemberitahuan itu, dan juga memperhatikan apa denominator umum dalam kecelakaan ini, yang mungkin tidak kasus untuk server Anda. Jadi penelitian ini sebelum mengambil tindakan apa pun, tetapi ini mungkin memberi Anda beberapa ide bagaimana memblokir permintaan ini.

Jadi saya jelas, Anda dapat memikirkan Meme it ERDDAP™ permintaan:

 https://baseURL/erddap/method/datasetID.filetype?constraint
 

Dalam kasus kami denominator umum dari kecelakaan di griddap atau permintaan meja dengan jenis file tertentu tetapi tidak ada batasan. Jadi pertanyaan adalah cara memblokir permintaan tanpa batasan? Kecuali itu tidak sederhana, karena ada sejumlah jenis file yang tidak perlu hambatan dan akan berperilaku dengan baik, jadi kita tidak ingin menghalangi mereka. Setelah berbicara ke Chris, dan tidak ada keraguan saya telah meninggalkan sesuatu, jenis file yang berperilaku dengan baik tanpa batasan adalah:

- Login
- .iso19115_2
- .iso19139_2007
- .iso19115_3_2016
-  .nc Login
-  .nc Login
- Login
- Login
- Login
- Login
- Login
-  .nc Login
- Login
- Login
- .iso19115
-  .nc Login (satu ini akan di rilis baru mendatang) Sitemap

## Login

Solusi "Saya menemukan" bekerja di Apache2, saya tidak tahu nginx tetapi saya membayangkan ada sesuatu yang mirip. Pada awalnya saya mencoba mod_security, tapi itu terlalu rumit dan tidak bekerja dengan baik. Solusinya adalah menggunakan mod_rewrite. Sekarang saya apa pun tetapi seorang ahli di ini, jadi sementara saya mendefinisikan apa yang saya mencoba untuk menyelesaikan, solusi, serta penjelasan, karena Claude. Login ChatGPT akan menyediakan jawaban yang sama pada dasarnya.

Langkah 1. Pastikan mod_rewrite dipasang dan diaktifkan. Karena bagaimana melakukan ini bervariasi dengan OS, ini adalah sesuatu yang dapat Anda tanyakan chatbot favorit Anda.

Langkah 2. Dalam file yang tepat yang mengkonfigurasi sl untuk apache2 (yang lagi bervariasi oleh OS) misalnya mungkin sesuatu seperti /etc/apache2/sites-enabled/ssl.conf, tambahkan berikut di bawah definisi VirtualHost yang tepat (catatan jika Anda menyalin ini hanya ada 4 baris, garis ketiga dapat dibungkus, tidak membungkusnya) 

```
RewriteEngine On
RewriteCond %{QUERY_STRING} ^$
RewriteCond %{REQUEST_URI} ^/erddap/(griddap|tabledap)/[^/?]+\\.(?!(?:croissant|iso19115_2|iso19139_2007|iso19115_3_2016|ncCFHeader|ncCFMAHeader|das|dds|html|graph|subset|ncHeader|help|fgdc|iso19115|ncoJsonHeader)$)[A-Za-z0-9_]+$
RewriteRule ^ - [R=429,L]
```

Langkah 3. Periksa bahwa konfigurasi berlaku: `sudo apache2ctl configtest` 

Login 4. Restart apache2: `sudo sistemctl restart apache2` 

Langkah 5. Periksa log Anda bahwa tidak ada yang diblokir yang seharusnya tidak, dan permintaan yang tepat tanpa batasan mengembalikan 429 tanpa pernah memukul jari Anda

## Login

Mengapa pekerjaan ini dan apa yang dilakukan ini - ini adalah penjelasan dari Claude.ai:

Promo `Login Sitemap` 
Nyalakan pemrosesan mod_rewrite untuk ruang lingkup ini. Tanpa itu, arahan RewriteCond/RewriteRule di bawah ini hanya diabaikan.

Garis 2 — `RewriteCond %&#123;QUERY_STRING&#125; ^$` 
Kondisi yang harus benar sebelum aturan di bawah berlaku. `Sitemap` Apakah semuanya setelah? di URL permintaan. ^$ adalah makna regex "start string segera diikuti oleh akhir string" — yaitu, string kosong. Jadi kondisi ini benar hanya ketika tidak ada string query sama sekali — tidak ada ekspresi subsetting/kontraint atas permintaan.

Login `RewriteCond %&#123;REQUEST_URI&#125; ^/erddap/ (Login |  tabledap ) /[^/?]+\\. (Sitemap (Sitemap) Sitemap) [A-Za-z0-9_]+$` 
Kondisi kedua, diperiksa terhadap `Facebook Twitter Google Plus Pinterest Email` - jalur permintaan literal sebagai klien mengirimkannya, selalu jalan penuh terlepas dari di mana di mengkonfigurasi kehidupan aturan ini (sengaja memilih untuk membiarkan pola RewriteRule itu sendiri melakukan pencocokan, karena pola-matching di dalam ` <Location> ` blok dapat berperilaku ambiguously - lihat catatan di bawah ini) Sitemap Breaking turun regex:

 `Sitemap` — harus dimulai dengan /erddap/
 (Login |  tabledap ) / — diikuti oleh salah satu dari dua ERDDAP™ metode akses

 `[^/?]+` Login datasetID : satu atau lebih karakter yang tidak / atau?

 `Login` - titik literal

 ` (Sitemap (Login | iso19115_2 | Login | Login) Sitemap) ` — lookahead negatif: "selama apa yang berikut bukan salah satu file yang tepat Jenis nama semua cara untuk akhir string. " Ini adalah fileTypes ERDDAP™ dapat secara sah melayani tanpa batasan (metadata, struktur, halaman bentuk, dll.) - lookahead adalah apa yang tidak termasuk mereka dari diblokir.

 `[A-Za-z0-9_]+$` - file yang sebenarnya Jenis ekstensi (huruf, digit, underscore) Diperlukan untuk menjalankan ujung string.
Jadi kondisi ini benar hanya ketika jalan adalah griddap / tabledap permintaan untuk beberapa file Jenis yang tidak ada di daftar tanpa hambatan yang aman.

Garis 4 — `RewriteRule ^ - [R=429,L]` 
Aturan itu sendiri. Karena kedua kondisi di atas harus sudah benar untuk Apache bahkan mengevaluasi garis ini, pola di sini tidak perlu memeriksa apa pun — ^ hanya pertandingan "start string," yang selalu benar. - berarti "tidak menulis ulang URL ke apa pun yang berbeda" (kami tidak mengarahkan ke mana saja, hanya hubung singkat permintaan) Sitemap Bendera:

 `G=429` - menanggapi dengan tindakan langsung-klaim HTTP yang membawa kode status 429 ("Too Banyak Permintaan") bukan melayani permintaan.
L — "Last": berhenti memproses aturan menulis ulang lebih lanjut setelah satu kebakaran ini.
Masukkan bersama: jika string query kosong, dan permintaannya adalah untuk griddap / tabledap Login Jenis yang tidak ada di daftar yang aman, segera kembali 429 - tanpa pernah menghubungi ERDDAP Login

Sitemap `Facebook Twitter Google Plus Pinterest Email` bukan membiarkan pola RewriteRule cocok jalan secara langsung (layak termasuk sebagai catatan untuk rekan-rekan, karena itu bagian non-obvious) : di dalam ` <Location> ` blok, apa pola RewriteRule telanjang sebenarnya akan cocok melawan dapat berperilaku secara tidak konsisten tergantung pada versi Apache dan konteks. Jelas menarik jalan penuh melalui RewriteCond `Facebook Twitter Google Plus Pinterest Email` langkah samping bahwa ambiguitas sepenuhnya - itu selalu literal, jalur permintaan lengkap, sehingga regex berperilaku persis seperti yang ditulis terlepas dari di mana aturan bersarang.


Saya telah menggunakan ini selama beberapa hari sekarang dan tampaknya bekerja dengan sangat baik, itu memblokir apa yang saya mencoba untuk memblokir dan tidak menghalangi apa yang tidak ingin saya blok. Kami ERDDAP™ telah menjadi lebih stabil.
