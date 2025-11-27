
# ✅ 2) SPRING BOOT MÜLAKAT SORULARI (BANKALARDA EN ÇOK SORULAN BÖLÜM)

Bu doküman, Spring Boot mülakatlarında (özellikle bankalarda) en sık sorulan konuları, sadece tanım vererek değil, işin mantığını ve örnek kodları anlatarak özetler.

---

## 1) Temel Spring Kavramları

### 1.1) IoC Container nedir? Mantığı nedir?

Klasik Java’da nesneleri genelde sınıfın içinde `new` ile oluşturursun:

```java
public class OrderService {
    private PaymentService paymentService = new PaymentService();
}
```

Bu yaklaşımda:

- `OrderService`, hangi `PaymentService` implementasyonunun kullanılacağını kendi belirler.
- `PaymentService`’i mock’lamak, değiştirilebilir yapmak, farklı implementasyonla denemek zorlaşır.
- Yüksek seviye sınıf (`OrderService`), düşük seviye sınıfa (`PaymentService`) sıkı sıkıya bağlıdır.

**Inversion of Control (IoC)**, “nesnelerin oluşturulması ve bağlanması kontrolünü uygulama kodundan alıp framework’e devretme” fikridir.  
**Spring IoC Container** (`ApplicationContext`), uygulamanın kalbidir ve şunları yapar:

- Bean tanımlarını okur (annotation scanning, `@Configuration` / `@Bean`, XML vb.).
- Hangi bean’in hangi bean’e bağımlı olduğunu belirleyerek bir nesne grafiği çıkarır.
- Bean’leri uygun sırayla oluşturur ve bağımlılıklarını enjekte eder (**Dependency Injection**).
- Bean’lerin yaşam döngüsünü yönetir (`init`, `destroy`, `@PostConstruct`, `@PreDestroy`).
- Scope’ları yönetir (`singleton`, `prototype`, `request`, `session`).
- AOP/proxy tabanlı davranışları uygular (`@Transactional`, `@Async`, security proxy’leri gibi).

Mantık şudur: Uygulama içindeki sınıflar birbiriyle `new` üzerinden sıkı bağlı olmak yerine,  
“ben şu türde bir servise ihtiyaç duyuyorum” der; o servisin hangi sınıftan, nasıl oluşturulacağına Spring karar verir.  
Böylece daha esnek, test edilebilir ve konfigüre edilebilir bir mimari ortaya çıkar.

**Kod örneği (basit IoC/DI kullanımı):**

```java
@Service
public class PaymentService {
    public void pay() {
        System.out.println("Ödeme alındı");
    }
}

@Service
public class OrderService {

    private final PaymentService paymentService;

    // IoC Container bu constructor'a PaymentService bean'ini enjekte eder.
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    public void createOrder() {
        // OrderService new PaymentService() DEMİYOR, Spring'in verdiği bean'i kullanıyor.
        paymentService.pay();
    }
}
```

---

### 1.2) Dependency Injection nedir? Kaç çeşit DI vardır?

Dependency, bir sınıfın çalışmak için ihtiyaç duyduğu başka sınıflardır.  
Örneğin `OrderService`, `PaymentService`’e ihtiyaç duyuyorsa, `PaymentService` bir dependency’dir.

**Dependency Injection (DI)**, bu bağımlılıkların sınıfın içinde `new` ile oluşturulmak yerine, Spring tarafından dışarıdan verilmesidir. Böylece:

- Bağımlılıkları değiştirmek (farklı implementasyon, fake/mock) kolaylaşır.
- Kod daha modüler ve test edilebilir olur.
- *“High-level modules should not depend on low-level modules, both should depend on abstractions”* ilkesine yaklaşmış olursun.

Spring’de üç temel DI stili kullanılır:

1. **Constructor Injection (Önerilen)**
2. **Setter Injection** (Opsiyonel veya sonradan atanabilir bağımlılıklar için mantıklı)
3. **Field Injection** (Mevcut ama best practice olarak önerilmiyor)

