# Java Development Guidelines

## Purpose
Establish coding standards and best practices for Java development in SAP Commerce Cloud, focusing on Spring Framework integration, service layer patterns, and maintainable code structure.

## Key Concepts

### Service Layer Architecture
SAP Commerce uses a **Service Layer** as the primary API for business logic. The Jalo layer is deprecated and should not be used directly.

**Core Services:**
- `ModelService`: CRUD operations for models
- `FlexibleSearchService`: Query execution
- `SessionService`: Session management
- `UserService`: User operations
- `CatalogService`: Catalog management

### Spring Framework Integration
SAP Commerce is built on Spring Framework 6.2, providing:
- Dependency Injection (DI)
- Aspect-Oriented Programming (AOP)
- Transaction management
- Bean lifecycle management

## Coding Standards

### 1. Interface + Implementation Pattern

Always define an interface for services and provide a default implementation.

```java
// Interface
package com.mycompany.core.service;

public interface ProductService {
    ProductModel findByCode(String code);
    List<ProductModel> findByCategory(CategoryModel category);
    void updateStock(ProductModel product, int quantity);
}

// Implementation
package com.mycompany.core.service.impl;

import org.springframework.stereotype.Service;

@Service("productService")
public class DefaultProductService implements ProductService {
    
    private final FlexibleSearchService flexibleSearchService;
    private final ModelService modelService;
    
    // Constructor injection (Spring 6 best practice)
    public DefaultProductService(
            FlexibleSearchService flexibleSearchService,
            ModelService modelService) {
        this.flexibleSearchService = flexibleSearchService;
        this.modelService = modelService;
    }
    
    @Override
    public ProductModel findByCode(String code) {
        final String query = "SELECT {pk} FROM {Product} WHERE {code} = ?code";
        final FlexibleSearchQuery fQuery = new FlexibleSearchQuery(query);
        fQuery.addQueryParameter("code", code);
        
        try {
            return flexibleSearchService.searchUnique(fQuery);
        } catch (ModelNotFoundException e) {
            throw new UnknownIdentifierException("Product not found: " + code);
        }
    }
    
    @Override
    public List<ProductModel> findByCategory(CategoryModel category) {
        final String query = "SELECT {p:pk} FROM {Product AS p} " +
                           "WHERE {p:supercategories} = ?category";
        final FlexibleSearchQuery fQuery = new FlexibleSearchQuery(query);
        fQuery.addQueryParameter("category", category);
        
        return flexibleSearchService.<ProductModel>search(fQuery).getResult();
    }
    
    @Override
    public void updateStock(ProductModel product, int quantity) {
        product.setStockLevel(quantity);
        modelService.save(product);
    }
}
```

### 2. Dependency Injection

**DO: Constructor Injection (Preferred)**
```java
@Service
public class DefaultOrderService implements OrderService {
    
    private final ModelService modelService;
    private final UserService userService;
    private final CalculationService calculationService;
    
    // Constructor injection - immutable dependencies
    public DefaultOrderService(
            ModelService modelService,
            UserService userService,
            CalculationService calculationService) {
        this.modelService = modelService;
        this.userService = userService;
        this.calculationService = calculationService;
    }
}
```

**DON'T: Field Injection**
```java
@Service
public class DefaultOrderService implements OrderService {
    
    @Autowired
    private ModelService modelService; // Avoid field injection
    
    @Autowired
    private UserService userService; // Hard to test, mutable
}
```

### 3. Spring Bean Configuration

