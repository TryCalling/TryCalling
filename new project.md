# 🧺🚚 Laundry & Water/Gas Delivery – Superapp (KH Market)

**Focus:** ABA PayWay & KHQR · **Frontend:** Flutter (Riverpod, GoRouter, Dio) · **Backend:** **Java Spring Boot** · **DB:** Postgres · **ORM:** JPA/Hibernate + Flyway · **Infra:** Redis · Docker

> Single source of truth to bootstrap the whole stack quickly and safely. Copy–paste friendly snippets included.

---

## 0) TL;DR

* **Use-cases:** Laundry (pickup → wash → deliver); Water/Gas delivery (now/later)
* **Actors:** Customer, Rider/Driver, Vendor/Branch, Admin/Ops
* **Payments:** ABA PayWay (redirect/WebView + webhook), KHQR (dynamic QR + webhook; fallback polling)
* **Security:** JWT + refresh, webhook signature verify, idempotency, Android **FLAG\_SECURE** on payment screens

---

## 1) High-Level Architecture

```
[ Flutter App ]  --HTTPS/JSON-->  [ Spring Boot API ]  --SQL-->  [ Postgres ]
        |                             |   \
        |                             |    \-- Redis (jobs/cache, pub/sub)
        |                             |    \-- S3/MinIO (files: receipts, weight slips)
        |                             |
        |                             |-->  ABA PayWay  <---(webhook)---|
        |                             |-->  KHQR / PSP  <---(webhook)---|
        |                                                         ^
        '---- Deep Link (ABA return) <-----------------------------'
```

### Backend (Detailed)

* **API Layer (Spring MVC):** versioned routes `/v1/**`, Swagger via springdoc-openapi at `/swagger-ui`.
* **Modules:**

  * `auth` – OTP login, JWT/refresh, rate limiting.
  * `catalog` – services (laundry/water/gas), pricing base, vendor radius.
  * `orders` – draft → final price → lifecycle state machine.
  * `payments` – **ABA** (HMAC-sign, checkout URL, webhook verify), **KHQR** (dynamic QR, webhook/poll), idempotency store.
  * `assignments` – auto-assign driver (distance + load + rating), reassign on timeout.
  * `ratings` – post-delivery feedback.
* **Persistence:** Postgres (JPA/Hibernate), Flyway migrations, indexes on `orders(order_no)`, `payments(intent_id)`, geo indexes (`vendors(lat,lng)` if PostGIS planned).
* **Cache/Jobs:** Redis for short-lived caches (catalog, OTP throttles) + background jobs (assignment, payment reconciler). Optional pub/sub for driver live updates.
* **Files:** S3/MinIO for weight slips, receipts (PDF/IMG). Store pointer in DB.
* **Observability:** Actuator health/metrics, structured logs (JSON), tracing (OpenTelemetry), dashboards (Prometheus/Grafana) — optional in prod.
* **Security:** JWT stateless, CORS allowlist, input validation (Bean Validation), webhook signature verification (ABA/KHQR), idempotency keys per webhook event, Android **FLAG\_SECURE** UI guidance documented for mobile.
* **Scalability:** Stateless API nodes (horizontal scale), Redis-backed background workers, DB connection pool sizing, backpressure on async jobs.
* **Environments:** `dev`, `staging`, `prod` with separate secrets, endpoints, and payment sandboxes.

### Frontend (Detailed)

* **Architecture pattern:** Feature-first clean structure (Domain–Data–Presentation) with **Riverpod** for state.

  * **Domain:** entities, repositories (abstract), use-cases.
  * **Data:** Dio clients, DTOs (**freezed/json\_serializable**), repositories impl, mappers, caching (Hive).
  * **Presentation:** feature folders (auth/catalog/checkout/payment/tracking/profile), widgets, theming.
* **Navigation:** **GoRouter** with guarded routes (auth-only, payment-only screens), deep links (`myapp://payment_result?...`) and unknown route handling.
* **Networking:** **Dio** with interceptors (auth header, refresh token, retry/backoff, logging), consistent error envelope parsing.
* **Storage:** `flutter_secure_storage` (tokens), **Hive** (cart, address, last intents), offline queue for failed requests.
* **Payments UI:**

  * **ABA:** open `checkoutUrl` via `webview_flutter` or `url_launcher`, listen deep link with `uni_links`, always confirm server state with `GET /orders/{id}`.
  * **KHQR:** display QR (from API) + countdown, poll `/payments/{id}` every 4s (fallback) until `paid/expired`.
