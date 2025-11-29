# 3) DATABASE SORULARI – Detaylı Açıklamalar

Bu doküman, temel veritabanı (database) kavramlarını **mantığıyla** ve **örneklerle** açıklamak için hazırlanmıştır.

Kapsam:

- Normalization (1NF / 2NF / 3NF)
- Index nedir, ne işe yarar, ne zaman zarar verir
- LEFT JOIN vs INNER JOIN farkı
- ACID nedir
- Isolation level’lar (READ COMMITTED, SERIALIZABLE…)
- Deadlock nedir, neden olur

---

## 1. Normalization nedir? 1NF / 2NF / 3NF

### 1.1. Normalization nedir? (Sezgisel açıklama)

**Normalization**, veritabanındaki tabloları:

- Gereksiz tekrar (duplicate data) olmasın,
- Silme / güncelleme / ekleme sırasında garip hatalar (anomaliler) oluşmasın,
- Veri tutarlı (consistent) kalsın

diye **kurallara göre parçalayıp düzenleme** işidir.

Temel amaçlar:

- Aynı bilgiyi mümkün olduğunca **tek bir yerde tutmak**
- İlişkili bilgileri **mantıklı tablolara bölmek**

Klasik problemler:

- **Update anomaly**: Aynı müşterinin adresi 5 satırda yazılı, birini güncellemeyi unutursan veri çelişkili olur.
- **Insert anomaly**: Henüz siparişi olmayan müşteriyi tabloya ekleyemiyorsan (çünkü tablo siparişle birlikte tutuluyor) tasarımda sorun vardır.
- **Delete anomaly**: Tek siparişi olan müşterinin siparişini silince müşterinin kendisi de kayboluyorsa yine sorun vardır.

Normalization bu sorunları azaltmak için **kurallar (normal form’lar)** tanımlar.

---

### 1.2. Örnek: Normalize edilmemiş tablo

**Orders**

| order_id | customer_id | customer_name | customer_city | product1 | product2 | product3 |
|----------|-------------|---------------|---------------|----------|----------|----------|
| 1        | 10          | Ali Yılmaz    | Ankara        | Kalem    | Defter   | NULL     |
| 2        | 11          | Ayşe Demir    | İstanbul      | Silgi    | NULL     | NULL     |

Sorunlar:

- Aynı müşteri her siparişte **tekrar tekrar** yazılıyor.
- `product1`, `product2`, `product3` gibi kolonlar **tekrarlayan alanlar** (repeating group).
- Yeni bir ürün daha eklemek için tabloya `product4` diye yeni kolon eklemen gerekir → kötü tasarım.

---

### 1.3. 1NF (Birinci Normal Form)

**1NF şartları:**

1. Her hücrede **sadece tek bir değer** olmalı (atomic value). `"Kalem, Defter"` gibi virgüllü listeler olmaz.
2. **Tekrarlayan kolonlar** olmamalı (`product1`, `product2`, `product3` gibi).

1NF’e getirmek için ürünleri satır bazında ayırırız:

**OrderItems (1NF sonrası)**

| order_id | customer_id | customer_name | customer_city | product |
|----------|-------------|---------------|---------------|---------|
| 1        | 10          | Ali Yılmaz    | Ankara        | Kalem   |
| 1        | 10          | Ali Yılmaz    | Ankara        | Defter  |
| 2        | 11          | Ayşe Demir    | İstanbul      | Silgi   |

Artık:

- `product1`, `product2`, `product3` yok,
- Her satırda **tek bir ürün** var.

---

### 1.4. 2NF (İkinci Normal Form)

**2NF için şartlar:**

1. Tablo **önce 1NF** olmalı.
2. Eğer **bileşik (composite) bir primary key** varsa (örneğin `order_id + product`), tablodaki kolonlar **key’in tamamına** bağlı olmalı. (*Partial dependency* olmamalı.)

`OrderItems` tablosunda primary key olsun: `(order_id, product)`.

- `customer_name`, `customer_city` → sadece `customer_id`’ye bağlı.
- Ama tablonun key’i `(order_id, product)` → bazı sütunlar key’in tamamına değil, key ile alakalı başka bir sütuna (`customer_id`) bağlı.

Bu 2NF’e aykırıdır.