**Constructor Injection örneği:**

```java
@Service
public class OrderService {

    private final PaymentService paymentService;
    private final NotificationService notificationService;

    public OrderService(PaymentService paymentService,
                        NotificationService notificationService) {
        this.paymentService = paymentService;
        this.notificationService = notificationService;
    }
}
```

**Setter Injection örneği:**

```java
@Service
public class OrderService {

    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Field Injection örneği:**

```java
@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService;
}
```

Field injection kısa görünse de, **test edilebilirlik ve okunabilirlik** açısından constructor injection kadar sağlıklı değildir.

---

### 1.3) `@Component`, `@Service`, `@Repository` farkı

Bu anotasyonların hepsi Spring’e “Bu sınıf bir bean, IoC Container’a ekle” demenin yollarıdır.  
Teknik olarak hepsi `@Component` tabanlıdır, ama semantik olarak katman ayrımı yapmak için kullanılır:

- `@Component`: En genel anotasyondur. Her türlü bileşen için kullanılabilir.
- `@Service`: İş mantığı (business logic) katmanı için kullanılır (`UserService`, `AccountService` gibi). Kod okuyan kişiye “Burada iş kuralı var” mesajı verir.
- `@Repository`: Veri erişim katmanı için (DAO, Spring Data JPA repository’leri vb.). Ayrıca Spring, `@Repository` anotasyonunu gördüğünde veritabanı hatalarını kendi `DataAccessException` hiyerarşisine çevirerek daha standart bir exception yönetimi sağlar.

**Kod örneği:**

```java
@Component
public class IdGenerator {
    public String generate() { return UUID.randomUUID().toString(); }
}

@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

---

### 1.4) `@Bean` ile `@Component` farkı

**`@Component`:**

- Sınıfın üzerine yazılır.
- Component scan ile otomatik bulunur ve bean olarak kaydedilir.
- Genelde kendi yazdığın servis, repo vb. sınıflar için kullanılır.

**`@Bean`:**

- Bir `@Configuration` sınıfının içindeki metoda yazılır.
- Metodun döndürdüğü nesne bean olur.
- Üçüncü parti kütüphanelerin sınıfları gibi, üzerine anotasyon koyamadığın nesneleri bean yapmak için kullanılır.

Mantık:  
`@Component` ile Spring’in kendisinin sınıfı bulmasını sağlarsın;  
`@Bean` ile “bu metodu çağır, dönen nesneyi bean yap” dersin. `@Bean` daha manuel ama esnektir.

**Kod örneği:**

```java
@Configuration
public class AppConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
}

@Component
public class PriceCalculator {
    // Spring bu sınıfı component-scan ile otomatik bulur.
}
```

---

### 1.5) `@Autowired` ile constructor injection farkı

Field injection örneği:

```java
@Service
public class MyService {

    @Autowired
    private UserRepository userRepository;
}
```

Constructor injection örneği:

```java
@Service
public class MyService {

    private final UserRepository userRepository;

    public MyService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

Spring 4.3+ ile, sınıfta **tek constructor** varsa `@Autowired` yazmasan bile Spring o constructor’ı kullanır.

Farkın mantığı:

- **Field injection**: Kısa ama bağımlılıkları gizler, test etmeyi zorlaştırır, immutable tasarımı engeller.
- **Constructor injection**: Zorunlu bağımlılıkları net gösterir, testte mock geçmeyi kolaylaştırır, `final` alanlar ile daha güvenli ve sağlam sınıflar yazmanı sağlar.

Mülakatta:  
“Ben genellikle constructor injection kullanıyorum, field injection’ı best practice olarak tercih etmiyorum.” demek güzel durur.

---

### 1.6) Singleton scope nedir? Spring’te default scope nedir?

**Scope**, bir bean’den uygulama boyunca kaç instance üretileceğini ifade eder.

**Singleton scope:**

- Spring konteynerinde bir bean tanımı için yalnızca tek bir instance oluşturulur.
- Uygulama boyunca her yerde aynı instance kullanılır.

Diğer örnek scope’lar:

- `prototype`: Her injection isteğinde yeni bir instance oluşturulur.
- `request`: Her HTTP request için ayrı instance (web projelerinde).
- `session`: Her HTTP session için ayrı instance.

**Spring’te default scope: `singleton`**’dır.  
Yani özel bir şey belirtmezsen tüm `@Service`, `@Repository`, `@Component` bean’lerin singleton olur.

Bankacılık uygulamalarında stateless servisler için singleton scope, performans ve kaynak kullanımı açısından idealdir.

**Kod örneği (scope değişimi):**

```java
@Component
@Scope("prototype")
public class PrototypeBean {
    // Her injection isteğinde yeni bir instance
}
```

---

## 2) Spring Boot

### 2.1) Auto-configuration nasıl çalışır?

Spring Boot’un en önemli özelliklerinden biri **auto-configuration**’dır.  
Amaç: “En az konfigurasyonla çalışan bir uygulama ayağa kaldırmak”.

`@SpringBootApplication` anotasyonu; `@Configuration`, `@EnableAutoConfiguration` ve `@ComponentScan` içerir.

`@EnableAutoConfiguration` üzerinden Spring Boot:

- Classpath’te hangi starter’lar ve kütüphaneler (örneğin `spring-boot-starter-web`, `spring-boot-starter-data-jpa`) olduğuna bakar.
- `META-INF` içindeki auto-configuration sınıflarını (`@Configuration`) tarar.
- Bu sınıfların üzerindeki `@ConditionalOnClass`, `@ConditionalOnMissingBean` gibi anotasyonlara göre hangi bean’lerin oluşturulacağına karar verir.

Örneğin proje `spring-boot-starter-web` içeriyorsa:

- `DispatcherServlet`, `HandlerMapping`, `HandlerAdapter` gibi MVC bileşenlerini otomatik konfigüre eder.
- Gömülü Tomcat başlatır, varsayılan olarak 8080 portunda dinler.

Mantık: Spring Boot “akıllı varsayılanlar” kullanarak konfigürasyon yükünü azaltır ama ihtiyaç olduğunda senin override etmeni engellemez.  
Yani auto-config, senin tanımladığın bean’leri ezmeyecek şekilde tasarlanmıştır.

**Kod örneği (basit Spring Boot main sınıfı):**

```java
@SpringBootApplication
public class BankApplication {
    public static void main(String[] args) {
        SpringApplication.run(BankApplication.class, args);
    }
}
```

---

### 2.2) `application.yml` nerelerde kullanılır? Mantığı nedir?

`application.yml`, Spring Boot uygulamasının **konfigürasyonlarının** tutulduğu dosyadır.  
`application.properties` ile aynı amaç için kullanılır, sadece YAML formatındadır.

Kullanım alanları:

- Sunucu portu (`server.port`)
- Veritabanı bağlantı ayarları (`spring.datasource.*`)
- JPA/Hibernate ayarları (`spring.jpa.*`)
- Log seviyeleri (`logging.level.*`)
- Cache, mail, security, message broker vb. için Spring Boot starter ayarları
- Kendi custom property’lerin (`myapp.*` gibi)

Örnek:

```yaml
server:
  port: 8081

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/bank
    username: dbuser
    password: secret
```

**Property okuma örneği:**

```java
@Component
@ConfigurationProperties(prefix = "bank")
public class BankProperties {
    private String name;
    private String country;
    // getter/setter
}
```

```yaml
# application.yml
bank:
  name: MyBank
  country: TR
```

---

### 2.3) Profile yapısı nasıl çalışır? (dev, test, prod)

Farklı ortamlar (**development**, **test**, **production**) için farklı ayar setleri kullanmak için **Spring Profiles** mekanizması vardır.

**Dosya bazlı profil kullanımı:**

- `application-dev.yml`
- `application-test.yml`
- `application-prod.yml`

**Aktif profil seçimi:**

- `application.yml` içinde:

```yaml
spring:
  profiles:
    active: dev
