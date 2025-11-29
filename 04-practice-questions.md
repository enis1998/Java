# 4) SORULAN GERÇEK PRATİK SORULAR – Detaylı Açıklamalar

Bu doküman, Java/Spring Boot geliştiricilerine sık sorulan **gerçek hayat** senaryolarını, temelden ve örneklerle açıklamak için hazırlanmıştır.

Ele aldığımız başlıklar:

1. Bir kullanıcı login olduğunda JWT nasıl üretilir?
2. Kart numarası için maskleme nasıl yapılır?
3. Rate limiting yapısını nasıl kurarsın?
4. Microservice communication (sync/async, circuit breaker)
5. Money transfer service nasıl tasarlanır? (idempotent, double spending, locking)

---

## 1) Bir kullanıcı login olduğunda JWT nasıl üretirsin?

### 1.1. JWT nedir, ne işe yarar?

**JWT (JSON Web Token)**, kullanıcının kimliğini ve bazı ek bilgileri (roller, yetkiler vb.) içeren, **imzalanmış** bir token formatıdır.

- Sunucu, kullanıcı doğru şekilde login olduğunda bir JWT üretir.
- Client (mobil uygulama, web front-end) bu token’ı alır ve sonraki isteklerde `Authorization: Bearer <token>` header’ında gönderir.
- Sunucu her istekte bu token’ı **verify** edip kullanıcının:
  - Kim olduğunu,
  - Hangi rollere sahip olduğunu
  anlar.

Avantajı:

- Sunucu, session’ı memory’de tutmak zorunda kalmaz (stateless).
- Özellikle microservice mimarilerde, farklı servisler JWT içindeki bilgilere bakarak authorization yapabilir.

---

### 1.2. Login akışı (yüksek seviyeli)

1. Kullanıcı `username` ve `password` ile `/login` endpoint’ine gelir.
2. Backend:
   - Username’i DB’den bulur,
   - Password’ü kontrol eder (hash karşılaştırması),
   - Başarılıysa, bir JWT üretir.

3. JWT’nin içinde genelde şunlar olur:
   - `sub` (subject) → username veya user id
   - `roles` → kullanıcının rolleri (ROLE_USER, ROLE_ADMIN…)
   - `iat` (issued at) → oluşturulma zamanı
   - `exp` (expiration) → token’ın geçerlilik bitiş zamanı

4. Backend bu token’ı imzalar (**sign** eder):
   - Secret key (HMAC) veya private key (RS256) kullanılarak.
   - İmzalı olduğu için client token’ı değiştirse bile sunucu verify ederken bunu fark eder.

5. Client, aldığı token’ı localStorage/cookie/memory gibi bir yerde tutar ve sonraki isteklerde gönderir.

Özet akış:

```text
username + password → doğrula → (username + roles + exp) → sign → JWT token
```

---

### 1.3. Basit bir JWT üretim örneği (Java – kavramsal)

```java
String token = Jwts.builder()
    .setSubject(username)                // sub claim
    .claim("roles", userRoles)           // custom claim
    .setIssuedAt(new Date())             // iat
    .setExpiration(expirationDate)       // exp
    .signWith(SignatureAlgorithm.HS256, secretKey) // sign
    .compact();
```

Burada önemli noktalar:

- `secretKey` sunucuda güvenli bir şekilde saklanmalı (config’de, environment variable’da vs.).
- `expirationDate` ile token’ın ne kadar süre geçerli olacağını belirlersin (örnek: 15 dakika).

---

### 1.4. Access token & Refresh token ilişkisi

Bankalarda bu konu **özellikle sorulur**:  
> “Refresh token’ı nasıl yönetirsiniz?”

**Access token**:

- Daha kısa süreli (ör. 15 dakika).
- Her istekle birlikte gönderilir.
- Ele geçirilirse (çalıntı olursa), saldırgan 15 dakika boyunca kullanabilir.

**Refresh token**:

