Genel Mimari Katmanları
Client (Tarayıcı/İstemci)
Kullanıcı, bir web tarayıcısı veya HTTP istemcisi ile sisteme istek gönderir. Protokol olarak HTTP veya HTTPS kullanılır.

SafeLine Katmanı (Mavi)
SafeLine, trafiği ilk karşılayan katmandır ve şu alt bileşenlere sahiptir:

safeline-tengine: Tengine (Nginx türevi) tabanlı proxy. Gelen istekleri alır, ilk WAF kontrollerini gerçekleştirir ve iç servislere yönlendirir.

safeline-detector: safeline-tengine’den gelen isteği detaylı kural setleriyle (SQLi, XSS vb.) değerlendirir. Tespit edilen saldırı girişimlerine göre yanıt döner veya isteği ileri gönderir.

safeline-redis: Detector’ın anlık kararlar için kullandığı hızlı veri önbelleği.

safeline-mgt UI: Yöneticilerin politika ve kural setlerini tanımladığı, istatistikleri görüntülediği web arayüzü.

safeline-pg (PostgreSQL): safeline-mgt ve detector bileşenlerinin kalıcı veri (kayıt, kural, kullanıcı, istatistik) saklamak için kullandığı veritabanı.


Caddy-Coraza Katmanı (Bej)
SafeLine’ın izin verdiği veya geçirdiği istekleri ikinci bir WAF katmanı olan Caddy-Coraza’ya yönlendirir:

caddy-coraza: Caddy web sunucusu üzerine entegre edilmiş Coraza WAF motoru.

OWASP CRS Kuralları: OWASP Core Rule Set (SQLi, XSS, RCE gibi) kuralları burada yüklenir ve uygulanır.

Statik İçerik (/site): İstekler eğer engellenmezse, /site altında barındırılan index.html vb. statik dosyaları sunar.

Coraza Logları: Engellenen veya şüpheli bulunan isteklerin detaylı kayıtlarını tutar.

2. Trafik Akışı Adımları
İlk İstek

Kullanıcı http://localhost veya https://localhost üzerinden bir istek gönderir.

SafeLine-tengine

Bu istek önce safeline-tengine’e ulaşır.

safeline-tengine, safeline-detector’a isteği yönlendirerek temel WAF kurallarını uygulatır.

SafeLine-detector & redis

safeline-detector, kural setlerini (Redis üzerinden konfigüre edilmiş hızlı erişim ile) kontrol eder.

Eğer kural ihlali yoksa istek “Allow” kararıyla geri döner; ihlal varsa safeline-tengine 403 Forbidden ile cevap verir.

Yönlendirme

“Allow” kararı aldığında safeline-tengine, isteği iç ağdaki Caddy-Coraza servisine iletir.

Not: WSL/Docker ortamında host.docker.internal:8080 adresi ile Caddy’nin çalıştığı porta yönlendirme yapılır.

Caddy-Coraza İşleme

Caddy, gelen isteği Coraza WAF ile birlikte işler.

Coraza, OWASP CRS veya custom ModSecurity kurallarını çalıştırır.

Eğer Coraza da isteği onaylarsa, Caddy statik içerik klasöründen (örn. index.html) yanıtı sunar.

Aksi halde Coraza “403 Forbidden” döner ve bu olay Coraza loglarında kaydedilir.

Yanıt

Sonuç olarak, ya safeline katmanında ya da Coraza katmanında engellenmiş bir 403 yanıtı, ya da başarılı bir 200 OK + içerik yanıtı istemciye geri iletilir.

3. Neden İki Katman?
Defence in Depth yaklaşımıyla, bir katmandan kaçabilen saldırı vektörünü diğer katman yakalar.

SafeLine, GUI üzerinden politika yönetimi, hızlı ön filtreleme ve gerçek zamanlı istatistikler sunar.

Coraza, OWASP CRS’in zengin kural seti ve ModSecurity uyumluluğu ile derinlemesine inceleme yapar.

Bu şekilde hem yüksek esneklikte politika yönetimi, hem de kapsamlı saldırı tespiti sağlanmış olur.
