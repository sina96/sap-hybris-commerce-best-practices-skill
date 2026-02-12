# Define Data Types

## Purpose
Master SAP Commerce's type system for defining data models, relationships, and attributes using items.xml. The type system is metadata-driven and generates Java model classes automatically.

## Key Concepts

### Type System Hierarchy
```
GenericItem (root)
├── Item
│   ├── Product
│   ├── Category
│   ├── Order
│   ├── User
│   └── [Custom Types]
├── Relation
└── EnumerationValue
```

### Type Categories
1. **ItemType**: Business entities (Product, Order, User)
2. **EnumType**: Enumeration values (OrderStatus, Gender)
3. **AtomicType**: Custom primitive types
4. **RelationType**: Many-to-many relationships
5. **CollectionType**: Typed collections
6. **MapType**: Typed maps

## Items.xml Structure

### Basic Template
```xml
<?xml version="1.0" encoding="UTF-8"?>
<items xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="items.xsd">
    
    <atomictypes>
        <!-- Custom atomic types -->
    </atomictypes>
    
    <collectiontypes>
        <!-- Custom collection types -->
    </collectiontypes>
    
    <enumtypes>
        <!-- Enumeration types -->
    </enumtypes>
    
    <maptypes>
        <!-- Custom map types -->
    </maptypes>
    
    <relations>
        <!-- Many-to-many relationships -->
    </relations>
    
    <itemtypes>
        <!-- Business entity types -->
    </itemtypes>
</items>
```

## Defining Item Types

### 1. Create New Item Type

```xml
<itemtypes>
    <itemtype code="CustomProduct"
              extends="Product"
              autocreate="true"
              generate="true"
              jaloclass="com.mycompany.core.jalo.CustomProduct">
        
        <description>Custom Product with additional attributes</description>
        
        <attributes>
            <!-- Simple attribute -->
            <attribute qualifier="manufacturer" type="java.lang.String">
                <description>Product manufacturer</description>
                <persistence type="property"/>
                <modifiers optional="true"/>
            </attribute>
            
            <!-- Localized attribute -->
            <attribute qualifier="marketingText" type="localized:java.lang.String">
                <description>Localized marketing text</description>
                <persistence type="property"/>
                <modifiers optional="true"/>
            </attribute>
            
            <!-- Numeric attribute with default -->
            <attribute qualifier="warrantyMonths" type="java.lang.Integer">
                <persistence type="property"/>
                <modifiers optional="true"/>
                <defaultvalue>Integer.valueOf(12)</defaultvalue>
            </attribute>
            
            <!-- Boolean attribute -->
            <attribute qualifier="featured" type="java.lang.Boolean">
                <persistence type="property"/>
                <modifiers optional="true"/>
                <defaultvalue>Boolean.FALSE</defaultvalue>
            </attribute>
            
            <!-- Date attribute -->
            <attribute qualifier="launchDate" type="java.util.Date">
                <persistence type="property"/>
                <modifiers optional="true"/>
            </attribute>
            
            <!-- Enum attribute -->
            <attribute qualifier="productStatus" type="ProductStatus">
                <persistence type="property"/>
                <modifiers optional="false" initial="true"/>
                <defaultvalue>em().getEnumerationValue("ProductStatus","ACTIVE")</defaultvalue>
            </attribute>
            
            <!-- Reference to another item -->
            <attribute qualifier="primaryCategory" type="Category">
                <persistence type="property"/>
                <modifiers optional="true"/>
            </attribute>
            
            <!-- Collection attribute -->
            <attribute qualifier="relatedProducts" type="ProductList">
                <persistence type="property"/>
                <modifiers optional="true"/>
            </attribute>
        </attributes>
        
        <indexes>
            <index name="manufacturerIdx">
                <key attribute="manufacturer"/>
            </index>
            <index name="statusDateIdx">
                <key attribute="productStatus"/>
                <key attribute="launchDate"/>
            </index>
        </indexes>
    </itemtype>
</itemtypes>
```

### 2. Extend Existing Item Type