- Daha uzun süreli (ör. 7 gün, 30 gün).
- Sadece **token yenileme** endpoint’inde kullanılır.
- Genelde **server side**’da saklanan bir kayıtla eşleştirilir.

Akış:

1. Kullanıcı login olur → Sunucu **access token + refresh token** üretir.
2. Access token kısa sürede expire olur (15–30 dk).
3. Expire olduğunda, client refresh token ile `/refresh` endpoint’ine gelir.
4. Sunucu:
   - Refresh token’ı doğrular,
   - DB veya Redis’teki kayıtla eşleşip eşleşmediğini kontrol eder,
   - Geçerliyse **yeni bir access token** üretir (gerekirse yeni refresh token da üretebilir).

**Neden böyle yapıyoruz?**

- Access token çalınsa bile kısa sürede geçersiz olur.
- Refresh token ayrı bir mekanizmayla ve daha sıkı kurallarla yönetilir (ip, device, revoke mekanizması vb.).
- Kullanıcı “logout” olduğunda, refresh token DB’den silinerek tüm access token akışının devamı kesilir.

---

## 2) Kart numarası için maskleme nasıl yapılır?

Kredi/banka kartı numarası, genelde 16 hanelidir ve aşağıdaki gibi gösterilir:

```text
5346 2301 5678 9123
```

Güvenlik ve gizlilik için bunu tamamen göstermeyiz. Maskleme (masking) örneği:

```text
5346 23** **** 9123
```

Amaç:

- İlk birkaç haneden hangi banka/kurum olduğu anlaşılabilir,
- Son 4 haneden kullanıcı kartını ayırt edebilir,
- Ortadaki hassas bilgiler yıldızlanır.

Basit maskeleme stratejisi:

- İlk 6 haneyi göster (`BIN` + 2 hane gibi düşünebilirsin),
- Son 4 haneyi göster,
- Ortadaki haneyi `*` ile değiştir.

Örneğin, 16 haneli bir string için:

```java
public String maskCardNumber(String cardNumber) {
    if (cardNumber == null || cardNumber.length() < 10) {
        return cardNumber; // veya exception
    }

    String digitsOnly = cardNumber.replaceAll("\s+", ""); // boşlukları kaldır
    int length = digitsOnly.length();

    String first6 = digitsOnly.substring(0, 6);
    String last4  = digitsOnly.substring(length - 4);

    int middleLength = length - 10; // 6 + 4
    StringBuilder middle = new StringBuilder();
    for (int i = 0; i < middleLength; i++) {
        middle.append('*');
    }

    return first6 + middle + last4;
}
```

Geri dönen string istenirse 4’lü gruplara bölünüp `"**** ****"` formatında da yazılabilir.

Önemli:

- Asla tam kart numarasını loglama, exception mesajına yazma, sentry’ye gönderme.
- Maskleme hem UI’de hem log’larda yapılmalı.

---

## 3) Rate limiting yapısını nasıl koyarsın?

**Rate limiting**, bir kullanıcının/istemcinin belirli bir süre içinde yaptığı istek sayısını **sınırlamak** anlamına gelir.

Örnek:

- “Bu API endpoint’i için her kullanıcıya **dakikada en fazla 100 istek** hakkı ver.”
- Fazlasını yapanlara `429 Too Many Requests` döndür.

Bankacılıkta neden önemli?

- Brute force saldırılarını sınırlandırmak (örneğin pin denemeleri, OTP denemeleri).
- DDoS tarzı aşırı istekleri yavaşlatmak.
- Genel sistem istikrarını korumak.

### 3.1. Redis + Bucket4j yaklaşımı

Sık kullanılan kombinasyon: **Redis + Bucket4j**

- **Redis** → Dağıtık ve hızlı bir key-value store. Tüm microservice instance’ları arasında paylaşılan ortak bir depo.
- **Bucket4j** → Token bucket algoritmasını Java’da kullanmanı sağlayan bir kütüphane.

Mantık:

- Her kullanıcı ya da IP için bir “bucket” (kova) tut.
- Bucket’ta belirli sayıda “token” vardır (örneğin 100 token).
- Her istek geldiğinde 1 token tüketilir.
- Zaman ilerledikçe bucket’a belli bir hızla token eklenir (örneğin 100 token/dakika).

Eğer bucket boşsa → rate limit aşılmıştır → `429` döndür.

Yapı:

1. İstekten kullanıcı id / client id / IP gibi bir kimlik belirle.
2. Redis’te o kimlik için bir bucket kaydı tut.
3. Bucket4j ile her istekte token dene, yetmiyorsa istek reddedilir.

Pseudocode mantık:

```java
public Response handleRequest(String clientId) {
    Bucket bucket = bucketService.resolveBucket(clientId); // Redis tabanlı

    if (bucket.tryConsume(1)) {
        // İstek kabul
        return processRequest();
    } else {
        // Rate limit aşıldı
        return tooManyRequestsResponse();
    }
}
```

Bu yapı sayesinde:

- Çok instance’lı bir microservice ortamında bile rate limiting merkezi / tutarlı çalışır.
- Redis ile tüm instance’lar aynı bucket durumunu paylaşır.

---

## 4) Microservice communication

Microservice dünyasında servisler arasında iletişim iki ana şekilde olur:

1. **Synchronous (senkron)** → REST (veya gRPC)
2. **Asynchronous (asenkron)** → Message broker (Kafka, RabbitMQ vb.)

Ayrıca **circuit breaker** gibi pattern’ler önemlidir.

### 4.1. Synchronous → REST

- Servis A, Servis B’nin endpoint’ini HTTP ile çağırır.
- A, B’den **anında cevap** bekler.
- Örnek: `AccountService` → `CustomerService`’den müşteri bilgisi ister.

Avantajlar:

- Basit ve anlaşılır.
- Request-response akışı net.

Dezavantajlar:

- B servisinin yavaşlaması A’yı da yavaşlatır.
- B tamamen çökerse A da hata alır → zincirleme sorun.

---

### 4.2. Asynchronous → Kafka

- Servis A, bir **mesaj** üretip bir topic’e yazar.
- Servis B, bu topic’i dinleyerek mesajı asenkron olarak işler.
- A, B’nin hemen cevap vermesini beklemez.

Örnek:

- Money transfer başarılı olduğunda “TRANSFER_COMPLETED” eventi Kafka’ya gönderilir.
- Notification servisi bu eventi dinler, SMS veya push bildirimi yollar.

Avantajlar:

- Gevşek bağlı (loose coupling).
- Yükü yayar, peak anlarda sistemi rahatlatır.
- Bir servis down olsa bile, mesajlar broker’da bekler, sonra işlenebilir.

Dezavantajlar:

- Veri tutarlılığını ve hataları takip etmek daha zordur.
- Debug / trace etmek daha karmaşıktır.

---

### 4.3. Circuit breaker → Resilience4j

Senkron REST çağrılarında, bir servis sürekli hata veriyor veya çok yavaşsa:

- Bunu fark edip **otomatik olarak o servise çağrı yapmayı kesmek**, sonra yavaş yavaş tekrar denemek isteyebilirsin.
- Buna **circuit breaker** deseni denir (sigortaya benzetebilirsin).

**Resilience4j**, Java dünyasında sık kullanılan bir kütüphanedir.

Çalışma mantığı:

- Başta circuit “closed” durumdadır → tüm istekler normal gider.
- Belli sayıda hata / timeout sonrası circuit “open” olur → hedef servise istek yapmayı bırakır, hemen fallback döner.
- Bir süre sonra circuit “half-open” olur → az sayıda deneme yapılır; başarılı olursa circuit tekrar “closed”a döner.

Bu sayede:

- Zaten bozuk olan servise gereksiz yere yük binmez,
- Çağıran servis boşuna beklememiş olur.

