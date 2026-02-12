# Dynamic Attributes

## Purpose
Implement computed attributes that are calculated at runtime without database storage. Dynamic attributes are ideal for derived values, aggregations, and expensive calculations that shouldn't be persisted.

## Key Concepts

### What are Dynamic Attributes?
- **Computed at runtime**: Values calculated on-demand
- **No database storage**: Not persisted in database columns
- **Read-only or read-write**: Can be computed-only or support setting
- **Performance consideration**: Calculation happens on every access

### Use Cases
- Calculated fields (full name from first + last name)
- Aggregations (total order value from entries)
- Formatted values (display price with currency)
- External system lookups
- Complex business logic results

## Defining Dynamic Attributes

### 1. items.xml Configuration

```xml
<itemtypes>
    <itemtype code="Product" autocreate="false" generate="false">
        <attributes>
            <!-- Read-only dynamic attribute -->
            <attribute qualifier="displayName" type="java.lang.String">
                <persistence type="dynamic" 
                           attributeHandler="productDisplayNameHandler"/>
                <modifiers read="true" write="false" search="false" optional="true"/>
            </attribute>
            
            <!-- Read-write dynamic attribute -->
            <attribute qualifier="formattedPrice" type="java.lang.String">
                <persistence type="dynamic" 
                           attributeHandler="productFormattedPriceHandler"/>
                <modifiers read="true" write="true" search="false" optional="true"/>
            </attribute>
            
            <!-- Dynamic attribute with localization -->
            <attribute qualifier="localizedDisplayName" type="localized:java.lang.String">
                <persistence type="dynamic" 
                           attributeHandler="productLocalizedDisplayNameHandler"/>
                <modifiers read="true" write="false" search="false" optional="true"/>
            </attribute>
        </attributes>
    </itemtype>
    
    <itemtype code="Order" autocreate="false" generate="false">
        <attributes>
            <!-- Calculated total from entries -->
            <attribute qualifier="calculatedTotal" type="java.lang.Double">
                <persistence type="dynamic" 
                           attributeHandler="orderCalculatedTotalHandler"/>
                <modifiers read="true" write="false" search="false" optional="true"/>
            </attribute>
        </attributes>
    </itemtype>
</itemtypes>
```

### 2. Attribute Handler Implementation

#### Read-Only Handler

```java
package com.mycompany.core.attribute;

import de.hybris.platform.servicelayer.model.attribute.AbstractDynamicAttributeHandler;
import org.springframework.stereotype.Component;

@Component("productDisplayNameHandler")
public class ProductDisplayNameHandler 
        extends AbstractDynamicAttributeHandler<String, ProductModel> {
    
    @Override
    public String get(ProductModel product) {
        if (product == null) {
            return null;
        }
        
        // Compute display name from code and name
        final String code = product.getCode();
        final String name = product.getName();
        
        if (name != null && code != null) {
            return String.format("%s - %s", code, name);
        }
        
        return code != null ? code : name;
    }
}
```

#### Read-Write Handler

```java
package com.mycompany.core.attribute;

import de.hybris.platform.servicelayer.model.attribute.DynamicAttributeHandler;
import org.springframework.stereotype.Component;

@Component("productFormattedPriceHandler")
public class ProductFormattedPriceHandler 
        implements DynamicAttributeHandler<String, ProductModel> {
    
    private final PriceService priceService;
    private final ModelService modelService;
    
    public ProductFormattedPriceHandler(
            PriceService priceService,
            ModelService modelService) {
        this.priceService = priceService;
        this.modelService = modelService;
    }
    
    @Override
    public String get(ProductModel product) {
        if (product == null) {
            return null;
        }
        
        final PriceInformation priceInfo = priceService.getPriceInformation(product);
        if (priceInfo != null) {
            final PriceValue priceValue = priceInfo.getPriceValue();
            return String.format("%s %.2f", 
                priceValue.getCurrencyIso(), 
                priceValue.getValue());
        }
        
        return null;
    }
    
    @Override
    public void set(ProductModel product, String formattedPrice) {
        // Parse formatted price and update base price
        if (formattedPrice == null || product == null) {
            return;
        }
        
        // Parse: "USD 99.99"
        final String[] parts = formattedPrice.split(" ");
        if (parts.length == 2) {
            final String currency = parts[0];
            final double value = Double.parseDouble(parts[1]);
            
            // Update price (implementation depends on your price model)
            updateProductPrice(product, currency, value);
            modelService.save(product);
        }
    }
    
    private void updateProductPrice(ProductModel product, String currency, double value) {
        // Implementation specific to your price structure
    }
}
```

