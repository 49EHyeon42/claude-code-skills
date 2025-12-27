---
name: kotlin-hexagonal-architecture
description: Implements hexagonal architecture (ports and adapters pattern) for Kotlin and Spring Boot applications. Use when architecting new features, refactoring existing code, or reviewing code for architectural compliance. Activates on requests mentioning hexagonal architecture, ports and adapters, clean architecture, domain-driven design, or architectural patterns.
---

# Kotlin Hexagonal Architecture

## Core Principles

Hexagonal Architecture separates business logic from external concerns through ports and adapters.

**Dependency Rule**: Dependencies point inward. Domain has zero dependencies. Application depends on Domain. Adapters depend on Application and Domain.

**Port**: Interface defining a contract. Input ports define use cases. Output ports define external dependencies.

**Adapter**: Implementation of a port. Input adapters receive external requests. Output adapters interact with external systems.

**Code Style**: All code follows ktlint conventions. Maximum line length is 120 characters. Use trailing commas in multiline parameter lists. Avoid wildcard imports. Follow Kotlin coding conventions for naming and structure.

## Package Structure

Single domain example (Order domain):

```
src/main/kotlin/com/company/product/order/
├── domain/
│   ├── model/                          # Entities and Value Objects
│   │   ├── Order.kt
│   │   ├── OrderItem.kt
│   │   ├── OrderId.kt
│   │   ├── OrderStatus.kt
│   │   └── Money.kt
│   ├── port/
│   │   ├── input/                      # Use Case interfaces (Driving Ports)
│   │   │   ├── CreateOrderUseCase.kt
│   │   │   ├── GetOrderUseCase.kt
│   │   │   └── CancelOrderUseCase.kt
│   │   └── output/                     # Repository interfaces (Driven Ports)
│   │       ├── OrderRepository.kt
│   │       └── PaymentGateway.kt
│   ├── service/                        # Domain services (optional)
│   │   └── PricingService.kt
│   ├── event/                          # Domain events
│   │   ├── OrderEvent.kt
│   │   ├── OrderSubmittedEvent.kt
│   │   └── OrderCancelledEvent.kt
│   └── exception/                      # Domain exceptions
│       ├── OrderNotFoundException.kt
│       └── InvalidOrderStateException.kt
├── application/
│   └── service/                        # Use Case implementations
│       ├── CreateOrderService.kt
│       ├── GetOrderService.kt
│       └── CancelOrderService.kt
├── adapter/
│   ├── input/
│   │   └── rest/                       # REST Controllers
│   │       ├── OrderController.kt
│   │       ├── request/                # Request DTOs
│   │       │   ├── CreateOrderRequest.kt
│   │       │   ├── OrderItemRequest.kt
│   │       │   └── CancelOrderRequest.kt
│   │       └── response/               # Response DTOs
│   │           ├── OrderResponse.kt
│   │           ├── OrderItemResponse.kt
│   │           └── OrderSummaryResponse.kt
│   └── output/
│       ├── persistence/                # JPA implementations
│       │   ├── OrderJpaRepository.kt
│       │   ├── OrderPersistenceAdapter.kt
│       │   └── entity/
│       │       ├── OrderJpaEntity.kt
│       │       └── OrderItemJpaEntity.kt
│       └── external/                   # External API clients
│           └── PaymentGatewayAdapter.kt
└── config/                             # Spring configuration
    └── OrderConfiguration.kt
```

## Domain Layer

### Entities

Use regular classes (not data classes) for entities with identity. Implement business logic within the entity.

```kotlin
class Order private constructor(
    val id: OrderId,
    private val items: MutableList<OrderItem>,
    private var status: OrderStatus,
) {
    val totalAmount: Money
        get() = items.fold(Money.ZERO) { acc, item -> acc + item.amount }

    fun addItem(item: OrderItem) {
        require(status == OrderStatus.DRAFT) {
            "Cannot add items to order in status $status"
        }
        require(items.size < MAX_ITEMS) {
            "Order cannot have more than $MAX_ITEMS items"
        }
        items.add(item)
    }

    fun submit(): OrderSubmittedEvent {
        require(status == OrderStatus.DRAFT) {
            "Only draft orders can be submitted"
        }
        require(items.isNotEmpty()) {
            "Cannot submit empty order"
        }
        status = OrderStatus.SUBMITTED
        return OrderSubmittedEvent(id, totalAmount)
    }

    companion object {
        private const val MAX_ITEMS = 100

        fun create(id: OrderId): Order {
            return Order(id, mutableListOf(), OrderStatus.DRAFT)
        }
    }
}
```

### Value Objects

Use data classes or inline value classes for value objects. Make them immutable.

```kotlin
@JvmInline
value class OrderId(val value: String) {
    init {
        require(value.isNotBlank()) { "OrderId cannot be blank" }
    }
}

data class Money(
    val amount: BigDecimal,
    val currency: Currency,
) {
    init {
        require(amount >= BigDecimal.ZERO) { "Money amount cannot be negative" }
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) {
            "Cannot add money with different currencies: $currency vs ${other.currency}"
        }
        return Money(amount + other.amount, currency)
    }

    companion object {
        val ZERO = Money(BigDecimal.ZERO, Currency.getInstance("USD"))
    }
}
```

### Input Ports (Use Cases)

Define interfaces for all use cases. Use command/query segregation. Return domain types or Result types.

