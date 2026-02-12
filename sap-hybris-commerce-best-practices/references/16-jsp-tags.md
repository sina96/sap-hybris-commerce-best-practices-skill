# JSP Tags

## Purpose
Create and use custom JSP tag libraries for reusable view logic in SAP Commerce storefronts.

## Standard Tags

### 1. Common Tag Libraries

```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt" %>
<%@ taglib prefix="fn" uri="http://java.sun.com/jsp/jstl/functions" %>
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags" %>
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
<%@ taglib prefix="cms" uri="http://hybris.com/tld/cmstags" %>
```

### 2. JSTL Core Tags

```jsp
<!-- Conditional -->
<c:if test="${product.available}">
    <span>In Stock</span>
</c:if>

<c:choose>
    <c:when test="${product.stockLevel > 10}">
        <span class="in-stock">In Stock</span>
    </c:when>
    <c:when test="${product.stockLevel > 0}">
        <span class="low-stock">Low Stock</span>
    </c:when>
    <c:otherwise>
        <span class="out-of-stock">Out of Stock</span>
    </c:otherwise>
</c:choose>

<!-- Iteration -->
<c:forEach items="${products}" var="product">
    <div class="product">${product.name}</div>
</c:forEach>

<!-- URL -->
<c:url value="/products/${product.code}" var="productUrl"/>
<a href="${productUrl}">View Product</a>
```

## Custom Tag Library

### 1. Tag Handler

```java
package com.mycompany.storefront.tags;

import javax.servlet.jsp.JspException;
import javax.servlet.jsp.tagext.TagSupport;

public class FormatPriceTag extends TagSupport {
    
    private Double value;
    private String currency;
    
    @Override
    public int doStartTag() throws JspException {
        try {
            String formatted = String.format("%s %.2f", currency, value);
            pageContext.getOut().write(formatted);
        } catch (Exception e) {
            throw new JspException("Error formatting price", e);
        }
        return SKIP_BODY;
    }
    
    public void setValue(Double value) {
        this.value = value;
    }
    
    public void setCurrency(String currency) {
        this.currency = currency;
    }
}
```

### 2. TLD Configuration

```xml
<!-- WEB-INF/tags/mycompany.tld -->
<?xml version="1.0" encoding="UTF-8"?>
<taglib xmlns="http://java.sun.com/xml/ns/javaee"
        version="2.1">
    
    <tlib-version>1.0</tlib-version>
    <short-name>mycompany</short-name>
    <uri>http://mycompany.com/tags</uri>
    
    <tag>
        <name>formatPrice</name>
        <tag-class>com.mycompany.storefront.tags.FormatPriceTag</tag-class>
        <body-content>empty</body-content>
        <attribute>
            <name>value</name>
            <required>true</required>
            <rtexprvalue>true</rtexprvalue>
        </attribute>
        <attribute>
            <name>currency</name>
            <required>true</required>
            <rtexprvalue>true</rtexprvalue>
        </attribute>
    </tag>
</taglib>
```

### 3. Usage

```jsp
<%@ taglib prefix="mycompany" uri="http://mycompany.com/tags" %>

<mycompany:formatPrice value="${product.price}" currency="USD"/>
```

## Best Practices

### DO
- Use JSTL tags when possible
- Create reusable custom tags
- Document tag attributes
- Handle errors gracefully
- Use meaningful tag names
- Test tags thoroughly
- Escape output properly
- Use tag files for simple tags
- Cache tag results when appropriate
- Follow naming conventions

### DON'T
- Put business logic in tags
- Create complex tags
- Skip error handling
- Ignore XSS vulnerabilities
- Use scriptlets
- Create duplicate tags
- Skip documentation
- Ignore performance
- Use deprecated tags
- Mix concerns

## Resources

- **JSTL**: Oracle JSTL documentation
- **Custom Tags**: JSP tag development guide
