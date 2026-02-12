# Interceptors

## Purpose
Implement model lifecycle hooks to execute logic before/after model operations (create, load, save, remove). Interceptors enable validation, data transformation, and business rule enforcement.

## Key Concepts

### Interceptor Types
1. **PrepareInterceptor**: Before model save (validation, defaults)
2. **ValidateInterceptor**: Validation before save
3. **InitDefaultsInterceptor**: Set default values on creation
4. **LoadInterceptor**: After model load from database
5. **RemoveInterceptor**: Before model removal

### Execution Order
```
Create: InitDefaults -> Prepare -> Validate -> Save
Load: Load from DB -> LoadInterceptor
Update: Prepare -> Validate -> Save
Remove: RemoveInterceptor -> Delete
```

## Interceptor Types

### 1. PrepareInterceptor

Executes before model save for data preparation and transformation.

```java
package com.mycompany.core.interceptor;

import de.hybris.platform.servicelayer.interceptor.InterceptorContext;
import de.hybris.platform.servicelayer.interceptor.InterceptorException;
import de.hybris.platform.servicelayer.interceptor.PrepareInterceptor;
import de.hybris.platform.core.model.product.ProductModel;
import org.springframework.stereotype.Component;

@Component
public class ProductPrepareInterceptor implements PrepareInterceptor<ProductModel> {
    
    @Override
    public void onPrepare(ProductModel product, InterceptorContext ctx) 
            throws InterceptorException {
        
        // Normalize product code to uppercase
        if (product.getCode() != null) {
            product.setCode(product.getCode().toUpperCase());
        }
        
        // Set modification timestamp
        product.setModifiedTime(new Date());
        
        // Calculate derived values
        if (product.getBasePrice() != null && product.getDiscountPercent() != null) {
            double discountedPrice = product.getBasePrice() * 
                (1 - product.getDiscountPercent() / 100.0);
            product.setFinalPrice(discountedPrice);
        }
    }
}
```

### 2. ValidateInterceptor

Validates model data before save.

```java
package com.mycompany.core.interceptor;

import de.hybris.platform.servicelayer.interceptor.InterceptorContext;
import de.hybris.platform.servicelayer.interceptor.InterceptorException;
import de.hybris.platform.servicelayer.interceptor.ValidateInterceptor;
import de.hybris.platform.core.model.product.ProductModel;
import org.apache.commons.lang3.StringUtils;
import org.springframework.stereotype.Component;

@Component
public class ProductValidateInterceptor implements ValidateInterceptor<ProductModel> {
    
    private static final int MAX_CODE_LENGTH = 50;
    private static final int MIN_NAME_LENGTH = 3;
    
    @Override
    public void onValidate(ProductModel product, InterceptorContext ctx) 
            throws InterceptorException {
        
        // Validate product code
        if (StringUtils.isBlank(product.getCode())) {
            throw new InterceptorException("Product code cannot be empty");
        }
        
        if (product.getCode().length() > MAX_CODE_LENGTH) {
            throw new InterceptorException(
                String.format("Product code too long: %d (max %d)", 
                    product.getCode().length(), MAX_CODE_LENGTH));
        }
        
        // Validate product name
        if (StringUtils.isBlank(product.getName())) {
            throw new InterceptorException("Product name cannot be empty");
        }
        
        if (product.getName().length() < MIN_NAME_LENGTH) {
            throw new InterceptorException(
                String.format("Product name too short: %d (min %d)", 
                    product.getName().length(), MIN_NAME_LENGTH));
        }
        
        // Validate price
        if (product.getBasePrice() != null && product.getBasePrice() < 0) {
            throw new InterceptorException("Product price cannot be negative");
        }
        
        // Validate stock level
        if (product.getStockLevel() != null && product.getStockLevel() < 0) {
            throw new InterceptorException("Stock level cannot be negative");
        }
        
        // Business rule: Featured products must have image
        if (Boolean.TRUE.equals(product.getFeatured()) && product.getPicture() == null) {
            throw new InterceptorException("Featured products must have an image");
        }
    }
}
```

### 3. InitDefaultsInterceptor

Sets default values when model is created.

