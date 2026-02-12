# Code Quality

## Purpose
Maintain high code quality through SOLID principles, design patterns, clean code practices, and extensibility patterns for SAP Commerce development.

## SOLID Principles

### 1. Single Responsibility Principle (SRP)

Each class should have one reason to change.

```java
// ❌ Wrong: Multiple responsibilities
public class ProductService {
    public void createProduct(ProductModel product) { }
    public void sendEmail(String to, String subject) { }
    public void generateReport() { }
    public void syncToExternalSystem() { }
}

// ✅ Correct: Single responsibility
public class ProductService {
    public void createProduct(ProductModel product) { }
    public ProductModel getProduct(String code) { }
}

public class EmailService {
    public void sendEmail(String to, String subject, String body) { }
}

public class ReportService {
    public void generateProductReport() { }
}

public class ExternalSyncService {
    public void syncProduct(ProductModel product) { }
}
```

### 2. Open/Closed Principle (OCP)

Open for extension, closed for modification.

```java
// ✅ Correct: Extensible design
public interface PriceCalculationStrategy {
    double calculatePrice(ProductModel product, int quantity);
}

public class StandardPriceStrategy implements PriceCalculationStrategy {
    @Override
    public double calculatePrice(ProductModel product, int quantity) {
        return product.getBasePrice() * quantity;
    }
}

public class VolumeDiscountStrategy implements PriceCalculationStrategy {
    @Override
    public double calculatePrice(ProductModel product, int quantity) {
        double basePrice = product.getBasePrice() * quantity;
        if (quantity >= 10) {
            return basePrice * 0.9; // 10% discount
        }
        return basePrice;
    }
}

@Service
public class PriceService {
    private final Map<String, PriceCalculationStrategy> strategies;
    
    public double calculatePrice(ProductModel product, int quantity, String strategyType) {
        PriceCalculationStrategy strategy = strategies.get(strategyType);
        return strategy.calculatePrice(product, quantity);
    }
}
```

### 3. Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types.

```java
// ✅ Correct: Proper inheritance
public interface OrderValidator {
    ValidationResult validate(OrderModel order);
}

public class MinimumAmountValidator implements OrderValidator {
    @Override
    public ValidationResult validate(OrderModel order) {
        // Always returns ValidationResult
        ValidationResult result = new ValidationResult();
        if (order.getTotalPrice() < 10.0) {
            result.addError("Minimum amount not met");
        }
        return result;
    }
}

public class StockValidator implements OrderValidator {
    @Override
    public ValidationResult validate(OrderModel order) {
        // Always returns ValidationResult
        ValidationResult result = new ValidationResult();
        // Validation logic
        return result;
    }
}
```

### 4. Interface Segregation Principle (ISP)

Clients shouldn't depend on interfaces they don't use.

```java
// ❌ Wrong: Fat interface
public interface ProductOperations {
    void create();
    void update();
    void delete();
    void export();
    void import();
    void sync();
    void validate();
}

// ✅ Correct: Segregated interfaces
public interface ProductCrudOperations {
    void create();
    void update();
    void delete();
}

public interface ProductImportExport {
    void export();
    void importData();
}

public interface ProductSync {
    void sync();
}

public interface ProductValidation {
    void validate();
}
```

### 5. Dependency Inversion Principle (DIP)

Depend on abstractions, not concretions.

```java
// ✅ Correct: Depend on abstractions
@Service
public class DefaultOrderService implements OrderService {
    
    private final PaymentService paymentService;  // Interface
    private final InventoryService inventoryService;  // Interface
    private final NotificationService notificationService;  // Interface
    
    public DefaultOrderService(
            PaymentService paymentService,
            InventoryService inventoryService,
            NotificationService notificationService) {
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
        this.notificationService = notificationService;
    }
}
```

## Design Patterns

### 1. Strategy Pattern

```java
public interface DiscountStrategy {
    double applyDiscount(double price);
}

@Component("percentageDiscount")
public class PercentageDiscountStrategy implements DiscountStrategy {
    @Override
    public double applyDiscount(double price) {
        return price * 0.9; // 10% off
    }
}

@Component("fixedDiscount")
public class FixedDiscountStrategy implements DiscountStrategy {
    @Override
    public double applyDiscount(double price) {
        return price - 10.0; // $10 off
    }
}

@Service
public class PricingService {
    private final Map<String, DiscountStrategy> strategies;
    
    public double calculateFinalPrice(double price, String discountType) {
        DiscountStrategy strategy = strategies.getOrDefault(
            discountType, new NoDiscountStrategy());
        return strategy.applyDiscount(price);
    }
}
```

### 2. Factory Pattern

```java
public interface PaymentProcessor {
    void processPayment(OrderModel order);
}

@Component
public class PaymentProcessorFactory {
    
    private final Map<String, PaymentProcessor> processors;
    
    public PaymentProcessorFactory(List<PaymentProcessor> processorList) {
        this.processors = processorList.stream()
            .collect(Collectors.toMap(
                p -> p.getClass().getSimpleName(),
                p -> p
            ));
    }
    
    public PaymentProcessor getProcessor(String type) {
        PaymentProcessor processor = processors.get(type);
        if (processor == null) {
            throw new IllegalArgumentException("Unknown payment type: " + type);
        }
        return processor;
    }
}
```

### 3. Template Method Pattern