```kotlin
interface CreateOrderUseCase {
    fun execute(command: CreateOrderCommand): Result<OrderId>
}

data class CreateOrderCommand(
    val customerId: CustomerId,
    val items: List<OrderItemData>,
)

data class OrderItemData(
    val productId: ProductId,
    val quantity: Int,
    val price: Money,
)
```

### Output Ports (Repository Interfaces)

Define repository interfaces in the domain layer. Use domain types only.

```kotlin
interface OrderRepository {
    fun save(order: Order)
    fun findById(id: OrderId): Order?
    fun findByCustomerId(customerId: CustomerId): List<Order>
}

interface PaymentGateway {
    fun processPayment(orderId: OrderId, amount: Money): Result<PaymentConfirmation>
}
```

### Domain Events

Use sealed classes or data classes for domain events.

```kotlin
sealed interface OrderEvent {
    val orderId: OrderId
}

data class OrderSubmittedEvent(
    override val orderId: OrderId,
    val totalAmount: Money,
) : OrderEvent

data class OrderCancelledEvent(
    override val orderId: OrderId,
    val reason: String,
) : OrderEvent
```

## Application Layer

### Use Case Implementation

Implement use cases as application services. Keep transaction boundaries here.

```kotlin
@Service
class CreateOrderService(
    private val orderRepository: OrderRepository,
    private val eventPublisher: EventPublisher,
) : CreateOrderUseCase {

    @Transactional
    override fun execute(command: CreateOrderCommand): Result<OrderId> {
        return runCatching {
            val orderId = OrderId(UUID.randomUUID().toString())
            val order = Order.create(orderId)

            command.items.forEach { itemData ->
                val item = OrderItem(
                    productId = itemData.productId,
                    quantity = itemData.quantity,
                    price = itemData.price,
                )
                order.addItem(item)
            }

            val event = order.submit()
            orderRepository.save(order)
            eventPublisher.publish(event)

            orderId
        }
    }
}
```

## Adapter Layer

### Input Adapters (REST Controllers)

Controllers translate HTTP requests to use case commands. Handle HTTP concerns only.

```kotlin
@RestController
@RequestMapping("/api/v1/orders")
class OrderController(
    private val createOrderUseCase: CreateOrderUseCase,
    private val getOrderUseCase: GetOrderUseCase,
) {

    @PostMapping
    fun createOrder(@RequestBody request: CreateOrderRequest): ResponseEntity<OrderResponse> {
        val command = request.toCommand()

        return createOrderUseCase.execute(command)
            .map { orderId ->
                ResponseEntity
                    .created(URI.create("/api/v1/orders/${orderId.value}"))
                    .body(OrderResponse(orderId.value))
            }
            .getOrElse { exception ->
                ResponseEntity.badRequest().build()
            }
    }

    @GetMapping("/{id}")
    fun getOrder(@PathVariable id: String): ResponseEntity<OrderResponse> {
        val orderId = OrderId(id)

        return getOrderUseCase.execute(orderId)
            .map { order -> ResponseEntity.ok(order.toResponse()) }
            .getOrElse { ResponseEntity.notFound().build() }
    }
}

data class CreateOrderRequest(
    val customerId: String,
    val items: List<OrderItemRequest>,
) {
    fun toCommand(): CreateOrderCommand {
        return CreateOrderCommand(
            customerId = CustomerId(customerId),
            items = items.map { it.toData() },
        )
    }
}
```

### Request and Response DTOs

DTOs live in the adapter layer and should never leak into domain or application layers.

**Request DTOs:**

Request DTOs receive data from external clients and convert it to domain commands.

```kotlin
data class CreateOrderRequest(
    val customerId: String,
    val items: List<OrderItemRequest>
) {
    fun toCommand(): CreateOrderCommand {
        return CreateOrderCommand(
            customerId = CustomerId(customerId),
            items = items.map { it.toData() }
        )
    }
}

data class OrderItemRequest(
    val productId: String,
    val quantity: Int,
    val price: BigDecimal,
    val currency: String
) {
    fun toData(): OrderItemData {
        return OrderItemData(
            productId = ProductId(productId),
            quantity = quantity,
            price = Money(price, Currency.getInstance(currency))
        )
    }
}

data class CancelOrderRequest(
    val reason: String
)
```

**Response DTOs:**

Response DTOs convert domain objects to external representation.

```kotlin
data class OrderResponse(
    val id: String,
    val customerId: String,
    val status: String,
    val items: List<OrderItemResponse>,
    val totalAmount: BigDecimal,
    val currency: String,
    val createdAt: String
)

data class OrderItemResponse(
    val productId: String,
    val quantity: Int,
    val price: BigDecimal,
    val currency: String
)

data class OrderSummaryResponse(
    val id: String,
    val status: String,
    val totalAmount: BigDecimal,
    val itemCount: Int
)

fun Order.toResponse(): OrderResponse {
    return OrderResponse(
        id = id.value,
        customerId = customerId.value,
        status = status.name,
        items = items.map { it.toResponse() },
        totalAmount = totalAmount.amount,
        currency = totalAmount.currency.currencyCode,
        createdAt = createdAt.toString()
    )
}

fun OrderItem.toResponse(): OrderItemResponse {
    return OrderItemResponse(
        productId = productId.value,
        quantity = quantity,
        price = price.amount,
        currency = price.currency.currencyCode
    )
}
```

**Key Principles:**

- DTOs contain only primitive types and collections (String, Int, List, etc.)
- DTOs are data classes with no business logic
- DTOs belong to the adapter layer, never in domain
- Conversion methods (toCommand, toResponse) are in the DTO or adapter layer
- Use extension functions for domain-to-DTO mapping
- Separate request and response DTOs even if fields are similar