```

- JVM argümanı ile:

```bash
-Dspring.profiles.active=prod
```

**Bean bazlı kullanım:**

```java
@Service
@Profile("dev")
public class DevMailService implements MailService { ... }

@Service
@Profile("prod")
public class ProdMailService implements MailService { ... }
```

Mantık:  
Banka gibi kurumsal ortamlarda dev, test, uat, prod gibi farklı ortamlar olur ve her birinin veritabanı, queue, cache, API endpoint’leri farklıdır.  
Profil yapısı sayesinde kodu değiştirmeden, sadece profil değiştirerek farklı ortam konfigürasyonlarıyla uygulamayı çalıştırabilirsin.

---

## 3) Spring MVC

### 3.1) `@Controller` vs `@RestController` farkı

**`@Controller`:**

- Genellikle view (HTML, Thymeleaf, JSP vb.) döndürmek için kullanılır.
- Metoddan dönen `String`, bir template ismine karşılık gelir.

**`@RestController`:**

- `@Controller + @ResponseBody` birleşimi gibidir.
- Metodun döndürdüğü nesne doğrudan HTTP response body’sine (genelde JSON) yazılır.
- REST API’lerde standart olarak kullanılır.

Bankacılıkta çoğu zaman JSON tabanlı API’ler yazdığın için `@RestController` görürsün.  
Web arayüzü olan legacy projelerde `@Controller` ile view döndürülebilir.

**Kod örneği:**

```java
@Controller
public class HomeController {

    @GetMapping("/")
    public String home() {
        return "home"; // home.html template ismi
    }
}

@RestController
@RequestMapping("/api")
public class AccountController {

    @GetMapping("/accounts/{id}")
    public AccountDto getAccount(@PathVariable Long id) {
        // JSON olarak döner
        return accountService.getAccount(id);
    }
}
```

---

### 3.2) `@RequestParam`, `@PathVariable` farkı

**`@RequestParam`:**

- URL’deki **query parametrelerini** veya form parametrelerini okur.
- Örnek: `/search?keyword=java&page=2`

**`@PathVariable`:**

- URL path içindeki **değişken parçaları** okur.
- Örnek: `/customers/123/accounts/5`

Mantık:

- `@PathVariable` genelde “kaynak kimliği” (resource id) için kullanılır.
- `@RequestParam` filtre, sayfalama, sıralama gibi ek parametreler için kullanılır.  
  Böylece API tasarımın daha RESTful ve okunabilir olur.

**Kod örneği:**

```java
@GetMapping("/customers/{id}")
public CustomerDto getCustomer(
        @PathVariable Long id,
        @RequestParam(required = false, defaultValue = "true") boolean includeAccounts) {
    // id: path variable
    // includeAccounts: query param
    return customerService.getCustomer(id, includeAccounts);
}
```

---

### 3.3) `@ExceptionHandler` nasıl kullanılır? Mantığı nedir?

`@ExceptionHandler`, belirli exception türleri için **özel bir HTTP response** üretmene yarar.

**Controller içinde lokal kullanım:**

```java
@GetMapping("/users/{id}")
public UserDto getUser(@PathVariable Long id) {
    return userService.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
}