```java
public abstract class AbstractOrderProcessor {
    
    public final void processOrder(OrderModel order) {
        validateOrder(order);
        calculateTotals(order);
        applyDiscounts(order);
        processPayment(order);
        updateInventory(order);
        sendConfirmation(order);
    }
    
    protected abstract void validateOrder(OrderModel order);
    protected abstract void applyDiscounts(OrderModel order);
    protected abstract void processPayment(OrderModel order);
    
    protected void calculateTotals(OrderModel order) {
        // Common implementation
    }
    
    protected void updateInventory(OrderModel order) {
        // Common implementation
    }
    
    protected void sendConfirmation(OrderModel order) {
        // Common implementation
    }
}

@Service
public class B2COrderProcessor extends AbstractOrderProcessor {
    @Override
    protected void validateOrder(OrderModel order) {
        // B2C specific validation
    }
    
    @Override
    protected void applyDiscounts(OrderModel order) {
        // B2C specific discounts
    }
    
    @Override
    protected void processPayment(OrderModel order) {
        // B2C payment processing
    }
}
```

## Clean Code Practices

### 1. Meaningful Names

```java
// ❌ Wrong
public class Mgr {
    public void proc(List<Obj> lst) {
        for (Obj o : lst) {
            int x = o.getVal();
            if (x > 0) {
                // ...
            }
        }
    }
}

// ✅ Correct
public class OrderManager {
    public void processOrders(List<OrderModel> orders) {
        for (OrderModel order : orders) {
            double totalAmount = order.getTotalPrice();
            if (totalAmount > 0) {
                // ...
            }
        }
    }
}
```

### 2. Small Functions

```java
// ✅ Correct: Small, focused functions
@Service
public class OrderService {
    
    public void placeOrder(OrderModel order) {
        validateOrder(order);
        calculateTotals(order);
        processPayment(order);
        updateInventory(order);
        sendConfirmation(order);
    }
    
    private void validateOrder(OrderModel order) {
        if (order.getEntries().isEmpty()) {
            throw new IllegalArgumentException("Order has no entries");
        }
    }
    
    private void calculateTotals(OrderModel order) {
        double total = order.getEntries().stream()
            .mapToDouble(e -> e.getBasePrice() * e.getQuantity())
            .sum();
        order.setTotalPrice(total);
    }
    
    private void processPayment(OrderModel order) {
        paymentService.charge(order);
    }
    
    private void updateInventory(OrderModel order) {
        inventoryService.reserve(order.getEntries());
    }
    
    private void sendConfirmation(OrderModel order) {
        emailService.sendOrderConfirmation(order);
    }
}
```

### 3. Comments

```java
// ❌ Wrong: Obvious comments
// Get the product
ProductModel product = productService.getProduct(code);

// Set the price
product.setPrice(99.99);

// ✅ Correct: Explain why, not what
// Apply temporary promotional pricing for holiday sale
product.setPrice(calculateHolidayPrice(product.getBasePrice()));

// Workaround for SAP Commerce bug #12345: 
// CatalogVersion must be set before saving
product.setCatalogVersion(catalogVersionService.getSessionCatalogVersion());
```

## Extensibility Patterns

### 1. Extension Points

```java
// Define extension point
public interface ProductEnrichmentHook {
    void enrich(ProductModel product);
}

@Service
public class ProductService {
    
    private final List<ProductEnrichmentHook> enrichmentHooks;
    
    public void createProduct(ProductModel product) {
        // Core logic
        modelService.save(product);
        
        // Extension point
        enrichmentHooks.forEach(hook -> hook.enrich(product));
    }
}

// Implement extension
@Component
public class PriceEnrichmentHook implements ProductEnrichmentHook {
    @Override
    public void enrich(ProductModel product) {
        // Add pricing data
    }
}
```

### 2. Decorator Pattern

```java
public interface OrderService {
    void placeOrder(OrderModel order);
}

@Service("coreOrderService")
public class CoreOrderService implements OrderService {
    @Override
    public void placeOrder(OrderModel order) {
        // Core implementation
    }
}

@Service("orderService")
@Primary
public class AuditedOrderService implements OrderService {
    
    private final OrderService delegate;
    private final AuditService auditService;
    
    public AuditedOrderService(
            @Qualifier("coreOrderService") OrderService delegate,
            AuditService auditService) {
        this.delegate = delegate;
        this.auditService = auditService;
    }
    
    @Override
    public void placeOrder(OrderModel order) {
        auditService.logBefore("placeOrder", order);
        delegate.placeOrder(order);
        auditService.logAfter("placeOrder", order);
    }
}
```

## Best Practices

### ✅ DO
- Follow SOLID principles
- Use design patterns appropriately
- Write self-documenting code
- Keep functions small and focused
- Use meaningful names
- Favor composition over inheritance
- Program to interfaces
- Make classes immutable when possible
- Use dependency injection
- Write testable code

### ❌ DON'T
- Create god classes
- Use magic numbers
- Duplicate code
- Create deep inheritance hierarchies
- Use global state
- Ignore exceptions
- Premature optimization
- Over-engineer solutions
- Skip code reviews
- Ignore technical debt

## Code Review Checklist

- [ ] Follows SOLID principles
- [ ] Uses appropriate design patterns
- [ ] Has meaningful names
- [ ] Functions are small and focused
- [ ] No code duplication
- [ ] Proper error handling
- [ ] Has unit tests
- [ ] Documented complex logic
- [ ] No hardcoded values
- [ ] Follows project conventions

## Resources

- **Clean Code** by Robert C. Martin
- **Design Patterns** by Gang of Four
- **Effective Java** by Joshua Bloch
- **Refactoring** by Martin Fowler
