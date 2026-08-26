# BAP (Bilimsel Araştırma Projeleri) Yönetim Sistemi'nde Kimlik Doğrulamasız SQL Enjeksiyonu Zafiyeti

## Genel Bakış

BAP Yönetim Sistemi, Türkiye'deki yükseköğretim kurumlarında bilimsel araştırma projelerinin başvuru, değerlendirme, bütçe ve takip süreçlerinin yürütülmesi amacıyla kullanılan, PHP tabanlı ticari bir web uygulamasıdır. Uygulamanın kamuya açık proje listeleme ekranı, sunucu taraflı DataTables altyapısı üzerinden çalışmakta ve istemciden gelen filtre koşullarını JSON biçiminde kabul etmektedir.

Bu filtre yapısında **`IN` operatörüyle** kullanılan sütunların değerleri, herhangi bir parametrik sorgu (prepared statement) kullanılmadan ve tip/biçim doğrulamasından geçirilmeden doğrudan SQL sorgusunun `IN (...)` listesine birleştirilmektedir. Bu durum, kimliği doğrulanmamış bir saldırganın sorgunun `WHERE` koşulunu değiştirmesine ve veritabanından keyfi bilgi çıkarmasına imkân tanımaktadır.

Zafiyetli uç nokta, adında da açıkça görüldüğü üzere **misafir (guest) bağlamında** çalışmakta; oturum açma, çerez veya herhangi bir kimlik doğrulama unsuru gerektirmemektedir.

Aynı zafiyet, **birbirinden bağımsız iki farklı kurumun kurulumunda** doğrulanmıştır. Söz konusu kurulumların işletim sistemi, veritabanı sürümü ve dağıtım mimarisi tamamen farklıdır (bir kurulum Windows üzerinde XAMPP + MariaDB 10.4.x, diğeri Ubuntu 24.04 üzerinde konteynerize MariaDB 10.11.x). Zafiyetin her iki ortamda da birebir aynı biçimde bulunması, sorunun altyapı veya sürüm kaynaklı bir yapılandırma hatası değil, **ürünün uygulama kodundan kaynaklanan bir tedarikçi/ürün seviyesi zafiyeti** olduğunu göstermektedir.

## Etki

Kimliği doğrulanmamış herhangi bir saldırgan, tek bir HTTP isteğiyle veritabanı sorgusunun mantığını değiştirebilmekte ve kör (blind) boolean tabanlı çıkarım yöntemiyle veritabanı içeriğini okuyabilmektedir:

- **Veritabanı içeriğinin tamamının okunması.** Sorgu, saldırgan denetimindeki bir koşulun doğru/yanlış olmasına göre farklı kayıt sayısı döndürmekte ve bu davranış güvenilir bir çıkarım orakılı (oracle) oluşturmaktadır. Bu yolla veritabanı sürümü, veritabanı adı, bağlanan kullanıcı, sunucu adı, şema sayısı ve tablo sayısı gibi bilgiler doğrulanmıştır. Aynı teknikle proje, personel ve bütçe tablolarındaki kayıtların içeriği de karakter karakter çıkarılabilir.
- **Aşırı yetkiyle çalışan veritabanı bağlantısı.** Her iki kurulumda da uygulamanın veritabanına **`root@localhost`** kullanıcısıyla bağlandığı tespit edilmiştir. Bu, tek bir kurumun yapılandırma hatası değil, ürünün kurulum deseninden kaynaklanan ikinci ve bağımsız bir zayıflıktır (CWE-250). Sonuç olarak enjeksiyon yalnızca uygulamanın kendi veritabanıyla sınırlı kalmamakta; aynı veritabanı örneğindeki **diğer tüm veritabanları** da erişilebilir hâle gelmektedir (test edilen kurulumlarda 6 şema).
- **Sunucu dosya sisteminden keyfi dosya okuma.** Test edilen kurulumlardan birinde veritabanı kullanıcısının `FILE` yetkisine sahip olduğu ve dosya okuma fonksiyonunun fiilen çalıştığı doğrulanmıştır. İşletim sistemi yapılandırma dosyalarının ve web kökündeki uygulama kaynak kodunun okunabildiği kanıtlanmıştır. Bu, veritabanı sınırının dışına taşan bir gizlilik ihlalidir.
- **Kod çalıştırmaya (RCE) giden gerçekçi zincir.** Enjeksiyon noktasının kendisi doğrudan kod çalıştırmaya izin vermemektedir (aşağıdaki *Sınırlayıcı Etkenler* bölümüne bakınız). Ancak dosya okuma yeteneğiyle uygulamanın veritabanı yapılandırma dosyası okunabildiğinde açık metin kimlik bilgileri elde edilebilir; veritabanı portunun dışarıya açık olduğu veya saldırganın iç ağa eriştiği kurulumlarda bu kimlik bilgileri gerçek bir SQL konsolundan dosya yazma veya UDF yükleme yoluyla sunucuda kod çalıştırmaya dönüştürülebilir. Eklenti dizini ve veri dizini yolları da aynı enjeksiyonla elde edilebildiğinden bu zincirin ön koşulları saldırgan tarafından hazır biçimde toplanabilmektedir.