2NF’e getirmek için müşteri ve siparişi parçalıyoruz:

**Customers**

| customer_id | customer_name | customer_city |
|-------------|---------------|---------------|
| 10          | Ali Yılmaz    | Ankara        |
| 11          | Ayşe Demir    | İstanbul      |

**Orders**

| order_id | customer_id |
|----------|-------------|
| 1        | 10          |
| 2        | 11          |

**OrderItems**

| order_id | product |
|----------|---------|
| 1        | Kalem   |
| 1        | Defter  |
| 2        | Silgi   |

---

### 1.5. 3NF (Üçüncü Normal Form)

**3NF şartları:**

1. Tablo **2NF** olmalı.
2. **Transitive dependency** olmamalı: Bir kolon, primary key’e **başka bir kolon üzerinden** bağlı olmamalı.

Örnek:

| customer_id | city_id | city_name |
|-------------|---------|-----------|

Burada:

- `customer_id` → `city_id`
- `city_id` → `city_name`
- Dolayısıyla `customer_id` → (dolaylı olarak) `city_name`

Bu **transitive dependency**’dir ve 3NF’e aykırıdır.

3NF için şehir bilgisini ayrı tabloya alırız:

**Cities**

| city_id | city_name |
|---------|-----------|
| 1       | Ankara    |
| 2       | İstanbul  |

**Customers**

| customer_id | customer_name | city_id |
|-------------|---------------|--------|

> **Özet:**  
> - **Normalization**: Tekrarlayan veriyi ve anomalileri azaltmak için tabloyu kurallara göre bölme süreci.  
> - **1NF**: Hücreler atomik, tekrarlayan kolon yok.  
> - **2NF**: Bileşik key’de partial dependency yok.  
> - **3NF**: Transitive dependency yok.

---

## 2. Index nedir? Ne işe yarar?

### 2.1. Index nedir?

**Index**, veritabanında bir veya birden fazla kolon üzerine oluşturulan,  
**arama ve sorgu hızını artıran** yardımcı bir veri yapısıdır.

Kitap analojisi:

- Kitabın sonundaki **“Kavram Dizini (Index)”** gibidir.
- “ACID” kelimesini bulmak için kitabı sayfa sayfa okumak yerine, dizinden bakıp doğrudan ilgili sayfaya atlarsın.

Veritabanında:

- Index **olmayan** bir tabloda arama → tüm satırlar tek tek kontrol edilir (**full table scan**).
- Index **olan** bir kolonda arama → index yapısı üzerinden hızlıca ilgili satıra atlanır.

---

### 2.2. Ne işe yarar?

- **SELECT sorgularını hızlandırır.**
- Özellikle `WHERE`, `JOIN` ve `ORDER BY` kullanılan kolonlarda etkilidir.
- Büyük tablolarda milisaniyeler ile saniyeler arasında ciddi fark yaratabilir.

> **Kısa cümle:**  
> Index, tablo üzerinde hızlı arama yapmamızı sağlayan bir yapı; kitap sonundaki dizin gibidir.

---

## 3. Index hangi durumlarda zarar verir?

Index her zaman “iyi” değildir; bazı durumlarda **performansı düşürür**.

### 3.1. INSERT / UPDATE / DELETE maliyeti

Index, tablodan ayrı bir veri yapısıdır. Bu yüzden:

- `INSERT`,
- `UPDATE`,
- `DELETE`

işlemlerinde sadece tablo değil, **index de güncellenir**.

Sonuç: Write-heavy (yazma ağırlıklı) sistemlerde ek yük oluşturur, işlemler yavaşlayabilir.

---

### 3.2. Düşük seçicilik (low cardinality) alanlar

Örnek:

- `gender` kolonu → sadece `'M'` / `'F'`.
- 10 milyon satırda 5 milyon M, 5 milyon F olsun.

```sql
SELECT * FROM Users WHERE gender = 'M';
```

Burada index çoğu zaman **anlamsızdır**:

- Zaten satırların yarısını okuyacaksın.
- Veritabanı full table scan’i tercih edebilir.
- Index hem yer kaplar, hem yazma maliyeti getirir.

---

### 3.3. Çok küçük tablolar ve gereksiz index’ler

