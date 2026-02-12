# WCMS Components

## Purpose
Create and configure Web Content Management System (WCMS) components for dynamic content management in SAP Commerce storefronts.

## Key Concepts

### WCMS Architecture
- **CMS Pages**: Page templates and content pages
- **CMS Components**: Reusable content blocks
- **CMS Slots**: Placeholders for components
- **Content Catalog**: Versioned content management

## Component Types

### 1. Simple Component

```xml
<!-- items.xml -->
<itemtype code="CustomBannerComponent"
          extends="SimpleCMSComponent"
          autocreate="true"
          generate="true">
    <attributes>
        <attribute qualifier="title" type="localized:java.lang.String">
            <persistence type="property"/>
        </attribute>
        <attribute qualifier="imageUrl" type="java.lang.String">
            <persistence type="property"/>
        </attribute>
        <attribute qualifier="linkUrl" type="java.lang.String">
            <persistence type="property"/>
        </attribute>
    </attributes>
</itemtype>
```

### 2. Component Controller

```java
@Controller
@RequestMapping("/view/CustomBannerComponentController")
public class CustomBannerComponentController extends AbstractCMSComponentController<CustomBannerComponentModel> {
    
    @Override
    protected void fillModel(HttpServletRequest request,
                            Model model,
                            CustomBannerComponentModel component) {
        model.addAttribute("title", component.getTitle());
        model.addAttribute("imageUrl", component.getImageUrl());
        model.addAttribute("linkUrl", component.getLinkUrl());
    }
}
```

### 3. Component JSP

```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>

<div class="custom-banner">
    <a href="${linkUrl}">
        <img src="${imageUrl}" alt="${title}" />
        <h2>${title}</h2>
    </a>
</div>
```

## ImpEx Configuration

```impex
# Component Type
INSERT_UPDATE CustomBannerComponent;uid[unique=true];name;title[lang=en];imageUrl;linkUrl;$contentCV
;homepageBanner;Homepage Banner;Welcome to Our Store;/images/banner.jpg;/products

# Page Template
INSERT_UPDATE PageTemplate;uid[unique=true];name;frontendTemplateName;$contentCV
;HomePageTemplate;Home Page Template;home/homePage

# Content Slot
INSERT_UPDATE ContentSlot;uid[unique=true];name;active;$contentCV
;HomepageBannerSlot;Homepage Banner Slot;true

# Assign Component to Slot
INSERT_UPDATE ContentSlotForPage;uid[unique=true];position;page(uid,$contentCV);contentSlot(uid,$contentCV);$contentCV
;HomepageBannerSlot-Homepage;BannerSlot;homepage;HomepageBannerSlot

# Add Component to Slot
INSERT_UPDATE ContentSlot;uid[unique=true];cmsComponents(uid,$contentCV);$contentCV
;HomepageBannerSlot;homepageBanner
```

## Best Practices

### ✅ DO
- Use localized attributes for content
- Create reusable components
- Use content slots for flexibility
- Version content properly
- Test components in Backoffice
- Document component attributes
- Use meaningful component names
- Implement responsive design
- Cache component rendering
- Use CMS restrictions

### ❌ DON'T
- Hardcode content in JSPs
- Skip localization
- Create monolithic components
- Ignore content versioning
- Mix business logic in components
- Skip component testing
- Use inline styles
- Ignore accessibility
- Create tight coupling
- Skip content approval workflow

## Resources

- **WCMS Documentation**: Help Portal → WCMS
- **Component Development**: SAP Commerce documentation