### Output Adapters (Persistence)

Separate JPA entities from domain entities. Use mappers to convert between them.

```kotlin
@Repository
interface OrderJpaRepository : JpaRepository<OrderJpaEntity, String>

@Component
class OrderPersistenceAdapter(
    private val jpaRepository: OrderJpaRepository,
) : OrderRepository {

    override fun save(order: Order) {
        val entity = order.toJpaEntity()
        jpaRepository.save(entity)
    }

    override fun findById(id: OrderId): Order? {
        return jpaRepository.findById(id.value)
            .map { it.toDomain() }
            .orElse(null)
    }

    override fun findByCustomerId(customerId: CustomerId): List<Order> {
        return jpaRepository.findByCustomerId(customerId.value)
            .map { it.toDomain() }
    }
}

@Entity
@Table(name = "orders")
class OrderJpaEntity(
    @Id
    val id: String,

    @Column(nullable = false)
    val customerId: String,

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    var status: OrderStatus,

    @OneToMany(cascade = [CascadeType.ALL], orphanRemoval = true)
    @JoinColumn(name = "order_id")
    val items: MutableList<OrderItemJpaEntity> = mutableListOf(),
) {
    fun toDomain(): Order {
        // Mapping logic using reflection or manual construction
        // This is implementation-specific
    }
}

fun Order.toJpaEntity(): OrderJpaEntity {
    // Mapping logic from domain to JPA
}
```

## Configuration

### Bean Configuration

Wire dependencies using Spring configuration.

```kotlin
@Configuration
class BeanConfiguration {

    @Bean
    fun createOrderUseCase(
        orderRepository: OrderRepository,
        eventPublisher: EventPublisher,
    ): CreateOrderUseCase {
        return CreateOrderService(orderRepository, eventPublisher)
    }

    @Bean
    fun orderRepository(
        jpaRepository: OrderJpaRepository,
    ): OrderRepository {
        return OrderPersistenceAdapter(jpaRepository)
    }
}
```

## Exception Handling

Handle exceptions at adapter boundaries. Convert domain exceptions to appropriate HTTP responses.

### Domain Exceptions

Define specific exceptions for business rule violations in the domain layer.

```kotlin
sealed class OrderException(message: String) : RuntimeException(message)

class OrderNotFoundException(orderId: OrderId) :
    OrderException("Order not found: ${orderId.value}")

class InvalidOrderStateException(
    orderId: OrderId,
    currentState: OrderStatus,
    requiredState: OrderStatus,
) : OrderException(
    "Order ${orderId.value} is in state $currentState, required $requiredState",
)

class OrderValidationException(message: String) : OrderException(message)

class InsufficientInventoryException(
    productId: ProductId,
    requested: Int,
    available: Int,
) : OrderException(
    "Insufficient inventory for product ${productId.value}: requested $requested, available $available",
)
```

### Global Exception Handler

Use @ControllerAdvice to handle exceptions at the adapter layer.

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException::class)
    fun handleOrderNotFound(ex: OrderNotFoundException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(
                ErrorResponse(
                    code = "ORDER_NOT_FOUND",
                    message = ex.message ?: "Order not found",
                    timestamp = Instant.now(),
                ),
            )
    }

    @ExceptionHandler(InvalidOrderStateException::class)
    fun handleInvalidOrderState(ex: InvalidOrderStateException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(
                ErrorResponse(
                    code = "INVALID_ORDER_STATE",
                    message = ex.message ?: "Invalid order state",
                    timestamp = Instant.now(),
                ),
            )
    }

    @ExceptionHandler(OrderValidationException::class)
    fun handleValidationError(ex: OrderValidationException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(
                ErrorResponse(
                    code = "VALIDATION_ERROR",
                    message = ex.message ?: "Validation failed",
                    timestamp = Instant.now(),
                ),
            )
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidationErrors(
        ex: MethodArgumentNotValidException,
    ): ResponseEntity<ValidationErrorResponse> {
        val errors = ex.bindingResult.fieldErrors.map { error ->
            FieldError(
                field = error.field,
                message = error.defaultMessage ?: "Invalid value",
                rejectedValue = error.rejectedValue?.toString(),
            )
        }

        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(
                ValidationErrorResponse(
                    code = "VALIDATION_ERROR",
                    message = "Request validation failed",
                    errors = errors,
                    timestamp = Instant.now(),
                ),
            )
    }

    @ExceptionHandler(Exception::class)
    fun handleGenericError(ex: Exception): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(
                ErrorResponse(
                    code = "INTERNAL_ERROR",
                    message = "An unexpected error occurred",
                    timestamp = Instant.now(),
                ),
            )
    }
}
```

### Error Response DTOs

Define standard error response structures in the adapter layer.

```kotlin
data class ErrorResponse(
    val code: String,
    val message: String,
    val timestamp: Instant,
)

data class ValidationErrorResponse(
    val code: String,
    val message: String,
    val errors: List<FieldError>,
    val timestamp: Instant,
)