```xml
<itemtypes>
    <!-- Extend Product without creating new type -->
    <itemtype code="Product" autocreate="false" generate="false">
        <description>Extended Product type</description>
        <attributes>
            <attribute qualifier="customField" type="java.lang.String">
                <persistence type="property"/>
                <modifiers optional="true"/>
            </attribute>
            
            <attribute qualifier="externalId" type="java.lang.String">
                <persistence type="property"/>
                <modifiers optional="true" unique="true"/>
            </attribute>
        </attributes>
        
        <indexes>
            <index name="externalIdIdx" unique="true">
                <key attribute="externalId"/>
            </index>
        </indexes>
    </itemtype>
</itemtypes>
```

## Attribute Modifiers

### Common Modifiers
```xml
<attribute qualifier="attributeName" type="java.lang.String">
    <persistence type="property"/>
    <modifiers 
        optional="true"           <!-- Can be null -->
        initial="false"           <!-- Can be set after creation -->
        write="true"              <!-- Can be modified -->
        read="true"               <!-- Can be read -->
        search="true"             <!-- Can be used in FlexibleSearch -->
        unique="false"            <!-- Must be unique -->
        encrypted="false"         <!-- Encrypt in database -->
        private="false"           <!-- Private to type -->
        partof="false"            <!-- Cascade delete -->
    />
</attribute>
```

### Persistence Types
```xml
<!-- Property: Stored in database column -->
<persistence type="property"/>

<!-- Dynamic: Computed at runtime (no DB storage) -->
<persistence type="dynamic" attributeHandler="myAttributeHandler"/>

<!-- Jalo: Stored via Jalo layer (deprecated) -->
<persistence type="jalo"/>
```

### Default Values
```xml
<!-- String default -->
<defaultvalue>"DEFAULT_VALUE"</defaultvalue>

<!-- Numeric default -->
<defaultvalue>Integer.valueOf(0)</defaultvalue>
<defaultvalue>Double.valueOf(0.0)</defaultvalue>

<!-- Boolean default -->
<defaultvalue>Boolean.TRUE</defaultvalue>

<!-- Enum default -->
<defaultvalue>em().getEnumerationValue("OrderStatus","PENDING")</defaultvalue>

<!-- Date default (current time) -->
<defaultvalue>new java.util.Date()</defaultvalue>
```

## Enumeration Types

### Define Enum Type
```xml
<enumtypes>
    <enumtype code="ProductStatus" 
              autocreate="true" 
              generate="true"
              dynamic="false">
        <description>Product status enumeration</description>
        <value code="ACTIVE"/>
        <value code="INACTIVE"/>
        <value code="DISCONTINUED"/>
        <value code="COMING_SOON"/>
    </enumtype>
    
    <!-- Dynamic enum (values can be added at runtime) -->
    <enumtype code="OrderStatus" 
              autocreate="true" 
              generate="true"
              dynamic="true">
        <value code="PENDING"/>
        <value code="CONFIRMED"/>
        <value code="SHIPPED"/>
        <value code="DELIVERED"/>
        <value code="CANCELLED"/>
    </enumtype>
</enumtypes>
```

### Using Enums in Code
```java
// Get enum value
ProductStatus status = ProductStatus.ACTIVE;

// Set enum on model
product.setProductStatus(ProductStatus.ACTIVE);

// Get enum from string
ProductStatus status = enumerationService.getEnumerationValue(
    ProductStatus.class, "ACTIVE");

// Get all enum values
Collection<ProductStatus> allStatuses = 
    enumerationService.getEnumerationValues(ProductStatus.class);
```

## Relations

### One-to-Many (via attribute)
```xml
<itemtypes>
    <itemtype code="Order" autocreate="false" generate="false">
        <attributes>
            <!-- One-to-many: Order has many OrderEntries -->
            <attribute qualifier="entries" type="OrderEntryList">
                <persistence type="property"/>
                <modifiers optional="true" partof="true"/>
            </attribute>
        </attributes>
    </itemtype>
    
    <itemtype code="OrderEntry" autocreate="false" generate="false">
        <attributes>
            <!-- Many-to-one: OrderEntry belongs to Order -->
            <attribute qualifier="order" type="Order">
                <persistence type="property"/>
                <modifiers optional="false"/>
            </attribute>
        </attributes>
    </itemtype>
</itemtypes>
```