- 100–200 satırlık küçük tablolarda full table scan zaten çok hızlıdır.
- Hiç kullanılmayan kolonlarda index tanımlı olması gereksiz disk ve bakım maliyetidir.

> **Özet:**  
> Index, okumayı hızlandırır ama yazma işlemlerinde ek maliyet getirir. Çok fazla insert/update/delete olan tablolarda, düşük seçicilikli kolonlarda ve küçük tablolarda gereksiz index performansa zarar verebilir.

---

## 4. LEFT JOIN vs INNER JOIN farkı

**Customers**

| id | name   |
|----|--------|
| 1  | Ali    |
| 2  | Ayşe   |
| 3  | Mehmet |

**Orders**

| id | customer_id | amount |
|----|-------------|--------|
| 1  | 1           | 100    |
| 2  | 1           | 50     |
| 3  | 2           | 70     |

Mehmet’in (id=3) henüz siparişi yok.

---

### 4.1. INNER JOIN

```sql
SELECT c.name, o.amount
FROM Customers c
INNER JOIN Orders o ON c.id = o.customer_id;
```

**INNER JOIN**:

- Sadece **her iki tabloda da eşleşen** kayıtları döner.

Sonuç:

| name | amount |
|------|--------|
| Ali  | 100    |
| Ali  | 50     |
| Ayşe | 70     |

Mehmet görünmez, çünkü Orders tablosunda karşılığı yoktur.

---

### 4.2. LEFT JOIN

```sql
SELECT c.name, o.amount
FROM Customers c
LEFT JOIN Orders o ON c.id = o.customer_id;
```

**LEFT JOIN**:

- Soldaki tablodaki (burada `Customers`) **tüm satırları** döner.
- Sağda eşleşme yoksa, sağ tarafı **NULL** ile doldurur.

Sonuç:

| name   | amount |
|--------|--------|
| Ali    | 100    |
| Ali    | 50     |
| Ayşe   | 70     |
| Mehmet | NULL   |

> **Özet:**  
> - **INNER JOIN**: “Sadece ilişkisi olan kayıtları getir.”  
> - **LEFT JOIN**: “Soldaki tablodaki tüm kayıtları getir, sağda karşılığı olmayanlar için NULL yaz.”

---

## 5. ACID nedir?

ACID, veritabanındaki **transaction (işlem)** kavramının 4 temel özelliğini anlatır:

1. **A – Atomicity (Bölünmezlik)**  
2. **C – Consistency (Tutarlılık)**  
3. **I – Isolation (Yalıtım)**  
4. **D – Durability (Dayanıklılık)**  

Transaction: Bir grup SQL işleminin **“ya hep ya hiç”** mantığıyla çalışmasıdır. Örneğin: A hesabından B hesabına para transferi.

---

### 5.1. Atomicity (Bölünmezlik)

- Transaction içindeki adımlar **bölünemez**.
- Ya **hepsi başarılı olur** ya da **hiçbiri uygulanmaz** (rollback).

Para transferi örneğinde:

- A’dan 100 TL düşülür,
- B’ye 100 TL eklenir,
- B’ye ekleme sırasında hata olursa, A’dan düşülen para da geri alınır.

---

### 5.2. Consistency (Tutarlılık)

- Transaction öncesi ve sonrası, veritabanı kurallara **uygun** olmalı.
- Örneğin: “Hiçbir hesabın bakiyesi eksi olamaz” kuralı, transaction sonunda hâlâ geçerli olmalı.

---

### 5.3. Isolation (Yalıtım)

- Aynı anda çalışan transaction’lar birbirini **bozmamalı**.
- Sen bir kayıt üzerinde işlem yaparken, başka transaction’ın yarım kalmış değişikliklerini görmemelisin.

Detayları isolation level’lar belirler (READ COMMITTED, SERIALIZABLE vb.).

---

### 5.4. Durability (Dayanıklılık)

- Transaction **commit** edildikten sonra,
- Sistem çökse bile değişiklikler **kalıcı** olmalıdır.
- Veritabanı bunu log ve disk yapılarıyla garanti eder.

> **Özet:**  
> ACID, transaction’ların güvenliğini tanımlar: Atomicity (ya hep ya hiç), Consistency (kurallara uygunluk), Isolation (eşzamanlı işlemler birbirini bozmaz), Durability (commit sonrası veri kalıcıdır).