data class FieldError(
    val field: String,
    val message: String,
    val rejectedValue: String?,
)
```

### Exception Handling Principles

- Domain exceptions extend a common sealed class or base exception
- Never catch exceptions in the domain layer
- Convert domain exceptions to HTTP responses only in @ControllerAdvice
- Use specific HTTP status codes (404 for not found, 409 for conflict, 400 for validation)
- Include error codes for client-side handling
- Log exceptions appropriately (domain exceptions at WARN, unexpected at ERROR)
- Never expose internal implementation details in error messages

## Validation

Validate at multiple layers with different purposes.

### Request DTO Validation

Use Bean Validation annotations for basic input validation at the adapter layer.

```kotlin
data class CreateOrderRequest(
    @field:NotBlank(message = "Customer ID is required")
    val customerId: String,

    @field:NotEmpty(message = "Order must have at least one item")
    @field:Valid
    val items: List<OrderItemRequest>,
)

data class OrderItemRequest(
    @field:NotBlank(message = "Product ID is required")
    val productId: String,

    @field:Min(value = 1, message = "Quantity must be at least 1")
    @field:Max(value = 1000, message = "Quantity cannot exceed 1000")
    val quantity: Int,

    @field:DecimalMin(value = "0.01", message = "Price must be positive")
    val price: BigDecimal,

    @field:Pattern(regexp = "[A-Z]{3}", message = "Currency must be a valid ISO code")
    val currency: String,
)
```

### Controller Validation

Enable validation in controllers using @Valid annotation.

```kotlin
@RestController
@RequestMapping("/api/v1/orders")
class OrderController(
    private val createOrderUseCase: CreateOrderUseCase,
) {

    @PostMapping
    fun createOrder(
        @Valid @RequestBody request: CreateOrderRequest,
    ): ResponseEntity<OrderResponse> {
        val command = request.toCommand()

        return createOrderUseCase.execute(command)
            .map { orderId ->
                ResponseEntity
                    .created(URI.create("/api/v1/orders/${orderId.value}"))
                    .body(OrderResponse(orderId.value))
            }
            .getOrElse { exception ->
                throw exception
            }
    }
}
```

### Domain Validation

Implement business rule validation in domain entities and value objects.

```kotlin
@JvmInline
value class OrderId(val value: String) {
    init {
        require(value.isNotBlank()) { "OrderId cannot be blank" }
        require(value.length <= 100) { "OrderId too long" }
    }
}

data class Money(val amount: BigDecimal, val currency: Currency) {
    init {
        require(amount >= BigDecimal.ZERO) {
            "Money amount cannot be negative: $amount"
        }
        require(amount.scale() <= 2) {
            "Money amount cannot have more than 2 decimal places"
        }
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) {
            "Cannot add money with different currencies: $currency vs ${other.currency}"
        }
        return Money(amount + other.amount, currency)
    }
}

class Order private constructor(
    val id: OrderId,
    private val items: MutableList<OrderItem>,
    private var status: OrderStatus,
) {
    fun addItem(item: OrderItem) {
        require(status == OrderStatus.DRAFT) {
            "Cannot add items to order in status $status"
        }
        require(items.size < MAX_ITEMS) {
            "Order cannot have more than $MAX_ITEMS items"
        }
        require(items.none { it.productId == item.productId }) {
            "Product ${item.productId.value} already in order"
        }
        items.add(item)
    }

    companion object {
        private const val MAX_ITEMS = 100
    }
}
```

### Custom Validators

Create custom validators for complex business rules.

```kotlin
@Component
class OrderValidator {

    fun validateOrderCreation(command: CreateOrderCommand): ValidationResult {
        val errors = mutableListOf<String>()

        if (command.items.isEmpty()) {
            errors.add("Order must have at least one item")
        }

        if (command.items.size > 100) {
            errors.add("Order cannot have more than 100 items")
        }

        val totalAmount = command.items.sumOf { it.price.amount * it.quantity.toBigDecimal() }
        if (totalAmount > BigDecimal("1000000")) {
            errors.add("Order total exceeds maximum allowed amount")
        }

        return if (errors.isEmpty()) {
            ValidationResult.Valid
        } else {
            ValidationResult.Invalid(errors)
        }
    }
}

sealed class ValidationResult {
    object Valid : ValidationResult()
    data class Invalid(val errors: List<String>) : ValidationResult()
}
```

### Validation Principles

- Adapter layer: validate format and basic constraints (Bean Validation)
- Domain layer: validate business rules and invariants (require, init blocks)
- Use @Valid for nested object validation
- Return validation errors with field-level detail
- Domain validation throws domain exceptions
- Adapter validation returns 400 Bad Request
- Never skip validation assuming upstream already validated

## Transaction Management

Manage transactions at the application layer boundary.

### Transaction Boundaries

Place @Transactional on application service methods, not in domain or adapters.

```kotlin
@Service
class CreateOrderService(
    private val orderRepository: OrderRepository,
    private val inventoryService: InventoryService,
    private val eventPublisher: EventPublisher,
) : CreateOrderUseCase {

    @Transactional
    override fun execute(command: CreateOrderCommand): Result<OrderId> {
        return runCatching {
            val orderId = OrderId(UUID.randomUUID().toString())
            val order = Order.create(orderId)

            command.items.forEach { itemData ->
                inventoryService.reserveStock(itemData.productId, itemData.quantity)

                val item = OrderItem(
                    productId = itemData.productId,
                    quantity = itemData.quantity,
                    price = itemData.price,
                )
                order.addItem(item)
            }

            val event = order.submit()
            orderRepository.save(order)
            eventPublisher.publish(event)

            orderId
        }
    }
}

