# Services

## Purpose
Implement business logic using SAP Commerce Service Layer with Spring dependency injection. Services encapsulate business operations and provide a clean API for data access and manipulation.

## Key Concepts

### Service Layer Architecture
```
Controllers/Facades
       |
       v
   Services (Business Logic)
       |
       v
   DAOs (Data Access)
       |
       v
   Models (Data)
```

### Core Platform Services
- `ModelService`: CRUD operations for models
- `FlexibleSearchService`: Query execution
- `SessionService`: Session management
- `UserService`: User operations
- `CatalogService`: Catalog management
- `I18NService`: Internationalization
- `EventService`: Event publishing
- `TransactionService`: Transaction management

## Creating Custom Services

### 1. Service Interface

```java
package com.mycompany.core.service;

import de.hybris.platform.core.model.product.ProductModel;
import de.hybris.platform.category.model.CategoryModel;
import java.util.List;
import java.util.Optional;

/**
 * Service for product operations
 */
public interface ProductService {
    
    /**
     * Find product by code
     * @param code product code
     * @return product model
     * @throws UnknownIdentifierException if not found
     */
    ProductModel getProductForCode(String code);
    
    /**
     * Find product by code (optional)
     * @param code product code
     * @return optional product model
     */
    Optional<ProductModel> findProductByCode(String code);
    
    /**
     * Find products by category
     * @param category category model
     * @return list of products
     */
    List<ProductModel> getProductsForCategory(CategoryModel category);
    
    /**
     * Search products by name
     * @param searchTerm search term
     * @return list of matching products
     */
    List<ProductModel> searchProductsByName(String searchTerm);
    
    /**
     * Create new product
     * @param code product code
     * @param name product name
     * @return created product
     */
    ProductModel createProduct(String code, String name);
    
    /**
     * Update product stock
     * @param product product model
     * @param quantity new stock quantity
     */
    void updateStock(ProductModel product, int quantity);
    
    /**
     * Check if product is available
     * @param product product model
     * @return true if available
     */
    boolean isProductAvailable(ProductModel product);
    
    /**
     * Get featured products
     * @param limit maximum number of products
     * @return list of featured products
     */
    List<ProductModel> getFeaturedProducts(int limit);
}
```

### 2. Service Implementation

```java
package com.mycompany.core.service.impl;

import com.mycompany.core.service.ProductService;
import com.mycompany.core.dao.ProductDao;
import de.hybris.platform.servicelayer.model.ModelService;
import de.hybris.platform.servicelayer.exceptions.UnknownIdentifierException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.List;
import java.util.Optional;

@Service("productService")
public class DefaultProductService implements ProductService {
    
    private static final Logger LOG = LoggerFactory.getLogger(DefaultProductService.class);
    
    private final ModelService modelService;
    private final ProductDao productDao;
    private final CatalogVersionService catalogVersionService;
    
    // Constructor injection (Spring 6 best practice)
    public DefaultProductService(
            ModelService modelService,
            ProductDao productDao,
            CatalogVersionService catalogVersionService) {
        this.modelService = modelService;
        this.productDao = productDao;
        this.catalogVersionService = catalogVersionService;
    }
    
    @Override
    @Transactional(readOnly = true)
    public ProductModel getProductForCode(String code) {
        LOG.debug("Finding product by code: {}", code);
        
        if (StringUtils.isBlank(code)) {
            throw new IllegalArgumentException("Product code cannot be blank");
        }
        
        final ProductModel product = productDao.findProductByCode(code);
        if (product == null) {
            throw new UnknownIdentifierException(
                "Product not found with code: " + code);
        }
        
        return product;
    }
    
    @Override
    @Transactional(readOnly = true)
    public Optional<ProductModel> findProductByCode(String code) {
        if (StringUtils.isBlank(code)) {
            return Optional.empty();
        }
        
        try {
            return Optional.of(getProductForCode(code));
        } catch (UnknownIdentifierException e) {
            LOG.debug("Product not found: {}", code);
            return Optional.empty();
        }
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<ProductModel> getProductsForCategory(CategoryModel category) {
        Objects.requireNonNull(category, "Category cannot be null");
        
        LOG.debug("Finding products for category: {}", category.getCode());
        return productDao.findProductsByCategory(category);
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<ProductModel> searchProductsByName(String searchTerm) {
        if (StringUtils.isBlank(searchTerm)) {
            return Collections.emptyList();
        }
        
        LOG.debug("Searching products by name: {}", searchTerm);
        return productDao.searchProductsByName(searchTerm);
    }
    
    @Override
    @Transactional
    public ProductModel createProduct(String code, String name) {
        LOG.info("Creating product: code={}, name={}", code, name);
        
        if (StringUtils.isBlank(code)) {
            throw new IllegalArgumentException("Product code is required");
        }
        
        // Check if product already exists
        if (productDao.findProductByCode(code) != null) {
            throw new IllegalStateException("Product already exists: " + code);
        }
        
        // Create product
        final ProductModel product = modelService.create(ProductModel.class);
        product.setCode(code);
        product.setName(name);
        product.setCatalogVersion(catalogVersionService.getSessionCatalogVersion());
        
        modelService.save(product);
        LOG.info("Product created successfully: {}", code);
        
        return product;
    }
    
    @Override
    @Transactional
    public void updateStock(ProductModel product, int quantity) {
        Objects.requireNonNull(product, "Product cannot be null");
        
        LOG.info("Updating stock for product {}: quantity={}", 
            product.getCode(), quantity);
        
        if (quantity < 0) {
            throw new IllegalArgumentException("Stock quantity cannot be negative");
        }
        
        product.setStockLevel(quantity);
        modelService.save(product);
    }
    
    @Override
    @Transactional(readOnly = true)
    public boolean isProductAvailable(ProductModel product) {
        if (product == null) {
            return false;
        }
        
        return product.getStockLevel() != null && product.getStockLevel() > 0;
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<ProductModel> getFeaturedProducts(int limit) {
        LOG.debug("Getting featured products: limit={}", limit);
        
        if (limit <= 0) {
            return Collections.emptyList();
        }
        
        return productDao.findFeaturedProducts(limit);
    }
}
```