### Many-to-Many (via relation)
```xml
<relations>
    <relation code="Product2CategoryRelation"
               localized="false"
               generate="true"
               autocreate="true">
        
        <description>Product to Category many-to-many relation</description>
        
        <sourceElement type="Product" qualifier="products" cardinality="many">
            <modifiers read="true" write="true" search="true" optional="true"/>
        </sourceElement>
        
        <targetElement type="Category" qualifier="categories" cardinality="many">
            <modifiers read="true" write="true" search="true" optional="true" partof="true"/>
        </targetElement>
    </relation>
    
    <!-- Relation with ordering -->
    <relation code="Category2ProductRelation"
               localized="false"
               generate="true"
               autocreate="true">
        
        <deployment table="Cat2ProdRel" typecode="12345"/>
        
        <sourceElement type="Category" qualifier="category" cardinality="one">
            <modifiers read="true" write="true" search="true" optional="false"/>
        </sourceElement>
        
        <targetElement type="Product" qualifier="products" cardinality="many" 
                       collectiontype="list" ordered="true">
            <modifiers read="true" write="true" search="true" optional="true" partof="true"/>
        </targetElement>
    </relation>
</relations>
```

### Using Relations in Code
```java
// Many-to-many: Add product to categories
CategoryModel category = categoryService.getCategoryForCode("electronics");
product.setCategories(Collections.singleton(category));
modelService.save(product);

// Access reverse relation
List<ProductModel> products = category.getProducts();

// One-to-many: Add order entry
OrderEntryModel entry = modelService.create(OrderEntryModel.class);
entry.setOrder(order);
entry.setProduct(product);
entry.setQuantity(1L);
order.setEntries(Collections.singletonList(entry));
modelService.save(order);
```

## Collection Types

### Define Collection Type
```xml
<collectiontypes>
    <collectiontype code="ProductList" 
                    elementtype="Product" 
                    autocreate="true" 
                    generate="true" 
                    type="list"/>
    
    <collectiontype code="CategorySet" 
                    elementtype="Category" 
                    autocreate="true" 
                    generate="true" 
                    type="set"/>
    
    <collectiontype code="StringCollection" 
                    elementtype="java.lang.String" 
                    autocreate="true" 
                    generate="true" 
                    type="collection"/>
</collectiontypes>
```

### Using Collections
```xml
<itemtypes>
    <itemtype code="CustomProduct" extends="Product" 
              autocreate="true" generate="true">
        <attributes>
            <attribute qualifier="relatedProducts" type="ProductList">
                <persistence type="property"/>
                <modifiers optional="true"/>
            </attribute>
            
            <attribute qualifier="tags" type="StringCollection">
                <persistence type="property"/>
                <modifiers optional="true"/>
            </attribute>
        </attributes>
    </itemtype>
</itemtypes>
```

## Map Types

### Define Map Type
```xml
<maptypes>
    <maptype code="LocalizedStringMap"
             argumenttype="Language"
             returntype="java.lang.String"
             autocreate="true"
             generate="true"/>
    
    <maptype code="ProductAttributeMap"
             argumenttype="java.lang.String"
             returntype="java.lang.Object"
             autocreate="true"
             generate="true"/>
</maptypes>
```

### Using Maps
```xml
<itemtypes>
    <itemtype code="CustomProduct" extends="Product" 
              autocreate="true" generate="true">
        <attributes>
            <attribute qualifier="customAttributes" type="ProductAttributeMap">
                <persistence type="property"/>
                <modifiers optional="true"/>
            </attribute>
        </attributes>
    </itemtype>
</itemtypes>
```

```java
// Using map in code
Map<String, Object> attributes = new HashMap<>();
attributes.put("color", "red");
attributes.put("size", "large");
product.setCustomAttributes(attributes);
modelService.save(product);
```

## Indexes

### Define Indexes
```xml
<itemtypes>
    <itemtype code="Product" autocreate="false" generate="false">
        <indexes>
            <!-- Simple index -->
            <index name="codeIdx">
                <key attribute="code"/>
            </index>
            
            <!-- Unique index -->
            <index name="externalIdIdx" unique="true">
                <key attribute="externalId"/>
            </index>
            
            <!-- Composite index -->
            <index name="catalogVersionIdx">
                <key attribute="catalogVersion"/>
                <key attribute="code"/>
            </index>
            
            <!-- Include additional columns -->
            <index name="productSearchIdx">
                <key attribute="code"/>
                <key attribute="name"/>
                <include attribute="description"/>
            </index>
        </indexes>
    </itemtype>
</itemtypes>
```