* **Location & Tracking:** permissions flow, background location for driver app (future), live status via polling or SSE/WebSocket (optional phase 2).
* **Internationalization:** Khmer/English (`intl`), number/date/currency formatting (KHR/USD), RTL-safe layout.
* **Quality:** widget tests (goldens for key screens), integration tests for payment flows, analytics hooks on critical events.
* **Build flavors:** `dev`, `stg`, `prod` with `--dart-define` (`API_BASE`, `SENTRY_DSN`, etc.).

### Cross-Cutting

* **API Contracting:** OpenAPI-first (springdoc), shared DTO naming, ISO-8601 UTC for timestamps, money as decimal with currency field, stable `order_no` reference across systems.
* **Error Model:**

  ```json
  {
    "error": {"code":"PAYMENT_WEBHOOK_INVALID","message":"Signature mismatch","traceId":"..."}
  }
  ```
* **Security Posture:** least-privilege DB roles, rotate secrets, rate-limit `/auth` & webhooks, audit trail (immutable ledger of payment events).

---

## 1.1 Project Folder Architecture (Monorepo)

```
repo-root/
  backend/                      # Spring Boot service
    src/main/java/com/yourco/washdrop/... (modules)
    src/main/resources/
      application.yml
      db/migration/             # Flyway SQL (V1__init.sql ...)
    pom.xml                     # or build.gradle
    Dockerfile
    README.md

  app/                          # Flutter mobile app
    lib/
      core/                     # theme, constants, helpers
      data/                     # dio clients, dto, repos impl
      domain/                   # entities, repos (abstract), usecases
      presentation/
        features/
          auth/
          catalog/
          checkout/
          payment/
          tracking/
          profile/
        widgets/
      app.dart, main.dart
    assets/                     # icons, lottie, fonts
    test/                       # unit/widget/integration tests
    analysis_options.yaml
    pubspec.yaml

  infra/
    docker-compose.yml          # postgres, redis, loki/promtail (optional)
    k8s/                        # manifests (optional)
    grafana-prometheus/         # observability stack (optional)

  docs/
    architecture.md             # extended diagrams
    api-contracts/              # OpenAPI exports
    adr/                        # Architecture Decision Records (ADR-001-...)

  scripts/
    dev.sh                      # spin up local stack
    db/seed.sql                 # sample data

  .github/workflows/
    ci-backend.yml              # build, test, flyway validate, docker
    ci-app.yml                  # flutter analyze, test, build

  .env.example                  # root env references
  README.md                     # product-level readme (this file)
```

**Why monorepo?** One PR spans mobile + backend contracts; easier versioning and CI. If you prefer split repos, keep `docs/api-contracts` as the shared source of truth and publish a versioned OpenAPI artifact.

**Module Boundaries & Ownership**

* Backend modules own their tables + APIs; cross-module calls via services, not repositories.
* Frontend features own their routers & providers; share only via `core` helpers and typed DTOs.

---

## 2) Domain Model (ERD – Minimal Viable)

(ERD – Minimal Viable)

```
users (id, name, phone, email, role[customer|driver|admin], status)
addresses (id, user_id -> users.id, label, lat, lng, details, is_default)
vendors (id, name, branch_code, lat, lng, radius_km)
services (id, type[laundry|water|gas], name, unit[kg|item], base_price, add_on_json)
inventory (id, vendor_id -> vendors.id, sku, name, unit, stock_qty, price) -- for water/gas
orders (id, order_no, user_id -> users.id, vendor_id -> vendors.id, type,
        subtotal, delivery_fee, discount, total, currency,
        payment_method[ABA|KHQR|COD], payment_status[pending|paid|failed|refunded],
        order_status[draft|placed|assigned|picked_up|in_process|ready|delivering|delivered|cancelled],
        address_id -> addresses.id, scheduled_at, created_at)
order_items (id, order_id -> orders.id, service_id -> services.id (nullable when sku), sku,
             qty, unit_price, amount, meta_json)
riders (id, user_id -> users.id, vehicle_type, capacity, active)
assignments (id, order_id -> orders.id, rider_id -> riders.id, status[assigned|accepted|rejected|enroute|arrived|completed])
payments (id, order_id -> orders.id, provider[ABA|KHQR], intent_id, checkout_url,
          amount, currency, provider_status, provider_ref, qr_payload, qr_image_url,
          expires_at, webhook_payload, verified_at)
ratings (id, order_id -> orders.id, user_id -> users.id, score, comment)
```

