# Controllers

## Purpose
Implement Spring MVC controllers for web requests and REST API endpoints. Controllers handle HTTP requests, validate input, and return responses.

## Spring MVC Controllers

### 1. Basic Controller

```java
package com.mycompany.storefront.controllers;

import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;

@Controller
@RequestMapping("/products")
public class ProductController {
    
    private final ProductFacade productFacade;
    
    public ProductController(ProductFacade productFacade) {
        this.productFacade = productFacade;
    }
    
    @GetMapping("/{code}")
    public String productDetail(@PathVariable String code, Model model) {
        ProductData product = productFacade.getProduct(code);
        model.addAttribute("product", product);
        return "product/detail";
    }
    
    @GetMapping
    public String productList(
            @RequestParam(required = false) String query,
            @RequestParam(defaultValue = "0") int page,
            Model model) {
        
        SearchPageData<ProductData> searchResult = 
            productFacade.searchProducts(query, page);
        
        model.addAttribute("searchResult", searchResult);
        return "product/list";
    }
}
```

### 2. Form Handling

```java
@Controller
@RequestMapping("/products")
public class ProductFormController {
    
    @GetMapping("/create")
    public String showCreateForm(Model model) {
        model.addAttribute("productForm", new ProductForm());
        return "product/create";
    }
    
    @PostMapping("/create")
    public String createProduct(
            @Valid @ModelAttribute ProductForm form,
            BindingResult bindingResult,
            RedirectAttributes redirectAttributes) {
        
        if (bindingResult.hasErrors()) {
            return "product/create";
        }
        
        productFacade.createProduct(form);
        redirectAttributes.addFlashAttribute("message", "Product created successfully");
        
        return "redirect:/products";
    }
}
```

## REST Controllers

### 1. REST API Controller

```java
package com.mycompany.occ.controllers;

import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import org.springframework.http.HttpStatus;

@RestController
@RequestMapping("/api/products")
public class ProductRestController {
    
    private final ProductFacade productFacade;
    
    @GetMapping("/{code}")
    public ResponseEntity<ProductData> getProduct(@PathVariable String code) {
        try {
            ProductData product = productFacade.getProduct(code);
            return ResponseEntity.ok(product);
        } catch (UnknownIdentifierException e) {
            return ResponseEntity.notFound().build();
        }
    }
    
    @GetMapping
    public ResponseEntity<List<ProductData>> searchProducts(
            @RequestParam String query) {
        List<ProductData> products = productFacade.searchProducts(query);
        return ResponseEntity.ok(products);
    }
    
    @PostMapping
    public ResponseEntity<ProductData> createProduct(
            @Valid @RequestBody ProductData productData) {
        ProductData created = productFacade.createProduct(productData);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
    
    @PutMapping("/{code}")
    public ResponseEntity<ProductData> updateProduct(
            @PathVariable String code,
            @Valid @RequestBody ProductData productData) {
        ProductData updated = productFacade.updateProduct(code, productData);
        return ResponseEntity.ok(updated);
    }
    
    @DeleteMapping("/{code}")
    public ResponseEntity<Void> deleteProduct(@PathVariable String code) {
        productFacade.deleteProduct(code);
        return ResponseEntity.noContent().build();
    }
}
```

### 2. Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UnknownIdentifierException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UnknownIdentifierException e) {
        ErrorResponse error = new ErrorResponse("NOT_FOUND", e.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException e) {
        ErrorResponse error = new ErrorResponse("VALIDATION_ERROR", e.getMessage());
        return ResponseEntity.badRequest().body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception e) {
        ErrorResponse error = new ErrorResponse("INTERNAL_ERROR", "An error occurred");
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

## Best Practices

### DO
- Use appropriate HTTP methods
- Validate input with @Valid
- Return proper HTTP status codes
- Handle exceptions gracefully
- Use DTOs for request/response
- Document API endpoints
- Use meaningful URL patterns
- Implement pagination for lists
- Use constructor injection
- Test controllers

### DON'T
- Put business logic in controllers
- Expose models directly
- Return null
- Ignore validation errors
- Use generic exception handlers only
- Skip security checks
- Create fat controllers
- Use field injection
- Hardcode URLs
- Skip error handling

## Resources

- **Spring MVC**: spring.io/guides/gs/serving-web-content
- **REST API**: spring.io/guides/gs/rest-service