### 3. Data Access Object (DAO)

```java
package com.mycompany.core.dao;

import de.hybris.platform.core.model.product.ProductModel;
import de.hybris.platform.category.model.CategoryModel;
import java.util.List;

/**
 * DAO for product data access
 */
public interface ProductDao {
    
    ProductModel findProductByCode(String code);
    
    List<ProductModel> findProductsByCategory(CategoryModel category);
    
    List<ProductModel> searchProductsByName(String searchTerm);
    
    List<ProductModel> findFeaturedProducts(int limit);
    
    List<ProductModel> findProductsByPriceRange(double minPrice, double maxPrice);
}
```

### 4. DAO Implementation

```java
package com.mycompany.core.dao.impl;

import com.mycompany.core.dao.ProductDao;
import de.hybris.platform.servicelayer.search.FlexibleSearchService;
import de.hybris.platform.servicelayer.search.FlexibleSearchQuery;
import de.hybris.platform.servicelayer.search.SearchResult;
import org.springframework.stereotype.Repository;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Repository("productDao")
public class DefaultProductDao implements ProductDao {
    
    private final FlexibleSearchService flexibleSearchService;
    
    public DefaultProductDao(FlexibleSearchService flexibleSearchService) {
        this.flexibleSearchService = flexibleSearchService;
    }
    
    @Override
    public ProductModel findProductByCode(String code) {
        final String query = "SELECT {pk} FROM {Product} WHERE {code} = ?code";
        
        final FlexibleSearchQuery fQuery = new FlexibleSearchQuery(query);
        fQuery.addQueryParameter("code", code);
        
        try {
            return flexibleSearchService.searchUnique(fQuery);
        } catch (ModelNotFoundException e) {
            return null;
        }
    }
    
    @Override
    public List<ProductModel> findProductsByCategory(CategoryModel category) {
        final String query = 
            "SELECT {p:pk} FROM {Product AS p} " +
            "WHERE {p:supercategories} = ?category";
        
        final FlexibleSearchQuery fQuery = new FlexibleSearchQuery(query);
        fQuery.addQueryParameter("category", category);
        
        final SearchResult<ProductModel> result = flexibleSearchService.search(fQuery);
        return result.getResult();
    }
    
    @Override
    public List<ProductModel> searchProductsByName(String searchTerm) {
        final String query = 
            "SELECT {pk} FROM {Product} " +
            "WHERE {name} LIKE ?searchTerm";
        
        final FlexibleSearchQuery fQuery = new FlexibleSearchQuery(query);
        fQuery.addQueryParameter("searchTerm", "%" + searchTerm + "%");
        
        final SearchResult<ProductModel> result = flexibleSearchService.search(fQuery);
        return result.getResult();
    }
    
    @Override
    public List<ProductModel> findFeaturedProducts(int limit) {
        final String query = 
            "SELECT {pk} FROM {Product} " +
            "WHERE {featured} = ?featured " +
            "ORDER BY {creationtime} DESC";
        
        final FlexibleSearchQuery fQuery = new FlexibleSearchQuery(query);
        fQuery.addQueryParameter("featured", Boolean.TRUE);
        fQuery.setCount(limit);
        
        final SearchResult<ProductModel> result = flexibleSearchService.search(fQuery);
        return result.getResult();
    }
    
    @Override
    public List<ProductModel> findProductsByPriceRange(double minPrice, double maxPrice) {
        final String query = 
            "SELECT {p:pk} FROM {Product AS p " +
            "JOIN PriceRow AS pr ON {p:pk} = {pr:product}} " +
            "WHERE {pr:price} >= ?minPrice AND {pr:price} <= ?maxPrice";
        
        final Map<String, Object> params = new HashMap<>();
        params.put("minPrice", minPrice);
        params.put("maxPrice", maxPrice);
        
        final FlexibleSearchQuery fQuery = new FlexibleSearchQuery(query, params);
        
        final SearchResult<ProductModel> result = flexibleSearchService.search(fQuery);
        return result.getResult();
    }
}
```