> **Laundry weight slip** → keep inside `order_items.meta_json` (e.g., `{kg:7.4, surcharge:1.5, stains:[...]}`).

---

## 3) API Design (REST – v1)

### Auth

* `POST /auth/login` (OTP mock or SMS provider) → `{accessToken, refreshToken}`
* `POST /auth/refresh` → rotate access token

### Catalog

* `GET /catalog/services?type=laundry|water|gas`
* `GET /vendors?lat=..&lng=..` (optional proximity)

### Orders

* `POST /orders` → create **draft**
* `POST /orders/{id}/price` → **final price** (laundry weight update)
* `POST /orders/{id}/assign` → auto-assign driver (or manual)
* `GET /orders/{id}` / `GET /orders?status=...`
* `PATCH /orders/{id}/status` → ops/driver progression

### Payments

* `POST /pay/aba/intent` → ABA intent (server-side)
* `POST /pay/khqr/intent` → **dynamic KHQR**
* `POST /pay/webhook/aba` → ABA webhook (server-to-server)
* `POST /pay/webhook/khqr` → KHQR webhook (server-to-server)
* `GET /payments/{id}` → check status (poll fallback)

### Ratings

* `POST /ratings` → score/comment

**Status Machines**

* Payment: `pending → processing → paid | failed`
* Order (laundry): `placed → assigned → picked_up → in_process → ready → delivering → delivered`

---

## 4) Sequence Diagrams

### ABA PayWay (redirect)

```
App -> API: POST /orders (draft)
App -> API: POST /pay/aba/intent {order_id, amount}
API -> ABA: create checkout (HMAC)
ABA -> API: {checkout_url}
API -> App: {checkout_url}
App -> ABA: open WebView/External
ABA -> App: deep link myapp://payment_result?status=success&order=...
ABA -> API: webhook /pay/webhook/aba (signed)
API -> DB: payments.provider_status=success; orders.payment_status=paid
App -> API: GET /orders/{id} (refresh)
```

### KHQR (dynamic QR)

```
App -> API: POST /pay/khqr/intent {order_id, amount}
API -> PSP: request dynamic QR
PSP -> API: {qr_payload, image_url, ref, expires_at}
API -> App: return QR data
App: show QR + countdown; poll /payments/{id}
PSP -> API: webhook /pay/webhook/khqr
API -> DB: mark paid + set verified_at
App -> API: GET /payments/{id} -> paid -> success
```

---

## 5) Backend (Spring Boot) – Project Structure

```
backend/
  src/main/java/com/yourco/washdrop/
    common/               # utils, errors, ApiResponse
    config/               # CORS, Jackson, MapStruct, Cache
    security/             # JWT filters, WebSecurityConfig
    auth/                 # OTP/login, token endpoints
    users/
    vendors/
    catalog/
    orders/
    assignments/
    payments/
      aba/                # ABA client + signer + DTO
      khqr/               # KHQR client + DTO
      webhook/            # webhook controllers
    ratings/
  src/main/resources/
    application.yml
    db/migration/         # Flyway SQL migrations V1__init.sql ...
  pom.xml (or build.gradle)
```

### 5.1 Maven Dependencies (pom.xml excerpt)

```xml
<dependencies>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-web</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-validation</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-security</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-cache</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-actuator</artifactId></dependency>
  <dependency><groupId>org.springdoc</groupId><artifactId>springdoc-openapi-starter-webmvc-ui</artifactId><version>2.6.0</version></dependency>
  <dependency><groupId>org.postgresql</groupId><artifactId>postgresql</artifactId></dependency>
  <dependency><groupId>org.springframework.boot</groupId><artifactId>spring-boot-starter-data-redis</artifactId></dependency>
  <dependency><groupId>io.jsonwebtoken</groupId><artifactId>jjwt-api</artifactId><version>0.11.5</version></dependency>
  <dependency><groupId>io.jsonwebtoken</groupId><artifactId>jjwt-impl</artifactId><version>0.11.5</version><scope>runtime</scope></dependency>
  <dependency><groupId>io.jsonwebtoken</groupId><artifactId>jjwt-jackson</artifactId><version>0.11.5</version><scope>runtime</scope></dependency>
  <dependency><groupId>org.projectlombok</groupId><artifactId>lombok</artifactId><optional>true</optional></dependency>
  <dependency><groupId>org.mapstruct</groupId><artifactId>mapstruct</artifactId><version>1.5.5.Final</version></dependency>
</dependencies>
```