#### Aggregation Handler

```java
package com.mycompany.core.attribute;

import de.hybris.platform.servicelayer.model.attribute.AbstractDynamicAttributeHandler;
import org.springframework.stereotype.Component;

@Component("orderCalculatedTotalHandler")
public class OrderCalculatedTotalHandler 
        extends AbstractDynamicAttributeHandler<Double, OrderModel> {
    
    @Override
    public Double get(OrderModel order) {
        if (order == null || order.getEntries() == null) {
            return 0.0;
        }
        
        return order.getEntries().stream()
            .filter(entry -> entry.getBasePrice() != null && entry.getQuantity() != null)
            .mapToDouble(entry -> entry.getBasePrice() * entry.getQuantity())
            .sum();
    }
}
```

### 3. Spring Bean Configuration

```xml
<!-- resources/mycompanycore-spring.xml -->
<beans>
    <!-- Handlers are auto-discovered via @Component -->
    <!-- Or define explicitly: -->
    
    <bean id="productDisplayNameHandler"
          class="com.mycompany.core.attribute.ProductDisplayNameHandler"/>
    
    <bean id="productFormattedPriceHandler"
          class="com.mycompany.core.attribute.ProductFormattedPriceHandler">
        <constructor-arg ref="priceService"/>
        <constructor-arg ref="modelService"/>
    </bean>
    
    <bean id="orderCalculatedTotalHandler"
          class="com.mycompany.core.attribute.OrderCalculatedTotalHandler"/>
</beans>
```

## Advanced Patterns

### 1. Localized Dynamic Attribute

```java
@Component("productLocalizedDisplayNameHandler")
public class ProductLocalizedDisplayNameHandler 
        extends AbstractDynamicAttributeHandler<String, ProductModel> {
    
    private final I18NService i18nService;
    
    public ProductLocalizedDisplayNameHandler(I18NService i18nService) {
        this.i18nService = i18nService;
    }
    
    @Override
    public String get(ProductModel product) {
        if (product == null) {
            return null;
        }
        
        final Locale currentLocale = i18nService.getCurrentLocale();
        final String localizedName = product.getName(currentLocale);
        
        return String.format("[%s] %s", 
            currentLocale.getLanguage().toUpperCase(), 
            localizedName);
    }
}
```

### 2. Cached Dynamic Attribute

```java
@Component("productCachedStockHandler")
public class ProductCachedStockHandler 
        extends AbstractDynamicAttributeHandler<Integer, ProductModel> {
    
    private final StockService stockService;
    private final CacheService cacheService;
    
    private static final String CACHE_KEY_PREFIX = "product_stock_";
    private static final int CACHE_TTL_SECONDS = 300; // 5 minutes
    
    public ProductCachedStockHandler(
            StockService stockService,
            CacheService cacheService) {
        this.stockService = stockService;
        this.cacheService = cacheService;
    }
    
    @Override
    public Integer get(ProductModel product) {
        if (product == null) {
            return 0;
        }
        
        final String cacheKey = CACHE_KEY_PREFIX + product.getCode();
        
        // Try cache first
        Integer cachedStock = cacheService.get(cacheKey);
        if (cachedStock != null) {
            return cachedStock;
        }
        
        // Calculate and cache
        final Integer stock = stockService.getStockLevel(product);
        cacheService.put(cacheKey, stock, CACHE_TTL_SECONDS);
        
        return stock;
    }
}
```

### 3. External System Lookup

```java
@Component("productExternalDataHandler")
public class ProductExternalDataHandler 
        extends AbstractDynamicAttributeHandler<String, ProductModel> {
    
    private final ExternalSystemClient externalClient;
    
    public ProductExternalDataHandler(ExternalSystemClient externalClient) {
        this.externalClient = externalClient;
    }
    
    @Override
    public String get(ProductModel product) {
        if (product == null || product.getExternalId() == null) {
            return null;
        }
        
        try {
            return externalClient.fetchProductData(product.getExternalId());
        } catch (ExternalSystemException e) {
            // Log error and return fallback
            LOG.error("Failed to fetch external data for product: {}", 
                product.getCode(), e);
            return null;
        }
    }
}
```

