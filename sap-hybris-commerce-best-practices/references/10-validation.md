# Validation Framework

## Purpose
Implement declarative validation using SAP Commerce validation framework and JSR-303 Bean Validation. Provides consistent validation across application layers.

## Key Concepts

### Validation Approaches
1. **Declarative Validation**: XML-based constraints
2. **JSR-303 Annotations**: Java annotation-based validation
3. **Interceptor Validation**: ValidateInterceptor for complex rules
4. **Custom Validators**: Reusable validation logic

### Validation Layers
- Model layer (interceptors)
- Service layer (programmatic)
- Facade layer (DTO validation)
- Controller layer (form validation)

## Declarative Validation

### 1. Validation XML Configuration

`resources/mycompanycore-validation.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<validation xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:noNamespaceSchemaLocation="validation.xsd">
    
    <!-- Product validation -->
    <constraints type="Product">
        <!-- Not null constraint -->
        <constraint type="de.hybris.platform.validation.validators.NotNullConstraint">
            <annotation qualifier="code"/>
            <message>Product code is required</message>
        </constraint>
        
        <!-- Size constraint -->
        <constraint type="de.hybris.platform.validation.validators.SizeConstraint">
            <annotation qualifier="code"/>
            <property name="min" value="3"/>
            <property name="max" value="50"/>
            <message>Product code must be between 3 and 50 characters</message>
        </constraint>
        
        <!-- Pattern constraint -->
        <constraint type="de.hybris.platform.validation.validators.PatternConstraint">
            <annotation qualifier="code"/>
            <property name="regexp" value="[A-Z0-9]+"/>
            <message>Product code must contain only uppercase letters and numbers</message>
        </constraint>
        
        <!-- Min/Max constraints -->
        <constraint type="de.hybris.platform.validation.validators.MinConstraint">
            <annotation qualifier="basePrice"/>
            <property name="value" value="0.01"/>
            <message>Price must be greater than 0</message>
        </constraint>
        
        <constraint type="de.hybris.platform.validation.validators.MaxConstraint">
            <annotation qualifier="stockLevel"/>
            <property name="value" value="99999"/>
            <message>Stock level cannot exceed 99999</message>
        </constraint>
    </constraints>
    
    <!-- Order validation -->
    <constraints type="Order">
        <constraint type="de.hybris.platform.validation.validators.NotNullConstraint">
            <annotation qualifier="user"/>
            <message>Order must have a user</message>
        </constraint>
        
        <constraint type="de.hybris.platform.validation.validators.NotEmptyConstraint">
            <annotation qualifier="entries"/>
            <message>Order must have at least one entry</message>
        </constraint>
    </constraints>
</validation>
```

### 2. Custom Constraint

```java
package com.mycompany.core.validation;

import de.hybris.platform.validation.annotations.Constraint;
import java.lang.annotation.*;

@Target({ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueProductCodeValidator.class)
public @interface UniqueProductCode {
    String message() default "Product code must be unique";
}
```

### 3. Custom Validator

```java
package com.mycompany.core.validation;

import de.hybris.platform.validation.model.constraints.AbstractConstraintModel;
import de.hybris.platform.validation.validators.AbstractConstraintValidator;
import de.hybris.platform.core.model.product.ProductModel;

public class UniqueProductCodeValidator 
        extends AbstractConstraintValidator<ProductModel> {
    
    private final ProductDao productDao;
    
    public UniqueProductCodeValidator(ProductDao productDao) {
        this.productDao = productDao;
    }
    
    @Override
    public void validate(ProductModel product, AbstractConstraintModel constraint) {
        String code = product.getCode();
        
        if (code == null) {
            return; // Let NotNull constraint handle this
        }
        
        ProductModel existing = productDao.findProductByCode(code);
        
        if (existing != null && !existing.getPk().equals(product.getPk())) {
            addViolation(constraint, product, "Product code already exists: " + code);
        }
    }
}
```

## JSR-303 Bean Validation

### 1. DTO with Validation Annotations

```java
package com.mycompany.facades.product.data;

import javax.validation.constraints.*;

public class ProductData {
    
    @NotNull(message = "Product code is required")
    @Size(min = 3, max = 50, message = "Code must be 3-50 characters")
    @Pattern(regexp = "[A-Z0-9]+", message = "Code must be uppercase alphanumeric")
    private String code;
    
    @NotBlank(message = "Product name is required")
    @Size(min = 3, max = 255, message = "Name must be 3-255 characters")
    private String name;
    
    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.01", message = "Price must be greater than 0")
    @DecimalMax(value = "999999.99", message = "Price cannot exceed 999999.99")
    private Double price;
    
    @Min(value = 0, message = "Stock level cannot be negative")
    @Max(value = 99999, message = "Stock level cannot exceed 99999")
    private Integer stockLevel;
    
    @Email(message = "Invalid email format")
    private String contactEmail;
    
    @Past(message = "Launch date must be in the past")
    private Date launchDate;
    
    @AssertTrue(message = "Terms must be accepted")
    private Boolean termsAccepted;
    
    // Getters and setters...
}
```

### 2. Controller Validation