### 5.2 JPA Entities (excerpt)

```java
// User.java
@Entity @Table(name = "users")
@Data @NoArgsConstructor @AllArgsConstructor
public class User {
  @Id @GeneratedValue(strategy = GenerationType.UUID)
  private String id;
  private String name;
  @Column(unique = true) private String phone;
  @Column(unique = true) private String email;
  @Enumerated(EnumType.STRING) private Role role = Role.customer;
  private String status = "active";
  @OneToMany(mappedBy = "user") private List<Address> addresses = new ArrayList<>();
}
public enum Role { customer, driver, admin }
```

```java
// Order.java
@Entity @Table(name = "orders")
@Data @NoArgsConstructor @AllArgsConstructor
public class Order {
  @Id @GeneratedValue(strategy = GenerationType.UUID)
  private String id;
  @Column(unique = true) private String orderNo;
  @ManyToOne @JoinColumn(name = "user_id") private User user;
  @ManyToOne @JoinColumn(name = "vendor_id") private Vendor vendor;
  @Enumerated(EnumType.STRING) private ServiceType type;
  @Column(precision = 10, scale = 2) private BigDecimal subtotal;
  @Column(precision = 10, scale = 2) private BigDecimal deliveryFee;
  @Column(precision = 10, scale = 2) private BigDecimal discount;
  @Column(precision = 10, scale = 2) private BigDecimal total;
  private String currency; // KHR|USD
  @Enumerated(EnumType.STRING) private PaymentMethod paymentMethod;
  @Enumerated(EnumType.STRING) private PaymentStatus paymentStatus = PaymentStatus.pending;
  @Enumerated(EnumType.STRING) private OrderStatus orderStatus = OrderStatus.placed;
  @ManyToOne @JoinColumn(name = "address_id") private Address address;
  private Instant scheduledAt;
  private Instant createdAt = Instant.now();
  @OneToMany(mappedBy = "order", cascade = CascadeType.ALL) private List<OrderItem> items = new ArrayList<>();
}
public enum ServiceType { laundry, water, gas }
public enum PaymentMethod { ABA, KHQR, COD }
public enum PaymentStatus { pending, processing, paid, failed, refunded }
public enum OrderStatus { draft, placed, assigned, picked_up, in_process, ready, delivering, delivered, cancelled }
```

```java
// Payment.java
@Entity @Table(name = "payments")
@Data
public class Payment {
  @Id @GeneratedValue(strategy = GenerationType.UUID)
  private String id;
  @ManyToOne @JoinColumn(name = "order_id") private Order order;
  @Enumerated(EnumType.STRING) private PaymentProvider provider; // ABA|KHQR
  private String intentId;
  private String checkoutUrl;
  @Column(precision = 10, scale = 2) private BigDecimal amount;
  private String currency;
  private String providerStatus;
  private String providerRef;
  @Column(columnDefinition = "text") private String qrPayload;
  private String qrImageUrl;
  private Instant expiresAt;
  @Column(columnDefinition = "text") private String webhookPayload;
  private Instant verifiedAt;
}
public enum PaymentProvider { ABA, KHQR }
```

### 5.3 Controllers (Spring MVC – excerpt)

```java
@RestController
@RequestMapping("/v1/pay")
@RequiredArgsConstructor
public class PaymentsController {
  private final PaymentsService paymentsService;

  @PostMapping("/aba/intent")
  @PreAuthorize("hasRole('customer')")
  public PaymentIntentDto createAba(@Valid @RequestBody CreatePaymentDto dto, Authentication auth) {
    return paymentsService.createAbaIntent(dto.getOrderId(), auth.getName());
  }

  @PostMapping("/webhook/aba")
  @PermitAll
  public ResponseEntity<String> abaWebhook(@RequestHeader Map<String,String> headers, @RequestBody String body) {
    paymentsService.handleAbaWebhook(headers, body);
    return ResponseEntity.ok("OK");
  }
}
```

```java
@RestController
@RequestMapping("/v1/orders")
@RequiredArgsConstructor
public class OrdersController {
  private final OrdersService ordersService;

  @PostMapping
  @PreAuthorize("hasRole('customer')")
  public OrderDto create(@Valid @RequestBody CreateOrderDto dto, Authentication auth) {
    return ordersService.createDraft(dto, auth.getName());
  }

  @PostMapping("/{id}/assign")
  @PreAuthorize("hasRole('admin')")
  public AssignmentDto autoAssign(@PathVariable String id) {
    return ordersService.autoAssign(id);
  }
}
```