Örnek kullanım senaryosu:

- `AccountService`, `CardService`’i çağırırken circuit breaker uygular.
- CardService çökerse, AccountService hızlıca fallback döner, “şu an bu bilgiye erişilemiyor” gibi kontrollü davranır.

---

## 5) Money transfer service nasıl tasarlanır?

Bu soru bankacılık mülakatlarında **çok kritik**:  
Para transferi yaparken **tutarlılık, güvenlik ve tekrar eden istekler** nasıl yönetilir?

Ana kavramlar:

- **Idempotent request**
- **Double spending koruması**
- **Optimistic / pessimistic locking**

---

### 5.1. Idempotent request nedir?

**Idempotent**, aynı isteği birden fazla kez gönderdiğinde **aynı sonucu** alman anlamına gelir.

Örnek:

- Kullanıcı bir EFT isteği gönderdi, ama network’te sorun oldu, butona tekrar bastı.
- Backend, aynı transfer isteğini **iki kere ayrı işlem olarak** gerçekleştirmemeli.
- Yani 100 TL’yi A’dan B’ye **iki kere** göndermemelisin.

Çözüm yaklaşımı:

- Her transfer isteğine bir **unique requestId** ver (örneğin UUID).
- Bu requestId’yi DB’de tut.
- Yeni transfer isteği geldiğinde, önce “bu requestId daha önce işlendi mi?” diye kontrol et.

Basit akış:

1. İstek: `{ requestId: "abc-123", fromAccount: X, toAccount: Y, amount: 100 }`
2. DB’de `requestId = "abc-123"` var mı?
   - Yoksa → işlemi yap, kayıt oluştur.
   - Varsa → işlemi **tekrar yapma**, önceki sonuç neyse onu döndür.

Böylece kullanıcı butona 3 kere bassa bile, backend bunu **tek işlem** gibi ele alır.

---

### 5.2. Double spending koruması

**Double spending**, aynı bakiyeyi iki farklı işlemde kullanmaya çalışmak demektir.

Örnek:

- A hesabında 100 TL var.
- Aynı anda iki transfer isteği geliyor:
  - İstek 1: A → B, 100 TL
  - İstek 2: A → C, 100 TL

Eğer doğru kilitleme yapılmazsa:

- İki işlem de “A’da 100 var, sorun yok” diye düşünüp onaylanabilir.
- Sonuçta A’dan 200 TL çıkmış gibi olur → hatalı.

Bunu engellemek için:

- İki işlem aynı anda A hesabını kullanırken, **veri tabanında kilit** kullanırsın.
- Bir transaction A hesabının bakiyesini güncellerken, diğeri bekler.

Bu noktada **optimistic** ve **pessimistic** locking devreye girer.

---

### 5.3. Pessimistic lock

**Pessimistic lock**:

- “Ben bu kaydı kullanacağım, başkası dokunmasın” mantığı.
- DB seviyesinde `SELECT ... FOR UPDATE` gibi komutlarla satırı kilitlersin.

Örnek akış:

1. Transaction 1:
   - `SELECT balance FROM account WHERE id = A FOR UPDATE`
   - A hesabını kilitledi.
   - Bakiye kontrolü + güncelleme yapar, commit eder.

2. Transaction 2:
   - `SELECT ... FOR UPDATE` çağrısı yapınca, A zaten kilitli olduğu için **bekler**.
   - T1 bitince kilit açılır, sonra T2 çalışır.
   - Böylece aynı anda iki işlem bakiyeyi yanlış kullanamaz.

Avantaj:

- Kesin ve net bir koruma sağlar.

Dezavantaj:

- Kilitler uzarsa performansı düşürür.
- Yüksek trafikli sistemlerde dikkatli kullanmak gerekir (ölçeklenebilirlik).

---

### 5.4. Optimistic lock

**Optimistic lock**:

- “Çakışma olmaz diye umut ederim, olursa yakalar ve retry ederim” mantığı.
- Genelde tabloda bir `version` kolonu veya `updated_at` timestamp ile çalışır.