```java
package com.mycompany.storefront.controllers;

import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;
import javax.validation.Valid;

@Controller
@RequestMapping("/products")
public class ProductController {
    
    private final ProductFacade productFacade;
    
    public ProductController(ProductFacade productFacade) {
        this.productFacade = productFacade;
    }
    
    @PostMapping("/create")
    public String createProduct(
            @Valid @ModelAttribute ProductForm form,
            BindingResult bindingResult,
            Model model) {
        
        // Check for validation errors
        if (bindingResult.hasErrors()) {
            model.addAttribute("errors", bindingResult.getAllErrors());
            return "product/create";
        }
        
        // Process valid form
        productFacade.createProduct(form);
        return "redirect:/products";
    }
}
```

### 3. Programmatic Validation

```java
package com.mycompany.core.service.impl;

import javax.validation.Validator;
import javax.validation.ConstraintViolation;
import java.util.Set;

@Service
public class DefaultProductService implements ProductService {
    
    private final Validator validator;
    
    public DefaultProductService(Validator validator) {
        this.validator = validator;
    }
    
    @Override
    public void createProduct(ProductData productData) {
        // Validate programmatically
        Set<ConstraintViolation<ProductData>> violations = 
            validator.validate(productData);
        
        if (!violations.isEmpty()) {
            StringBuilder errors = new StringBuilder();
            for (ConstraintViolation<ProductData> violation : violations) {
                errors.append(violation.getMessage()).append("; ");
            }
            throw new ValidationException("Validation failed: " + errors);
        }
        
        // Process valid data
        createProductModel(productData);
    }
}
```

## Custom Validation Logic

### 1. Service Layer Validation

```java
@Service
public class DefaultOrderValidationService implements OrderValidationService {
    
    private final List<OrderValidator> validators;
    
    public DefaultOrderValidationService(List<OrderValidator> validators) {
        this.validators = validators;
    }
    
    @Override
    public ValidationResult validate(OrderModel order) {
        ValidationResult result = new ValidationResult();
        
        for (OrderValidator validator : validators) {
            ValidationResult validatorResult = validator.validate(order);
            result.addErrors(validatorResult.getErrors());
            
            if (!validatorResult.isValid() && validator.isBlocking()) {
                break;
            }
        }
        
        return result;
    }
}

// Individual validators
@Component
public class OrderMinimumAmountValidator implements OrderValidator {
    
    private static final double MINIMUM_ORDER_AMOUNT = 10.0;
    
    @Override
    public ValidationResult validate(OrderModel order) {
        ValidationResult result = new ValidationResult();
        
        if (order.getTotalPrice() < MINIMUM_ORDER_AMOUNT) {
            result.addError("order.minimum.amount", 
                String.format("Order total must be at least $%.2f", MINIMUM_ORDER_AMOUNT));
        }
        
        return result;
    }
    
    @Override
    public boolean isBlocking() {
        return true;
    }
}

@Component
public class OrderStockValidator implements OrderValidator {
    
    private final StockService stockService;
    
    @Override
    public ValidationResult validate(OrderModel order) {
        ValidationResult result = new ValidationResult();
        
        for (OrderEntryModel entry : order.getEntries()) {
            if (!stockService.isAvailable(entry.getProduct(), entry.getQuantity())) {
                result.addError("order.stock.unavailable",
                    "Product not available: " + entry.getProduct().getCode());
            }
        }
        
        return result;
    }
}
```

### 2. Validation Result

```java
package com.mycompany.core.validation;

import java.util.*;

public class ValidationResult {
    
    private final List<ValidationError> errors = new ArrayList<>();
    private final List<ValidationWarning> warnings = new ArrayList<>();
    
    public void addError(String code, String message) {
        errors.add(new ValidationError(code, message));
    }
    
    public void addWarning(String code, String message) {
        warnings.add(new ValidationWarning(code, message));
    }
    
    public boolean isValid() {
        return errors.isEmpty();
    }
    
    public boolean hasWarnings() {
        return !warnings.isEmpty();
    }
    
    public List<ValidationError> getErrors() {
        return Collections.unmodifiableList(errors);
    }
    
    public List<ValidationWarning> getWarnings() {
        return Collections.unmodifiableList(warnings);
    }
    
    public void merge(ValidationResult other) {
        this.errors.addAll(other.errors);
        this.warnings.addAll(other.warnings);
    }
}

public class ValidationError {
    private final String code;
    private final String message;
    
    public ValidationError(String code, String message) {
        this.code = code;
        this.message = message;
    }
    
    // Getters...
}
```

## Best Practices

### ✅ DO
- Use declarative validation when possible
- Validate at appropriate layers
- Provide clear error messages
- Use JSR-303 for DTOs
- Create reusable validators
- Test validation logic
- Document validation rules
- Use validation groups for different scenarios
- Validate early (fail fast)
- Return structured validation results

### ❌ DON'T
- Skip validation
- Duplicate validation logic
- Use generic error messages
- Perform heavy operations in validators
- Throw generic exceptions
- Mix validation with business logic
- Validate in multiple places inconsistently
- Ignore validation results
- Hardcode validation rules
- Skip null checks

## Resources

- **Validation Framework**: Help Portal → Validation
- **JSR-303**: Bean Validation specification
- **Spring Validation**: Spring Framework documentation