### 4. Complex Business Logic

```java
@Component("orderEligibilityHandler")
public class OrderEligibilityHandler 
        extends AbstractDynamicAttributeHandler<Boolean, OrderModel> {
    
    private final UserService userService;
    private final PromotionService promotionService;
    private final InventoryService inventoryService;
    
    public OrderEligibilityHandler(
            UserService userService,
            PromotionService promotionService,
            InventoryService inventoryService) {
        this.userService = userService;
        this.promotionService = promotionService;
        this.inventoryService = inventoryService;
    }
    
    @Override
    public Boolean get(OrderModel order) {
        if (order == null) {
            return false;
        }
        
        // Check multiple conditions
        final boolean userEligible = checkUserEligibility(order.getUser());
        final boolean promotionValid = checkPromotionValidity(order);
        final boolean stockAvailable = checkStockAvailability(order);
        final boolean paymentValid = checkPaymentValidity(order);
        
        return userEligible && promotionValid && stockAvailable && paymentValid;
    }
    
    private boolean checkUserEligibility(UserModel user) {
        // Complex user eligibility logic
        return user != null && userService.isActive(user);
    }
    
    private boolean checkPromotionValidity(OrderModel order) {
        // Check if applied promotions are still valid
        return promotionService.validatePromotions(order);
    }
    
    private boolean checkStockAvailability(OrderModel order) {
        // Verify all items are in stock
        return inventoryService.checkAvailability(order.getEntries());
    }
    
    private boolean checkPaymentValidity(OrderModel order) {
        // Validate payment information
        return order.getPaymentInfo() != null;
    }
}
```

## Using Dynamic Attributes

### In Code

```java
@Service
public class ProductDisplayService {
    
    public void demonstrateDynamicAttributes(ProductModel product) {
        // Read dynamic attribute (triggers handler.get())
        String displayName = product.getDisplayName();
        System.out.println("Display Name: " + displayName);
        
        // Read-write dynamic attribute
        String formattedPrice = product.getFormattedPrice();
        System.out.println("Formatted Price: " + formattedPrice);
        
        // Set dynamic attribute (triggers handler.set())
        product.setFormattedPrice("USD 149.99");
        modelService.save(product);
        
        // Localized dynamic attribute
        String localizedName = product.getLocalizedDisplayName();
        System.out.println("Localized Name: " + localizedName);
    }
}
```

### In FlexibleSearch

```java
// Dynamic attributes cannot be used in FlexibleSearch WHERE clause
String query = "SELECT {pk} FROM {Product} WHERE {displayName} = ?name"; // FAILS

// Use persisted attributes for queries
String query = "SELECT {pk} FROM {Product} WHERE {code} = ?code";
List<ProductModel> products = flexibleSearchService.<ProductModel>search(query).getResult();

// Then filter by dynamic attribute in Java
List<ProductModel> filtered = products.stream()
    .filter(p -> "Expected Value".equals(p.getDisplayName()))
    .collect(Collectors.toList());
```

## Performance Considerations

### 1. Lazy Loading

Dynamic attributes are calculated on every access. Avoid repeated calls:

```java
// Wrong: Multiple calculations
for (ProductModel product : products) {
    log.info("Name: {}", product.getDisplayName()); // Calculated
    log.info("Display: {}", product.getDisplayName()); // Calculated again
}

// Correct: Cache result
for (ProductModel product : products) {
    String displayName = product.getDisplayName(); // Calculated once
    log.info("Name: {}", displayName);
    log.info("Display: {}", displayName);
}
```

### 2. Expensive Operations

For expensive calculations, consider caching:

```java
@Component("expensiveCalculationHandler")
public class ExpensiveCalculationHandler 
        extends AbstractDynamicAttributeHandler<String, ProductModel> {
    
    private final LoadingCache<String, String> cache = CacheBuilder.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build(new CacheLoader<String, String>() {
            @Override
            public String load(String productCode) {
                return performExpensiveCalculation(productCode);
            }
        });
    
    @Override
    public String get(ProductModel product) {
        try {
            return cache.get(product.getCode());
        } catch (ExecutionException e) {
            LOG.error("Cache error", e);
            return null;
        }
    }
    
    private String performExpensiveCalculation(String productCode) {
        // Expensive operation
        return "result";
    }
}
```

