# Events

## Purpose
Implement event-driven architecture for loose coupling between components. Events enable asynchronous processing, auditing, and integration without direct dependencies.

## Key Concepts

### Event System
- **Publisher**: Publishes events via EventService
- **Listener**: Subscribes to and handles events
- **Event**: Data object containing event information
- **Asynchronous**: Events processed in separate threads
- **Transactional**: Events can be transaction-aware

### Use Cases
- Audit logging
- Email notifications
- Cache invalidation
- External system integration
- Workflow triggers
- Analytics tracking

## Creating Events

### 1. Define Event Class

```java
package com.mycompany.core.event;

import de.hybris.platform.servicelayer.event.events.AbstractEvent;
import de.hybris.platform.core.model.order.OrderModel;

/**
 * Event published when an order is placed
 */
public class OrderPlacedEvent extends AbstractEvent {
    
    private final OrderModel order;
    private final String userEmail;
    private final double totalAmount;
    
    public OrderPlacedEvent(OrderModel order) {
        this.order = order;
        this.userEmail = order.getUser().getUid();
        this.totalAmount = order.getTotalPrice();
    }
    
    public OrderModel getOrder() {
        return order;
    }
    
    public String getUserEmail() {
        return userEmail;
    }
    
    public double getTotalAmount() {
        return totalAmount;
    }
    
    @Override
    public String toString() {
        return String.format("OrderPlacedEvent[order=%s, user=%s, amount=%.2f]",
            order.getCode(), userEmail, totalAmount);
    }
}
```

### 2. Publish Event

```java
package com.mycompany.core.service.impl;

import de.hybris.platform.servicelayer.event.EventService;
import com.mycompany.core.event.OrderPlacedEvent;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class DefaultOrderService implements OrderService {
    
    private final ModelService modelService;
    private final EventService eventService;
    
    public DefaultOrderService(
            ModelService modelService,
            EventService eventService) {
        this.modelService = modelService;
        this.eventService = eventService;
    }
    
    @Transactional
    @Override
    public void placeOrder(OrderModel order) {
        // Validate and save order
        validateOrder(order);
        calculateTotals(order);
        order.setStatus(OrderStatus.CONFIRMED);
        modelService.save(order);
        
        // Publish event
        LOG.info("Publishing OrderPlacedEvent for order: {}", order.getCode());
        eventService.publishEvent(new OrderPlacedEvent(order));
    }
}
```

### 3. Create Event Listener

```java
package com.mycompany.core.event.listener;

import de.hybris.platform.servicelayer.event.impl.AbstractEventListener;
import com.mycompany.core.event.OrderPlacedEvent;
import org.springframework.stereotype.Component;

@Component
public class OrderPlacedEventListener extends AbstractEventListener<OrderPlacedEvent> {
    
    private final EmailService emailService;
    private final AuditService auditService;
    
    public OrderPlacedEventListener(
            EmailService emailService,
            AuditService auditService) {
        this.emailService = emailService;
        this.auditService = auditService;
    }
    
    @Override
    protected void onEvent(OrderPlacedEvent event) {
        LOG.info("Handling OrderPlacedEvent: {}", event);
        
        try {
            // Send confirmation email
            sendConfirmationEmail(event);
            
            // Create audit log
            createAuditLog(event);
            
            LOG.info("OrderPlacedEvent handled successfully");
        } catch (Exception e) {
            LOG.error("Error handling OrderPlacedEvent", e);
            // Don't throw - event processing should not fail the transaction
        }
    }
    
    private void sendConfirmationEmail(OrderPlacedEvent event) {
        EmailData email = new EmailData();
        email.setTo(event.getUserEmail());
        email.setSubject("Order Confirmation - " + event.getOrder().getCode());
        email.setBody(buildEmailBody(event));
        
        emailService.send(email);
    }
    
    private void createAuditLog(OrderPlacedEvent event) {
        auditService.log(
            "ORDER_PLACED",
            event.getOrder().getCode(),
            event.getUserEmail(),
            String.format("Order placed: %.2f", event.getTotalAmount())
        );
    }
    
    private String buildEmailBody(OrderPlacedEvent event) {
        return String.format(
            "Thank you for your order!\\n\\n" +
            "Order Number: %s\\n" +
            "Total Amount: $%.2f\\n\\n" +
            "We will send you a shipping confirmation soon.",
            event.getOrder().getCode(),
            event.getTotalAmount()
        );
    }
}
```

## Event Types

### 1. Business Events

```java
// Product updated event
public class ProductUpdatedEvent extends AbstractEvent {
    private final ProductModel product;
    private final String updatedBy;
    
    public ProductUpdatedEvent(ProductModel product, String updatedBy) {
        this.product = product;
        this.updatedBy = updatedBy;
    }
    
    // Getters...
}

// User registered event
public class UserRegisteredEvent extends AbstractEvent {
    private final UserModel user;
    private final String registrationSource;
    
    public UserRegisteredEvent(UserModel user, String source) {
        this.user = user;
        this.registrationSource = source;
    }
    
    // Getters...
}

// Payment processed event
public class PaymentProcessedEvent extends AbstractEvent {
    private final OrderModel order;
    private final PaymentStatus status;
    private final String transactionId;
    
    public PaymentProcessedEvent(OrderModel order, PaymentStatus status, String txId) {
        this.order = order;
        this.status = status;
        this.transactionId = txId;
    }
    
    // Getters...
}
```