```java
package com.mycompany.core.interceptor;

import de.hybris.platform.servicelayer.interceptor.InitDefaultsInterceptor;
import de.hybris.platform.servicelayer.interceptor.InterceptorContext;
import de.hybris.platform.servicelayer.interceptor.InterceptorException;
import de.hybris.platform.core.model.product.ProductModel;
import org.springframework.stereotype.Component;

@Component
public class ProductInitDefaultsInterceptor 
        implements InitDefaultsInterceptor<ProductModel> {
    
    private final ConfigurationService configurationService;
    
    public ProductInitDefaultsInterceptor(ConfigurationService configurationService) {
        this.configurationService = configurationService;
    }
    
    @Override
    public void onInitDefaults(ProductModel product, InterceptorContext ctx) 
            throws InterceptorException {
        
        // Set default active status
        if (product.getActive() == null) {
            product.setActive(Boolean.TRUE);
        }
        
        // Set default featured status
        if (product.getFeatured() == null) {
            product.setFeatured(Boolean.FALSE);
        }
        
        // Set default stock level
        if (product.getStockLevel() == null) {
            int defaultStock = configurationService.getConfiguration()
                .getInt("mycompany.product.defaultStock", 0);
            product.setStockLevel(defaultStock);
        }
        
        // Set creation timestamp
        product.setCreationTime(new Date());
        
        // Set default approval status
        if (product.getApprovalStatus() == null) {
            product.setApprovalStatus(ApprovalStatus.PENDING);
        }
    }
}
```

### 4. LoadInterceptor

Executes after model is loaded from database.

```java
package com.mycompany.core.interceptor;

import de.hybris.platform.servicelayer.interceptor.LoadInterceptor;
import de.hybris.platform.servicelayer.interceptor.InterceptorContext;
import de.hybris.platform.servicelayer.interceptor.InterceptorException;
import de.hybris.platform.core.model.product.ProductModel;
import org.springframework.stereotype.Component;

@Component
public class ProductLoadInterceptor implements LoadInterceptor<ProductModel> {
    
    private final CacheService cacheService;
    private final AuditService auditService;
    
    public ProductLoadInterceptor(
            CacheService cacheService,
            AuditService auditService) {
        this.cacheService = cacheService;
        this.auditService = auditService;
    }
    
    @Override
    public void onLoad(ProductModel product, InterceptorContext ctx) 
            throws InterceptorException {
        
        // Warm up cache
        cacheService.put("product:" + product.getCode(), product);
        
        // Track product views (be careful with performance)
        if (shouldTrackView(ctx)) {
            auditService.logProductView(product);
        }
        
        // Decrypt sensitive data
        if (product.getEncryptedData() != null) {
            String decrypted = decrypt(product.getEncryptedData());
            product.setDecryptedData(decrypted);
        }
    }
    
    private boolean shouldTrackView(InterceptorContext ctx) {
        // Only track in specific contexts to avoid performance issues
        return ctx.getAttribute("trackView") == Boolean.TRUE;
    }
}
```

### 5. RemoveInterceptor

Executes before model is removed.

```java
package com.mycompany.core.interceptor;

import de.hybris.platform.servicelayer.interceptor.RemoveInterceptor;
import de.hybris.platform.servicelayer.interceptor.InterceptorContext;
import de.hybris.platform.servicelayer.interceptor.InterceptorException;
import de.hybris.platform.core.model.product.ProductModel;
import org.springframework.stereotype.Component;

@Component
public class ProductRemoveInterceptor implements RemoveInterceptor<ProductModel> {
    
    private final OrderService orderService;
    private final AuditService auditService;
    
    public ProductRemoveInterceptor(
            OrderService orderService,
            AuditService auditService) {
        this.orderService = orderService;
        this.auditService = auditService;
    }
    
    @Override
    public void onRemove(ProductModel product, InterceptorContext ctx) 
            throws InterceptorException {
        
        // Prevent deletion if product has orders
        if (orderService.hasOrders(product)) {
            throw new InterceptorException(
                "Cannot delete product with existing orders: " + product.getCode());
        }
        
        // Create audit log
        auditService.log(
            "PRODUCT_DELETED",
            product.getCode(),
            getCurrentUser(),
            "Product deleted: " + product.getName()
        );
        
        // Clean up related data
        cleanupProductData(product);
    }
    
    private void cleanupProductData(ProductModel product) {
        // Remove from cache
        cacheService.evict("product:" + product.getCode());
        
        // Clean up media files
        if (product.getPicture() != null) {
            mediaService.deleteMedia(product.getPicture());
        }
    }
}
```

## Advanced Patterns

### 1. Conditional Validation