@Service
class CancelOrderService(
    private val orderRepository: OrderRepository,
    private val inventoryService: InventoryService,
) : CancelOrderUseCase {

    @Transactional
    override fun execute(command: CancelOrderCommand): Result<Unit> {
        return runCatching {
            val order = orderRepository.findById(command.orderId)
                ?: throw OrderNotFoundException(command.orderId)

            order.cancel(command.reason)

            order.items.forEach { item ->
                inventoryService.releaseStock(item.productId, item.quantity)
            }

            orderRepository.save(order)
        }
    }
}
```

### Read-Only Transactions

Use @Transactional(readOnly = true) for query operations.

```kotlin
@Service
class GetOrderService(
    private val orderRepository: OrderRepository,
) : GetOrderUseCase {

    @Transactional(readOnly = true)
    override fun execute(orderId: OrderId): Order? {
        return orderRepository.findById(orderId)
    }
}

@Service
class ListOrdersService(
    private val orderRepository: OrderRepository,
) : ListOrdersUseCase {

    @Transactional(readOnly = true)
    override fun execute(customerId: CustomerId): List<Order> {
        return orderRepository.findByCustomerId(customerId)
    }
}
```

### Transaction Propagation

Control transaction propagation for nested calls.

```kotlin
@Service
class ComplexOrderService(
    private val orderRepository: OrderRepository,
    private val paymentService: PaymentService,
    private val notificationService: NotificationService,
) {

    @Transactional
    fun processOrder(orderId: OrderId): Result<Unit> {
        return runCatching {
            val order = orderRepository.findById(orderId)
                ?: throw OrderNotFoundException(orderId)

            // This runs in the same transaction
            paymentService.processPayment(order.id, order.totalAmount)

            order.markAsPaid()
            orderRepository.save(order)

            // This runs in a separate transaction (fire-and-forget)
            notificationService.sendOrderConfirmation(order)
        }
    }
}

@Service
class NotificationService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun sendOrderConfirmation(order: Order) {
        // This runs in its own transaction
        // Failure here won't rollback the order transaction
    }
}
```

### Transaction Management Principles

- @Transactional belongs only in application layer (use case implementations)
- Never use @Transactional in domain layer (domain must be framework-agnostic)
- Never use @Transactional in adapters (controllers, repositories)
- Use readOnly = true for queries to optimize database performance
- Keep transactions short and focused on a single use case
- Use Propagation.REQUIRES_NEW sparingly for independent operations
- Let exceptions propagate to trigger automatic rollback
- Avoid catching exceptions inside @Transactional methods unless rethrowing
- Test transaction boundaries with integration tests

## Auditing

Track who created and modified entities and when. Use this pattern in all production entities.

### When to Use

**Always Use:**
- All domain entities in production systems
- Any entity that needs change tracking
- Compliance and audit requirements
- Debugging and troubleshooting

**Pattern Applies To:**
- All aggregate roots
- Important value objects that persist separately
- Any entity with business significance

### Base Auditing Entity

Create a base class for auditing fields in the domain layer.

```kotlin
abstract class AuditableEntity(
    var createdAt: Instant = Instant.now(),
    var createdBy: String? = null,
    var updatedAt: Instant = Instant.now(),
    var updatedBy: String? = null,
)
```

### JPA Entity with Auditing

Apply auditing to JPA entities using Spring Data JPA auditing.

```kotlin
@Entity
@Table(name = "orders")
@EntityListeners(AuditingEntityListener::class)
class OrderJpaEntity(
    @Id
    val id: String,

    @Column(nullable = false)
    val customerId: String,

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    var status: OrderStatus,

    @OneToMany(cascade = [CascadeType.ALL], orphanRemoval = true)
    @JoinColumn(name = "order_id")
    val items: MutableList<OrderItemJpaEntity> = mutableListOf(),

    @CreatedDate
    @Column(nullable = false, updatable = false)
    var createdAt: Instant = Instant.now(),

    @CreatedBy
    @Column(updatable = false)
    var createdBy: String? = null,

    @LastModifiedDate
    @Column(nullable = false)
    var updatedAt: Instant = Instant.now(),

    @LastModifiedBy
    var updatedBy: String? = null,
)
```

### Enable JPA Auditing

Configure Spring Data JPA auditing in the configuration layer.

```kotlin
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
class JpaAuditingConfiguration {

    @Bean
    fun auditorProvider(): AuditorAware<String> {
        return AuditorAwareImpl()
    }
}

class AuditorAwareImpl : AuditorAware<String> {