### 3. Batch Processing

Avoid dynamic attributes in batch processing when possible:

```java
// Wrong: Dynamic attribute in loop
for (ProductModel product : allProducts) {
    String displayName = product.getDisplayName(); // Calculated 10,000 times
    // Process...
}

// Correct: Use persisted attributes or pre-calculate
for (ProductModel product : allProducts) {
    String name = product.getName(); // From database
    String code = product.getCode(); // From database
    String displayName = code + " - " + name; // Calculated in Java
    // Process...
}
```

## Best Practices

### DO
- Use for calculated/derived values
- Keep handlers lightweight and fast
- Implement caching for expensive operations
- Use read-only when possible
- Document calculation logic
- Handle null values gracefully
- Use dependency injection in handlers
- Test handlers thoroughly
- Consider performance impact
- Use for values that change frequently

### DON'T
- Use for frequently queried attributes
- Perform database queries in handlers (use services)
- Store state in handler instances
- Use in FlexibleSearch WHERE clauses
- Perform expensive operations without caching
- Modify other models in get() method
- Throw unchecked exceptions
- Use for simple concatenations (use getters)
- Access dynamic attributes in loops unnecessarily
- Use when persisted attribute would suffice

## Common Pitfalls

### 1. Using in FlexibleSearch

```java
// Wrong: Dynamic attribute in query
String query = "SELECT {pk} FROM {Product} WHERE {displayName} LIKE ?pattern";
// FAILS: displayName is not a database column

// Correct: Query persisted attributes, filter in Java
String query = "SELECT {pk} FROM {Product}";
List<ProductModel> products = flexibleSearchService.<ProductModel>search(query).getResult();
List<ProductModel> filtered = products.stream()
    .filter(p -> p.getDisplayName().contains(pattern))
    .collect(Collectors.toList());
```

### 2. Performance in Loops

```java
// Wrong: Repeated calculation
for (ProductModel product : products) {
    if (product.getCalculatedValue() > threshold) { // Calculated
        log.info("Value: {}", product.getCalculatedValue()); // Calculated again
    }
}

// Correct: Calculate once
for (ProductModel product : products) {
    Double value = product.getCalculatedValue(); // Calculated once
    if (value > threshold) {
        log.info("Value: {}", value);
    }
}
```

### 3. Null Handling

```java
// Wrong: No null check
@Override
public String get(ProductModel product) {
    return product.getCode() + " - " + product.getName(); // NPE if name is null
}

// Correct: Handle nulls
@Override
public String get(ProductModel product) {
    if (product == null) {
        return null;
    }
    
    final String code = product.getCode();
    final String name = product.getName();
    
    if (code == null && name == null) {
        return null;
    }
    
    return String.format("%s - %s", 
        code != null ? code : "", 
        name != null ? name : "");
}
```

## Testing Dynamic Attributes

```java
@UnitTest
class ProductDisplayNameHandlerTest {
    
    private ProductDisplayNameHandler handler;
    
    @BeforeEach
    void setUp() {
        handler = new ProductDisplayNameHandler();
    }
    
    @Test
    void testGet_WithCodeAndName() {
        // Given
        ProductModel product = new ProductModel();
        product.setCode("PROD001");
        product.setName("Test Product");
        
        // When
        String result = handler.get(product);
        
        // Then
        assertEquals("PROD001 - Test Product", result);
    }
    
    @Test
    void testGet_WithNullProduct() {
        // When
        String result = handler.get(null);
        
        // Then
        assertNull(result);
    }
    
    @Test
    void testGet_WithNullName() {
        // Given
        ProductModel product = new ProductModel();
        product.setCode("PROD001");
        product.setName(null);
        
        // When
        String result = handler.get(product);
        
        // Then
        assertEquals("PROD001", result);
    }
}
```

## Resources

- **SAP Commerce Documentation**: Dynamic Attributes
- **Performance Tuning**: Help Portal -> Performance
- **Caching Strategies**: Platform caching documentation