## Spring Configuration

### XML Configuration

```xml
<!-- resources/mycompanycore-spring.xml -->
<beans>
    <!-- Enable component scanning -->
    <context:component-scan base-package="com.mycompany.core"/>
    
    <!-- Or define beans explicitly -->
    <bean id="productDao" 
          class="com.mycompany.core.dao.impl.DefaultProductDao">
        <constructor-arg ref="flexibleSearchService"/>
    </bean>
    
    <bean id="productService" 
          class="com.mycompany.core.service.impl.DefaultProductService">
        <constructor-arg ref="modelService"/>
        <constructor-arg ref="productDao"/>
        <constructor-arg ref="catalogVersionService"/>
    </bean>
</beans>
```

### Java Configuration

```java
package com.mycompany.core.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ServiceConfig {
    
    @Bean
    public ProductDao productDao(FlexibleSearchService flexibleSearchService) {
        return new DefaultProductDao(flexibleSearchService);
    }
    
    @Bean
    public ProductService productService(
            ModelService modelService,
            ProductDao productDao,
            CatalogVersionService catalogVersionService) {
        return new DefaultProductService(modelService, productDao, catalogVersionService);
    }
}
```

## Transaction Management

### Declarative Transactions

```java
@Service
public class DefaultOrderService implements OrderService {
    
    // Read-only transaction (optimization)
    @Transactional(readOnly = true)
    @Override
    public OrderModel getOrderByCode(String code) {
        return orderDao.findByCode(code);
    }
    
    // Read-write transaction
    @Transactional
    @Override
    public void placeOrder(OrderModel order) {
        validateOrder(order);
        calculateTotals(order);
        modelService.save(order);
        eventService.publishEvent(new OrderPlacedEvent(order));
    }
    
    // Transaction with specific propagation
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    @Override
    public void createAuditLog(String message) {
        // Always runs in new transaction
        AuditLogModel log = modelService.create(AuditLogModel.class);
        log.setMessage(message);
        modelService.save(log);
    }
    
    // Transaction with rollback rules
    @Transactional(rollbackFor = Exception.class)
    @Override
    public void processPayment(OrderModel order) {
        // Rollback on any exception
        paymentService.charge(order);
        order.setPaymentStatus(PaymentStatus.PAID);
        modelService.save(order);
    }
}
```

### Programmatic Transactions

```java
@Service
public class DefaultOrderService implements OrderService {
    
    private final TransactionTemplate transactionTemplate;
    
    public DefaultOrderService(PlatformTransactionManager transactionManager) {
        this.transactionTemplate = new TransactionTemplate(transactionManager);
    }
    
    @Override
    public void processOrder(OrderModel order) {
        transactionTemplate.execute(status -> {
            try {
                validateOrder(order);
                calculateTotals(order);
                modelService.save(order);
                return null;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

## Service Patterns

### 1. Validation Service

```java
@Service
public class DefaultOrderValidationService implements OrderValidationService {
    
    private final List<OrderValidator> validators;
    
    public DefaultOrderValidationService(List<OrderValidator> validators) {
        this.validators = validators;
    }
    
    @Override
    public ValidationResult validate(OrderModel order) {
        final ValidationResult result = new ValidationResult();
        
        for (OrderValidator validator : validators) {
            final ValidationResult validatorResult = validator.validate(order);
            result.merge(validatorResult);
            
            if (!validatorResult.isValid() && validator.isBlocking()) {
                break; // Stop on first blocking error
            }
        }
        
        return result;
    }
}
```

### 2. Calculation Service

```java
@Service
public class DefaultPriceCalculationService implements PriceCalculationService {
    