Zafiyetin istismarı için özel bir araç, ayrıcalıklı hesap veya kullanıcı etkileşimi gerekmemektedir.

## Teknik Ayrıntı

Zafiyetli uç nokta, sunucu taraflı DataTables isteklerini karşılayan misafir bağlamındaki proje listeleme adresidir:

```http
POST /?act=guest&act2=projeler&durum=devam&grid=1&table_id=Proje HTTP/1.1
Host: bap.<kurum>.edu.tr
Content-Type: application/x-www-form-urlencoded

data=<DataTables filtre ve sütun tanımlarını içeren JSON dizisi>
```

İstek gövdesindeki JSON yapısı, hangi sütuna hangi karşılaştırma operatörünün uygulanacağını belirten bir operatör haritası (`sConditionOperators`) ve bu sütunlara ait filtre değerlerini taşır. **`IN` operatörüne bağlanan sütunların** değerleri bir dizi olarak gönderilmekte ve bu dizi elemanları SQL sorgusundaki `IN (...)` listesine kaçış uygulanmadan yerleştirilmektedir. Parantez dengesi saldırgan tarafından korunduğu sürece, koşul ifadesinin tamamı değiştirilebilmektedir.

Zafiyet, doğrulanmış sütun üzerinden aşağıdaki davranışla kanıtlanmıştır *(sayısal değerler kurulumdan kurulumuma değişmekte, ancak doğru/yanlış ayrımı her iki kurulumda da net biçimde gözlenmektedir)*:

| Gönderilen koşul | Dönen kayıt sayısı (Kurulum A) | Dönen kayıt sayısı (Kurulum B) |Dönen kayıt sayısı (Kurulum c) |
|---|---|---|---|
| Her zaman **doğru** olan enjekte edilmiş koşul | 331 | 88 | 149 |
| Her zaman **yanlış** olan enjekte edilmiş koşul | 19 | 28 | 0 | 


Doğru ve yanlış koşulların birbirinden ayrılabilir iki farklı kayıt sayısı üretmesi, kör boolean tabanlı çıkarım için yeterli ve kararlı bir orakıl oluşturmaktadır. Bu orakıl kullanılarak her iki kurulumda da veritabanı sürümü, veritabanı adı, bağlanan kullanıcı (`root@localhost`), sunucu makine adı, şema ve tablo sayıları doğrulanmıştır.

Veritabanı yönetim sistemi, sunucuya özgü fonksiyonların davranış farkıyla **MySQL/MariaDB** olarak tespit edilmiştir; diğer veritabanı sistemlerine ait eşdeğer fonksiyonlar sunucu hatası üretmektedir.

### Sınırlayıcı Etkenler

Uygulamada eksik ve tutarsız da olsa bazı savunma unsurları bulunmakta; bunlar zafiyeti ortadan kaldırmamakta, yalnızca istismar tekniğini biçimlendirmektedir:

- **Tek tırnak karakteri kaçırılmaktadır.** Bu nedenle enjeksiyonda dizgi sabiti kullanılamamakta; saldırgan karşılaştırmalarını sayısal fonksiyonlar ve onaltılık (hex) sabitler üzerinden kurmaktadır. Bu, veri çıkarımını engellememekte yalnızca istek sayısını artırmaktadır.
- **Yorum karakterleri sunucu hatası üretmektedir.** Sorgunun kalanı kesilemediğinden saldırganın parantez dengesini koruması gerekmektedir; bu da salt bir zorluk unsurudur.
- **Yığın sorgu (stacked query) desteklenmemektedir.** İkinci bir SQL ifadesi çalıştırılamamaktadır.
- Enjeksiyon noktası `WHERE` koşulu içindeki bir alt bağlam olduğundan, dosya yazma ifadesi bu noktadan çağrılamamaktadır.

