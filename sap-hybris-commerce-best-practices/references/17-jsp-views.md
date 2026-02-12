# JSP Views

## Purpose
Create JSP view templates for SAP Commerce storefronts using JSTL, Spring tags, and best practices for maintainable, secure views.

## Basic JSP Structure

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<%@ taglib prefix="template" tagdir="/WEB-INF/tags/responsive/template" %>

<template:page pageTitle="${product.name}">
    <div class="product-detail">
        <h1>${product.name}</h1>
        <p>${product.description}</p>
        <span class="price">${product.price.formattedValue}</span>
    </div>
</template:page>
```

## Common Patterns

### 1. Product Display

```jsp
<div class="product-grid">
    <c:forEach items="${products}" var="product">
        <div class="product-item">
            <img src="${product.imageUrl}" alt="${product.name}"/>
            <h3>${product.name}</h3>
            <p class="price">${product.price.formattedValue}</p>
            <c:if test="${product.available}">
                <button class="add-to-cart">Add to Cart</button>
            </c:if>
        </div>
    </c:forEach>
</div>
```

### 2. Form Handling

```jsp
<form:form method="POST" modelAttribute="productForm">
    <div class="form-group">
        <form:label path="code">Product Code</form:label>
        <form:input path="code" cssClass="form-control"/>
        <form:errors path="code" cssClass="error"/>
    </div>
    
    <div class="form-group">
        <form:label path="name">Product Name</form:label>
        <form:input path="name" cssClass="form-control"/>
        <form:errors path="name" cssClass="error"/>
    </div>
    
    <button type="submit">Create Product</button>
</form:form>
```

### 3. Localization

```jsp
<spring:message code="product.name.label"/>
<spring:message code="product.price.label" arguments="${product.price}"/>
```

## Best Practices

### ✅ DO
- Use JSTL tags instead of scriptlets
- Escape output to prevent XSS
- Use Spring message tags for i18n
- Create reusable tag files
- Use meaningful variable names
- Separate concerns (logic in controllers)
- Use CSS classes for styling
- Implement responsive design
- Test views in multiple browsers
- Use template inheritance

### ❌ DON'T
- Use Java scriptlets (<% %>)
- Put business logic in JSPs
- Hardcode text (use i18n)
- Ignore XSS vulnerabilities
- Create monolithic JSPs
- Use inline styles
- Skip error handling
- Ignore accessibility
- Mix HTML and Java code
- Skip validation

## Resources

- **JSP Best Practices**: Oracle JSP documentation
- **JSTL**: JSTL tag reference
- **Spring Tags**: Spring MVC tag library
