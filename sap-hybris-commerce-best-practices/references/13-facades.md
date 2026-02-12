# Facades

## Purpose
Implement the Facade pattern to provide a simplified API for frontend layers using DTOs (Data Transfer Objects) and converters. Facades decouple frontend from backend models.

## Key Concepts

### Facade Pattern
```
Controller -> Facade -> Service -> DAO -> Model
              |
              v
            DTO (Data Transfer Object)
```

### Benefits
- Decouples frontend from backend models
- Provides simplified API
- Enables versioning
- Improves security (no model exposure)
- Facilitates testing

## Creating Facades

### 1. Data Transfer Object (DTO)

```java
package com.mycompany.facades.product.data;

import java.io.Serializable;
import java.util.List;

public class ProductData implements Serializable {
    
    private String code;
    private String name;
    private String description;
    private PriceData price;
    private Integer stockLevel;
    private String imageUrl;
    private List<CategoryData> categories;
    private boolean available;
    
    // Getters and setters
    public String getCode() { return code; }
    public void setCode(String code) { this.code = code; }
    
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    // ... other getters/setters
}

public class PriceData implements Serializable {
    private String currencyIso;
    private Double value;
    private String formattedValue;
    
    // Getters and setters
}

public class CategoryData implements Serializable {
    private String code;
    private String name;
    private String url;
    
    // Getters and setters
}
```

### 2. Converter

```java
package com.mycompany.facades.product.converters;

import de.hybris.platform.servicelayer.dto.converter.Converter;
import de.hybris.platform.core.model.product.ProductModel;
import com.mycompany.facades.product.data.ProductData;
import org.springframework.stereotype.Component;

@Component
public class ProductConverter implements Converter<ProductModel, ProductData> {
    
    private final Converter<PriceInformation, PriceData> priceConverter;
    private final Converter<CategoryModel, CategoryData> categoryConverter;
    private final PriceService priceService;
    
    public ProductConverter(
            Converter<PriceInformation, PriceData> priceConverter,
            Converter<CategoryModel, CategoryData> categoryConverter,
            PriceService priceService) {
        this.priceConverter = priceConverter;
        this.categoryConverter = categoryConverter;
        this.priceService = priceService;
    }
    
    @Override
    public ProductData convert(ProductModel source) {
        return convert(source, new ProductData());
    }
    
    @Override
    public ProductData convert(ProductModel source, ProductData target) {
        if (source == null) {
            return null;
        }
        
        target.setCode(source.getCode());
        target.setName(source.getName());
        target.setDescription(source.getDescription());
        target.setStockLevel(source.getStockLevel());
        target.setAvailable(source.getStockLevel() != null && source.getStockLevel() > 0);
        
        // Convert price
        PriceInformation priceInfo = priceService.getPriceInformation(source);
        if (priceInfo != null) {
            target.setPrice(priceConverter.convert(priceInfo));
        }
        
        // Convert image
        if (source.getPicture() != null) {
            target.setImageUrl(source.getPicture().getURL());
        }
        
        // Convert categories
        if (source.getSupercategories() != null) {
            List<CategoryData> categories = source.getSupercategories().stream()
                .map(categoryConverter::convert)
                .collect(Collectors.toList());
            target.setCategories(categories);
        }
        
        return target;
    }
}
```

### 3. Populator (Alternative to Converter)

```java
package com.mycompany.facades.product.populators;

import de.hybris.platform.converters.Populator;
import de.hybris.platform.servicelayer.dto.converter.ConversionException;
import com.mycompany.facades.product.data.ProductData;
import org.springframework.stereotype.Component;

@Component
public class ProductBasicPopulator implements Populator<ProductModel, ProductData> {
    
    @Override
    public void populate(ProductModel source, ProductData target) throws ConversionException {
        target.setCode(source.getCode());
        target.setName(source.getName());
        target.setDescription(source.getDescription());
        target.setStockLevel(source.getStockLevel());
    }
}

@Component
public class ProductPricePopulator implements Populator<ProductModel, ProductData> {
    
    private final PriceService priceService;
    private final Converter<PriceInformation, PriceData> priceConverter;
    
    @Override
    public void populate(ProductModel source, ProductData target) throws ConversionException {
        PriceInformation priceInfo = priceService.getPriceInformation(source);
        if (priceInfo != null) {
            target.setPrice(priceConverter.convert(priceInfo));
        }
    }
}
```

### 4. Facade Interface

```java
package com.mycompany.facades.product;

import com.mycompany.facades.product.data.ProductData;
import java.util.List;

public interface ProductFacade {
    
    /**
     * Get product by code
     */
    ProductData getProduct(String code);
    
    /**
     * Search products by query
     */
    List<ProductData> searchProducts(String query);
    
    /**
     * Get products by category
     */
    List<ProductData> getProductsByCategory(String categoryCode);
    
    /**
     * Get featured products
     */
    List<ProductData> getFeaturedProducts(int limit);
    
    /**
     * Check product availability
     */
    boolean isProductAvailable(String code);
}
```

### 5. Facade Implementation