    override fun getCurrentAuditor(): Optional<String> {
        return Optional.ofNullable(
            SecurityContextHolder.getContext()
                .authentication
                ?.name
                ?: "system",
        )
    }
}
```

### Domain Model with Audit Info

Include audit information in domain entities when needed for business logic.

```kotlin
class Order private constructor(
    val id: OrderId,
    private val items: MutableList<OrderItem>,
    private var status: OrderStatus,
    val createdAt: Instant,
    val createdBy: String?,
) {
    fun canBeCancelledBy(userId: String): Boolean {
        // Business rule: only creator can cancel within 24 hours
        val isCreator = createdBy == userId
        val isWithin24Hours = Instant.now().isBefore(createdAt.plus(24, ChronoUnit.HOURS))
        return isCreator && isWithin24Hours
    }

    companion object {
        fun create(id: OrderId, createdBy: String): Order {
            return Order(
                id = id,
                items = mutableListOf(),
                status = OrderStatus.DRAFT,
                createdAt = Instant.now(),
                createdBy = createdBy,
            )
        }
    }
}
```

### Auditing Principles

- Always include audit fields in JPA entities
- Use @CreatedDate, @CreatedBy, @LastModifiedDate, @LastModifiedBy annotations
- Enable @EnableJpaAuditing in configuration
- Implement AuditorAware to provide current user
- Include audit info in domain model only when needed for business logic
- Use Instant for timestamps (UTC, timezone-independent)
- Make createdAt and createdBy immutable (updatable = false)
- Default to "system" when no authenticated user exists

## Event Publishing

Publish domain events to decouple business logic from side effects. Use for medium to large-scale projects.

### When to Use

**Use For:**
- Medium to large-scale projects with complex business logic
- Multiple systems or modules need to react to domain changes
- Asynchronous processing is required for performance
- Frequent addition of side effects (notifications, analytics, integrations)
- Loose coupling between bounded contexts
- Eventual consistency is acceptable

**Do NOT Use For:**
- Simple CRUD applications
- Small projects with straightforward 1:1 relationships
- When direct method calls are sufficient and clearer
- Early-stage projects (add later when complexity grows)
- Tightly coupled operations that must succeed or fail together
- When immediate consistency is required

### Domain Events

Define domain events in the domain layer as data classes.

```kotlin
sealed interface OrderEvent {
    val orderId: OrderId
    val occurredAt: Instant
}

data class OrderCreatedEvent(
    override val orderId: OrderId,
    val customerId: CustomerId,
    val totalAmount: Money,
    override val occurredAt: Instant = Instant.now(),
) : OrderEvent

data class OrderSubmittedEvent(
    override val orderId: OrderId,
    val totalAmount: Money,
    override val occurredAt: Instant = Instant.now(),
) : OrderEvent

data class OrderCancelledEvent(
    override val orderId: OrderId,
    val reason: String,
    override val occurredAt: Instant = Instant.now(),
) : OrderEvent
```

### Publishing Events

Publish events from application services after persisting changes.

```kotlin
@Service
class CreateOrderService(
    private val orderRepository: OrderRepository,
    private val eventPublisher: ApplicationEventPublisher,
) : CreateOrderUseCase {

    @Transactional
    override fun execute(command: CreateOrderCommand): Result<OrderId> {
        return runCatching {
            val orderId = OrderId(UUID.randomUUID().toString())
            val order = Order.create(orderId)

            command.items.forEach { itemData ->
                val item = OrderItem(
                    productId = itemData.productId,
                    quantity = itemData.quantity,
                    price = itemData.price,
                )
                order.addItem(item)
            }

            order.submit()
            orderRepository.save(order)

            // Publish event after successful persistence
            eventPublisher.publishEvent(
                OrderCreatedEvent(
                    orderId = order.id,
                    customerId = command.customerId,
                    totalAmount = order.totalAmount,
                ),
            )

            orderId
        }
    }
}
```

### Event Handlers

Handle events in separate components for different concerns.

```kotlin
@Component
class OrderNotificationEventHandler(
    private val emailService: EmailService,
) {

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleOrderCreated(event: OrderCreatedEvent) {
        emailService.sendOrderConfirmation(event.orderId)
    }

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleOrderCancelled(event: OrderCancelledEvent) {
        emailService.sendCancellationNotification(event.orderId)
    }
}