```java
@Component
public class ConditionalProductInterceptor implements ValidateInterceptor<ProductModel> {
    
    @Override
    public void onValidate(ProductModel product, InterceptorContext ctx) 
            throws InterceptorException {
        
        // Only validate if specific attribute changed
        if (ctx.isModified(product, ProductModel.PRICE)) {
            validatePriceChange(product, ctx);
        }
        
        // Only validate for new products
        if (ctx.isNew(product)) {
            validateNewProduct(product);
        }
        
        // Skip validation in specific contexts
        if (ctx.getAttribute("skipValidation") == Boolean.TRUE) {
            return;
        }
        
        performStandardValidation(product);
    }
    
    private void validatePriceChange(ProductModel product, InterceptorContext ctx) 
            throws InterceptorException {
        // Get original price
        Double oldPrice = ctx.getDirtyValues(product).get(ProductModel.PRICE);
        Double newPrice = product.getPrice();
        
        // Validate price change
        if (oldPrice != null && newPrice != null) {
            double changePercent = Math.abs((newPrice - oldPrice) / oldPrice * 100);
            if (changePercent > 50) {
                throw new InterceptorException(
                    "Price change exceeds 50%: " + changePercent + "%");
            }
        }
    }
}
```

### 2. Cascading Operations

```java
@Component
public class OrderPrepareInterceptor implements PrepareInterceptor<OrderModel> {
    
    private final ModelService modelService;
    
    @Override
    public void onPrepare(OrderModel order, InterceptorContext ctx) 
            throws InterceptorException {
        
        // Update order entries
        if (order.getEntries() != null) {
            for (OrderEntryModel entry : order.getEntries()) {
                // Calculate entry total
                if (entry.getQuantity() != null && entry.getBasePrice() != null) {
                    double total = entry.getQuantity() * entry.getBasePrice();
                    entry.setTotalPrice(total);
                    modelService.save(entry);
                }
            }
        }
        
        // Calculate order total
        calculateOrderTotal(order);
    }
}
```

### 3. External System Integration

```java
@Component
public class ProductExternalSyncInterceptor implements PrepareInterceptor<ProductModel> {
    
    private final ExternalSystemService externalService;
    private final ConfigurationService configurationService;
    
    @Override
    public void onPrepare(ProductModel product, InterceptorContext ctx) 
            throws InterceptorException {
        
        // Only sync if enabled
        boolean syncEnabled = configurationService.getConfiguration()
            .getBoolean("mycompany.externalSync.enabled", false);
        
        if (!syncEnabled) {
            return;
        }
        
        // Only sync if specific fields changed
        if (ctx.isModified(product, ProductModel.PRICE) || 
            ctx.isModified(product, ProductModel.STOCKLEVEL)) {
            
            try {
                externalService.syncProduct(product);
            } catch (ExternalSystemException e) {
                // Log error but don't fail save
                LOG.error("Failed to sync product to external system", e);
            }
        }
    }
}
```

## Spring Configuration

```xml
<!-- resources/mycompanycore-spring.xml -->
<beans>
    <!-- Interceptors are auto-discovered via @Component -->
    
    <!-- Or define explicitly -->
    <bean id="productPrepareInterceptor"
          class="com.mycompany.core.interceptor.ProductPrepareInterceptor">
        <property name="interceptorMapping">
            <bean class="de.hybris.platform.servicelayer.interceptor.impl.InterceptorMapping">
                <property name="interceptor" ref="productPrepareInterceptor"/>
                <property name="typeCode" value="Product"/>
            </bean>
        </property>
    </bean>
    
    <bean id="productValidateInterceptor"
          class="com.mycompany.core.interceptor.ProductValidateInterceptor">
        <property name="interceptorMapping">
            <bean class="de.hybris.platform.servicelayer.interceptor.impl.InterceptorMapping">
                <property name="interceptor" ref="productValidateInterceptor"/>
                <property name="typeCode" value="Product"/>
            </bean>
        </property>
    </bean>
</beans>
```

## InterceptorContext Usage

```java
@Component
public class AdvancedProductInterceptor implements PrepareInterceptor<ProductModel> {
    
    @Override
    public void onPrepare(ProductModel product, InterceptorContext ctx) 
            throws InterceptorException {
        
        // Check if model is new
        if (ctx.isNew(product)) {
            LOG.info("New product being created: {}", product.getCode());
        }
        
        // Check if specific attribute was modified
        if (ctx.isModified(product, ProductModel.PRICE)) {
            LOG.info("Price changed for product: {}", product.getCode());
        }
        
        // Get original values
        Map<String, Object> dirtyValues = ctx.getDirtyValues(product);
        Object oldPrice = dirtyValues.get(ProductModel.PRICE);
        
        // Set context attribute for other interceptors
        ctx.setAttribute("priceChanged", Boolean.TRUE);
        
        // Get context attribute
        Boolean skipValidation = (Boolean) ctx.getAttribute("skipValidation");
        
        // Check if model was removed
        if (ctx.isRemoved(product)) {
            LOG.info("Product marked for removal: {}", product.getCode());
        }
    }
}
```