### 5.4 Security (JWT) – essentials

```java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {
  private final JwtAuthFilter jwtAuthFilter;

  @Bean SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.csrf(csrf -> csrf.disable())
       .authorizeHttpRequests(auth -> auth
         .requestMatchers("/v1/pay/webhook/**", "/v3/api-docs/**", "/swagger-ui/**").permitAll()
         .anyRequest().authenticated())
       .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
       .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
  }
}
```

### 5.5 Flyway Migration (V1\_\_init.sql excerpt)

```sql
CREATE TABLE users (
  id uuid PRIMARY KEY,
  name text NOT NULL,
  phone text UNIQUE NOT NULL,
  email text UNIQUE,
  role text NOT NULL DEFAULT 'customer',
  status text NOT NULL DEFAULT 'active'
);

CREATE TABLE vendors (
  id uuid PRIMARY KEY,
  name text NOT NULL,
  branch_code text UNIQUE NOT NULL,
  lat double precision NOT NULL,
  lng double precision NOT NULL,
  radius_km double precision NOT NULL DEFAULT 5
);
/* + tables: addresses, services, inventory, orders, order_items, payments, riders, assignments, ratings */
```

### 5.6 Payments Service (sketch)

```java
@Service @RequiredArgsConstructor
public class PaymentsService {
  private final AbaClient abaClient;  // wraps HMAC signing & HTTP
  private final PaymentRepository repo;
  private final OrdersRepository orders;

  @Transactional
  public PaymentIntentDto createAbaIntent(String orderId, String userId) {
    Order order = orders.findByIdOrThrow(orderId);
    AbaCheckoutSession ses = abaClient.createCheckout(order.getOrderNo(), order.getTotal(), order.getCurrency());
    Payment p = new Payment();
    p.setOrder(order); p.setProvider(PaymentProvider.ABA);
    p.setIntentId(ses.id()); p.setCheckoutUrl(ses.url());
    p.setAmount(order.getTotal()); p.setCurrency(order.getCurrency());
    repo.save(p);
    return PaymentIntentDto.from(p);
  }

  @Transactional
  public void handleAbaWebhook(Map<String,String> headers, String body) {
    if (!abaClient.verify(headers, body)) throw new AccessDeniedException("bad signature");
    AbaWebhook evt = abaClient.parse(body);
    Payment p = repo.findByIntentId(evt.intentId());
    p.setProviderStatus("success"); p.setProviderRef(evt.txnId()); p.setVerifiedAt(Instant.now());
    repo.save(p);
    // also mark order paid
  }
}
```

---

## 6) Mobile (Flutter) – Project Structure

```
app/
  lib/
    main.dart
    app.dart                  // ProviderScope + GoRouter
    core/                     // theme, constants, helpers
    data/                     // models(freezed), api(dio), repos impl
    domain/                   // entities, repos abstract, usecases
    presentation/
      features/
        auth/
        catalog/
        checkout/
        payment/
        tracking/
        profile/
      widgets/
```

### 6.1 Packages

`go_router`, `flutter_riverpod`, `dio`, `freezed`, `json_serializable`, `hive`, `flutter_secure_storage`, `intl`, `cached_network_image`, `qr_flutter`, `webview_flutter`/`url_launcher`

### 6.2 KHQR Polling Page (pseudo)

* Use a `FutureProvider` to create intent
* Poll `/payments/{id}` every \~4s with a `Timer`; navigate to success when `paid`

### 6.3 ABA WebView

* `webview_flutter` or `url_launcher`
* Deep link via `uni_links`; after return always `GET /orders/{id}` to trust webhook

---

## 7) Config

### 7.1 Backend `application.yml`

```yaml
server:
  port: 3001
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/washdrop
    username: postgres
    password: postgres
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        jdbc.lob.non_contextual_creation: true
  flyway:
    enabled: true
  redis:
    host: localhost
    port: 6379
jwt:
  secret: change_me
  refreshSecret: change_me_too
payments:
  aba:
    merchantId: your_id
    apiKey: your_key
    hmacSecret: your_hmac
    checkoutBase: https://checkout-sandbox.abapayments.com
    returnUrl: https://your.api/pay/aba/return
    webhookSecret: shared_webhook_secret
  khqr:
    apiBase: https://psp-sandbox.example.com
    merchantId: your_mid
    apiKey: your_key
    webhookSecret: shared_webhook_secret
```