@Component
class OrderInventoryEventHandler(
    private val inventoryService: InventoryService,
) {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleOrderCreated(event: OrderCreatedEvent) {
        inventoryService.reserveStock(event.orderId)
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleOrderCancelled(event: OrderCancelledEvent) {
        inventoryService.releaseStock(event.orderId)
    }
}

@Component
class OrderAnalyticsEventHandler(
    private val analyticsService: AnalyticsService,
) {

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleOrderCreated(event: OrderCreatedEvent) {
        analyticsService.trackEvent("order_created", event.orderId.value)
    }
}
```

### Enable Async Processing

Configure async processing for event handlers.

```kotlin
@Configuration
@EnableAsync
class AsyncConfiguration {

    @Bean
    fun taskExecutor(): Executor {
        val executor = ThreadPoolTaskExecutor()
        executor.corePoolSize = 5
        executor.maxPoolSize = 10
        executor.queueCapacity = 100
        executor.setThreadNamePrefix("async-event-")
        executor.initialize()
        return executor
    }
}
```

### Event Publishing Principles

- Define events in the domain layer as immutable data classes
- Publish events from application services, not from domain entities
- Use ApplicationEventPublisher from Spring Framework
- Use @TransactionalEventListener with AFTER_COMMIT phase
- Use @Async for non-critical handlers (notifications, analytics)
- Keep handlers focused on a single concern
- Never throw exceptions from async event handlers
- Log failures in event handlers for monitoring
- Consider event store for audit trail if needed
- Events represent facts that already happened (past tense naming)

## Optimistic Locking

Prevent lost updates in concurrent modifications using version-based locking. Use when concurrent access is expected.

### When to Use

**Use For:**
- Entities modified by multiple users concurrently
- Long-running transactions or user think time
- Distributed systems with multiple instances
- Shopping carts, inventory, reservations, bookings
- Any entity where lost updates would cause business problems

**Do NOT Use For:**
- Read-only entities
- Entities modified by a single user or process
- High-contention scenarios (use pessimistic locking instead)
- Simple CRUD with no concurrent access

### JPA Entity with Versioning

Add @Version field to JPA entities.

```kotlin
@Entity
@Table(name = "orders")
class OrderJpaEntity(
    @Id
    val id: String,

    @Column(nullable = false)
    val customerId: String,

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    var status: OrderStatus,

    @Version
    @Column(nullable = false)
    var version: Long = 0,

    @OneToMany(cascade = [CascadeType.ALL], orphanRemoval = true)
    @JoinColumn(name = "order_id")
    val items: MutableList<OrderItemJpaEntity> = mutableListOf(),
)
```

### Domain Model with Version

Include version in domain model when needed for business logic.

```kotlin
class Order private constructor(
    val id: OrderId,
    private val items: MutableList<OrderItem>,
    private var status: OrderStatus,
    val version: Long,
) {
    fun addItem(item: OrderItem) {
        require(status == OrderStatus.DRAFT) {
            "Cannot add items to order in status $status"
        }
        items.add(item)
    }

    companion object {
        fun create(id: OrderId): Order {
            return Order(
                id = id,
                items = mutableListOf(),
                status = OrderStatus.DRAFT,
                version = 0,
            )
        }
    }
}
```

### Handling Version Conflicts

Handle OptimisticLockingFailureException in the adapter layer.

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(OptimisticLockingFailureException::class)
    fun handleOptimisticLocking(
        ex: OptimisticLockingFailureException,
    ): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(
                ErrorResponse(
                    code = "CONCURRENT_MODIFICATION",
                    message = "The resource was modified by another user. Please refresh and try again.",
                    timestamp = Instant.now(),
                ),
            )
    }
}
```

### Retry Strategy

Implement retry logic for transient conflicts in application services.

```kotlin
@Service
class UpdateOrderService(
    private val orderRepository: OrderRepository,
) : UpdateOrderUseCase {

    @Transactional
    override fun execute(command: UpdateOrderCommand): Result<Unit> {
        return retry(maxAttempts = 3) {
            val order = orderRepository.findById(command.orderId)
                ?: throw OrderNotFoundException(command.orderId)

            order.addItem(command.item)
            orderRepository.save(order)
        }
    }

    private fun <T> retry(maxAttempts: Int, block: () -> T): Result<T> {
        var lastException: Exception? = null

        repeat(maxAttempts) { attempt ->
            try {
                return Result.success(block())
            } catch (ex: OptimisticLockingFailureException) {
                lastException = ex
                if (attempt < maxAttempts - 1) {
                    Thread.sleep(100L * (attempt + 1)) // Exponential backoff
                }
            }
        }

        return Result.failure(
            lastException ?: IllegalStateException("Retry failed"),
        )
    }
}
```

### Optimistic Locking Principles

- Add @Version field to all entities with concurrent access
- Use Long or Integer for version field
- JPA automatically increments version on each update
- Never manually set or modify the version field
- Handle OptimisticLockingFailureException in @ControllerAdvice
- Return 409 Conflict status code for version conflicts
- Implement retry logic with exponential backoff for transient conflicts
- Maximum 3 retry attempts recommended
- Log version conflicts for monitoring
- Inform users to refresh and retry on conflict
- Use pessimistic locking for high-contention scenarios instead

## Testing Strategy

### Domain Testing

Test domain logic without any infrastructure dependencies.

```kotlin
class OrderTest {

    @Test
    fun `should add item to draft order`() {
        val order = Order.create(OrderId("123"))
        val item = OrderItem(
            productId = ProductId("P1"),
            quantity = 2,
            price = Money(BigDecimal("10.00"), Currency.getInstance("USD"))
        )

        order.addItem(item)

        assertEquals(Money(BigDecimal("20.00"), Currency.getInstance("USD")), order.totalAmount)
    }

    @Test
    fun `should not add item to submitted order`() {
        val order = Order.create(OrderId("123"))
        order.submit()

        assertThrows<IllegalArgumentException> {
            order.addItem(createTestItem())
        }
    }
}
```

### Use Case Testing

Test use cases with mocked output ports.

```kotlin
class CreateOrderServiceTest {

    private val orderRepository = mockk<OrderRepository>()
    private val eventPublisher = mockk<EventPublisher>()
    private val useCase = CreateOrderService(orderRepository, eventPublisher)

    @Test
    fun `should create order successfully`() {
        every { orderRepository.save(any()) } just Runs
        every { eventPublisher.publish(any()) } just Runs

        val command = CreateOrderCommand(
            customerId = CustomerId("C1"),
            items = listOf(createTestItemData())
        )

        val result = useCase.execute(command)

        assertTrue(result.isSuccess)
        verify { orderRepository.save(any()) }
        verify { eventPublisher.publish(any<OrderSubmittedEvent>()) }
    }
}
```

### Integration Testing

Test adapters with real infrastructure (using Testcontainers).

```kotlin
@SpringBootTest
@Testcontainers
class OrderPersistenceAdapterIT {

    @Container
    val postgres = PostgreSQLContainer<Nothing>("postgres:15")

    @Autowired
    lateinit var adapter: OrderRepository

    @Test
    fun `should save and retrieve order`() {
        val order = Order.create(OrderId("123"))

        adapter.save(order)
        val retrieved = adapter.findById(OrderId("123"))

        assertNotNull(retrieved)
        assertEquals(order.id, retrieved?.id)
    }
}
```

## Common Patterns

### Result Type for Error Handling

Use Kotlin Result or create custom Result type.

```kotlin
sealed class DomainResult<out T> {
    data class Success<T>(val value: T) : DomainResult<T>()
    data class Failure(val error: DomainError) : DomainResult<Nothing>()
}

sealed interface DomainError {
    val message: String
}

data class OrderNotFoundError(override val message: String) : DomainError
data class ValidationError(override val message: String) : DomainError
```

### Query vs Command Separation

Separate read and write operations.

```kotlin
// Command (writes)
interface CreateOrderUseCase {
    fun execute(command: CreateOrderCommand): Result<OrderId>
}

// Query (reads)
interface GetOrderQuery {
    fun execute(orderId: OrderId): Order?
}

interface ListOrdersQuery {
    fun execute(customerId: CustomerId): List<OrderSummary>
}
```

## Anti-Patterns to Avoid

### Avoid: Exposing JPA Entities Directly

**Wrong:**
```kotlin
@RestController
class OrderController(private val jpaRepository: OrderJpaRepository) {
    @GetMapping("/{id}")
    fun getOrder(@PathVariable id: String): OrderJpaEntity {
        return jpaRepository.findById(id).get()  // Exposes JPA entity
    }
}
```

**Correct:**
```kotlin
@RestController
class OrderController(private val getOrderUseCase: GetOrderUseCase) {
    @GetMapping("/{id}")
    fun getOrder(@PathVariable id: String): OrderResponse {
        return getOrderUseCase.execute(OrderId(id))
            .toResponse()
    }
}
```

### Avoid: Domain Dependencies on Spring

**Wrong:**
```kotlin
@Entity  // Spring/JPA annotation in domain
class Order(val id: OrderId) {
    @Autowired  // Spring dependency in domain
    lateinit var orderRepository: OrderRepository
}
```

**Correct:**
```kotlin
// Pure domain class, no annotations
class Order(val id: OrderId) {
    // Business logic only
}
```

### Avoid: Business Logic in Adapters

**Wrong:**
```kotlin
@RestController
class OrderController(private val repository: OrderRepository) {
    @PostMapping
    fun createOrder(@RequestBody request: CreateOrderRequest): OrderResponse {
        val order = Order.create(OrderId(UUID.randomUUID().toString()))

        // Business logic in controller (WRONG)
        request.items.forEach { order.addItem(it.toOrderItem()) }
        order.submit()

        repository.save(order)
        return order.toResponse()
    }
}
```

**Correct:**
```kotlin
@RestController
class OrderController(private val createOrderUseCase: CreateOrderUseCase) {
    @PostMapping
    fun createOrder(@RequestBody request: CreateOrderRequest): OrderResponse {
        // Delegate to use case
        return createOrderUseCase.execute(request.toCommand())
            .toResponse()
    }
}
```

## Code Review Checklist

When reviewing code for hexagonal architecture compliance:

**Architecture:**
- [ ] Domain layer has zero dependencies on Spring, JPA, or external libraries
- [ ] All use cases are defined as interfaces in domain/port/input
- [ ] All external dependencies are defined as interfaces in domain/port/output
- [ ] Package structure follows the defined convention (domain/application/adapter)
- [ ] No circular dependencies between layers

**Domain Layer:**
- [ ] Entities use regular classes, not data classes
- [ ] Value Objects use data classes or inline value classes
- [ ] Business logic is in domain entities and services, not in controllers or adapters
- [ ] Domain exceptions are used for business rule violations
- [ ] Domain events are defined in domain layer (if using event publishing)

**Application Layer:**
- [ ] Transaction boundaries are in application layer, not domain
- [ ] Use cases are implemented as application services
- [ ] @Transactional annotations only in application services
- [ ] Events are published after successful persistence (if applicable)

**Adapter Layer:**
- [ ] Controllers only handle HTTP concerns and delegate to use cases
- [ ] JPA entities are separate from domain entities
- [ ] Mappers convert between JPA entities and domain entities
- [ ] Request/Response DTOs are in adapter layer
- [ ] Exception handling is done in @ControllerAdvice

**Validation:**
- [ ] Request DTOs use Bean Validation annotations
- [ ] Domain entities validate business rules in constructors and methods
- [ ] Validation errors return appropriate HTTP status codes

**Auditing (Production Code):**
- [ ] All JPA entities include audit fields (createdAt, createdBy, updatedAt, updatedBy)
- [ ] @EntityListeners(AuditingEntityListener::class) is present
- [ ] @EnableJpaAuditing is configured
- [ ] AuditorAware is implemented to provide current user

**Event Publishing (Medium/Large Projects):**
- [ ] Domain events are immutable data classes
- [ ] Events published from application services, not domain entities
- [ ] @TransactionalEventListener with AFTER_COMMIT phase
- [ ] @Async used for non-critical handlers
- [ ] Event handlers are focused on single concern

**Optimistic Locking (Concurrent Access):**
- [ ] @Version field present in entities with concurrent access
- [ ] OptimisticLockingFailureException handled in @ControllerAdvice
- [ ] Version conflicts return 409 Conflict status
- [ ] Retry logic implemented for transient conflicts

**Testing:**
- [ ] Tests for domain logic have no Spring dependencies
- [ ] Use case tests use mocked output ports
- [ ] Integration tests verify adapter implementations

**Code Quality:**
- [ ] Code follows ktlint conventions
- [ ] Maximum line length 120 characters
- [ ] Trailing commas in multiline parameter lists
- [ ] No wildcard imports

## Summary

Hexagonal Architecture provides clear separation of concerns and maintainable code structure. Follow these principles consistently:

1. Domain is pure and has no external dependencies
2. All external interactions go through ports
3. Adapters implement ports and handle technical concerns
4. Business logic lives in the domain layer
5. Use cases orchestrate domain objects to fulfill business requirements
6. Testing becomes easier with clear boundaries and dependency injection
