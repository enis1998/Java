# Java Mülakat Notları

Bu doküman, Java backend mülakatlarında çok sık sorulan temel konuları ve concurrency kavramlarını özetler.  
Her başlık altında hem kavramsal açıklama hem de örnek Java kodları bulunur.

---

## İçindekiler

1. [Java Temelleri](#1-java-temelleri)  
   1.1 [List, Set, Map farkları](#11-list-set-map-farkları)  
   1.2 [HashMap nasıl çalışır](#12-hashmap-nasıl-çalışır-bucket-hashing-collision)  
   1.3 [HashMap vs ConcurrentHashMap](#13-hashmap-ve-concurrenthashmap-farkı)  
   1.4 [String, StringBuilder, StringBuffer](#14-string-stringbuilder-stringbuffer-farkı)  
   1.5 [final, finally, finalize](#15-final-finally-finalize-farkı)  
   1.6 [== ve equals() farkı](#16--ve-equals-farkı)  
   1.7 [Immutable class nasıl yazılır](#17-immutable-class-nasıl-yazılır)  
   1.8 [Overloading vs overriding](#18-overloading-vs-overriding-farkı)  
   1.9 [static nedir, neler statik olabilir](#19-static-nedir-neler-statik-olabilir)  
   1.10 [Java memory modeli (Stack, Heap, Metaspace)](#110-java-memory-modeli-stack-heap-metaspace)  
   1.11 [Garbage Collector ne zaman tetiklenir](#111-garbage-collector-ne-zaman-tetiklenir)  
   1.12 [Shallow copy vs deep copy](#112-shallow-copy-vs-deep-copy)  

2. [Threading & Concurrency](#2-threading--concurrency)  
   2.1 [Thread oluşturma yolları](#21-thread-oluşturma-yolları)  
   2.2 [Thread nedir, backendte nerede karşımıza çıkar](#22-thread-nedir-backendte-nerede-karşımıza-çıkar)  
   2.3 [Process vs Thread](#23-process-vs-thread)  
   2.4 [Request bitince thread ölür mü](#24-request-bitince-thread-ölür-mü-thread-pool)  
   2.5 [synchronized nasıl çalışır](#25-synchronized-nasıl-çalışır)  
   2.6 [Deadlock nedir, nasıl önlenir](#26-deadlock-nedir-nasıl-önlenir)  
   2.7 [wait(), sleep(), notify() farkı](#27-wait-sleep-notify-farkı)  
   2.8 [ExecutorService nedir](#28-executorservice-nedir)  
   2.9 [Future, Callable farkı](#29-future-callable-farkı)  

---

## 1) Java Temelleri

### 1.1. List, Set, Map farkları?

**List**
- Sıralı koleksiyondur, index vardır: `list.get(0)` gibi.
- Duplicate’e izin verir (aynı eleman birden fazla olabilir).
- Örnek: `ArrayList`, `LinkedList`.

**Set**
- Matematiksel küme gibi düşünebilirsin.
- Duplicate’e izin vermez, her eleman en fazla bir kez bulunur.
- Genelde index yoktur, önemli olan “bu eleman var mı yok mu?” sorusudur.
- Örnek: `HashSet`, `LinkedHashSet`, `TreeSet`.

**Map**
- `key -> value` ikililerini tutar.
- Key’ler benzersizdir, aynı key’i tekrar koyarsan eski value’yu ezer.
- Value’lar duplicate olabilir.
- Örnek: `HashMap`, `LinkedHashMap`, `TreeMap`.

List vs Set vs Map

| Özellik | List | Set | Map |
| --- | --- | --- | --- |
| Eleman yapısı | Tek eleman | Tek eleman | Key–Value çifti |
| Duplicate (tekrar) | Var (aynı eleman olabilir) | Yok (unique) | Key: yok, Value: olabilir |
| Index ile erişim | Var (get(0)) | Yok | Yok (key ile erişim: get(key)) |
| Sıra | İnsertion order (çoğu impl.) | Impl’a göre (HashSet, LinkedHashSet vs) | Impl’a göre (HashMap, LinkedHashMap vs) |
| Tipik kullanım | Sıralı liste, sıra önemli | Kümeler, benzersizlik önemli | Sözlük, key’den value bulma önemli |


**Kısa kod örneği:**

```java
List<String> list = new ArrayList<>();
list.add("A");
list.add("A"); // duplicate serbest

Set<String> set = new HashSet<>();
set.add("A");
set.add("A"); // duplicate eklenmez

Map<Integer, String> map = new HashMap<>();
map.put(1, "Enis");
map.put(1, "Kaan"); // "Enis" üzerine yazılır, key 1 tekil
```

---

### 1.2. HashMap nasıl çalışır? (bucket, hashing, collision)

- İçeride **bucket**’lardan oluşan bir dizi vardır.
- `put(k, v)` süreci:
  1. `k.hashCode()` hesaplanır.
  2. Hash, dizi boyuna göre indexe indirgenir (örn: `(n - 1) & hash`).
  3. Bu indexteki bucket’a bakılır:
     - Boşsa yeni node eklenir.
     - Doluysa **collision (çakışma)** vardır; aynı bucket’ta linked list veya ağaç yapısı (Java 8+) kullanılır.

- `get(k)` süreci:
  1. Aynı hash ve index hesaplanır.
  2. Bucket içindeki node’larda `equals()` ile key karşılaştırılır.
  3. Eşleşen bulunduğunda value döner.

**Varsayılan ayarlar:**
- Başlangıç kapasitesi: 16  
- Load factor: 0.75  
- `size > capacity * loadFactor` durumunda rehash/re-size olur (kapasite 2 katına çıkar).

**Basit örnek:**

```java
Map<String, Integer> ages = new HashMap<>();
ages.put("Enis", 27);
ages.put("Ahmet", 30);

int age = ages.get("Enis"); // hash -> bucket -> equals -> 27
```

---

### 1.3. HashMap ve ConcurrentHashMap farkı?

**HashMap**
- Thread-safe değildir.
- Çoklu thread aynı anda yazarsa:
  - Race condition, bozuk veri, eski sürümlerde infinite loop görülebilir.
- `null` key ve multiple `null` value’ya izin verir.

**ConcurrentHashMap**
- Thread-safe `Map` implementasyonudur.
- Segment/bucket bazlı lock veya CAS ile ince taneli eşzamanlılık.
- `null` key ve `null` value’ya izin vermez.
- Iterator’lar **weakly consistent**:
  - Çalışırken başka thread değişiklik yapabilir ama `ConcurrentModificationException` fırlatmaz.

**Örnek kullanım:**

```java
// Tek thread veya low-concurrency
Map<String, Integer> map = new HashMap<>();

// Yüksek concurrency gereken yer
Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();
```

---

### 1.4. String, StringBuilder, StringBuffer farkı?

**String**
- Immutable’dır: değiştirildiğinde yeni nesne oluşur.
- Sık concat ile çok nesne üretimi → performans sorunu.

**StringBuilder**
- Mutable’dır, synchronized değildir.
- Tek thread içinde çok concat yaparken en performanslı seçenektir.

**StringBuffer**
- Mutable + synchronized → thread-safe.
- Genelde eski/legacy kodlarda görülür.

**Kod örneği:**

```java
String s = "Hello";
s += " World"; // Yeni String oluşturulur

StringBuilder sb = new StringBuilder("Hello");
sb.append(" World"); // Aynı nesne üzerinde değişiklik
String result = sb.toString();
```

---

### 1.5. final, finally, finalize farkı?

**`final`**
- Değişkende: Tek sefer atanabilir (referans ise hangi objeye işaret ettiği değişmez).
- Metotta: Override edilemez.
- Sınıfta: Extend edilemez.

```java
final int x = 10;
// x = 20; // derleme hatası

final class MyClass { }
// class Child extends MyClass {} // hata
```

**`finally`**
- `try`/`catch` sonrasında **her durumda** çalışması istenen blok.
- Kaynak kapatma için kullanılır.

```java
try {
    // riskli işlem
} catch (Exception e) {
    // hata yönetimi
} finally {
    // her durumda çalışır
}
```

**`finalize()`**
- GC, nesneyi toplamadan önce teorik olarak çağırabilir.
- Ne zaman çağrılacağı garanti değil, performans sorunlu, modern Java’da deprecate.
- Yerine `try-with-resources` veya `close()` kullanılır.

---

### 1.6. `==` ve `equals()` farkı?

**Primitive tiplerde**
- `==` değerleri karşılaştırır.

**Referans tiplerinde**
- `==` → aynı nesne (referans) mi?  
- `equals()` → mantıksal eşitlik (içerik) mi?

```java
String a = new String("abc");
String b = new String("abc");

System.out.println(a == b);        // false (farklı nesneler)
System.out.println(a.equals(b));   // true  (aynı içerik)
```

---

### 1.7. Immutable class nasıl yazılır?

Adımlar:
1. Sınıfı mümkünse `final` yap.
2. Alanları `private` ve `final` yap.
3. Değerleri constructor’da ata.
4. Setter yazma.
5. Mutable alanlar varsa (List, Date vs.) defensive copy kullan.

**Örnek:**

```java
public final class Person {
    private final String name;
    private final int age;
    private final List<String> tags;

    public Person(String name, int age, List<String> tags) {
        this.name = name;
        this.age = age;
        // defensive copy
        this.tags = new ArrayList<>(tags);
    }

    public String getName() { return name; }
    public int getAge() { return age; }

    public List<String> getTags() {
        return new ArrayList<>(tags); // dışarıya kopya ver
    }
}
```

---

### 1.8. Overloading vs overriding farkı?

**Overloading (aynı sınıf içinde)**
- Metot ismi aynı, parametre listesi farklı.

```java
void print(String s) { }
void print(int i) { }
void print(String s, int i) { }
```

**Overriding (inheritance ile)**
- Alt sınıf, üst sınıftaki metodu **aynı imza** ile yeniden yazar.

```java
class Animal {
    void speak() {
        System.out.println("Animal");
    }
}

class Dog extends Animal {
    @Override
    void speak() {
        System.out.println("Woof");
    }
}
```

---

## static nedir neler statik olabilir? (method, variable, inner class)?

`static`, bir üyenin **sınıfa ait** olduğunu söyler; yani o üye nesneye (instance’a) değil, doğrudan sınıfa bağlıdır.
Normalde: her `new` dediğinde ayrı bir nesne ve o nesneye ait alanlar oluşur.
`static` ise "bu alan/metot tüm nesneler için ortak" demektir.

### 1.9.1. Statik olabilenler

1. **Static field (sınıf değişkeni)**
- Tüm instance’lar tarafından **paylaşılan tek bir kopya** vardır.
- Örnek: sayaç, global config, cache benzeri ortak bilgiler.

```java
public class Counter {
    public static int count = 0; // tüm nesneler için ortak

    public Counter() {
        count++;
    }
}

Counter c1 = new Counter();
Counter c2 = new Counter();
System.out.println(Counter.count); // 2
```

2. **Static method**
- Sınıf üzerinden çağrılır: `Math.max(…)`, `Collections.sort(…)` gibi.
- `this` kullanamaz, çünkü herhangi bir instance’a bağlı değildir.
- Sadece static üyeleri **doğrudan** kullanabilir.

```java
public class MathUtil {
    public static int square(int x) {
        return x * x;
    }
}

int result = MathUtil.square(5);
```

3. **Static nested class**
- Bir sınıfın içinde ama dış sınıfın instance’ına bağlı olmayan iç sınıftır.
- `Outer.Inner` gibi kullanılır, `new Outer()` oluşturmadan da yaratılabilir.

```java
public class Outer {
    static class Inner {
        void hello() {
            System.out.println("Static nested class");
        }
    }
}

Outer.Inner inner = new Outer.Inner();
inner.hello();
```

> Fark: `static class Inner` → outer’ın `this`’ine erişemez. 
> Static olmayan inner class ise outer instance’ına bağlıdır ve `Outer.this`’e erişebilir.

4. **Static block**
- Sınıf **ilk kez yüklenirken** bir defa çalışır.
- Genelde static alanların karmaşık ilk değer atamalarında veya config yüklemede kullanılır.

```java
public class Config {
    public static final Map<String, String> SETTINGS = new HashMap<>();

    static {
        SETTINGS.put("env", "prod");
        SETTINGS.put("version", "1.0.0");
        System.out.println("Config sınıfı yüklenirken bu blok 1 kez çalışır");
    }
}
```

### 1.9.2. Static ile ilgili önemli noktalar

- **Static metod, non-static alana doğrudan erişemez** (çünkü `this` yok):

```java
public class Example {
    private int instanceValue = 10;
    private static int staticValue = 20;

    public static void foo() {
        // instanceValue; // HATA: instance alanına doğrudan erişemez
        System.out.println(staticValue); // OK
    }
}
```

- Static alanlar global state gibi davranır:
  - Fazla mutable static alan **testleri zorlaştırır** ve
  - Concurrency (thread-safety) problemlerine yol açabilir.
- Bu yüzden çoğunlukla:
  - `static final` sabitler,
  - Dikkatli tasarlanmış, thread-safe cache/config yapıları için kullanılır.

- Bellek tarafı kabaca şöyle düşünülebilir:
  - Static alanlar sınıf yüklenmesine bağlıdır, sınıf başına tek kopya olur.
  - `new` ile her instance oluşturulduğunda bu static alanlar yeniden oluşmaz, sadece referans verilir.

```

---

## Java memory modeli (Stack, Heap, Metaspace)?

Burada genelde iki şey karışıyor:
- Fiziksel bellek alanları (Stack, Heap, Metaspace, native memory…),
- **Java Memory Model (JMM)** denen, thread’ler arası görünürlük kuralları.

Mülakatta çoğunlukla ilkini (Stack/Heap/Metaspace) anlatmak yeterli olur; ama JMM’den de kısaca bahsetmek artı yazar.

### 1.10.1. Stack

- Her **thread’in kendine ait** bir stack’i vardır.
- Şunlar stack’te tutulur:
  - Metot çağrı bilgileri (call frame),
  - Yerel primitive değişkenler (`int i`, `double x`),
  - Nesne referanslarının **kendisi** (pointer gibi).
- LIFO (Last In First Out) çalışır: metoda girince frame eklenir, çıkınca kaldırılır.

```java
public int add(int a, int b) {
    int sum = a + b; // a, b, sum -> stack’te
    return sum;
}
```

- Eğer `Person p = new Person();` dersen:
  - `p` referansı stack’te tutulur,
  - `new Person()` ile oluşturulan gerçek nesne heap’te durur.

### 1.10.2. Heap

- Tüm `new` ile oluşturulan nesneler ve array’ler heap’te tutulur.
- Heap **tüm thread’ler tarafından ortak kullanılır**.
- Garbage Collector (GC) heap üzerinde çalışır ve artık erişilmeyen nesneleri temizler.

Genelde generational yapıya sahiptir:
- **Young Generation** (Eden + Survivor)
  - Yeni nesneler önce buraya düşer.
  - Çoğu nesne çok kısa yaşadığı için burada sık ve hızlı GC yapılır.
- **Old Generation**
  - Uzun yaşayan nesneler young’tan buraya taşınır (promotion).
  - Burada daha seyrek ama daha ağır GC çalışır.

### 1.10.3. Metaspace

- Class metadata’ları (sınıf tanımları, method imzaları, bazı static bilgiler) burada tutulur.
- Java 8 öncesinde `PermGen` vardı, Java 8+ ile `Metaspace` geldi.
- Metaspace, **native memory** tarafını kullanır; yani boyutu JVM argümanlarıyla sınırlandırılabilir.

### 1.10.4. Native memory ve diğer bölgeler (kısaca)

- JVM’in kendisinin ve JIT compiler’ın kullandığı alanlar,
- NIO `DirectByteBuffer` gibi heap dışı (off-heap) buffer’lar,
- Her thread’in stack’leri vs. yine process’in toplam RAM kullanımına dahildir ama heap’ten ayrıdır.

### 1.10.5. Java Memory Model (JMM)’e kısa değiniş

“Java memory modeli” dendiğinde bazen şunu da kastederler:
- Bir thread’in yaptığı yazma işlemleri diğer thread’ler tarafından **ne zaman görülebilir?**
- `volatile`, `synchronized`, `final` alanlar bu görünürlük kurallarını nasıl etkiler?

Özet:
- JMM, compiler ve CPU’nun yaptığı optimizasyonlara rağmen,
- Programcının belli kurallara uyduğunda (ör. `happens-before` ilişkisi) kodun **tahmin edilebilir** davranmasını garanti eder.
- Concurrency konularında (`volatile`, `Atomic*`, lock-free yapılar) bu model çok önemlidir.


---

### 1.11. Garbage Collector ne zaman tetiklenir?

- GC’nin tam zamanı JVM tarafından belirlenir.
- Nesneler referanslanmıyorsa **garbage** kabul edilir.
- Genelde:
  - Heap dolmaya yaklaşınca,
  - Young/Old generation dolduğunda,
  - JVM stratejisine göre tetiklenir.

`System.gc()`:
- Sadece “GC çalıştır” önerisidir, garanti değil.
- Production’da kullanılmaması önerilir.

---

### 1.12. Shallow copy vs deep copy

**Shallow copy**
- Nesnenin kendisi kopyalanır, içindeki referanslar **aynı nesneleri** işaret etmeye devam eder.

**Deep copy**
- Nesne ve içindeki referanslanan nesneler de **yeniden kopyalanır**.

**Örnek:**

```java
class Address {
    String city;
    Address(String city) { this.city = city; }
}

class Person {
    String name;
    Address address;

    Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    // Deep copy constructor
    Person(Person other) {
        this.name = other.name;
        this.address = new Address(other.address.city);
    }
}

public static void main(String[] args) {
    Address addr = new Address("Istanbul");
    Person p1 = new Person("Enis", addr);

    // Shallow copy
    Person shallow = new Person(p1.name, p1.address);

    // Deep copy
    Person deep = new Person(p1);

    shallow.address.city = "Ankara";

    System.out.println(p1.address.city);   // Ankara (shallow ortak)
    System.out.println(deep.address.city); // Istanbul (deep bağımsız)
}
```

---

## 2) Threading & Concurrency

### 2.3. Process vs Thread

**Process**
- Çalışan programın ayrı bir örneği.
- Kendi adres alanı, heap, stack.
- Diğer process’lerle bellek paylaşmaz (IPC gerek).

**Thread**
- Aynı process içindeki yürütme akışı.
- Heap ve global veriyi diğer thread’lerle paylaşır.
- Her thread’in kendi stack’i vardır.

Backend’te:
- Genelde: 1 JVM process + içinde birden fazla thread (request thread’leri, async thread’ler).

Bunu biraz daha detaylı düşünelim:

#### 2.3.1. Process (İşlem)

- Her process’in:
  - Kendi **virtual address space**’i,
  - Kendi heap’i ve global verileri,
  - Kendi dosya tanıtıcıları vardır.
- Process’ler arasında veri paylaşımı için:
  - Socket, pipe, dosya, HTTP, message queue vb. kullanılır (IPC).
- Process oluşturmak ve context switch yapmak, thread’e göre daha maliyetlidir.

#### 2.3.2. Thread (İş parçacığı)

- Bir process’in içindeki yürütme akışıdır.
- Aynı process’teki thread’ler:
  - Aynı heap’i paylaşır,
  - Sadece kendi stack’leri ayrıdır.
- Daha hafiftir, fakat:
  - Paylaşılan veri yüzünden race condition, deadlock vb. concurrency sorunları ortaya çıkar,
  - `synchronized`, lock, `volatile`, `java.util.concurrent` yapıları gerekir.

#### 2.3.3. Özet tablo

| Özellik             | Process                                       | Thread                                         |
|---------------------|-----------------------------------------------|------------------------------------------------|
| Adres alanı         | Her process için ayrı                         | Aynı process içinde ortak                      |
| Heap                | Her process’in kendi heap’i                   | Tüm thread’ler aynı heap’i paylaşır            |
| Stack               | Process içinde birden çok stack olabilir      | Her thread’in kendi stack’i vardır             |
| İzolasyon           | Yüksek (bellek paylaşımı yok)                 | Düşük (heap ortak, dikkatli senkronizasyon gerek) |
| Oluşturma maliyeti  | Görece pahalı                                 | Daha ucuz                                      |
| İletişim            | IPC (socket, pipe, dosya, HTTP, MQ vb.)      | Paylaşılan bellek + senkronizasyon              |
| Çökme etkisi        | Bir process çökerse içindeki tüm thread’ler gider | Bir thread çökerse process genelde yaşamaya devam eder |

#### 2.3.4. Spring Boot / backend açısından

- Process seviyesi:
  - Her microservice genelde ayrı bir JVM process (ayrı container/pod).
- Thread seviyesi:
  - Her HTTP isteği web server thread pool’undan bir thread kullanır.
  - `@Async`, `@Scheduled`, MQ consumer gibi işler ayrı thread havuzlarından çalışır.

---

### 2.1. Thread oluşturma yolları?

Önce klasik kısa örneği görelim, sonra daha detaylı açıklamaya geçelim.

```java
// 1) Thread extend ederek
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread çalışıyor: " + Thread.currentThread().getName());
    }
}

MyThread t1 = new MyThread();
t1.start();

// 2) Runnable implement ederek
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable çalışıyor: " + Thread.currentThread().getName());
    }
}

Thread t2 = new Thread(new MyTask());
t2.start();

// 3) Lambda ile Runnable
Thread t3 = new Thread(() -> {
    System.out.println("Lambda thread: " + Thread.currentThread().getName());
});
t3.start();
```

Gerçek hayatta genelde:

```java
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> {
    System.out.println("Thread pool içinde: " + Thread.currentThread().getName());
});
executor.shutdown();
```

Şimdi bunları biraz daha sistematik anlatalım:

#### 2.1.1. Thread sınıfını extend etmek

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread çalışıyor: " + Thread.currentThread().getName());
    }
}

MyThread t = new MyThread();
t.start(); // run() asenkron çalışır
```

- Artı: Basit, demo için anlaşılır.
- Eksi: Java tek kalıtım destekler; `Thread`’i extend edersen başka sınıftan extend edemezsin.

#### 2.1.2. Runnable arayüzünü implement etmek

```java
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable çalışıyor: " + Thread.currentThread().getName());
    }
}

Thread t = new Thread(new MyTask());
t.start();
```

Lambda ile kullanımı:

```java
Runnable task = () -> {
    System.out.println("Lambda ile runnable");
};

Thread t2 = new Thread(task);
t2.start();
```

- `Runnable`:
  - `void run()` metodu vardır,
  - Sonuç döndürmez, checked exception fırlatamaz.

#### 2.1.3. Callable + Future + ExecutorService

Sonuç döndüren işler için `Callable` daha uygundur:

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};

Future<Integer> future = executor.submit(task);

// başka işler...

Integer result = future.get(); // sonuç hazır değilse burada bekler
System.out.println("Sonuç: " + result);

executor.shutdown();
```

- `Callable<V>` → `V call() throws Exception;`
- `Future<V>` → asenkron işin sonucunu ve durumunu temsil eder.

#### 2.1.4. ExecutorService ile thread pool kullanmak

Manuel `new Thread(...)` yerine çoğu gerçek projede thread pool kullanılır:

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

for (int i = 0; i < 10; i++) {
    int jobId = i;
    executor.submit(() -> {
        System.out.println("İş " + jobId + " - " + Thread.currentThread().getName());
    });
}

executor.shutdown();
```

- Thread’ler tekrar tekrar kullanılır → thread oluşturma/yıkma maliyeti azalır.
- Aynı anda en fazla kaç thread çalışacağını kontrol edebilirsin.

---

### 2.2. Thread nedir, backend’te nerede karşımıza çıkar?

- Process = Uygulama (örneğin JVM).
- Thread = Aynı uygulama içinde çalışan “işçi”.

Backend’te:
- Tomcat/Jetty/Undertow gibi server’lar bir **thread pool** açar.
- Her HTTP isteği, bu havuzdaki bir worker thread üzerinde çalışır.
- Sen Spring controller yazarken:
  - Her istek aslında farklı bir thread tarafından işlenir.

Uzun süren işler (rapor, PDF, mail):
- Kullanıcıya hemen cevap döndürmek için:
  - İş arka planda farklı bir thread’e verilir (`@Async`, `ExecutorService`).

Ek olarak:
- Message queue tüketicileri (Kafka listener, RabbitMQ consumer),
- `@Scheduled` job’lar,
- Background batch işler
hep ayrı thread havuzlarında çalışır.

---

### 5. Spring Boot’ta gerçekçi bir örnek: `@Async` ile arka plan işi

**Senaryo:**
- Kullanıcı kayıt olurken,
- Controller hemen “kayıt başarılı” cevabını dönsün,
- Ama arkada “Hoş geldin” maili gönderilsin (mail geç sürebilir).

Eğer maili aynı thread’de gönderirsen:
- Kullanıcı 2–3 saniye response bekler.

`@Async` ile:
- Mail işini başka bir thread’e verirsin,
- HTTP response hemen döner,
- Mail arkada gönderilir.

#### 5.1. Config: `@EnableAsync`

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "emailTaskExecutor")
    public Executor emailTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("email-");
        executor.initialize();
        return executor;
    }
}
```

#### 5.2. Servis: `@Async` ile mail gönderme

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class EmailService {

    @Async("emailTaskExecutor")
    public void sendWelcomeEmail(String email) {
        System.out.println("Hoş geldin maili hazırlanıyor: " + email
                + " - Thread: " + Thread.currentThread().getName());
        try {
            Thread.sleep(2000); // 2 sn süren ağır iş gibi düşün
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println("Hoş geldin maili gönderildi: " + email
                + " - Thread: " + Thread.currentThread().getName());
    }
}
```

#### 5.3. Controller: Kullanıcı kayıt olurken asenkron mail çağrısı

```java
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api")
public class RegistrationController {

    private final EmailService emailService;

    public RegistrationController(EmailService emailService) {
        this.emailService = emailService;
    }

    @PostMapping("/register")
    public String register(@RequestParam String email) {
        // 1) Kullanıcıyı veritabanına kaydet (örnek):
        // userRepository.save(new User(email));

        // 2) Hoş geldin mailini asenkron gönder:
        emailService.sendWelcomeEmail(email);

        // 3) Kullanıcıya hemen cevap döndür:
        return "Kaydın alındı, mail kısa süre içinde gönderilecek.";
    }
}
```

#### 5.4. Çalışma sırası

1. `/api/register?email=test@test.com` endpoint’ine POST isteği geliyor.
2. Controller, web server’ın bir request thread’i üzerinde çalışıyor (ör. `http-nio-8080-exec-1`).
3. `emailService.sendWelcomeEmail(email)` çağrıldığında:
   - Spring, bu metodu `@Async` olduğu için **kendi thread pool’una** bir iş olarak ekliyor,
   - Örneğin `email-1` isimli başka bir thread bu işi almaya başlıyor.
4. Request thread’i maili beklemeden, hemen HTTP cevabını döndürüyor.
5. Arkada `email-1` thread’i 2 saniye çalışıp maili gönderiyor (veya log basıyor).

Log’larda şuna benzer şeyler görürsün:

```text
Hoş geldin maili hazırlanıyor: test@test.com - Thread: email-1
Hoş geldin maili gönderildi: test@test.com - Thread: email-1
```

Bu sayede:
- Kullanıcı arayüzü hızlı cevap alır,
- Uzun süren iş (mail gönderme) arkada, farklı bir thread pool’da yürür.

### 2.4. Request bitince thread ölür mü? (Thread pool)

- Genelde **hayır**.
- Server başlarken bir thread havuzu oluşturur (ör: 200 thread).
- Her request geldiğinde:
  - Boşta olan bir thread isteği işler.
- İstek bitince:
  - Thread ölmez, sadece tekrar **boşa çıkar** ve yeni istek bekler.

> “Her istek için yeni thread yaratılıp sonra öldürülmez; var olan thread’ler havuzda tekrar tekrar kullanılır.”

Bu sayede:
- Thread oluşturma/yıkma maliyeti düşer,
- Maksimum eşzamanlı istek sayısını havuz boyutuyla kontrol edebilirsin.

---

### 2.6. Deadlock nedir, nasıl önlenir?

**Deadlock**
- İki veya daha fazla thread’in, birbirinden beklediği lock’lar yüzünden sonsuza kadar beklemesi.

**Örnek:**

```java
Object lock1 = new Object();
Object lock2 = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lock1) {
        System.out.println("T1 lock1 aldı");
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        synchronized (lock2) {
            System.out.println("T1 lock2 aldı");
        }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lock2) {
        System.out.println("T2 lock2 aldı");
        try { Thread.sleep(100); } catch (InterruptedException e) {}
        synchronized (lock1) {
            System.out.println("T2 lock1 aldı");
        }
    }
});
```

Bu durumda:
- T1 `lock1`’i alıp `lock2`’yi beklerken,
- T2 `lock2`’yi alıp `lock1`’i beklerse → deadlock.

Önlemek için:
1. Lock’ları **her zaman aynı sırayla** al (örn. her yerde önce `lock1`, sonra `lock2`).
2. Gereksiz lock kullanma.
3. Lock’lı bölgeyi minimum tut.
4. `java.util.concurrent` yapılarını kullan (`ReentrantLock.tryLock()`, `Semaphore`, `ConcurrentHashMap` vb.).

---

### 2.8. ExecutorService nedir?

- Java’da **thread pool** yönetmek için kullanılan framework.
- Thread’leri manuel yönetmek yerine, işleri havuza verirsin.

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

executor.execute(() -> {
    System.out.println("Runnable çalışıyor");
});

Future<Integer> future = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;
});

// başka işler...

Integer result = future.get(); // sonuç hazır değilse bekler
System.out.println("Sonuç: " + result);

executor.shutdown();
```

Avantajlar:
- Thread oluşturma/yıkma maliyeti azalır.
- Aynı anda kaç thread çalışacağını kontrol edebilirsin.
- `Future` ile işi takip edip sonucunu alabilirsin.

---

### 2.9. Future, Callable farkı?

**`Callable<V>`**
- Bir işi temsil eden fonksiyonel arayüz:
  ```java
  V call() throws Exception;
  ```
- Sonuç döndürür (`V`) ve checked exception fırlatabilir.

**`Future<V>`**
- Asenkron işin sonucunu/durumunu temsil eder.
- `ExecutorService.submit(Callable)` ile elde edilir.

**Örnek:**

```java
ExecutorService executor = Executors.newSingleThreadExecutor();

Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};

Future<Integer> future = executor.submit(task);

// başka işler...

if (future.isDone()) {
    Integer result = future.get();
    System.out.println("Sonuç: " + result);
}

executor.shutdown();
```

**Analojik:**
- `Callable` → yapılacak işin tarifi  
- `Future`   → o işin sonucu hazır olunca elinde tuttuğun fiş / makbuz

---

### 2.5. synchronized nasıl çalışır?

- `synchronized`, aynı anda **tek thread**’in belirli bir blok/metodu çalıştırmasını sağlar.
- Nesne üzerinde **monitor lock** alır.

```java
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++; // aynı anda tek thread girebilir
    }

    public int getCount() {
        return count;
    }
}
```

Veya blok bazlı:

```java
public void increment() {
    synchronized (this) {
        count++;
    }
}

public static synchronized void foo() {
    // class-level lock
}
```

Temel amaç:
- Paylaşılan verinin aynı anda iki thread tarafından bozulmamasını sağlamak.

---

### 2.7. wait(), sleep(), notify() farkı

**`sleep(long millis)`**
- `Thread.sleep(...)` → thread’i belirtilen süre uyutur.
- Lock bırakmaz.
- Zamanlama için kullanılır.

**`wait()`**
- Her `Object`’te vardır.
- Sadece `synchronized` blok içinde çağrılabilir.
- Çağıran thread:
  - Monitor lock’u bırakır,
  - WAITING durumuna geçer,
  - `notify()`/`notifyAll()` veya timeout ile uyanır.

**`notify()` / `notifyAll()`**
- Aynı lock üzerinde `wait()` eden thread’leri uyandırır.
- `notify()` → bir thread,
- `notifyAll()` → hepsi.

Kısaca:
- `sleep` → Thread class’ına ait, lock serbest bırakmaz.
- `wait` / `notify` → Object’in monitor mekanizması, `synchronized` ile birlikte kullanılır.

---