    private final TaxService taxService;
    private final DiscountService discountService;
    
    public DefaultPriceCalculationService(
            TaxService taxService,
            DiscountService discountService) {
        this.taxService = taxService;
        this.discountService = discountService;
    }
    
    @Override
    public PriceData calculatePrice(ProductModel product, int quantity) {
        // Base price
        double basePrice = product.getBasePrice() * quantity;
        
        // Apply discounts
        double discount = discountService.calculateDiscount(product, quantity);
        double discountedPrice = basePrice - discount;
        
        // Calculate tax
        double tax = taxService.calculateTax(product, discountedPrice);
        double finalPrice = discountedPrice + tax;
        
        return PriceData.builder()
            .basePrice(basePrice)
            .discount(discount)
            .tax(tax)
            .finalPrice(finalPrice)
            .build();
    }
}
```

### 3. Integration Service

```java
@Service
public class DefaultExternalSystemService implements ExternalSystemService {
    
    private final RestTemplate restTemplate;
    private final ConfigurationService configurationService;
    
    public DefaultExternalSystemService(
            RestTemplate restTemplate,
            ConfigurationService configurationService) {
        this.restTemplate = restTemplate;
        this.configurationService = configurationService;
    }
    
    @Override
    public ExternalProductData fetchProductData(String externalId) {
        final String apiUrl = configurationService.getConfiguration()
            .getString("external.api.url");
        final String endpoint = apiUrl + "/products/" + externalId;
        
        try {
            return restTemplate.getForObject(endpoint, ExternalProductData.class);
        } catch (RestClientException e) {
            LOG.error("Failed to fetch product data: {}", externalId, e);
            throw new ExternalSystemException("Product fetch failed", e);
        }
    }
}
```

## Best Practices

### DO
- Use interface + implementation pattern
- Use constructor injection
- Keep services focused (Single Responsibility)
- Use `@Transactional` for transaction boundaries
- Validate input parameters
- Use proper exception handling
- Log important business events
- Write unit tests for services
- Use DAOs for data access
- Document public methods

### DON'T
- Use field injection
- Mix concerns in single service
- Perform UI logic in services
- Catch generic `Exception` without re-throwing
- Hardcode configuration values
- Access Jalo layer directly
- Create circular dependencies
- Perform heavy operations in constructors
- Skip transaction management
- Expose internal implementation details

## Testing Services

### Unit Test

```java
@ExtendWith(MockitoExtension.class)
class DefaultProductServiceTest {
    
    @Mock
    private ModelService modelService;
    
    @Mock
    private ProductDao productDao;
    
    @Mock
    private CatalogVersionService catalogVersionService;
    
    @InjectMocks
    private DefaultProductService productService;
    
    @Test
    void testGetProductForCode_Success() {
        // Given
        String code = "TEST123";
        ProductModel expectedProduct = new ProductModel();
        expectedProduct.setCode(code);
        
        when(productDao.findProductByCode(code)).thenReturn(expectedProduct);
        
        // When
        ProductModel result = productService.getProductForCode(code);
        
        // Then
        assertNotNull(result);
        assertEquals(code, result.getCode());
        verify(productDao).findProductByCode(code);
    }
    
    @Test
    void testGetProductForCode_NotFound() {
        // Given
        String code = "NOTFOUND";
        when(productDao.findProductByCode(code)).thenReturn(null);
        
        // When/Then
        assertThrows(UnknownIdentifierException.class, 
            () -> productService.getProductForCode(code));
    }
}
```

### Integration Test

```java
@IntegrationTest
class ProductServiceIntegrationTest extends ServicelayerTest {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private ModelService modelService;
    
    @Before
    public void setUp() throws Exception {
        createCoreData();
        createDefaultCatalog();
    }
    
    @Test
    public void testCreateProduct() {
        // When
        ProductModel product = productService.createProduct("TEST001", "Test Product");
        
        // Then
        assertNotNull(product);
        assertEquals("TEST001", product.getCode());
        assertEquals("Test Product", product.getName());
        
        // Verify persisted
        modelService.refresh(product);
        assertNotNull(product.getPk());
    }
}
```

## Resources

- **Service Layer Documentation**: Help Portal -> Service Layer
- **Spring Framework**: spring.io/docs
- **Transaction Management**: Spring transaction documentation