### 2. System Events

```java
// Cache invalidation event
public class CacheInvalidationEvent extends AbstractEvent {
    private final String cacheRegion;
    private final String cacheKey;
    
    public CacheInvalidationEvent(String region, String key) {
        this.cacheRegion = region;
        this.cacheKey = key;
    }
    
    // Getters...
}

// Data import completed event
public class DataImportCompletedEvent extends AbstractEvent {
    private final String importType;
    private final int recordsProcessed;
    private final boolean success;
    
    public DataImportCompletedEvent(String type, int records, boolean success) {
        this.importType = type;
        this.recordsProcessed = records;
        this.success = success;
    }
    
    // Getters...
}
```

## Advanced Patterns

### 1. Multiple Listeners

```java
// Email notification listener
@Component
public class OrderEmailListener extends AbstractEventListener<OrderPlacedEvent> {
    
    @Override
    protected void onEvent(OrderPlacedEvent event) {
        sendConfirmationEmail(event);
    }
}

// Analytics listener
@Component
public class OrderAnalyticsListener extends AbstractEventListener<OrderPlacedEvent> {
    
    @Override
    protected void onEvent(OrderPlacedEvent event) {
        trackOrderPlaced(event);
    }
}

// Inventory listener
@Component
public class OrderInventoryListener extends AbstractEventListener<OrderPlacedEvent> {
    
    @Override
    protected void onEvent(OrderPlacedEvent event) {
        reserveInventory(event.getOrder());
    }
}
```

### 2. Conditional Event Handling

```java
@Component
public class ConditionalOrderListener extends AbstractEventListener<OrderPlacedEvent> {
    
    private final ConfigurationService configurationService;
    
    public ConditionalOrderListener(ConfigurationService configurationService) {
        this.configurationService = configurationService;
    }
    
    @Override
    protected void onEvent(OrderPlacedEvent event) {
        // Only process if feature is enabled
        boolean enabled = configurationService.getConfiguration()
            .getBoolean("mycompany.feature.orderNotifications", true);
        
        if (!enabled) {
            LOG.debug("Order notifications disabled, skipping event");
            return;
        }
        
        // Only process for high-value orders
        if (event.getTotalAmount() >= 1000.0) {
            notifyHighValueOrder(event);
        }
    }
}
```

### 3. Asynchronous Event Processing

```java
@Component
public class AsyncOrderListener extends AbstractEventListener<OrderPlacedEvent> {
    
    private final ExecutorService executorService;
    
    public AsyncOrderListener() {
        this.executorService = Executors.newFixedThreadPool(5);
    }
    
    @Override
    protected void onEvent(OrderPlacedEvent event) {
        // Process event asynchronously
        executorService.submit(() -> {
            try {
                processOrderAsync(event);
            } catch (Exception e) {
                LOG.error("Async processing failed", e);
            }
        });
    }
    
    private void processOrderAsync(OrderPlacedEvent event) {
        // Long-running operation
        LOG.info("Processing order asynchronously: {}", event.getOrder().getCode());
        // ... expensive operation ...
    }
    
    @PreDestroy
    public void shutdown() {
        executorService.shutdown();
    }
}
```

### 4. Event Chaining

```java
// First event
public class OrderValidatedEvent extends AbstractEvent {
    private final OrderModel order;
    
    public OrderValidatedEvent(OrderModel order) {
        this.order = order;
    }
    
    public OrderModel getOrder() {
        return order;
    }
}

// Listener that publishes another event
@Component
public class OrderValidationListener extends AbstractEventListener<OrderPlacedEvent> {
    
    private final OrderValidationService validationService;
    private final EventService eventService;
    
    @Override
    protected void onEvent(OrderPlacedEvent event) {
        OrderModel order = event.getOrder();
        
        // Validate order
        if (validationService.validate(order)) {
            // Publish next event in chain
            eventService.publishEvent(new OrderValidatedEvent(order));
        }
    }
}

// Next listener in chain
@Component
public class OrderProcessingListener extends AbstractEventListener<OrderValidatedEvent> {
    
    @Override
    protected void onEvent(OrderValidatedEvent event) {
        // Process validated order
        processOrder(event.getOrder());
    }
}
```

### 5. Transaction-Aware Events

