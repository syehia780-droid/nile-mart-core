# NileMart

A small online shop that runs **in the console and in the browser at the same time**, built with plain
**Spring Framework 6** (no Spring Boot). The goal of the project is to understand *who creates the objects and who
connects them* (IoC and Dependency Injection), and how a web page reaches those objects (Spring MVC).

## What it does

Customers open carts, add products from a catalog, and check out. The shop adds 14% VAT and a shipping fee that
depends on the governorate, reduces the stock for all lines or none, saves an order numbered `NM-YYYYMMDD-NNNNN`,
and sends an SMS and an email confirmation.

## Tech

| Piece | Used for |
|---|---|
| Java 17+ | records, enums, streams, `Optional`, text blocks |
| Spring Context 6 | IoC container, dependency injection, scopes, lifecycle callbacks |
| Spring Web + Spring MVC 6 | `@Controller`, `DispatcherServlet`, views, `redirect:` |
| Thymeleaf | HTML pages (given) |
| Embedded Tomcat | web server started from `main`, port 8080 |
| Lombok | less boilerplate (never on domain logic) |
| Maven | build |

## Run it

```bash
mvn compile exec:java
```

Then open <http://localhost:8080/>. The console menu runs in the same terminal. Option `0` stops the web server and
closes the Spring container (the `@PreDestroy` messages print).

Requirements: JDK 17 or newer, Maven 3.9+.

## Spring ideas used, by stage

| Stage | What | Spring ideas |
|---|---|---|
| 1 | Catalog, pricing, `ProductService` wired by hand with `new` | interfaces, constructor injection, loose coupling |
| 2 | The container builds the graph | `@Configuration`, `@ComponentScan`, stereotypes, `@Value`, `@PropertySource` |
| 3 | Shipping and notifications | `@Primary`, `@Qualifier`, `List<T>` injection, `@Order` |
| 4 | One cart per customer | `prototype` scope, `@PostConstruct`, `@PreDestroy` |
| 5 | Checkout and the console | `@Bean` (for a JDK class), everything together |
| 6 | The web pages | `@Controller`, `@GetMapping`/`@PostMapping`, `Model`, Post/Redirect/Get, `@ControllerAdvice`, `ObjectProvider` |

## Architecture

```
 Browser ──► DispatcherServlet ──► Controllers ─┐
                                                ├──► CartRegistry, CheckoutService, ProductService, OrderRegistry
 Console (StoreConsole) ────────────────────────┘                     │
                                                       ShippingQuoteService, TaxCalculator, NotificationSenders
                                                                       │
                                                           ProductRepository (in memory, loaded from CSV)
```

Both "front doors" use the same beans from the same container, so a cart created in the console shows up in the
browser and the other way around.

## Project layout

```
org.route
├─ Main, WebServer
├─ config/        AppConfig, WebConfig
├─ common/        StoreException family, Money, Check
├─ model/         Product, Category, Customer, Order, OrderLine, Zone, Governorate
├─ repository/    Repository<T,ID>, ProductRepository, InMemoryProductRepository
├─ pricing/       PricingService + Standard / Promotional
├─ shipping/      ShippingCalculator + Zone / Free, ShippingQuoteService
├─ notification/  NotificationSender, AbstractNotificationSender, Sms / Email
├─ service/       ProductService
├─ cart/          ShoppingCart, CartRegistry, CartView, LineView
├─ checkout/      CheckoutService, TaxCalculator, OrderRegistry, Quote
├─ controller/    Product-, Cart-, Checkout-, Order-Controller, GlobalExceptionHandler (+ form/)
└─ console/       StoreConsole
```

## Business rules in one minute

* Money is `BigDecimal`, 2 decimals, `HALF_UP`.
* VAT 14%. Shipping by zone (50 / 65 / 70 / 85 / 120 EGP); free from 5,000.00 EGP, except the `REMOTE` zone.
* Maximum 10 per product in a cart, never more than the stock. A quantity below 1 is rejected.
* Checkout re-checks the stock of every line, then reduces all lines or none.
* Every `POST` ends with a redirect to a `GET` page (Post/Redirect/Get).
* Every `StoreException` shows the error page with its own HTTP status: 400, 404 or 409.

## Troubleshooting

* `Name for argument of type [int] not specified` → compile with `-parameters` (already set in `pom.xml`; in an
  IDE, reimport the Maven project).
* `Port 8080 already in use` → stop the other program that uses the port.