Bu dört etken birlikte, **bu HTTP parametresi üzerinden tek adımda kod çalıştırmayı** engellemektedir. Ancak veri okuma, dosya okuma ve ayrıcalıklı veritabanı erişimi tamamen mümkündür; dolayısıyla zafiyetin şiddeti bu etkenlerle azalmamaktadır.

### Aynı İstekte Denenip Zafiyetli Bulunmayan Parametreler

Zafiyetin kapsamının doğru belirlenmesi açısından, aynı istek gövdesindeki diğer parametrelerin güvenli olduğu tespit edilmiştir: serbest metin arama alanı parametrik sorgu kullanmakta; görüntülenecek sütun listeleri bilinen sütun adlarına karşı beyaz listeye tabi tutulmakta; sıralama sütunu ve yönü enjekte edilememekte; operatör adları sabit bir haritadan okunmaktadır. `LIKE` operatörüne bağlanan sütun değerleri de kaçırılmaktadır.

**Zafiyet münhasıran `IN` operatörüne bağlanan sütunların değerlerindedir.** Doğrulama `tur` sütunu üzerinden yapılmış olmakla birlikte, aynı operatöre bağlanan diğer sütunlar (`fakulte`, `bolum`, `bilimDali`, `Kategori`) da aynı kod yolunu kullandığından etkilenmiş kabul edilmelidir.

## İlişkili Zayıflıklar

- **CWE-89** — SQL Komutunda Kullanılan Özel Elemanların Uygun Olmayan Biçimde Etkisizleştirilmesi ("SQL Injection")
- **CWE-306** — Kritik İşlev İçin Kimlik Doğrulamasının Eksikliği
- **CWE-250** — Gereğinden Fazla Yetkiyle Çalıştırma *(uygulamanın veritabanına `root` kullanıcısıyla bağlanması)*
- **CWE-20** — Uygun Olmayan Girdi Doğrulaması

## Şiddet

**Kritik — CVSS 3.1 Temel Puan 9.8** (`AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`)

Puanlama; zafiyetin ağ üzerinden (`AV:N`), düşük karmaşıklıkla (`AC:L`), **hiçbir ayrıcalık** (`PR:N`) ve **kullanıcı etkileşimi** (`UI:N`) gerektirmeden istismar edilebilmesi esas alınarak yapılmıştır.

Test kapsamında fiilen gösterilen etki **gizliliğin tamamen ihlali** (veritabanı içeriğinin ve sunucu dosyalarının okunabilmesi) olmuştur; bütünlük ve erişilebilirlik etkileri, uygulamanın veritabanına `root` kullanıcısıyla bağlanması nedeniyle mevcut yetki düzeyinden türetilmiştir. Yalnızca doğrulanmış gizlilik etkisi esas alınarak daha muhafazakâr bir puanlama tercih edilmesi hâlinde `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` (**7.5 — Yüksek**) kullanılabilir.

## Etkilenen Sürümler

1.9.25

## Etkilenen Bileşen

Sunucu taraflı DataTables isteklerini karşılayan misafir bağlamındaki proje listeleme uç noktasında, `IN` operatörüne bağlanan sütun filtrelerinin SQL sorgusuna dönüştürüldüğü kod yolu.

## Azaltıcı Önlemler

Üretici düzeltmesi yayımlanana kadar işletmecilerin şunları uygulaması önerilir:

1. **Acil önlem olarak**, misafir bağlamındaki proje listeleme uç noktasına erişim kısıtlanmalı veya bu uç nokta kimlik doğrulaması arkasına alınmalıdır. Uygulama kamuya açık bir listeleme sunuyorsa, filtreleme özelliği geçici olarak devre dışı bırakılmalıdır.
2. Ters vekil sunucu (reverse proxy) veya WAF katmanında, ilgili isteğin gövdesindeki filtre değerleri denetlenmeli; sayısal olmayan, parantez, boşluk veya SQL anahtar sözcüğü içeren değerler reddedilmelidir. Bu önlemin bir kara liste olduğu ve atlatılabileceği unutulmamalı, kalıcı çözümün yerine geçtiği varsayılmamalıdır.
3. **Veritabanı bağlantı kullanıcısı derhal düşürülmelidir.** Uygulama `root` yerine, yalnızca kendi veritabanı üzerinde `SELECT`/`INSERT`/`UPDATE`/`DELETE` yetkisine sahip, **`FILE` yetkisi bulunmayan** ayrılmış bir kullanıcıyla bağlanmalıdır. Bu tek adım, dosya okuma ve diğer veritabanlarına erişim etkilerini ortadan kaldırır ve zafiyetin şiddetini belirgin biçimde düşürür.
4. Veritabanı sunucusunda dosya okuma/yazma işlemleri, ilgili yapılandırma seçeneğiyle boş veya erişilemeyen bir dizine sınırlandırılmalıdır.
5. Veritabanı portu ve varsa veritabanı yönetim arayüzleri internete kapalı tutulmalı, yalnızca yerel arayüzden erişilebilir olmalıdır.
6. Uygulama günlükleri ve web sunucusu günlükleri, ilgili uç noktaya gelen olağandışı yoğunluktaki istekler açısından geriye dönük olarak incelenmelidir; kör çıkarım saldırıları binlerce benzer istek üretir ve günlüklerde belirgin bir iz bırakır.
7. Erişim kayıtlarında istismar izine rastlanması hâlinde, `root` erişimiyle ulaşılabilen tüm veritabanlarının ihlal edilmiş kabul edilmesi ve ilgili mevzuat kapsamında değerlendirme yapılması gerekir.

Kalıcı çözüm için üreticinin şunları yapması gerekmektedir:

- Filtre değerlerinin tamamı, dizi elemanları dâhil olmak üzere **parametrik sorgu (prepared statement)** ile bağlanmalıdır. `IN` listeleri için eleman sayısı kadar yer tutucu üretilmeli, değerler asla dizgi birleştirmeyle sorguya eklenmemelidir.
- Filtre değerleri, kaçış uygulanmadan önce **tip düzeyinde doğrulanmalıdır**; sayısal kimlik bekleyen sütunlarda sayısal olmayan değerler kabul edilmemeli ve istek reddedilmelidir.
- Kaçış (escaping) tabanlı savunmaya güvenilmemelidir. Mevcut kodda tek tırnağın kaçırılıyor olması, sayısal bağlamdaki enjeksiyonu engellememektedir; sayısal bağlamda tek geçerli savunma parametrik sorgu ve tip doğrulamasıdır.
- Aynı kod yolunu kullanan **tüm operatörler ve tüm sütunlar** (`IN`, `BETWEEN`, `LIKE` vb.) gözden geçirilmeli, düzeltme tek bir sütunla sınırlı tutulmamalıdır.
- Ürün kurulum dokümantasyonu, veritabanına `root` kullanıcısıyla bağlanmayı öngörmeyecek biçimde güncellenmeli; kurulum betikleri en az ayrıcalık ilkesine uygun ayrılmış bir veritabanı kullanıcısı oluşturmalıdır.
- Hata durumlarında istemciye dönen yanıtlar gözden geçirilmelidir; mevcut kurulumlarda yığın izi (stack trace) sızdırılmıyor olması olumlu bir uygulamadır ve korunmalıdır.

## Sorumlu Açıklama Notu

Bu doküman, zafiyetin tetiklenmesine ait somut yük (payload) ve istismar kodlarını **bilinçli olarak içermemektedir**; yalnızca zafiyetin sınıfı, konumu ve etkisi tanımlanmıştır. Teknik kanıtlar, koordineli açıklama süreci kapsamında üretici ve ilgili ulusal siber olaylara müdahale ekibiyle doğrudan paylaşılmıştır.

Testler, ilgili kurumların yazılı izniyle yürütülen yetkili sızma testleri kapsamında gerçekleştirilmiştir. Veri çıkarımı, zafiyetin varlığını kanıtlamaya yetecek asgari düzeyde (sürüm, veritabanı adı, kullanıcı, sayaç bilgileri) tutulmuş; kişisel veri içeren tabloların içeriği çekilmemiştir.

## CVE Kimliği

Henüz atanmadı — CVE talebi beklemede

## Bulan

Hasan Hüseyin UYAR – Netlore Security

## Açıklama Takvimi

| Tarih | Olay |
|---|---|
| 2026-08-23 | Zafiyet, yetkili bir sızma testi sırasında ilk kurulumda tespit edildi ve doğrulandı |
| 2026-08-24 | Aynı zafiyet, bağımsız ikinci bir kurumun kurulumunda doğrulandı; ürün seviyesi zafiyet olarak sınıflandırıldı |