```java
@Component
public class TransactionalOrderListener extends AbstractEventListener<OrderPlacedEvent> {
    
    private final TransactionTemplate transactionTemplate;
    
    public TransactionalOrderListener(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
    }
    
    @Override
    protected void onEvent(OrderPlacedEvent event) {
        // Execute in new transaction
        transactionTemplate.execute(status -> {
            try {
                processOrderInTransaction(event);
                return null;
            } catch (Exception e) {
                status.setRollbackOnly();
                LOG.error("Transaction failed", e);
                throw e;
            }
        });
    }
    
    private void processOrderInTransaction(OrderPlacedEvent event) {
        // Operations that require transaction
        updateInventory(event.getOrder());
        createShipment(event.getOrder());
    }
}
```

## Spring Configuration

```xml
<!-- resources/mycompanycore-spring.xml -->
<beans>
    <!-- Event listeners are auto-discovered via @Component -->
    
    <!-- Or define explicitly -->
    <bean id="orderPlacedEventListener"
          class="com.mycompany.core.event.listener.OrderPlacedEventListener"
          parent="abstractEventListener">
        <constructor-arg ref="emailService"/>
        <constructor-arg ref="auditService"/>
    </bean>
    
    <bean id="orderEmailListener"
          class="com.mycompany.core.event.listener.OrderEmailListener"
          parent="abstractEventListener">
        <constructor-arg ref="emailService"/>
    </bean>
</beans>
```

## Best Practices

### ✅ DO
- Use events for loose coupling
- Keep event listeners lightweight
- Handle exceptions in listeners
- Use meaningful event names
- Document event contracts
- Use events for cross-cutting concerns
- Consider async processing for expensive operations
- Test event publishing and handling
- Use events for audit trails
- Log event processing

### ❌ DON'T
- Use events for synchronous workflows
- Perform heavy operations in listeners
- Throw unchecked exceptions from listeners
- Create circular event dependencies
- Use events for direct method calls
- Modify event objects in listeners
- Skip error handling
- Use events for simple method calls
- Create too many events
- Ignore event processing failures

## Common Pitfalls

### 1. Exception Handling

```java
// ❌ Wrong: Uncaught exception fails transaction
@Override
protected void onEvent(OrderPlacedEvent event) {
    sendEmail(event); // May throw exception
}

// ✅ Correct: Handle exceptions
@Override
protected void onEvent(OrderPlacedEvent event) {
    try {
        sendEmail(event);
    } catch (Exception e) {
        LOG.error("Failed to send email", e);
        // Don't rethrow - event processing should not fail transaction
    }
}
```

### 2. Heavy Processing

```java
// ❌ Wrong: Blocking operation in listener
@Override
protected void onEvent(OrderPlacedEvent event) {
    // Blocks for 10 seconds
    externalService.syncOrder(event.getOrder());
}

// ✅ Correct: Async processing
@Override
protected void onEvent(OrderPlacedEvent event) {
    executorService.submit(() -> {
        externalService.syncOrder(event.getOrder());
    });
}
```

### 3. Transaction Issues

```java
// ❌ Wrong: Modifying models without transaction
@Override
protected void onEvent(OrderPlacedEvent event) {
    OrderModel order = event.getOrder();
    order.setProcessed(true);
    modelService.save(order); // May fail outside transaction
}

// ✅ Correct: Use transaction template
@Override
protected void onEvent(OrderPlacedEvent event) {
    transactionTemplate.execute(status -> {
        OrderModel order = event.getOrder();
        modelService.refresh(order);
        order.setProcessed(true);
        modelService.save(order);
        return null;
    });
}
```

## Testing Events

### Unit Test

```java
@ExtendWith(MockitoExtension.class)
class OrderPlacedEventListenerTest {
    
    @Mock
    private EmailService emailService;
    
    @Mock
    private AuditService auditService;
    
    @InjectMocks
    private OrderPlacedEventListener listener;
    
    @Test
    void testOnEvent_Success() {
        // Given
        OrderModel order = createTestOrder();
        OrderPlacedEvent event = new OrderPlacedEvent(order);
        
        // When
        listener.onEvent(event);
        
        // Then
        verify(emailService).send(any(EmailData.class));
        verify(auditService).log(eq("ORDER_PLACED"), anyString(), anyString(), anyString());
    }
}
```

### Integration Test

```java
@IntegrationTest
class OrderEventIntegrationTest extends ServicelayerTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private EventService eventService;
    
    private List<OrderPlacedEvent> capturedEvents = new ArrayList<>();
    
    @Before
    public void setUp() {
        // Register test listener
        eventService.registerEventListener(new AbstractEventListener<OrderPlacedEvent>() {
            @Override
            protected void onEvent(OrderPlacedEvent event) {
                capturedEvents.add(event);
            }
        });
    }
    
    @Test
    public void testOrderPlacedEventPublished() {
        // Given
        OrderModel order = createTestOrder();
        
        // When
        orderService.placeOrder(order);
        
        // Then
        assertEquals(1, capturedEvents.size());
        OrderPlacedEvent event = capturedEvents.get(0);
        assertEquals(order.getCode(), event.getOrder().getCode());
    }
}
```

## Resources

- **Event Service**: Help Portal → Event System
- **Spring Events**: Spring Framework documentation
- **Async Processing**: Platform threading documentation