## Best Practices

### DO
- Keep interceptors lightweight and fast
- Use appropriate interceptor type for task
- Validate in ValidateInterceptor
- Set defaults in InitDefaultsInterceptor
- Handle exceptions gracefully
- Use InterceptorContext to check modifications
- Log important operations
- Test interceptors thoroughly
- Document business rules
- Use conditional logic to minimize overhead

### DON'T
- Perform heavy operations (database queries, external calls)
- Modify other models without care
- Throw generic exceptions
- Skip null checks
- Create circular dependencies
- Use interceptors for business logic (use services)
- Perform async operations
- Access session-dependent data
- Modify immutable attributes
- Create infinite loops

## Common Pitfalls

### 1. Infinite Loops

```java
// Wrong: Causes infinite loop
@Override
public void onPrepare(ProductModel product, InterceptorContext ctx) 
        throws InterceptorException {
    product.setModifiedTime(new Date());
    modelService.save(product); // Triggers interceptor again!
}

// Correct: Check if already modified
@Override
public void onPrepare(ProductModel product, InterceptorContext ctx) 
        throws InterceptorException {
    if (!ctx.isModified(product, ProductModel.MODIFIEDTIME)) {
        product.setModifiedTime(new Date());
    }
}
```

### 2. Performance Issues

```java
// Wrong: Heavy operation in interceptor
@Override
public void onPrepare(ProductModel product, InterceptorContext ctx) 
        throws InterceptorException {
    // Expensive database query
    List<OrderModel> orders = orderDao.findOrdersForProduct(product);
    // Process orders...
}

// Correct: Use events for heavy operations
@Override
public void onPrepare(ProductModel product, InterceptorContext ctx) 
        throws InterceptorException {
    // Light validation only
    if (product.getPrice() < 0) {
        throw new InterceptorException("Invalid price");
    }
    
    // Publish event for heavy processing
    if (ctx.isModified(product, ProductModel.PRICE)) {
        eventService.publishEvent(new ProductPriceChangedEvent(product));
    }
}
```

### 3. Transaction Issues

```java
// Wrong: Modifying other models without transaction
@Override
public void onPrepare(ProductModel product, InterceptorContext ctx) 
        throws InterceptorException {
    CategoryModel category = product.getPrimaryCategory();
    category.setProductCount(category.getProductCount() + 1);
    modelService.save(category); // May cause issues
}

// Correct: Be careful with cascading saves
@Override
public void onPrepare(ProductModel product, InterceptorContext ctx) 
        throws InterceptorException {
    // Only modify if necessary and within same transaction
    if (ctx.isNew(product)) {
        // Mark for update, don't save directly
        product.getPrimaryCategory().setNeedsUpdate(true);
    }
}
```

## Testing Interceptors

```java
@ExtendWith(MockitoExtension.class)
class ProductValidateInterceptorTest {
    
    @Mock
    private InterceptorContext ctx;
    
    @InjectMocks
    private ProductValidateInterceptor interceptor;
    
    @Test
    void testValidate_ValidProduct() throws InterceptorException {
        // Given
        ProductModel product = new ProductModel();
        product.setCode("TEST001");
        product.setName("Test Product");
        product.setBasePrice(99.99);
        
        // When/Then - should not throw
        assertDoesNotThrow(() -> interceptor.onValidate(product, ctx));
    }
    
    @Test
    void testValidate_InvalidCode() {
        // Given
        ProductModel product = new ProductModel();
        product.setCode(""); // Invalid
        product.setName("Test Product");
        
        // When/Then
        InterceptorException exception = assertThrows(
            InterceptorException.class,
            () -> interceptor.onValidate(product, ctx)
        );
        
        assertTrue(exception.getMessage().contains("code cannot be empty"));
    }
}
```

## Resources

- **Interceptor Documentation**: Help Portal -> Interceptors
- **Model Lifecycle**: Platform documentation
- **Performance Best Practices**: Tuning guide
