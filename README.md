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

### 1.9. static nedir, neler statik olabilir?

`static` = üye sınıfa ait, instance’a değil.

Static olabilenler:
- Static field (değişken)
- Static method
- Static nested class
- Static block

```java
public class Counter {
    public static int count = 0;

    public static void increment() {
        count++;
    }

    static {
        // sınıf yüklenirken bir defa çalışır
        System.out.println("Class loaded");
    }
}
```

---

### 1.10. Java memory modeli (Stack, Heap, Metaspace)

**Stack**
- Her thread’in kendi stack’i var.
- Metot çağrıları, yerel değişkenler, referansların kendisi.

**Heap**
- Tüm nesneler/array’ler burada.
- Tüm thread’ler ortak kullanır.
- GC burada çalışır.

**Metaspace**
- Class metadata’ları (tanımlar, method imzaları).
- Java 8+’ta `PermGen` yerine geldi.

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

### 2.1. Thread oluşturma yolları?

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

---

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

---

### 2.4. Request bitince thread ölür mü? (Thread pool)

- Genelde **hayır**.
- Server başlarken bir thread havuzu oluşturur (ör: 200 thread).
- Her request geldiğinde:
  - Boşta olan bir thread isteği işler.
- İstek bitince:
  - Thread ölmez, sadece tekrar **boşa çıkar** ve yeni istek bekler.

> “Her istek için yeni thread yaratılıp sonra öldürülmez; var olan thread’ler havuzda tekrar tekrar kullanılır.”

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

Önlemek için:
1. Lock’ları **her zaman aynı sırayla** al (örn. her yerde önce `lock1`, sonra `lock2`).
2. Gereksiz lock kullanma.
3. Lock’lı bölgeyi minimum tut.
4. `java.util.concurrent` yapılarını kullan (`ReentrantLock.tryLock()`, `Semaphore`, `ConcurrentHashMap` vb.).

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
- `Future` ile sonucu takip edebilirsin.

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
