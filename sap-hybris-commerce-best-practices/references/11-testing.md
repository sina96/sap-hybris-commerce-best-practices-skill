# Automated Testing

## Purpose
Implement comprehensive testing strategy for SAP Commerce applications using unit tests, integration tests, and end-to-end tests.

## Key Concepts

### Test Types
1. **Unit Tests**: Test individual classes in isolation
2. **Integration Tests**: Test with SAP Commerce platform
3. **Service Layer Tests**: Test services with database
4. **Web Tests**: Test controllers and web layer

### Testing Framework
- **JUnit 5**: Test framework
- **Mockito**: Mocking framework
- **ServicelayerTest**: Base class for integration tests
- **Spring Test**: Spring testing support

## Unit Tests

### 1. Service Unit Test

```java
package com.mycompany.core.service;

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
    private ProductDao productDao;
    
    @Mock
    private ModelService modelService;
    
    @InjectMocks
    private DefaultProductService productService;
    
    @Test
    void testGetProductForCode_Success() {
        // Given
        String code = "TEST001";
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
    
    @Test
    void testCreateProduct_Success() {
        // Given
        String code = "NEW001";
        String name = "New Product";
        ProductModel product = new ProductModel();
        
        when(productDao.findProductByCode(code)).thenReturn(null);
        when(modelService.create(ProductModel.class)).thenReturn(product);
        
        // When
        ProductModel result = productService.createProduct(code, name);
        
        // Then
        assertNotNull(result);
        verify(modelService).save(product);
    }
}
```

## Integration Tests

### 1. Service Integration Test

```java
package com.mycompany.core.service;

import de.hybris.bootstrap.annotations.IntegrationTest;
import de.hybris.platform.servicelayer.ServicelayerTest;
import de.hybris.platform.servicelayer.model.ModelService;
import org.junit.Before;
import org.junit.Test;
import javax.annotation.Resource;
import static org.junit.Assert.*;

@IntegrationTest
public class ProductServiceIntegrationTest extends ServicelayerTest {
    
    @Resource
    private ProductService productService;
    
    @Resource
    private ModelService modelService;
    
    @Before
    public void setUp() throws Exception {
        createCoreData();
        createDefaultCatalog();
        importCsv("/test/testdata.impex", "UTF-8");
    }
    
    @Test
    public void testGetProductForCode() {
        // When
        ProductModel product = productService.getProductForCode("TEST001");
        
        // Then
        assertNotNull(product);
        assertEquals("TEST001", product.getCode());
        assertEquals("Test Product", product.getName());
    }
    
    @Test
    public void testCreateProduct() {
        // When
        ProductModel product = productService.createProduct("NEW001", "New Product");
        
        // Then
        assertNotNull(product);
        assertNotNull(product.getPk());
        
        // Verify persisted
        modelService.refresh(product);
        assertEquals("NEW001", product.getCode());
    }
    
    @Test
    public void testUpdateStock() {
        // Given
        ProductModel product = productService.getProductForCode("TEST001");
        
        // When
        productService.updateStock(product, 100);
        
        // Then
        modelService.refresh(product);
        assertEquals(Integer.valueOf(100), product.getStockLevel());
    }
}
```

### 2. DAO Integration Test

```java
@IntegrationTest
public class ProductDaoIntegrationTest extends ServicelayerTest {
    
    @Resource
    private ProductDao productDao;
    
    @Before
    public void setUp() throws Exception {
        importCsv("/test/products.impex", "UTF-8");
    }
    
    @Test
    public void testFindProductByCode() {
        ProductModel product = productDao.findProductByCode("TEST001");
        assertNotNull(product);
        assertEquals("TEST001", product.getCode());
    }
    
    @Test
    public void testFindProductsByCategory() {
        CategoryModel category = createCategory("electronics");
        List<ProductModel> products = productDao.findProductsByCategory(category);
        
        assertFalse(products.isEmpty());
        assertTrue(products.size() >= 1);
    }
}
```

## Test Data Setup

### 1. ImpEx Test Data

`testsrc/test/testdata.impex`:

```impex
$catalogVersion=catalogVersion(catalog(id[default='testCatalog']),version[default='Staged'])[unique=true]

INSERT_UPDATE Catalog;id[unique=true];name[lang=en];defaultCatalog
;testCatalog;Test Catalog;true

INSERT_UPDATE CatalogVersion;catalog(id)[unique=true];version[unique=true];active
;testCatalog;Staged;true

INSERT_UPDATE Product;code[unique=true];name;$catalogVersion
;TEST001;Test Product 1
;TEST002;Test Product 2
;TEST003;Test Product 3

INSERT_UPDATE Category;code[unique=true];name;$catalogVersion
;electronics;Electronics
;clothing;Clothing
```

### 2. Test Data Builder

```java
public class ProductTestDataBuilder {
    
    private String code = "TEST001";
    private String name = "Test Product";
    private Double price = 99.99;
    private Integer stockLevel = 10;
    
    public static ProductTestDataBuilder aProduct() {
        return new ProductTestDataBuilder();
    }
    
    public ProductTestDataBuilder withCode(String code) {
        this.code = code;
        return this;
    }
    
    public ProductTestDataBuilder withName(String name) {
        this.name = name;
        return this;
    }
    
    public ProductTestDataBuilder withPrice(Double price) {
        this.price = price;
        return this;
    }
    
    public ProductTestDataBuilder withStockLevel(Integer stockLevel) {
        this.stockLevel = stockLevel;
        return this;
    }
    
    public ProductModel build(ModelService modelService) {
        ProductModel product = modelService.create(ProductModel.class);
        product.setCode(code);
        product.setName(name);
        product.setPrice(price);
        product.setStockLevel(stockLevel);
        return product;
    }
}

// Usage
ProductModel product = ProductTestDataBuilder.aProduct()
    .withCode("CUSTOM001")
    .withName("Custom Product")
    .withPrice(149.99)
    .build(modelService);
```

## Testing Best Practices

### ✅ DO
- Write tests for all services
- Use meaningful test names
- Follow AAA pattern (Arrange, Act, Assert)
- Test both success and failure cases
- Use test data builders
- Clean up test data
- Run tests in CI/CD
- Mock external dependencies
- Test edge cases
- Maintain test coverage

### ❌ DON'T
- Skip tests
- Test implementation details
- Use production data
- Create test dependencies
- Ignore failing tests
- Write flaky tests
- Test framework code
- Duplicate test logic
- Use Thread.sleep()
- Commit commented tests

## Running Tests

```bash
# All tests
ant alltests

# Unit tests only
ant unittests

# Integration tests only
ant integrationtests

# Specific test
ant integrationtests -Dtestclasses.extensions=mycompanycore

# With coverage
ant alltests -Dtestclasses.coverage=true
```

## Resources

- **Testing Guide**: Help Portal → Testing
- **JUnit 5**: junit.org/junit5
- **Mockito**: mockito.org
