---
name: apized
description: This skill should be used when the user asks to build an API, add a model, add an endpoint, create a new entity, implement a behavior, configure security, add federation, write a repository extension, or work with any part of the apized framework. Activates on questions like "how do I add a new model", "create a REST endpoint", "add a behavior", "configure permissions", or "use @Apized".
metadata:
  version: 1.1.0
  last_synced_commit: 43f59a4a0a13da2eb5fda99aff4fe2da4d0f8a2e
---

# Apized Framework Guide

Apized is an annotation-driven JVM framework that auto-generates REST API infrastructure (controller, service, repository, deserializer) from a single annotated model class.

## Quick Reference

| Section | What it covers |
|---|---|
| [Gradle Setup](#gradle-setup) | Plugin + optional module dependencies |
| [Model Definition](#model-definition) | `@Apized`, `BaseModel`, scope hierarchy, `layers`, `operations` |
| [Service Extensions](#service-extensions) | Shared logic between behaviours |
| [Repository Extensions](#repository-extensions) | Custom queries, dynamic finders, `@Query` |
| [Behaviours](#behaviours) | Lifecycle hooks, ordering, touched fields, original state |
| [Context Access](#context-access) | `ApizedContext` — request, security, audit, events |
| [REST Query Features](#rest-query-features) | `?fields=`, `?search=`, pagination |
| [Security & Permissions](#security--permissions) | Permission format, UserResolver, `@Owner`, enrichers, filters |
| [Federation](#federation) | Cross-API references with `@Federation` |
| [Audit & Events](#audit--events) | Automatic trails, custom events, RabbitMQ config |
| [Controller Extensions](#controller-extensions) | Override generated actions |
| [Behaviour Pipeline on Custom Endpoints](#triggering-the-behaviour-pipeline-from-custom-endpoints) | `@MicronautBehaviourExecution` / `@SpringBehaviourExecution` |
| [Custom Controllers](#custom-controllers) | Fully custom endpoints |
| [Tracing](#tracing-module) | `@Traced` + OpenTelemetry |
| [Distributed Lock](#distributed-lock-module) | `LockFactory` + ShedLock |
| [Testing](#test-module) | Cucumber BDD integration tests |
| [MCP Integration](#mcp-integration-micronaut-mcp--spring-mcp) | Auto-generated AI tool endpoints via `mcp = true` |

## Core Concept

Annotate a model with `@Apized` → annotation processor generates all layers at compile time.

## Gradle Setup (consumer project)

**Micronaut:**
```gradle
plugins {
  id 'org.apized.micronaut' version "$apizedVersion"
}

dependencies {
  annotationProcessor 'io.micronaut.openapi:micronaut-openapi'              // optional: OpenAPI docs
  implementation "org.apized:micronaut-tracing:$apizedVersion"              // optional: OTEL tracing
  implementation "org.apized:micronaut-messaging-rabbitmq:$apizedVersion"   // optional: RabbitMQ events
  implementation "org.apized:micronaut-distributed-lock:$apizedVersion"     // optional: distributed locks
  implementation "org.apized:micronaut-mcp:$apizedVersion"                  // optional: MCP tool generation
}
```

The Micronaut plugin reads `apized.properties` at the project root to select the SQL dialect and automatically add the matching JDBC driver and Flyway dependency:

```properties
# apized.properties
dialect=POSTGRES   # H2 | MYSQL | POSTGRES | SQL_SERVER | ORACLE | ANSI (default)
```

**Spring:**
```gradle
plugins {
  id 'org.apized.spring' version "$apizedVersion"
}

dependencies {
  implementation "org.apized:spring-tracing:$apizedVersion"              // optional: OTEL tracing
  implementation "org.apized:spring-messaging-rabbitmq:$apizedVersion"   // optional: RabbitMQ events
  implementation "org.apized:spring-distributed-lock:$apizedVersion"     // optional: distributed locks
  implementation "org.apized:spring-mcp:$apizedVersion"                  // optional: MCP tool generation
}
```

The apized Gradle plugin handles annotation processor wiring and code generation configuration automatically.

## Model Definition

```java
@Entity
@Getter @Setter
@Apized(
  layers = {Layer.CONTROLLER, Layer.SERVICE, Layer.REPOSITORY},
  operations = {Action.LIST, Action.GET, Action.CREATE, Action.UPDATE, Action.DELETE},
  audit = true,
  event = true,
  maxPageSize = 50,
  mcp = true,
  extensions = MyRepositoryExtension.class
)
public class Product extends BaseModel {
  @NotBlank
  private String name;

  private BigDecimal price;

  @ManyToOne
  private Category category;

  @OneToMany(mappedBy = "product", orphanRemoval = true)
  private List<Review> reviews;
}
```

**Rules:**
- Extend `BaseModel` (provides `id` UUID PK, `version` Long for optimistic locking — if the client sends it the update will fail if stale (useful for sensitive operations like decrementing a stock counter); if omitted the framework applies a request-scoped optimistic lock, `createdBy`, `createdAt`, `lastUpdatedBy`, `lastUpdatedAt`, `metadata` JSON-like Map/List for unstructured data)
- Annotate with `@Entity` (JPA) and `@Apized`
- Use Lombok `@Getter @Setter`
- `layers` controls which classes are generated. Default: all three (`CONTROLLER`, `SERVICE`, `REPOSITORY`). Omit a layer to suppress it — e.g. `layers = {Layer.SERVICE, Layer.REPOSITORY}` generates no HTTP endpoints, useful for internal-only models.
- `operations` controls which CRUD endpoints are exposed. Default: all five (`LIST`, `GET`, `CREATE`, `UPDATE`, `DELETE`).
- Request bodies for `CREATE` and `UPDATE` are automatically validated with Bean Validation (`@Valid`). Annotate model fields with `@NotBlank`, `@NotNull`, `@Size`, etc. as needed.
- `@ManyToMany` relationships are managed automatically — the framework generates `add/remove` methods for the join table and calls them when the relationship field is included in a PUT request. No custom code needed.
- `mcp` — when `true` (default), generates a `{Type}McpTools` bean that exposes all enabled CRUD operations as MCP tool calls. Set `mcp = false` to suppress generation for a specific model. Requires `micronaut-mcp` / `spring-mcp` on the classpath to activate.
- `extensions` accepts an array — pass multiple extension classes: `extensions = {RepoExtension.class, ServiceExtension.class}`.
- Use `scope` in `@Apized` to define the parent model and establish a hierarchy (used for endpoint URL nesting and integration test tooling):

```java
@Entity
@Apized(scope = Organization.class)
public class Department extends BaseModel { ... }
```

## Service Extensions

Add custom methods to the generated service, or override standard CRUD actions. The extension must be a concrete bean — the generated service injects it and delegates calls to it.

```java
@Singleton   // @Component for Spring
@Apized.Extension(layer = Layer.SERVICE)
public class ProductServiceExtension {
  public BigDecimal calculateDiscountedPrice(Product product, double discountPct) {
    return product.getPrice().multiply(BigDecimal.valueOf(1 - discountPct));
  }
}
```

Reference it via `@Apized(extensions = ProductServiceExtension.class)`. The generated `ProductService` will inject this bean and expose `calculateDiscountedPrice` as a public method, available to all behaviours that inject `ProductService`.

Use `@Apized.Extension.Action` to replace a standard CRUD action entirely:

```java
@Singleton   // @Component for Spring
@Apized.Extension(layer = Layer.SERVICE)
public class ProductServiceExtension {
  @Inject ProductRepository repository;

  @Apized.Extension.Action(Action.LIST)
  public Page<Product> list(int page, int pageSize, List<SearchTerm> search, List<SortTerm> sort) {
    // custom list logic — replaces the generated list method
    return repository.findActiveProducts(page, pageSize);
  }
}
```

## Repository Extensions

Add custom queries by creating an interface and referencing it in `@Apized(extensions = ...)`:

```java
@Apized.Extension(layer = Layer.REPOSITORY)
public interface ProductRepositoryExtension {
  List<Product> findByCategory(UUID categoryId);

  Page<Product> findByPriceGreaterThan(BigDecimal price, Pageable pageable);
}
```

Use `exclude` to suppress generated methods you don't want (works on any layer):
```java
@Apized.Extension(layer = Layer.REPOSITORY, exclude = {"someGeneratedMethod"})
```

## Behaviours

Implement custom logic before/after any CRUD operation. Behaviours must be registered as beans — `@Singleton` in Micronaut, `@Component` in Spring:

```java
@Singleton   // @Component for Spring
@Behaviour(
  model = Product.class,
  layer = Layer.SERVICE,
  when = {When.BEFORE, When.AFTER},
  actions = {Action.CREATE, Action.UPDATE},
  order = 100   // lower runs first
)
public class ProductValidationBehaviour implements BehaviourHandler<Product> {

  // Inject dependencies normally — @Inject (Micronaut) or @Autowired (Spring)
  @Inject CategoryRepository categoryRepository;

  @Override
  public void preCreate(Execution<Product> execution, Product input) {
    // runs BEFORE create
  }

  @Override
  public void postCreate(Execution<Product> execution, Product output) {
    // runs AFTER create
  }

  // also: preList, postList, preGet, postGet, preUpdate, postUpdate, preDelete, postDelete
}
```

`Execution<T>` contains `id` (UUID), `input` (T), and `output` (T).

### Touched fields and original state

`_getModelMetadata()` exposes two key helpers inside behaviours:

```java
@Override
public void preUpdate(Execution<Order> execution, Order input) {
  // Check which fields were actually sent in the request
  if (input._getModelMetadata().getTouched().contains("password")) {
    input.setPassword(BCrypt.hashpw(input.getPassword(), BCrypt.gensalt()));
  }

  // Force a computed field to be persisted even if client didn't send it
  input.setVerificationCode(generateCode());
  input._getModelMetadata().getTouched().add("verificationCode");

  // Access the pre-update snapshot for delta detection
  Order original = (Order) input._getModelMetadata().getOriginal();
  if (!input.getStatus().equals(original.getStatus())) {
    // status changed — react to the transition
  }
}
```

### Behaviour execution order

The `order` parameter controls execution sequence across all behaviours for the same model+action. **Lower values run first; negative values run before positive ones:**

```java
@Behaviour(model = Order.class, when = When.BEFORE, actions = Action.CREATE, order = -1000)
public class GenerateReference implements BehaviourHandler<Order> { ... }  // runs 1st

@Behaviour(model = Order.class, when = When.BEFORE, actions = Action.CREATE, order = -999)
public class EnrichFares implements BehaviourHandler<Order> { ... }  // runs 2nd

@Behaviour(model = Order.class, when = When.BEFORE, actions = Action.CREATE, order = 100)
public class ProcessPayment implements BehaviourHandler<Order> { ... }  // runs last
```

## Key Annotations Reference

| Annotation | Purpose |
|---|---|
| `@Apized` | Mark model for code generation |
| `@Apized.Extension` | Add custom methods to a generated layer |
| `@Behaviour` | Register a lifecycle hook |
| `@Federation` | Reference entity from an external API |
| `@PermissionEnricher` | Custom permission evaluation logic |
| `@AuditField` / `@AuditIgnore` | Control audit field tracking |
| `@EventField` / `@EventIgnore` | Control event field payload |
| `@Owner` | Mark ownership for permission checks |

## Key Enums

```java
Layer   → CONTROLLER, SERVICE, REPOSITORY
Action  → LIST, GET, CREATE, UPDATE, DELETE, NO_OP
When    → BEFORE, AFTER
```

## Context Access

Use `ApizedContext` (static, thread-local) inside behaviors or services:

```java
ApizedContext.getRequest()    // headers, path variables, timestamp
ApizedContext.getSecurity()   // current user, token, permissions
ApizedContext.getAudit()      // current audit entries
ApizedContext.getEvent()      // current events
ApizedContext.getFederation() // federation cache and config
ApizedContext.getSerde()      // internal framework use only — do not use in application code
```

## REST Query Features

All generated endpoints support these query parameters:

| Feature | Syntax | Example |
|---|---|---|
| Field filtering | `?fields=f1,f2` | `?fields=name` |
| Model drilling | `?fields=f1,rel.f2` | `?fields=name,employees.name` |
| Partial PUT | `?fields=f1` + partial body | send only changed fields |
| Search | `?search=field<op>value` | `?search=name=Org%20A` |
| Nested search | `?search=rel.field<op>value` | `?search=employee.name~=Sen` |
| Pagination | `?page=1&pageSize=50` | page is 1-based; default pageSize is 50, capped by `maxPageSize` on `@Apized` |
| Sorting | `?sort=field>,other<` | `>` = ASC, `<` = DESC; comma-separated; default ASC if no suffix |

Search operators for `?search=`: `=` (eq), `!=` (ne), `~=` (like/contains), `>` (gt), `>=` (gte), `<` (lt), `<=` (lte), `<>` (in — e.g. `status<>ACTIVE,PENDING`), `<!>` (nin — e.g. `status<!>DRAFT,CANCELLED`). Multiple search terms are comma-separated.

Model drilling works on both GET and PUT. LIST responses return a `Page<T>`:

```json
{
  "content": [...],
  "page": 1,
  "pageSize": 50,
  "totalPages": 5,
  "total": 98
}
```

## Search & Sorting (programmatic)

```java
List<SearchTerm> search = List.of(
  SearchTerm.builder().field("name").op(SearchOperation.like).value("phone").build(),
  SearchTerm.builder().field("price").op(SearchOperation.gte).value(10.0).build()
);
List<SortTerm> sort = List.of(
  SortTerm.builder().field("name").direction(SortDirection.ASC).build()
);
Page<Product> results = service.list(1, 50, search, sort);
```

Search operators: `eq`, `ne`, `like`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`

## Security & Permissions

Permission format: `{slug}.{entity}.{action}[.{id}][.{field}][.{value}]` — wildcards supported.

| Permission | Meaning |
|---|---|
| `*` | God permission — anything on any service |
| `sample` | Everything in the sample service |
| `sample.organization` | All operations on all organizations |
| `sample.organization.create` | Can create organizations |
| `sample.organization.update.{id}` | Can update specific organization |
| `sample.organization.update.*.name` | Can update name of any organization |
| `sample.address.update.*.country.PT` | Can update any address country to PT only |

Permissions are evaluated for **every affected model and field** in a request (including nested models via model drilling).

**Configuration:**
```yaml
apized:
  slug: myapp          # REQUIRED — used as the first segment of all permission strings
  cookie: token        # cookie name for token auth (default: token)
  token: ${APP_TOKEN}  # this service's own identity token for inter-service calls (federation, RabbitMQ, ESB); required when security is enabled
  exclusions:          # paths that bypass security/filter processing
    - /health.*
    - /swagger.*
```

**Authentication:** `Authorization: Bearer <token>` header, or cookie (name set by `apized.cookie`).

**UserResolver:** Implement `UserResolver` to replace the default `MemoryUserResolver`. It builds `User` + `Role` objects for each request. The `User` object accepts a `metadata` map for carrying extra data (e.g. external system IDs) through the request lifecycle.

> **Default (dev only):** `MemoryUserResolver` is active when no custom `UserResolver` is registered. It ignores the token and always returns a hardcoded admin user with `permissions: ["*"]`. Replace it in production.

**Micronaut:**
```java
@Singleton
@Replaces(MemoryUserResolver.class)
public class DBUserResolver implements UserResolver {
  @Override
  public User getUser(String token) {
    return User.builder()
      .id(userId)
      .name("Alice Smith")                          // full display name
      .username("user@example.com")
      .permissions(permissions)                     // direct permission strings
      .roles(roles)                                 // optional; isAllowed() checks roles too
      .metadata(Map.of("stripeCustomerId", stripeId))
      .build();
  }
}
```

**Spring:**
```java
@Component
@Primary
public class DBUserResolver implements UserResolver {
  @Override
  public User getUser(String token) {
    return User.builder()
      .id(userId)
      .name("Alice Smith")                          // full display name
      .username("user@example.com")
      .permissions(permissions)                     // direct permission strings
      .roles(roles)                                 // optional; isAllowed() checks roles too
      .metadata(Map.of("stripeCustomerId", stripeId))
      .build();
  }
}
```

You can also execute code as a specific user via `userResolver.runAs()`:

```java
userResolver.runAs(targetUser, () -> {
  // all ApizedContext.getSecurity() calls inside here see targetUser
  orderService.create(order);
});
```

```java
// Manual permission check inside a behaviour/service:
ApizedContext.getSecurity().getUser().isAllowed("myapp.product.create");
```

**Inferred (runtime) permissions** via three mechanisms:

### 1. @Owner
```java
@Owner(actions = { Action.GET }, permissions = @Permission(action = Action.UPDATE, fields = "owner"))
private UUID owner;
```

The `fields` value in `@Permission` supports a `"field.VALUE"` syntax to make permissions conditional on a field's current value:

```java
// Owner can update the order, but only the listed fields, and status only when set to COMPLETE
@Owner(
  actions = { Action.LIST, Action.GET, Action.CREATE },
  permissions = @Permission(
    action = Action.UPDATE,
    fields = { "status.COMPLETE", "orderItems", "customer", "paymentMethod", "metadata" }
  )
)
private UUID owner;
```

### 2. PermissionEnricher (model-scoped)

**Micronaut:**
```java
@Singleton
@PermissionEnricher(Booking.class)
public class BookingPermissionEnricher implements PermissionEnricherHandler<Booking> {
  @Inject BookingRepository bookingRepository;
  @Inject ApizedConfig config;

  @Override
  public boolean enrich(Class<Model> type, Action action, Execution<Booking> execution) {
    User user = ApizedContext.getSecurity().getUser();
    Booking booking = bookingRepository.get(execution.getId()).orElseThrow();
    if (booking.getOwner().equals(user.getId())) {
      user.getInferredPermissions().add(config.getSlug() + ".booking.get." + booking.getId());
      return true;
    }
    return false;
  }
}
```

**Spring:**
```java
@Component
@PermissionEnricher(Booking.class)
public class BookingPermissionEnricher implements PermissionEnricherHandler<Booking> {
  @Autowired BookingRepository bookingRepository;
  @Autowired ApizedConfig config;

  @Override
  public boolean enrich(Class<Model> type, Action action, Execution<Booking> execution) {
    User user = ApizedContext.getSecurity().getUser();
    Booking booking = bookingRepository.get(execution.getId()).orElseThrow();
    if (booking.getOwner().equals(user.getId())) {
      user.getInferredPermissions().add(config.getSlug() + ".booking.get." + booking.getId());
      return true;
    }
    return false;
  }
}
```

### 3. Server Filter (global)

**Micronaut:**
```java
@ServerFilter(Filter.MATCH_ALL_PATTERN)
class MyPermissionFilter extends ApizedServerFilter {
  @RequestFilter
  @ExecuteOn(TaskExecutors.BLOCKING)
  void filterRequest(HttpRequest<?> request) {
    if (shouldExclude(request.getServletPath())) return;
    User user = ApizedContext.getSecurity().getUser();
    // add to user.getInferredPermissions() based on path variables etc.
  }
  @Override public int getOrder() { return ServerFilterPhase.SECURITY.after(); }
}
```

**Spring:**
```java
@Component
class MyPermissionFilter extends ApizedServerFilter {
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
      throws ServletException, IOException {
    if (!shouldExclude(request.getServletPath())) {
      User user = ApizedContext.getSecurity().getUser();
      // add to user.getInferredPermissions() based on path variables etc.
    }
    chain.doFilter(request, response);
  }
  // Built-in order: HttpRequestFilter=1, SecurityFilter=2, SerdeFilter=3 — run after security
  @Override public int getOrder() { return 4; }
}
```

## Federation (cross-API references)

Every apized server instance acts as an API gateway. You can reference models that live on a remote service using `@Federation` — the framework will transparently fetch them when requested via model drilling.

**Important:** The root object of any query must be requested from the server that owns that model directly. Federation only resolves nested references.

### Declaring a federated field

```java
@Entity
@Getter @Setter
@Apized
public class Order extends BaseModel {
  // 'catalog' is the federation alias; Item is the remote type
  // uri template resolves {catalogItemId} from the field of that name on this entity
  @Federation(value = "catalog", type = "Item", uri = "/items/{catalogItemId}")
  private Item catalogItem;

  private UUID catalogItemId;
}
```

When a client requests `?fields=catalogItem.name`, apized calls the remote catalog service to resolve `Item` and inlines the result.

### Configuration

```yaml
# application.yml
apized:
  federation:
    catalog:                               # alias used in @Federation(value = ...)
      baseUrl: https://catalog-api.example.com
      headers:                             # optional; if Authorization absent, falls back to global apized.token
        Authorization: "Bearer ${CATALOG_TOKEN}"
      queryParams:                         # optional extra query params appended to every federation call
        - "version=2"
```

### FederationContext

The `FederationContext` maintains a per-request cache of already-fetched federated models to avoid duplicate remote calls:

```java
ApizedContext.getFederation()  // access cache and federation config inside behaviours
```

## Audit & Events

- **Audit**: Every CREATE/UPDATE/DELETE is recorded automatically when `audit = true`. Pass `X-Reason` header to attach a reason.
- **Events**: Published on CREATE/UPDATE/DELETE when `event = true`. Format: `{slug}.{entityType}.{action}d` (e.g., `myapp.product.created`).

Event delivery requires a messaging adapter. For RabbitMQ, add `micronaut-messaging-rabbitmq` (or `spring-messaging-rabbitmq`) and configure:

```yaml
rabbitmq:
  exchange: apized   # topic exchange name (default: "apized"); routing key = event type
  # standard rabbitmq connection properties (uri, host, port, etc.)
```

`ApizedStartupEvent` is fired by the framework after initialization. Use it to bootstrap data (e.g. ensure default roles/users exist):

```java
// Micronaut
@EventListener
void onStartup(ApizedStartupEvent event) { ensureDefaults(); }

// Spring
@EventListener
void onStartup(ApizedStartupEvent event) { ensureDefaults(); }
```

You can inject custom headers into all outgoing event messages for the current request:

```java
// adds to AMQP message headers alongside timestamp, token, and path variables
ApizedContext.getEvent().getHeaders().put("tenantId", tenantId.toString());
```

You can also publish custom events manually from a behaviour — they are queued in the request scope and delivered only if the request succeeds:

```java
ApizedContext.getEvent().add(new Event(
  ApizedContext.getRequest().getId(),  // correlationId
  "notification.email.order-complete", // event type / routing key
  Map.of("orderId", order.getId(), "userId", order.getOwner())  // payload
));
```

Use `@AuditIgnore` / `@EventIgnore` to exclude a field from the snapshot, and `@AuditField` / `@EventField` to include a computed/virtual value (can be placed on a method):

```java
public class Order extends BaseModel {
  private BigDecimal unitPrice;
  private Integer quantity;

  @AuditIgnore   // never appears in audit snapshots
  @EventIgnore   // never appears in event payloads
  private String internalNote;

  @AuditField    // included in audit snapshot even though it's not a persisted column
  @EventField
  public BigDecimal getTotal() {
    return unitPrice.multiply(BigDecimal.valueOf(quantity));
  }
}
```

For relationships, use `@EventField({"id", "name"})` to restrict which sub-fields appear in the event payload (avoids large nested payloads):

```java
@EventField({"id", "name"})   // only id and name of each role are included in events
@ManyToMany
private List<Role> roles;
```

## Generated Classes

For a model `Product`, the processor generates:
- `ProductRepository` (implements `ModelRepository<Product>`)
- `ProductService` (implements `ModelService<Product>`) — also exposes `batchCreate`, `batchUpdate`, `batchDelete` for bulk operations (service-layer only, not exposed as HTTP endpoints)
- `ProductController` (implements `ModelController<Product>`)
- `ProductDeserializer`
- `ProductMcpTools` — only when `mcp = true` (default) and `micronaut-mcp` / `spring-mcp` is on the classpath

All are annotated with `@Generated` — do not edit them directly. Use extensions and behaviors instead.

`ProductService` exposes two fetch methods with different semantics:
- `get(id)` — runs through the full behaviour pipeline (use from controllers or external callers)
- `find(id)` — bypasses behaviours (use inside behaviours to avoid triggering the pipeline recursively)
- `searchOne(List<SearchTerm> search)` — returns `Optional<T>` for finding a single record by field criteria instead of by UUID (e.g. find a user by email)

## Triggering the Behaviour Pipeline from Custom Endpoints

Use `@MicronautBehaviourExecution` (Micronaut) or `@SpringBehaviourExecution` (Spring) to run the behaviour pipeline for a given model+layer+action from a custom controller method:

**Micronaut:**
```java
@Controller("/search/trips")
public class SuggestionsController {
  @Inject TripService service;

  @Get
  @MicronautBehaviourExecution(execution =
    @BehaviourExecution(model = Trip.class, layer = Layer.CONTROLLER, action = Action.LIST)
  )
  public Page<Trip> suggest(@QueryValue String location) {
    List<SearchTerm> search = List.of(
      SearchTerm.builder().field("origin").op(SearchOperation.like).value(location).build()
    );
    return service.list(1, 10, search, List.of());
  }
}
```

**Spring:**
```java
@RestController
@RequestMapping("/search/trips")
public class SuggestionsController {
  @Autowired TripService service;

  @GetMapping
  @SpringBehaviourExecution(execution =
    @BehaviourExecution(model = Trip.class, layer = Layer.CONTROLLER, action = Action.LIST)
  )
  public Page<Trip> suggest(@RequestParam String location) {
    List<SearchTerm> search = List.of(
      SearchTerm.builder().field("origin").op(SearchOperation.like).value(location).build()
    );
    return service.list(1, 10, search, List.of());
  }
}
```

This ensures any behaviours registered for `Trip` / `CONTROLLER` / `LIST` fire around your custom method, just as they would for the generated endpoint.

## Controller Extensions

Use `@Apized.Extension(layer = Layer.CONTROLLER)` to override a generated action (e.g. replace hard-delete with soft-delete):

```java
@Apized.Extension(layer = Layer.CONTROLLER)
public class RouteControllerExtension {
  @Inject RouteService routeService;

  @Apized.Extension.Action(Action.DELETE)
  public Route delete(UUID id) {
    Route route = routeService.get(id);
    route.setDeleted(true);
    return routeService.update(id, route);
  }
}
```

Reference it via `@Apized(extensions = RouteControllerExtension.class)`. The framework calls your method instead of the generated DELETE.

## Custom Controllers

For operations that don't fit CRUD (e.g. login, password reset, token exchange), write a fully custom controller and inject the generated services.

**Micronaut:**
```java
@Controller("/users/{username}/password")
@ExecuteOn(TaskExecutors.BLOCKING)   // required for any blocking I/O (JPA, JDBC)
public class PasswordResetController {
  @Inject UserRepository userRepository;
  @Inject UserService userService;

  @Delete
  @Status(HttpStatus.ACCEPTED)
  public void requestReset(String username) {
    userRepository.findByUsername(username).ifPresent(user -> {
      user.setResetCode(CodeGenerator.generate());
      userService.update(user.getId(), user);
    });
  }

  @Post
  public void resetPassword(String username, @Body ResetRequest body) {
    User user = userRepository.findByUsername(username).orElseThrow(NotFoundException::new);
    if (!user.getResetCode().equals(body.code())) throw new UnauthorizedException();
    user.setPassword(body.newPassword());
    userService.update(user.getId(), user);
  }
}
```

**Spring:**
```java
@RestController
@RequestMapping("/users/{username}/password")
public class PasswordResetController {
  @Autowired UserRepository userRepository;
  @Autowired UserService userService;

  @DeleteMapping
  @ResponseStatus(HttpStatus.ACCEPTED)
  public void requestReset(@PathVariable String username) {
    userRepository.findByUsername(username).ifPresent(user -> {
      user.setResetCode(CodeGenerator.generate());
      userService.update(user.getId(), user);
    });
  }

  @PostMapping
  public void resetPassword(@PathVariable String username, @RequestBody ResetRequest body) {
    User user = userRepository.findByUsername(username).orElseThrow(NotFoundException::new);
    if (!user.getResetCode().equals(body.code())) throw new UnauthorizedException();
    user.setPassword(body.newPassword());
    userService.update(user.getId(), user);
  }
}
```

Custom controllers have full access to `ApizedContext`, all generated services, and repository extensions.

## Complete Example

### Parent model (top-level)

```java
@Entity
@Getter @Setter
@Apized(
  operations = {Action.LIST, Action.GET, Action.CREATE, Action.UPDATE, Action.DELETE},
  audit = true,
  event = true
)
public class Organization extends BaseModel {
  @NotBlank
  private String name;

  @OneToMany(mappedBy = "organization", orphanRemoval = true)
  private List<Department> departments;
}
```

### Child model (scoped under Organization)

```java
@Entity
@Getter @Setter
@Apized(
  scope = Organization.class,
  operations = {Action.LIST, Action.GET, Action.CREATE, Action.UPDATE, Action.DELETE}
)
public class Department extends BaseModel {
  @NotBlank
  private String name;

  @ManyToOne
  private Organization organization;

  @ManyToOne
  private Employee manager;
}
```

This generates endpoints like `GET /organizations/{organization}/departments/{id}`.

### Behaviour on the child

```java
@Singleton   // @Component for Spring
@Behaviour(
  model = Department.class,
  layer = Layer.SERVICE,
  when = When.BEFORE,
  actions = Action.CREATE
)
public class DepartmentCreationBehaviour implements BehaviourHandler<Department> {
  @Override
  public void preCreate(Execution<Department> execution, Department input) {
    // e.g. default the manager to the current user
    UUID currentUser = ApizedContext.getSecurity().getUser().getId();
    if (input.getManager() == null) {
      Employee mgr = new Employee();
      mgr.setId(currentUser);
      input.setManager(mgr);
    }
  }
}
```

### Custom repository query

```java
@Apized.Extension(layer = Layer.REPOSITORY)
public interface DepartmentRepositoryExtension {
  Page<Department> findByOrganizationAndManagerId(UUID organizationId, UUID managerId, Pageable pageable);
}
```

Reference it via `@Apized(extensions = DepartmentRepositoryExtension.class)` on the model.

---

## Tracing Module (`micronaut-tracing` / `spring-tracing`)

Apply `@Traced` to any method or class to create an OpenTelemetry span automatically:

```java
// Simple usage — span name defaults to "ClassName::methodName"
@Traced
public void processOrder(UUID orderId) { ... }

// Custom name + kind + attributes referencing method parameters
@Traced(
  value = "payment.charge",
  kind = TraceKind.CLIENT,
  attributes = {
    @Traced.Attribute(key = "order.id", arg = "orderId"),
    @Traced.Attribute(key = "gateway", value = "stripe")
  }
)
public void chargeCard(UUID orderId, BigDecimal amount) { ... }
```

`TraceKind` values: `INTERNAL` (default), `SERVER`, `CLIENT`, `PRODUCER`, `CONSUMER`.

Requires a `Tracer` bean in the context (e.g. via the OpenTelemetry SDK). If no `Tracer` bean is present the interceptor is inactive.

---

## Distributed Lock Module (`micronaut-distributed-lock` / `spring-distributed-lock`)

Inject `LockFactory` to run code under a named distributed lock (backed by ShedLock over JDBC). Use `@Inject` (Micronaut) or `@Autowired` (Spring).

```java
@Inject LockFactory lockFactory;  // or @Autowired for Spring

// Basic — acquire lock, run task
lockFactory.executeWithLock("generate-invoices", () -> {
  invoiceService.generateMonthly();
});

// With retry — keep trying to acquire lock for up to 30 seconds
lockFactory.executeWithLockRetry("send-notifications", Duration.ofSeconds(30), () -> {
  notificationService.sendPending();
});

// Full control — lock held at least 5s, at most 10min, with retry
lockFactory.executeWithLock("sync-catalog", true, Duration.ofSeconds(5), Duration.ofMinutes(10), () -> {
  catalogService.sync();
});
```

Requires a `shedlock` table in the database (created by your Flyway/Liquibase migration).

---

## Test Module (`micronaut-test` / `spring-test`)

Use the Apized test module for HTTP-level Cucumber integration tests. It boots the real application, discovers generated model services, sends requests with REST Assured, understands `@Apized(scope = ...)` hierarchies, and resets the database and registered service mocks before each scenario. Prefer a real disposable database over repository mocks.

### Recommended test layout

```text
src/test/groovy/com/yourcompany/integration/
  IntegrationTests.groovy
  TestController.groovy                 # only when tests need internal state
  mocks/TestUserResolver.groovy
  mocks/ExternalClientMock.groovy
  steps/ProductSteps.groovy             # only domain-specific language
src/test/resources/
  application-test.yml
  cucumber.properties                   # optional IDE/CLI defaults
  features/<domain>/*.feature
  payloads/...                           # optional request/response fixtures
```

Keep reusable CRUD, login, context, mock-expectation, and response assertions in the framework's built-in steps. Add project steps only for domain workflows or custom endpoints.

### Gradle and runner setup

The Apized Gradle plugin may already supply the matching test module. If it is not on the test classpath, add it explicitly:

```groovy
dependencies {
  testImplementation "org.apized:micronaut-test:$apizedVersion" // Micronaut
  // testImplementation "org.apized:spring-test:$apizedVersion" // Spring
}
```

The Groovy runner is:

```groovy
package com.yourcompany.integration

import io.cucumber.junit.Cucumber
import io.cucumber.junit.CucumberOptions
import io.micronaut.test.extensions.junit5.annotation.MicronautTest
import org.junit.runner.RunWith

@MicronautTest
@RunWith(Cucumber)
@CucumberOptions(
  plugin = ['pretty', 'html:target/features'],
  features = ['src/test/resources/features'],
  glue = ['org.apized', 'com.yourcompany']
)
class IntegrationTests {}
```

`org.apized` in `glue` is mandatory: it discovers the built-in steps and the `MicronautTestServer`/`SpringBootTestServer` hooks that start the embedded server and initialize `IntegrationConfig`. Add the application's package when custom steps are outside `org.apized`. For Spring, use the same Cucumber runner pattern and make `SpringBootTestServer` discoverable through glue; set `SPRING_MAIN_CLASS` to the fully-qualified application class because that server uses it to boot Spring.

JUnit Vintage must remain available because `@RunWith(Cucumber)` is a JUnit 4 runner; the Apized test module normally provides it transitively.

### Java + Micronaut migration recipe

Use Java 21 with Gradle 8.5 or newer. Add the test module and Java annotation processor explicitly so Micronaut can discover concrete Java test beans:

```gradle
dependencies {
  testImplementation "org.apized:micronaut-test:$apizedVersion"
  testAnnotationProcessor "io.micronaut:micronaut-inject-java"
}
```

Keep the Java runner in the test source set and constrain feature discovery to the module's own `src/test/resources/features` directory:

```java
package com.yourcompany.integration;

import io.cucumber.junit.Cucumber;
import io.cucumber.junit.CucumberOptions;
import io.micronaut.test.extensions.junit5.annotation.MicronautTest;
import org.junit.runner.RunWith;

@MicronautTest
@RunWith(Cucumber.class)
@CucumberOptions(
    features = "classpath:features",
    glue = {"org.apized", "com.yourcompany"},
    plugin = {"pretty", "html:target/features"})
public class IntegrationTests {}
```

The `org.apized` glue remains required for framework steps and lifecycle hooks; add the application's package for its custom steps. Do not point `features` at a repository-wide directory, because a module must not accidentally execute another module's feature files.

Concrete Java test beans must use Micronaut bean annotations such as `@Singleton` or `@Controller` and must have generated Micronaut bean metadata. Replace the concrete production resolver—not the `UserResolver` interface—with the test resolver:

```java
import io.micronaut.context.annotation.Replaces;
import jakarta.inject.Singleton;

@Singleton
@Replaces(DBUserResolver.class)
public class TestUserResolver extends AbstractMicronautUserResolverMock {
  // Provide the test users required by this module's features.
}
```

When a scenario needs to inspect test state, expose only opaque test support through a test-only controller. Extend `MicronautTestController` and use the `/integration` prefix: built-in login and reset support also requires that prefix.

```java
import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.Get;

@Controller("/integration")
public class IntegrationTestController extends MicronautTestController {
  @Get("/reset-state")
  public void resetOpaqueState() {
    reset();
  }

  @Get("/verification-state")
  public Object readOpaqueVerificationState() {
    return verificationState();
  }
}
```

Keep these endpoints minimal and test-only: return opaque reset or verification data rather than exposing domain internals. Use the concrete reset and verification accessors supplied by the module version in use when their names differ.

| Symptom | Check |
| --- | --- |
| Cucumber runner or built-in steps are missing | Use `@RunWith(Cucumber.class)`, retain JUnit Vintage, and include `org.apized` plus the application package in `glue`. |
| Java replacement is ignored or unavailable | Add `testAnnotationProcessor "io.micronaut:micronaut-inject-java"`; annotate concrete test beans with `@Singleton`/`@Controller` so Micronaut generates bean metadata; replace `DBUserResolver.class`, not `UserResolver`. |
| `/integration` endpoints return 404 | Keep the test controller in the test source set, annotate it with `@Controller("/integration")`, and extend `MicronautTestController`. |
| Features from another module run or local features are not found | Use `features = "classpath:features"` and keep this module's files under `src/test/resources/features`. |
| Gradle or Java startup/processor failures | Run Java 21 with Gradle 8.5 or newer. |

Optional `src/test/resources/cucumber.properties` for IDE/CLI discovery:

```properties
cucumber.glue=org.apized,com.yourcompany
cucumber.plugin=pretty
```

Run all tests with `./gradlew test` (or `./gradlew :server:test` in a multi-project build). To support CI/test partitioning, a project can pass exclusions into Gradle and map them to `test.exclude(...)`, for example with `-PexcludeTests=...`.

### Test profile and real database

Use the `test` profile and a random port. For Micronaut with PostgreSQL Testcontainers JDBC:

```yaml
# src/test/resources/application-test.yml
datasources:
  default:
    url: jdbc:tc:postgresql:16:///app
    driverClassName: org.testcontainers.jdbc.ContainerDatabaseDriver
    username: test
    password: test

micronaut:
  server:
    port: 0

endpoints:
  all:
    port: 0
    enabled: false
```

Use the same database family as production so migrations, constraints, native column types, and queries are exercised. The framework test controller truncates non-Flyway tables between scenarios for H2, MySQL, PostgreSQL, Oracle, and SQL Server, then re-runs startup initialization. Do not depend on records created by another scenario; put shared setup in `Background` or startup fixtures.

### Authentication fixture

For a static set of users, replace the application's resolver and extend the framework mock:

```groovy
@Singleton
@Replaces(DBUserResolver)
class TestUserResolver extends AbstractMicronautUserResolverMock {
  @Override
  Map<String, User> getKnownUsers() {
    [
      administrator: new User(permissions: ['*']),
      alice: new User(permissions: ['myapp.product.list'])
    ]
  }
}
```

Use `@Component`/`@Primary` and `AbstractSpringUserResolverMock` for Spring. The base mock assigns missing UUIDs and publishes the alias map into the integration context, so `Given I login as alice` obtains a token for that exact test user.

If users are themselves persisted during scenarios, use a dynamic fixture pattern instead of keeping only a static map:

1. Replace the production resolver, but inject/delegate to production conversion logic where appropriate.
2. Keep both alias-to-user and UUID-to-user maps in the inherited mock.
3. Register seeded users after application startup.
4. Register newly created users in a test-only `AFTER CREATE` service behaviour so later steps can immediately `login as <alias>`.
5. Resolve database users first, then fall back to the inherited in-memory users/anonymous user.

This keeps authentication realistic while making scenario-created identities usable. Reset `inferredPermissions` per resolution; never let inferred permissions leak between requests or scenarios.

### Built-in Cucumber steps

```gherkin
# Authentication and scoped URL context
Given I login as administrator
Given the context is
  | organization | ${organization.id} |

# Generated CRUD; aliases save responses for later interpolation/assertion
When I list the products
When I list the products as productPage
When I create an empty product
When I create a product as widget with
  | name     | Widget                   |
  | category | [ id: '${category.id}' ] |
When I get a product with id ${widget.id} as fetchedWidget
When I update a product with id ${widget.id} with
  | name | Updated Widget |
When I delete a product with id ${widget.id}

# Field expansion/model drilling and request headers
Given the responses are expanded to contain category.name
Given the request contains header X-Reason with value "integration test"

# Status and body assertions
Then the request succeeds
Then the request fails
Then the productPage request succeeds
Then the response contains 3 elements
Then the response element 0 contains
  | name | Widget |
Then the response contains element with
  | name | Widget |
Then the response does not contain element with
  | name | Deleted Widget |
Then the response path "errors" contains element with
  | field   | name             |
  | message | must not be null |
Then the response path "errors" contains 2 elements
Then the response matches products/widget.json
```

Important expression rules:

- Refer to stored values with `${alias.id}`, `${alias.roles[0]}`, etc. Do not use `<alias.id>` or `{alias.id}` outside a Scenario Outline example placeholder.
- DataTable values are evaluated as Groovy-like literals, allowing lists/maps such as `[ '${role.id}' ]` and `[ [ id: '${role.id}' ] ]`.
- Use `/regex/` for a full regular-expression match; `/.*Not allowed.*/` is typical for variable error text.
- Booleans and numeric strings are compared as their response types.
- Response paths are dot-separated (`errors.0.message`, `content.0.name`), not JSONPath bracket syntax.
- `_` means the current scalar/object, useful when asserting primitive lists.
- `response matches file.json` loads `/payloads/file.json` from the test classpath.

For validation matrices, prefer `Scenario Outline` with explicit input, expected status, response path, field, and message columns. Cover success boundaries as well as null, too-short/long, invalid enum, duplicate-key, and authorization failures.

### Custom steps for custom endpoints

Extend `AbstractSteps`, initialize the shared runner/context once, use `testRunner.getClient(context)`, evaluate URLs and payload values through `context.eval(...)`, and always record the result with `context.addResponse(...)` so built-in assertions continue to work:

```groovy
class ProductSteps extends AbstractSteps {
  static TestRunner testRunner
  static IntegrationContext context

  @BeforeAll
  static void setup() {
    testRunner = IntegrationConfig.testRunner
    context = testRunner.context
  }

  @When('^I activate product ([^\\s]+)$')
  void activate(String id) {
    Response response = testRunner.getClient(context)
      .post(context.eval("/products/$id/activation").toString())

    context.addResponse(
      'product',
      (response.statusCode().intdiv(100) == 2),
      response.asString(),
      null
    )
  }
}
```

Use an `executeAs(user, closure)` helper for fixture creation that temporarily switches identity, and restore the old token in `finally`. Keep feature text business-oriented; do not duplicate generic HTTP mechanics in every project step.

For stateful protocols (for example passkeys), keep simulator state as instance fields so it is per scenario, while the framework runner/context stay static. Store parsed intermediate responses by alias and use the same simulator/key material throughout the scenario.

### External service mocks

Replace the real client/adapter with a bean that extends `AbstractServiceIntegrationMock`. Give it a stable `mockedServiceName`, record every invocation, and return configured expectations:

```groovy
@Singleton
@Replaces(CatalogClient)
class CatalogClientMock extends AbstractServiceIntegrationMock implements CatalogClient {
  CatalogClientMock(ObjectMapper mapper) { super(mapper) }

  @Override
  String getMockedServiceName() { 'CatalogClient' }

  @Override
  ProductDetails getProduct(UUID id) {
    execute('getProduct', [id: id], ProductDetails) { null }
  }
}
```

Drive and verify it from Gherkin:

```gherkin
Given I expect service CatalogClient to respond with
  | getProduct | { "name": "Widget" } |
When I get a product with id ${product.id}
And I get the executions of getProduct from service CatalogClient as catalogCalls
Then the catalogCalls response contains 1 element
And the catalogCalls response element 0 contains
  | id | ${product.id} |
```

Expectation values can use `classpath:/payloads/...`; mock response templates may interpolate invocation arguments. All expectations and execution history are cleared before each scenario. Mock true boundaries (OAuth clients, ESB/message adapters, remote APIs), not repositories or generated services.

### Test-only controller

When a workflow needs an opaque value that the public API intentionally hides (verification code, reset token, captured event), add a controller under `src/test` extending `MicronautTestController` or `SpringTestController`, and expose the smallest read-only endpoint needed by the steps:

```groovy
@Transactional
@Controller('/integration')
class TestController extends MicronautTestController {
  @Inject UserService userService

  @Get('/users/{userId}/verificationCode')
  HttpResponse<String> verificationCode(UUID userId) {
    HttpResponse.ok(userService.get(userId).verificationCode)
  }
}
```

Because it lives in test sources, it cannot ship in the production artifact. Use it only to bridge otherwise inaccessible state; test public behavior through public endpoints.

### Coverage checklist

For each generated model or custom workflow, cover:

- happy-path CRUD/custom action and persisted response;
- bean-validation boundaries and structured `errors` assertions;
- anonymous, allowed, owner/inferred-permission, and forbidden access;
- relationship add/replace/remove and model drilling where relevant;
- behaviour side effects (hashing/defaults/events) through observable results;
- external calls through expectations plus captured argument assertions;
- transaction/database constraints using the real test database;
- scenario isolation (each scenario passes alone and in the full suite).

Do not assert secrets or irreversible transformed values directly when the public contract hides them. Assert that the clear text is absent, authenticate through the real workflow, inspect an intentionally test-only endpoint, or verify downstream calls instead.

---

## MCP Integration (`micronaut-mcp` / `spring-mcp`)

When `mcp = true` (the default) and the MCP module is on the classpath, the annotation processor generates a `{Type}McpTools` bean that exposes each enabled CRUD operation as an MCP tool callable by AI agents.

### Generated tools

For a model `Product` with all five operations enabled, the following MCP tools are registered:

| Tool name | Operation |
|---|---|
| `product_list` | LIST — with `page`, `pageSize`, `search`, `sort`, `fields` params |
| `product_get` | GET — with `id`, `fields` params |
| `product_create` | CREATE — with `it` (the model object), `fields` params |
| `product_update` | UPDATE — with `id`, `it`, `fields` params |
| `product_delete` | DELETE — with `id` param |

Tool names are `{snake_case_type}_{action}`. Only operations declared in `@Apized(operations = ...)` are generated.

### Authentication

The `McpContextInitializer` bean (provided by the MCP module) re-initialises the apized security context from the `Authorization: Bearer <token>` header of the incoming MCP request, delegating to the registered `UserResolver`. It is only active when a `UserResolver` bean is present.

### Custom HTTP endpoints need explicit MCP tools (Micronaut)

`mcp = true` generates tools **only** for enabled model CRUD actions. A custom Micronaut controller route—such as `POST` or `DELETE /users/{uuid}/permissions/{permission}`—is an HTTP endpoint and is **not** automatically registered as an MCP tool.

Expose custom operations with Micronaut MCP's `@Tool` and `@ToolArg` annotations, preferably in a dedicated `@Singleton` adapter rather than directly on the HTTP controller. Delegate from the adapter to the same generated service or domain logic used by the endpoint:

```java
import io.micronaut.context.annotation.Singleton;
import io.micronaut.mcp.annotations.Tool;
import io.micronaut.mcp.annotations.ToolArg;

@Singleton
public class UserPermissionMcpTools {
  private final UserService userService;

  public UserPermissionMcpTools(UserService userService) {
    this.userService = userService;
  }

  @Tool(name = "user_add_permission", description = "Grant a permission to a user")
  public User addPermission(
      @ToolArg(name = "userId") UUID userId,
      @ToolArg(name = "permission") String permission) {
    return userService.addPermission(userId, permission);
  }
}
```

Before invoking Apized-backed logic, initialise the request context for the MCP invocation with `McpContextInitializer.init()` when the integration requires explicit initialisation. This preserves authentication and permission checks; do not bypass the generated service by writing directly to the repository. Choose stable, descriptive tool names and validate tool arguments just as you would validate HTTP input.

### Disabling MCP for a specific model

```java
@Apized(mcp = false)
public class InternalModel extends BaseModel { ... }
```