Örnek:

`account` tablosu:

| id | balance | version |
|----|---------|---------|
| A  | 100     | 1       |

Akış:

1. T1 ve T2 aynı anda A hesabını okur → balance=100, version=1
2. T1, 100 TL düşer, yeni balance=0 yapar, version’ı 2’ye çıkar:
   ```sql
   UPDATE account
   SET balance = 0, version = 2
   WHERE id = 'A' AND version = 1;
   ```
   - Bu update başarılı oldu.

3. T2 de update etmeye çalışır:
   ```sql
   UPDATE account
   SET balance = 0, version = 2
   WHERE id = 'A' AND version = 1;
   ```
   - Ama version artık 2 olduğu için bu update **0 satır** etkiler.
   - Buradan anlarız ki, arada bir başkası bu hesabı değiştirmiş.

Sonuç:

- T2, bir **OptimisticLockException** vb. alır.
- Uygulama isterse bu durumda işlemi tekrar deneyebilir, kullanıcıya hata dönebilir.

Avantaj:

- Kilit tutma yok, concurrency daha yüksek.
- Çakışma azsa çok performanslı.

Dezavantaj:

- Çakışma çoksa sürekli retry gerekebilir.

---

### 5.5. Money transfer servisini toparlayalım

Gerçekçi bir bankacılık senaryosunda şunları söylersen çok iyi puan alırsın:

- **Idempotency**:
  - Her transfer isteği için `requestId` kullanırım.
  - Aynı `requestId` ile tekrar gelen isteği yeni transfer olarak gerçekleştirmem.
- **Double spending koruması**:
  - Hesap bakiyesini güncellerken optimistic/pessimistic locking kullanırım.
  - Böylece aynı bakiyeyi iki kere harcamaya izin vermem.
- **Transaction yönetimi**:
  - Para transferi, mutlaka tek bir transaction içinde çalışır (ACID).
  - Kaynak ve hedef hesap güncellemeleri “ya hep ya hiç” mantığıyla yapılır.
- **Audit & log**:
  - Her transfer için audit kaydı (kim, ne zaman, hangi IP’den, hangi cihazdan) tutulur.
- **Retry & error handling**:
  - Network hatası, downstream servis hatası gibi durumlar için retry mekanizmaları ve circuit breaker uygularım.

---

## 6) Kısa mülakat cevapları (özet halinde)

Son olarak mülakatta hızlı kullanabileceğin özet cümleler:

- **JWT üretimi:**  
  “Login sonrası kullanıcıyı doğrularım, sonra username ve roller gibi claim’lerle, belirli bir exp süresi olan JWT üretirim. Access token kısa süreli olur, refresh token daha uzun süreli olup DB/Redis’te tutulur ve token yenileme için kullanılır.”

- **Kart maskleme:**  
  “Kart numarasını log’larda ve UI’da tam göstermem, genelde ilk 6 ve son 4 haneyi gösterip ortasını `*` ile maskelerim: 5346 23** **** 9123 gibi.”

- **Rate limiting:**  
  “Dağıtık ortamda Redis + Bucket4j ile token bucket bazlı rate limiting kurarım. Örneğin IP veya kullanıcı bazında dakikada 100 istek hakkı tanımlar, aşılırsa 429 dönerim.”

- **Microservice communication:**  
  “Senkron iletişimde REST kullanırım, kritik external çağrılarda Resilience4j ile circuit breaker uygularım. Asenkron olay akışı için Kafka gibi bir message broker kullanır, event-driven mimari ile servisleri gevşek bağlı tutarım.”

- **Money transfer service:**  
  “Transfer servisini idempotent tasarlarım (requestId ile), double spending’e karşı optimistic/pessimistic locking uygularım. Tüm para transferi tek bir transaction içinde çalışır, böylece ACID garantisiyle tutarlı ve güvenli olur.”