@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<String> handleUserNotFound(UserNotFoundException ex) {
    return ResponseEntity.status(HttpStatus.NOT_FOUND)
                         .body(ex.getMessage());
}
```

Global kullanım için `@ControllerAdvice` veya `@RestControllerAdvice` ile tüm controller’lar için tek yerden hata yönetimi yapabilirsin.

Mantık:  
Bankacılık gibi kritik sistemlerde hataları kontrolsüz şekilde dışarı sızdırmak istemezsin.  
`@ExceptionHandler` ile hem kullanıcıya anlamlı mesajlar vermek hem de log’ta detaylı kayıt tutmak mümkündür.

---

### 3.4) Interceptor nedir? Mantığı nedir?

**Interceptor**, Spring MVC’de request–response akışına controller’dan önce ve sonra girebilen bir bileşendir.

`HandlerInterceptor` arayüzü ile:

- `preHandle`: Controller çağrılmadan hemen önce çalışır.
- `postHandle`: Controller çalışıp `ModelAndView` döndükten sonra, view render edilmeden önce çalışır.
- `afterCompletion`: Tüm işlem tamamlandıktan sonra (view render dahil) çalışır.

Kullanım örnekleri:

- Request loglama (kullanıcı hangi endpoint’e ne zaman geldi).
- Audit (kim, hangi işlemi yaptı).
- Basit güvenlik kontrolleri veya header kontrolü.

Interceptor, servlet filter’lara göre Spring MVC seviyesinde çalışır ve handler (controller metodu) bilgisine daha yakındır;  
bu yüzden MVC’ye özel cross-cutting işler için idealdir.

**Kod örneği:**

```java
public class LoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        System.out.println("Request URI: " + request.getRequestURI());
        return true; // false dönersek request controller'a gitmez
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoggingInterceptor())
                .addPathPatterns("/api/**");
    }
}
```

---

## 4) Spring Data JPA

### 4.1) Hibernate lifecycle evreleri

Hibernate’in entity yaşam döngüsü basitçe dört ana evrede özetlenir:

1. **Transient**  
   - `new` ile oluşturduğun ama henüz `EntityManager`/`Session` tarafından yönetilmeyen nesne.  
   - Örnek: `User u = new User();`

2. **Persistent**  
   - `entityManager.persist(u)` veya `repository.save(u)` çağrıldıktan sonra yönetilen entity.  
   - Üzerindeki değişiklikler transaction commit edildiğinde otomatik olarak veritabanına yansır (**dirty checking**).

3. **Detached**  
   - Entity bağlamdan (session/`EntityManager` context) kopmuştur; örneğin transaction bitmiştir.  
   - Artık yapılan değişiklikler otomatik olarak veritabanına gitmez.  
   - Geri bağlamak için `merge` kullanırsın.

4. **Removed**  
   - Silinmek üzere işaretlenmiş entity. `remove` çağrılmıştır.  
   - Transaction commit edildiğinde veritabanından gerçekten silinir.

Mantık:  
Bu evreleri bilmek, özellikle performans ve veri tutarlılığı açısından önemlidir.  
Bankacılıkta yanlış zamanda detach olmak, beklenmedik güncel olmayan veri kullanmana yol açabilir.

**Kod örneği (yaşam döngüsü akışı):**

```java
@Transactional
public void demo(EntityManager em) {
    User user = new User(); // Transient
    em.persist(user);       // Persistent
    em.flush();             // Değişiklikler DB'ye gider
    em.detach(user);        // Detached
    em.remove(user);        // Removed (commit ile DB'den silinir)
}
```

---

### 4.2) `@Entity`, `@Id`, `@GeneratedValue`

- `@Entity`  
  Sınıfın bir JPA entity’si olduğunu belirtir. Veritabanında bir tabloya karşılık gelir.

- `@Id`  
  Primary key alanını belirtir.

- `@GeneratedValue`  
  ID’nin nasıl üretileceğini belirtir (`IDENTITY`, `SEQUENCE`, `TABLE`, `AUTO`).

**Örnek:**

```java
@Entity
public class Account {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String iban;
}
```

Mantık:  
ID yönetimini elle yapmak yerine, JPA/Hibernate’in bunu otomatik halletmesini sağlarsın.  
Banka sistemlerinde kimliklendirme ve ilişki yönetimi çok önemli olduğundan, ID stratejini doğru seçmek kritik olabilir.

---

### 4.3) Lazy vs Eager fetch farkı

JPA ilişkilerinde (`@OneToMany`, `@ManyToOne` vb.) **Lazy** ve **Eager** fetch tipleri performansı doğrudan etkiler.

**EAGER fetch:**

- Entity yüklendiğinde ilişkili tüm nesneler hemen yüklenir.
- Basit senaryolarda pratik olabilir ama ilişkiler derinleştikçe büyük ve maliyetli join’lere sebep olabilir.

**LAZY fetch:**

- İlişkili nesneler, alanına gerçekten erişene kadar yüklenmez.
- Eriştiğin anda ek bir sorgu atılır.

Varsayılanlar (Hibernate implementasyonunda):

- `@ManyToOne` ve `@OneToOne` → genelde EAGER  
- `@OneToMany` ve `@ManyToMany` → genelde LAZY

Mantık:  
Banka gibi yüksek trafik ve büyük veri barındıran sistemlerde, gereksiz join ve veri taşımasından kaçınmak için çoğu ilişkiyi LAZY yapmak,  
ihtiyaç olduğunda fetch join veya `@EntityGraph` ile toplu çekmek performans açısından çok önemlidir.

**Kod örneği:**

```java
@Entity
public class Customer {