## Localized Attributes

### Define Localized Attribute
```xml
<itemtypes>
    <itemtype code="Product" autocreate="false" generate="false">
        <attributes>
            <!-- Localized string -->
            <attribute qualifier="name" type="localized:java.lang.String">
                <persistence type="property"/>
                <modifiers optional="false"/>
            </attribute>
            
            <!-- Localized text (CLOB) -->
            <attribute qualifier="description" type="localized:java.lang.String">
                <persistence type="property">
                    <columntype>
                        <value>HYBRIS.LONG_STRING</value>
                    </columntype>
                </persistence>
                <modifiers optional="true"/>
            </attribute>
        </attributes>
    </itemtype>
</itemtypes>
```

### Using Localized Attributes
```java
// Set localized value for current session locale
product.setName("Product Name");

// Set localized value for specific locale
Locale english = Locale.ENGLISH;
Locale german = Locale.GERMAN;

product.setName("Product Name", english);
product.setName("Produktname", german);

modelService.save(product);

// Get localized value
String name = product.getName(); // Current session locale
String germanName = product.getName(german);
```

## Generated Model Classes

After defining types in items.xml, run `ant clean all` to generate model classes:

```java
// Generated model class
package com.mycompany.core.model;

import de.hybris.platform.core.model.product.ProductModel;

public class CustomProductModel extends ProductModel {
    
    // Generated constants
    public static final String _TYPECODE = "CustomProduct";
    public static final String MANUFACTURER = "manufacturer";
    public static final String WARRANTYMONTHS = "warrantyMonths";
    
    // Generated getters/setters
    public String getManufacturer() { /* ... */ }
    public void setManufacturer(String value) { /* ... */ }
    
    public Integer getWarrantyMonths() { /* ... */ }
    public void setWarrantyMonths(Integer value) { /* ... */ }
    
    // ... other generated methods
}
```

## Best Practices

### ✅ DO
- Use meaningful type and attribute names
- Add descriptions to types and attributes
- Define indexes for frequently queried attributes
- Use enums for fixed value sets
- Use relations for many-to-many associations
- Set appropriate modifiers (optional, unique, etc.)
- Use localized attributes for user-facing text
- Version control items.xml
- Run `ant clean all` after changes
- Perform system update after build

### ❌ DON'T
- Modify generated model classes
- Use reserved type codes (< 10000)
- Create circular type dependencies
- Skip indexes on foreign keys
- Use dynamic persistence without good reason
- Hardcode enum values in code
- Change type codes after deployment
- Remove attributes (mark as deprecated instead)
- Use overly generic type names
- Skip system update after schema changes

## Common Pitfalls

### 1. Missing System Update
```bash
# ❌ Wrong: Build without update
ant clean all
# Schema changes not applied!

# ✅ Correct: Build and update
ant clean all
ant initialize  # or updatesystem
```

### 2. Modifying Generated Classes
```java
// ❌ Wrong: Modifying generated model
public class CustomProductModel extends ProductModel {
    // Custom method added here - will be overwritten!
    public String getDisplayName() { /* ... */ }
}

// ✅ Correct: Use service layer
@Service
public class ProductDisplayService {
    public String getDisplayName(ProductModel product) { /* ... */ }
}
```

### 3. Incorrect Default Values
```xml
<!-- ❌ Wrong: Invalid syntax -->
<defaultvalue>0</defaultvalue>

<!-- ✅ Correct: Proper Java expression -->
<defaultvalue>Integer.valueOf(0)</defaultvalue>
```

### 4. Missing Indexes
```xml
<!-- ❌ Wrong: No index on frequently queried field -->
<attribute qualifier="externalId" type="java.lang.String">
    <persistence type="property"/>
    <modifiers optional="true" unique="true"/>
</attribute>

<!-- ✅ Correct: Add unique index -->
<attribute qualifier="externalId" type="java.lang.String">
    <persistence type="property"/>
    <modifiers optional="true" unique="true"/>
</attribute>
<indexes>
    <index name="externalIdIdx" unique="true">
        <key attribute="externalId"/>
    </index>
</indexes>
```

## Resources

- **SAP Commerce Type System**: Help Portal → Type System
- **items.xsd**: Schema definition for items.xml validation
- **Platform items.xml**: Reference for standard types