**XML Configuration** (`resources/[extension]-spring.xml`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- Enable component scanning -->
    <context:component-scan base-package="com.mycompany.core"/>
    
    <!-- Service beans -->
    <bean id="productService" 
          class="com.mycompany.core.service.impl.DefaultProductService">
        <constructor-arg ref="flexibleSearchService"/>
        <constructor-arg ref="modelService"/>
    </bean>
    
    <!-- Or use @Service annotation and component scanning -->
</beans>
```

**Java Configuration** (Spring 6 style):
```java
package com.mycompany.core.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ServiceConfig {
    
    @Bean
    public ProductService productService(
            FlexibleSearchService flexibleSearchService,
            ModelService modelService) {
        return new DefaultProductService(flexibleSearchService, modelService);
    }
}
```

### 4. Exception Handling

Use SAP Commerce exception hierarchy:

```java
import de.hybris.platform.servicelayer.exceptions.UnknownIdentifierException;
import de.hybris.platform.servicelayer.exceptions.ModelNotFoundException;
import de.hybris.platform.servicelayer.exceptions.AmbiguousIdentifierException;

public class DefaultProductService implements ProductService {
    
    @Override
    public ProductModel findByCode(String code) {
        if (StringUtils.isBlank(code)) {
            throw new IllegalArgumentException("Product code cannot be blank");
        }
        
        try {
            final String query = "SELECT {pk} FROM {Product} WHERE {code} = ?code";
            final FlexibleSearchQuery fQuery = new FlexibleSearchQuery(query);
            fQuery.addQueryParameter("code", code);
            
            return flexibleSearchService.searchUnique(fQuery);
        } catch (ModelNotFoundException e) {
            throw new UnknownIdentifierException(
                "Product not found with code: " + code, e);
        } catch (AmbiguousIdentifierException e) {
            throw new IllegalStateException(
                "Multiple products found with code: " + code, e);
        }
    }
}
```

### 5. Logging

Use SLF4J with proper log levels:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class DefaultOrderService implements OrderService {
    
    private static final Logger LOG = LoggerFactory.getLogger(DefaultOrderService.class);
    
    private final ModelService modelService;
    
    public DefaultOrderService(ModelService modelService) {
        this.modelService = modelService;
    }
    
    @Override
    public void placeOrder(OrderModel order) {
        LOG.debug("Placing order: {}", order.getCode());
        
        try {
            validateOrder(order);
            calculateTotals(order);
            modelService.save(order);
            
            LOG.info("Order placed successfully: {}", order.getCode());
        } catch (Exception e) {
            LOG.error("Failed to place order: {}", order.getCode(), e);
            throw new OrderException("Order placement failed", e);
        }
    }
    
    private void validateOrder(OrderModel order) {
        LOG.trace("Validating order: {}", order.getCode());
        // Validation logic
    }
}
```

**Log Levels:**
- `TRACE`: Very detailed debugging (method entry/exit)
- `DEBUG`: Detailed debugging information
- `INFO`: Important business events
- `WARN`: Potentially harmful situations
- `ERROR`: Error events that might still allow the application to continue

### 6. Transaction Management

Use `@Transactional` for transaction boundaries:

```java
import org.springframework.transaction.annotation.Transactional;

@Service
public class DefaultOrderService implements OrderService {
    
    private final ModelService modelService;
    private final InventoryService inventoryService;
    
    public DefaultOrderService(
            ModelService modelService,
            InventoryService inventoryService) {
        this.modelService = modelService;
        this.inventoryService = inventoryService;
    }
    
    @Transactional
    @Override
    public void placeOrder(OrderModel order) {
        // All operations in single transaction
        validateOrder(order);
        reserveInventory(order);
        calculateTotals(order);
        modelService.save(order);
    }
    
    @Transactional(readOnly = true)
    @Override
    public OrderModel findByCode(String code) {
        // Read-only transaction (optimization)
        return flexibleSearchService.searchUnique(createQuery(code));
    }
}
```

### 7. Null Safety

Use proper null checks and Optional:

```java
import java.util.Optional;
import org.apache.commons.lang3.StringUtils;

@Service
public class DefaultProductService implements ProductService {
    
    @Override
    public Optional<ProductModel> findByCodeOptional(String code) {
        if (StringUtils.isBlank(code)) {
            return Optional.empty();
        }
        
        try {
            ProductModel product = findByCode(code);
            return Optional.of(product);
        } catch (UnknownIdentifierException e) {
            return Optional.empty();
        }
    }
    
    @Override
    public void updateProduct(ProductModel product) {
        Objects.requireNonNull(product, "Product cannot be null");
        
        if (StringUtils.isBlank(product.getCode())) {
            throw new IllegalArgumentException("Product code is required");
        }
        
        modelService.save(product);
    }
}
```

## Best Practices

### DO
- Use constructor injection for dependencies
- Follow interface + implementation pattern
- Use `@Service` annotation with meaningful bean names
- Log important business events at INFO level
- Use `@Transactional` for transaction boundaries
- Validate input parameters
- Use proper exception hierarchy
- Write unit tests for all services
- Use FlexibleSearch for queries (not direct SQL)
- Keep services focused (Single Responsibility Principle)

### DON'T
- Use field injection (`@Autowired` on fields)
- Access Jalo layer directly
- Hardcode configuration values
- Catch generic `Exception` without re-throwing
- Perform heavy operations in constructors
- Use `System.out.println()` for logging
- Modify generated model classes
- Use raw SQL queries
- Create circular dependencies between services
- Mix business logic with presentation logic

## Common Pitfalls

### 1. Session Context Issues
```java
// Wrong: Session context not set
public void processOrder(String orderCode) {
    OrderModel order = findByCode(orderCode);
    // May fail if session context not set
}

// Correct: Use SessionService or run in proper context
public void processOrder(String orderCode) {
    sessionService.executeInLocalView(() -> {
        OrderModel order = findByCode(orderCode);
        // Process order
    });
}
```

### 2. Lazy Loading Issues
```java
// Wrong: Accessing lazy-loaded collection outside transaction
@Transactional(readOnly = true)
public List<ProductModel> getProducts(CategoryModel category) {
    return category.getProducts(); // Returns proxy
}
// Later access may fail: category.getProducts().size()

// Correct: Initialize collection within transaction
@Transactional(readOnly = true)
public List<ProductModel> getProducts(CategoryModel category) {
    List<ProductModel> products = category.getProducts();
    products.size(); // Force initialization
    return products;
}
```

### 3. Model Refresh
```java
// Wrong: Using stale model
public void updatePrice(ProductModel product, double newPrice) {
    product.setPrice(newPrice);
    modelService.save(product);
    // product may be stale if modified elsewhere
}

// Correct: Refresh model before update
public void updatePrice(ProductModel product, double newPrice) {
    modelService.refresh(product);
    product.setPrice(newPrice);
    modelService.save(product);
}
```

## Testing

### Unit Test Example
```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class DefaultProductServiceTest {
    
    @Mock
    private FlexibleSearchService flexibleSearchService;
    
    @Mock
    private ModelService modelService;
    
    @InjectMocks
    private DefaultProductService productService;
    
    @Test
    void testFindByCode_Success() {
        // Given
        String code = "TEST123";
        ProductModel expectedProduct = new ProductModel();
        expectedProduct.setCode(code);
        
        when(flexibleSearchService.searchUnique(any(FlexibleSearchQuery.class)))
            .thenReturn(expectedProduct);
        
        // When
        ProductModel result = productService.findByCode(code);
        
        // Then
        assertNotNull(result);
        assertEquals(code, result.getCode());
        verify(flexibleSearchService).searchUnique(any(FlexibleSearchQuery.class));
    }
    
    @Test
    void testFindByCode_NotFound() {
        // Given
        String code = "NOTFOUND";
        when(flexibleSearchService.searchUnique(any(FlexibleSearchQuery.class)))
            .thenThrow(new ModelNotFoundException("Not found"));
        
        // When/Then
        assertThrows(UnknownIdentifierException.class, 
            () -> productService.findByCode(code));
    }
}
```

## Resources

- **Spring Framework Documentation**: spring.io/docs
- **SAP Commerce Service Layer**: Help Portal -> Service Layer
- **Effective Java** by Joshua Bloch: Best practices for Java development