    @Id
    @GeneratedValue
    private Long id;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Account> accounts;
}
```

---

### 4.4) N+1 problem nedir? Nasıl çözülür?

**N+1 problemi**, genellikle LAZY ilişki ve loop’lar yüzünden ortaya çıkan performans problemidir.

Senaryo:

- 1 sorgu ile N adet `Customer` çekersin.
- Sonra her `customer.getAccounts()` dediğinde ayrı bir sorgu atılır.
- Sonuç: 1 (müşterileri çekmek için) + N (her müşteri için hesapları çekmek için) ⇒ **N+1 sorgu**.

Bu bankacılık gibi sistemlerde büyük veri setlerinde ciddi performans sorunlarına yol açar.

**Çözüm yöntemleri:**

- JPQL veya Spring Data ile **fetch join** kullanmak:

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {

    @Query("SELECT c FROM Customer c JOIN FETCH c.accounts")
    List<Customer> findAllWithAccounts();
}
```

- `@EntityGraph` kullanarak ilişkili veriyi tek sorguda çekmek.
- Gerekirse DTO projection ile sadece ihtiyaç duyulan alanları çekmek.

Mantık:  
Amaç, aynı iş için gereksiz tekrarlı sorgular atmamak, veriyi mümkün olduğunca az sorguyla ama kontrollü şekilde çekmektir.

---

### 4.5) Transaction yönetimi (`@Transactional` propagation seviyeleri)

`@Transactional`, bir metodu veritabanı transaction’ı içinde çalıştırmak için kullanılır.  
Bankacılıkta transaction yönetimi özellikle önemlidir; örneğin bir hesaptan para çekip diğerine yatırırken işlemlerden biri başarısız olursa tamamı geri alınmalıdır.

**Propagation**, transactional bir metodun başka bir transactional metod tarafından çağrıldığında nasıl davranacağını belirler.

En önemli propagation seviyeleri:

- `REQUIRED` (varsayılan):  
  Mevcut bir transaction varsa ona katılır, yoksa yeni bir transaction başlatır.

- `REQUIRES_NEW`:  
  Her zaman yeni bir transaction başlatır, varsa mevcut transaction’ı askıya alır.  
  Özellikle loglama veya audit gibi ana transaction’dan bağımsız kayıtların tutulması gereken durumlarda kullanılır.

- `SUPPORTS`:  
  Transaction varsa onun içinde çalışır, yoksa transaction açmadan çalışır.

Mantık:  
Banka uygulamalarında hangi işlemlerin birlikte, atomik şekilde gerçekleşmesi gerektiğini belirleyip propagation seviyelerini buna göre seçmek veri tutarlılığı için kritiktir.

**Kod örneği:**

```java
@Service
public class TransferService {

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // REQUIRED: Aynı transaction içinde para çekme ve yatırma
        accountService.debit(fromId, amount);
        accountService.credit(toId, amount);
    }
}
```

---

## 5) Spring Security