```java
package com.mycompany.facades.product.impl;

import de.hybris.platform.servicelayer.dto.converter.Converter;
import com.mycompany.core.service.ProductService;
import com.mycompany.facades.product.ProductFacade;
import com.mycompany.facades.product.data.ProductData;
import org.springframework.stereotype.Service;

@Service("productFacade")
public class DefaultProductFacade implements ProductFacade {
    
    private final ProductService productService;
    private final Converter<ProductModel, ProductData> productConverter;
    
    public DefaultProductFacade(
            ProductService productService,
            Converter<ProductModel, ProductData> productConverter) {
        this.productService = productService;
        this.productConverter = productConverter;
    }
    
    @Override
    public ProductData getProduct(String code) {
        ProductModel product = productService.getProductForCode(code);
        return productConverter.convert(product);
    }
    
    @Override
    public List<ProductData> searchProducts(String query) {
        List<ProductModel> products = productService.searchProductsByName(query);
        return products.stream()
            .map(productConverter::convert)
            .collect(Collectors.toList());
    }
    
    @Override
    public List<ProductData> getProductsByCategory(String categoryCode) {
        CategoryModel category = categoryService.getCategoryForCode(categoryCode);
        List<ProductModel> products = productService.getProductsForCategory(category);
        return convertProducts(products);
    }
    
    @Override
    public List<ProductData> getFeaturedProducts(int limit) {
        List<ProductModel> products = productService.getFeaturedProducts(limit);
        return convertProducts(products);
    }
    
    @Override
    public boolean isProductAvailable(String code) {
        try {
            ProductModel product = productService.getProductForCode(code);
            return productService.isProductAvailable(product);
        } catch (UnknownIdentifierException e) {
            return false;
        }
    }
    
    private List<ProductData> convertProducts(List<ProductModel> products) {
        return products.stream()
            .map(productConverter::convert)
            .collect(Collectors.toList());
    }
}
```

## Spring Configuration

```xml
<!-- resources/mycompanyfacades-spring.xml -->
<beans>
    <!-- Converters -->
    <bean id="productConverter" 
          class="com.mycompany.facades.product.converters.ProductConverter">
        <constructor-arg ref="priceConverter"/>
        <constructor-arg ref="categoryConverter"/>
        <constructor-arg ref="priceService"/>
    </bean>
    
    <!-- Or use populators -->
    <bean id="productConverter" parent="abstractPopulatingConverter">
        <property name="targetClass" value="com.mycompany.facades.product.data.ProductData"/>
        <property name="populators">
            <list>
                <ref bean="productBasicPopulator"/>
                <ref bean="productPricePopulator"/>
                <ref bean="productImagePopulator"/>
            </list>
        </property>
    </bean>
    
    <!-- Facades -->
    <bean id="productFacade" 
          class="com.mycompany.facades.product.impl.DefaultProductFacade">
        <constructor-arg ref="productService"/>
        <constructor-arg ref="productConverter"/>
    </bean>
</beans>
```

## Advanced Patterns

### 1. Pagination

```java
public interface ProductFacade {
    SearchPageData<ProductData> searchProducts(SearchQuery query);
}

public class SearchQuery {
    private String query;
    private int pageNumber;
    private int pageSize;
    private String sortCode;
    
    // Getters/setters
}

public class SearchPageData<T> {
    private List<T> results;
    private PaginationData pagination;
    private List<SortData> sorts;
    
    // Getters/setters
}

@Service
public class DefaultProductFacade implements ProductFacade {
    
    @Override
    public SearchPageData<ProductData> searchProducts(SearchQuery query) {
        SearchPageData<ProductModel> searchResult = 
            productService.searchProducts(query);
        
        SearchPageData<ProductData> result = new SearchPageData<>();
        result.setResults(convertProducts(searchResult.getResults()));
        result.setPagination(searchResult.getPagination());
        result.setSorts(searchResult.getSorts());
        
        return result;
    }
}
```

### 2. Composite DTOs

```java
public class ProductDetailData extends ProductData {
    private List<ReviewData> reviews;
    private List<ProductData> relatedProducts;
    private List<VariantData> variants;
    private StockData stock;
    
    // Getters/setters
}

@Component
public class ProductDetailConverter implements Converter<ProductModel, ProductDetailData> {
    
    @Override
    public ProductDetailData convert(ProductModel source) {
        ProductDetailData target = new ProductDetailData();
        
        // Basic product data
        productBasicConverter.convert(source, target);
        
        // Additional details
        target.setReviews(convertReviews(source.getReviews()));
        target.setRelatedProducts(convertRelatedProducts(source.getRelatedProducts()));
        target.setVariants(convertVariants(source.getVariants()));
        target.setStock(convertStock(source));
        
        return target;
    }
}
```

## Best Practices

### DO
- Use DTOs for all facade responses
- Keep DTOs simple and serializable
- Use converters/populators for transformation
- Version DTOs for API compatibility
- Document DTO fields
- Use meaningful DTO names
- Keep facades thin (delegate to services)
- Return immutable collections when possible
- Handle null values gracefully
- Test converters thoroughly

### DON'T
- Expose models directly to frontend
- Put business logic in facades
- Create circular dependencies
- Use models in DTOs
- Skip null checks in converters
- Create deep DTO hierarchies
- Duplicate conversion logic
- Ignore performance (N+1 queries)
- Mix concerns in facades
- Return null (use Optional or empty collections)

## Resources

- **Facade Pattern**: Design Patterns documentation
- **DTO Pattern**: Enterprise patterns
- **Converter Framework**: SAP Commerce documentation