---

## 6. Isolation level’lar (READ COMMITTED, SERIALIZABLE…)

Isolation, aynı anda çalışan transaction’ların **birbirini ne kadar görebileceğini** belirler.

Bazı tipik problemler:

- **Dirty read**: Henüz commit edilmemiş veriyi okumak.
- **Non-repeatable read**: Aynı satırı iki kez okuduğunda, arada başka transaction değiştirdiği için farklı değer görmek.
- **Phantom read**: Aynı koşulla satır listesini iki kez okuduğunda, arada başka transaction yeni satırlar eklediği için ikinci okumada ekstra satırlar görmek.

---

### 6.1. READ UNCOMMITTED

- En zayıf seviye.
- Henüz commit edilmemiş veriyi bile görebilirsin (**dirty read** mümkündür).
- Kritik sistemlerde tercih edilmez.

---

### 6.2. READ COMMITTED

- **Sadece commit edilmiş veriyi** görürsün.
- **Dirty read yoktur.**
- Fakat **non-repeatable read** olabilir (aynı satırı iki kez okuduğunda arada güncellenmiş olabilir).

Birçok veritabanında varsayılan seviyedir.

---

### 6.3. REPEATABLE READ

- Bir satırı okuduysan, transaction bitene kadar aynı satırı aynı değerle görürsün.
- **Non-repeatable read** engellenir.
- Ancak **phantom read** hâlâ olabilir (aynı kriterle satır listesi çektiğinde ikinci seferde yeni satırlar gelebilir).

---

### 6.4. SERIALIZABLE

- En güçlü seviye.
- Sanki tüm transaction’lar **teker teker** (seri) çalışıyormuş gibi davranır.
- Dirty read, non-repeatable read ve phantom read tamamıyla engellenir.
- Bunun karşılığında daha fazla kilit kullanılır, performans maliyeti artar.

---

### 6.5. Özet tablo

| Level            | Dirty Read | Non-Repeatable Read | Phantom Read |
|------------------|-----------:|---------------------:|-------------:|
| READ UNCOMMITTED | Var        | Var                  | Var          |
| READ COMMITTED   | Yok        | Var                  | Var          |
| REPEATABLE READ  | Yok        | Yok                  | Var          |
| SERIALIZABLE     | Yok        | Yok                  | Yok          |

---

## 7. Deadlock neden olur?

### 7.1. Deadlock nedir?

**Deadlock**, iki (veya daha fazla) transaction’ın **birbirinin kilitlediği kaynağı bekleyip sonsuza kadar beklemesi** durumudur.

Basit senaryo:

- T1: Row A’yı kilitler, sonra Row B’yi kilitlemek ister.
- T2: Row B’yi kilitler, sonra Row A’yı kilitlemek ister.

Her iki transaction da diğerinin bırakmasını bekler → **Deadlock** oluşur.

---

### 7.2. Bankacılık örneği

- T1: Hesap A’dan B’ye para gönderiyor. Önce A’yı, sonra B’yi kilitliyor.
- T2: Hesap B’den A’ya para gönderiyor. Önce B’yi, sonra A’yı kilitliyor.

İkisi aynı anda çalışırsa:

- T1 → A kilitli, B’yi bekliyor.
- T2 → B kilitli, A’yı bekliyor.
- Hiçbiri ilerleyemiyor → Deadlock.

Veritabanı genellikle bunu algılar, transaction’lardan birini **rollback** eder.

---

### 7.3. Deadlock’un ana nedenleri

- **Kilit alma sırasının tutarsız olması** (bir yerde önce A sonra B, başka yerde önce B sonra A kilitlemek)
- **Uzun süren transaction’lar** (çok fazla kaynağı uzun süre kilitli tutmak)
- **Aynı kaynaklara farklı sıralarla erişen transaction’ların** aynı anda çalışması

> **Özet cümle:**  
> “Deadlock, iki transaction’ın birbirinin kilitlediği kaynağı beklemesi sonucu oluşan karşılıklı kilitlenme durumudur. Genelde tutarsız lock sıralaması ve uzun süren transaction’lar sebep olur; veritabanı birini kurban seçip rollback eder.”