### 5.1) FilterChain nasıl çalışır?

Spring Security, gelen HTTP isteğini controller’a göndermeden önce bir güvenlik filtre zincirinden (`springSecurityFilterChain`) geçirir.

Akış kabaca şöyledir:

1. Request, servlet container’a gelir ve önce security filter chain’e girer.
2. Sırayla çeşitli filtreler çalışır (örneğin JWT doğrulama filtresi, `UsernamePasswordAuthenticationFilter`, exception çeviren filtreler vb.).
3. Uygun filtre, kullanıcının kimliğini doğrular (`Authentication` nesnesi oluşturur) ve `SecurityContext`’e koyar.
4. Request daha sonra `DispatcherServlet`’e ve oradan uygun controller’a yönlendirilir.
5. Authorization aşamasında, `SecurityContext`’teki kullanıcı bilgileri ve rollerine göre erişim kontrolü yapılır.

Mantık:  
Güvenlik, uygulamanın en dış katmanlarından birinde (filter seviyesinde) uygulanarak istenmeyen istekler controller seviyesine bile gelmeden durdurulur.  
Banka gibi hassas uygulamalarda bu kritik öneme sahiptir.

**Kod örneği (SecurityFilterChain config):**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}
```

---

### 5.2) JWT nasıl doğrulanır? Mantığı nedir?

JWT (JSON Web Token), stateless kimlik doğrulama için kullanılan token formatıdır.  
Genelde `Authorization: Bearer <token>` header’ında taşınır.

Doğrulama adımları:

1. Header’dan token alınır.
2. Token `header.payload.signature` olarak üç parçaya ayrılır.
3. Sunucudaki secret key veya public key ile **signature doğrulanır**.
   - Eğer token’ın içeriği (payload) değiştirilmişse signature tutmaz ve token geçersiz sayılır.
4. Token’ın süresi (`exp` claim’i) kontrol edilir; süresi dolmuş token reddedilir.
5. Gerekli claim’ler (kullanıcı adı, roller, yetkiler vb.) okunur.
6. Token geçerliyse bir `Authentication` nesnesi oluşturulur ve `SecurityContext`’e konur.

Mantık:  
Sunucu, her istek için veritabanına gidip session kontrolü yapmak yerine JWT içindeki imzalı, değiştirilmesi zor bilgiyi kullanır.  
Bu da özellikle microservice ve yüksek trafikli sistemlerde ölçeklenebilir, stateless bir güvenlik modeli sağlar.

**Kod örneği (basit JWT filter iskeleti):**

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
                                    throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            // 1) Token'ı çöz
            // 2) İmzayı secret key ile doğrula
            // 3) Süre (exp) dolmuş mu kontrol et
            // 4) Kullanıcı bilgilerini al ve Authentication oluştur
            UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(username, null, authorities);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        filterChain.doFilter(request, response);
    }
}
```

---

### 5.3) Authentication vs Authorization farkı

**Authentication (Kimlik Doğrulama):**

- “Sen kimsin?” sorusuna cevap verir.
- Örneğin kullanıcı adı/şifre doğrulama, JWT token’in geçerli olup olmadığının kontrolü.

**Authorization (Yetkilendirme):**

- “Bu kişi ne yapabilir?” sorusuna cevap verir.
- Örneğin kullanıcının admin rolü olup olmadığı, belirli bir endpoint’e veya kaynağa erişip erişemeyeceği.

Mantık:  
Önce kullanıcının **kim** olduğundan emin olursun (authentication),  
sonra bu kullanıcının hangi işlemleri yapmaya **yetkili** olduğunu kontrol edersin (authorization).  
Bankacılıkta her iki adım da son derece sıkı politikalarla yönetilir.

**Kod örneği (method-level authorization):**

```java
@Service
public class AccountService {

    @PreAuthorize("hasRole('ADMIN')")
    public void closeAccount(Long id) {
        // Sadece ADMIN rolündeki kullanıcılar bu metodu çağırabilir.
    }
}
```