### 7.2 Flutter run

```bash
flutter run --dart-define=API_BASE=http://10.0.2.2:3001
```

---

## 8) Docker (local infra)

```yaml
# backend/docker-compose.yml
version: '3.9'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: washdrop
    ports: ["5432:5432"]
  redis:
    image: redis:7
    ports: ["6379:6379"]
```

---

## 9) Payment Integration Details

**ABA PayWay**

* Sign payload (HMAC) with `payments.aba.hmacSecret`
* Fields: `merchant_id`, `amount`, `currency`, `order_id`, `return_url`
* Store `intentId` + `checkoutUrl`
* **Mark paid only on webhook**; deep-link is UX only
* Use **idempotency** when processing webhooks

**KHQR (Dynamic)**

* Request QR with **exact amount**, currency (KHR/USD), unique reference (e.g., `ORD-2025-000123`)
* Return `qr_payload`, `qrImageUrl`, `expiresAt`
* Prefer webhook; else poll `/payments/{id}`

**Anti-Fraud**

* Lock order editing after intent creation
* Short-lived QR (5–15 min)
* Verify webhook signature & log raw bodies

---

## 10) Security Checklist

* JWT access (short) + refresh (long), rotate on suspicious activity
* Store tokens in **SecureStorage** (mobile)
* Payment screens: Android **FLAG\_SECURE**; iOS screen capture handling if needed
* Rate limit `/auth` & `/pay/webhook/*`
* DTO validation (Bean Validation), sanitize outputs
* Observability: request/response + payment events audit (immutable ledger)

---

## 11) Analytics & Growth

* Events: `search_service`, `select_slot`, `start_payment`, `payment_success`, `order_delivered`
* Push: pickup/delivery reminders; payment failure nudges
* Promo: first order discount, referral, coupon (%, fixed)

---

## 12) QA & Edge Cases

* Payment success but deep-link lost → rely on webhook + auto refresh
* Rider cancels → auto-reassign
* Inventory low (gas/water) → backorder/reschedule
* Laundry weight change → re-price before payment

---

## 13) OpenAPI

Use **springdoc-openapi**: Swagger UI at `/swagger-ui`.
Annotate controllers with `@Operation(summary = "...")` as needed.

---

## 14) Local Dev: Commands

**Backend**

```bash
# in backend/
mvn -v
docker compose up -d                # Postgres + Redis
mvn spring-boot:run                 # Flyway auto-runs migrations
```

**Mobile**

```bash
# in app/
flutter pub get
flutter run --dart-define=API_BASE=http://10.0.2.2:3001
```

---

## 15) Roadmap (6 Weeks)

* **W1:** API scaffold + DB; Auth; Catalog; Address
* **W2:** Orders draft→price; schedule picker; cart state
* **W3:** ABA integration + webhook + deep link
* **W4:** KHQR dynamic + polling fallback; driver assign + tracking
* **W5:** Laundry weight update; inventory (water/gas); receipts
* **W6:** QA; analytics; retries; release (Play Store internal, TestFlight internal)

---

## 16) Go-Live Checklist

* ✅ Webhook endpoints secured (shared secret / signature verify)
* ✅ Payment idempotency & retries
* ✅ SLA monitoring for delivery windows
* ✅ Error/empty states + retry buttons
* ✅ Logs & alerts for payment failures
* ✅ Privacy policy; T\&C; refund policy

---

## 17) Driver Auto-Assign (logic sketch)

```java
// choose best driver by distance, load, rating
List<Rider> candidates = ridersRepo.findActiveWithinRadius(vendorLat, vendorLng);
Rider best = candidates.stream()
  .map(r -> new Scored(r, w1*distance(addr, r.loc()) + w2*currentLoad(r) + w3*(5 - r.rating())))
  .sorted(Comparator.comparing(Scored::score)).findFirst().get();
assignmentsRepo.assign(orderId, best.rider().id());
```

---

> Want a **Spring Boot starter** (entities, DTOs, controllers, Flyway V1, JWT, ABA/KHQR client stubs) and a **Flutter starter** (GoRouter + Riverpod + Dio, ABA WebView + KHQR screen)? I can generate both next.
