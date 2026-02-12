# Create Extensions

## Purpose
Understand SAP Commerce extension architecture, types, structure, and how to create custom extensions for modular, maintainable code organization.

## Key Concepts

### Extension Architecture
SAP Commerce is built on an **extension-based architecture** where functionality is organized into modular extensions. Each extension is a self-contained module with its own:
- Data model (items.xml)
- Business logic (services, facades)
- Configuration (properties, Spring beans)
- Resources (localization, ImpEx)
- Web layer (controllers, views)

### Extension Types

1. **Core Extension**: Business logic, services, data model
2. **Facades Extension**: DTOs, converters, facade layer
3. **Storefront Extension**: Web controllers, JSP views, frontend
4. **Backoffice Extension**: Backoffice configuration and customization
5. **OCC Extension**: REST API endpoints (Omnichannel Commerce)
6. **Test Extension**: Integration and unit tests

## Extension Structure

### Standard Directory Layout
```
mycompany/
|-- bin/
|   `-- custom/
|       |-- mycompanycore/              # Core extension
|       |   |-- src/                    # Java source code
|       |   |-- resources/              # Configuration, ImpEx, i18n
|       |   |   |-- mycompanycore-items.xml
|       |   |   |-- mycompanycore-spring.xml
|       |   |   |-- localization/
|       |   |   `-- impex/
|       |   |-- testsrc/                # Unit tests
|       |   |-- web/                    # Web resources (if applicable)
|       |   |-- extensioninfo.xml       # Extension metadata
|       |   |-- project.properties      # Build configuration
|       |   `-- build.xml               # Ant build script
|       |
|       |-- mycompanyfacades/           # Facades extension
|       |   |-- src/
|       |   |-- resources/
|       |   |-- testsrc/
|       |   `-- extensioninfo.xml
|       |
|       `-- mycompanystorefront/        # Storefront extension
|           |-- web/
|           |   |-- src/                # Controllers
|           |   `-- webroot/            # JSP, CSS, JS
|           |-- resources/
|           `-- extensioninfo.xml
```

## Creating a Core Extension

### 1. Generate Extension Template

Use the `extgen` tool to generate extension scaffolding:

```bash
cd [HYBRIS_HOME]/bin/platform
. ./setantenv.sh
ant extgen
```

Follow the prompts:
1. Choose template: `yempty` (for core) or `yacceleratorstorefront` (for storefront)
2. Enter extension name: `mycompanycore`
3. Enter package name: `com.mycompany.core`

### 2. Extension Metadata (`extensioninfo.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<extensioninfo xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:noNamespaceSchemaLocation="extensioninfo.xsd">
    
    <extension abstractclassprefix="Generated"
               classprefix="Mycompany"
               name="mycompanycore"
               usemaven="false">
        
        <description>MyCompany Core Extension</description>
        
        <!-- Required extensions -->
        <requires-extension name="core"/>
        <requires-extension name="commerceservices"/>
        <requires-extension name="basecommerce"/>
        
        <!-- Core module configuration -->
        <coremodule generated="true" manager="com.mycompany.core.jalo.MycompanyCoreManager"
                    packageroot="com.mycompany.core"/>
        
        <!-- Web module (if applicable) -->
        <webmodule jspcompile="false" webroot="/mycompanycore"/>
        
        <!-- HMC module (deprecated, use Backoffice) -->
        <!-- <hmcmodule/> -->
    </extension>
</extensioninfo>
```

### 3. Data Model (`resources/mycompanycore-items.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<items xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="items.xsd">
    
    <atomictypes>
        <atomictype class="com.mycompany.core.enums.OrderStatus"
                    extends="java.lang.String"
                    autocreate="true"
                    generate="true"/>
    </atomictypes>
    
    <enumtypes>
        <enumtype code="OrderStatus" autocreate="true" generate="true">
            <value code="PENDING"/>
            <value code="CONFIRMED"/>
            <value code="SHIPPED"/>
            <value code="DELIVERED"/>
            <value code="CANCELLED"/>
        </enumtype>
    </enumtypes>
    
    <itemtypes>
        <!-- Extend existing Product type -->
        <itemtype code="Product" autocreate="false" generate="false">
            <attributes>
                <attribute qualifier="customField" type="java.lang.String">
                    <persistence type="property"/>
                    <modifiers optional="true"/>
                </attribute>
            </attributes>
        </itemtype>
        
        <!-- Create new custom type -->
        <itemtype code="CustomOrder"
                  extends="Order"
                  autocreate="true"
                  generate="true"
                  jaloclass="com.mycompany.core.jalo.CustomOrder">
            <description>Custom Order Type</description>
            <attributes>
                <attribute qualifier="orderStatus" type="OrderStatus">
                    <persistence type="property"/>
                    <modifiers optional="false" initial="true"/>
                    <defaultvalue>em().getEnumerationValue("OrderStatus","PENDING")</defaultvalue>
                </attribute>
                <attribute qualifier="trackingNumber" type="java.lang.String">
                    <persistence type="property"/>
                    <modifiers optional="true"/>
                </attribute>
            </attributes>
        </itemtype>
    </itemtypes>
</items>
```

### 4. Spring Configuration (`resources/mycompanycore-spring.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd
           http://www.springframework.org/schema/util
           http://www.springframework.org/schema/util/spring-util.xsd">

    <!-- Enable component scanning -->
    <context:component-scan base-package="com.mycompany.core"/>
    
    <!-- System setup -->
    <bean id="mycompanyCoreSystemSetup"
          class="com.mycompany.core.setup.MycompanyCoreSystemSetup">
        <constructor-arg ref="mycompanyCoreService"/>
    </bean>
    
    <!-- Services -->
    <bean id="customOrderService"
          class="com.mycompany.core.service.impl.DefaultCustomOrderService">
        <constructor-arg ref="modelService"/>
        <constructor-arg ref="flexibleSearchService"/>
    </bean>
    
    <!-- Event listeners -->
    <bean id="orderStatusEventListener"
          class="com.mycompany.core.event.OrderStatusEventListener"
          parent="abstractEventListener">
        <constructor-arg ref="customOrderService"/>
    </bean>
</beans>
```

### 5. Build Configuration (`project.properties`)

```properties
# Extension name
extension.name=mycompanycore

# Extension dependencies
extension.dependencies=core,commerceservices,basecommerce

# Build configuration
build.development=true

# Web application settings (if web module)
# webapp.main.contextroot=/mycompanycore

# Ant build settings
javac.source=21
javac.target=21
javac.debug=true
javac.debuglevel=lines,vars,source
```

### 6. Local Configuration (`config/localextensions.xml`)

Add your extension to the local extensions list:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<hybrisconfig xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xsi:noNamespaceSchemaLocation="resources/schemas/extensions.xsd">
    <extensions>
        <!-- SAP Commerce accelerator -->
        <path dir="${HYBRIS_BIN_DIR}" autoload="false"/>
        
        <!-- Custom extensions -->
        <extension name="mycompanycore"/>
        <extension name="mycompanyfacades"/>
        <extension name="mycompanystorefront"/>
    </extensions>
</hybrisconfig>
```

## Extension Dependencies

### Dependency Graph Example
```
mycompanycore
|-- core (platform)
|-- commerceservices
`-- basecommerce

mycompanyfacades
|-- mycompanycore
|-- commercefacades
`-- converters

mycompanystorefront
|-- mycompanyfacades
|-- acceleratorstorefrontcommons
`-- addonsupport
```

### Best Practices for Dependencies
- Keep dependencies minimal and explicit
- Avoid circular dependencies
- Core extension should not depend on facades or storefront
- Facades can depend on core
- Storefront depends on facades

## System Setup Class

Create a system setup class for initialization and updates:

```java
package com.mycompany.core.setup;

import de.hybris.platform.core.initialization.SystemSetup;
import de.hybris.platform.core.initialization.SystemSetupContext;
import de.hybris.platform.core.initialization.SystemSetupParameter;
import de.hybris.platform.core.initialization.SystemSetupParameterMethod;
import org.springframework.stereotype.Component;

@SystemSetup(extension = "mycompanycore")
@Component
public class MycompanyCoreSystemSetup {
    
    private static final String IMPORT_CORE_DATA = "importCoreData";
    private static final String IMPORT_SAMPLE_DATA = "importSampleData";
    
    private final MycompanyCoreService mycompanyCoreService;
    
    public MycompanyCoreSystemSetup(MycompanyCoreService mycompanyCoreService) {
        this.mycompanyCoreService = mycompanyCoreService;
    }
    
    @SystemSetupParameterMethod
    @Override
    public List<SystemSetupParameter> getInitializationOptions() {
        final List<SystemSetupParameter> params = new ArrayList<>();
        
        params.add(createBooleanSystemSetupParameter(
            IMPORT_CORE_DATA, "Import Core Data", true));
        params.add(createBooleanSystemSetupParameter(
            IMPORT_SAMPLE_DATA, "Import Sample Data", true));
        
        return params;
    }
    
    @SystemSetup(type = SystemSetup.Type.ESSENTIAL, process = SystemSetup.Process.ALL)
    public void createEssentialData(final SystemSetupContext context) {
        // Essential data that must always be present
        importImpexFile(context, "/impex/essential-data.impex");
    }
    
    @SystemSetup(type = SystemSetup.Type.PROJECT, process = SystemSetup.Process.ALL)
    public void createProjectData(final SystemSetupContext context) {
        if (getBooleanSystemSetupParameter(context, IMPORT_CORE_DATA)) {
            importImpexFile(context, "/impex/core-data.impex");
        }
        
        if (getBooleanSystemSetupParameter(context, IMPORT_SAMPLE_DATA)) {
            importImpexFile(context, "/impex/sample-data.impex");
        }
    }
    
    private void importImpexFile(SystemSetupContext context, String file) {
        logInfo(context, "Importing: " + file);
        getSetupImpexService().importImpexFile(
            String.format("/%s%s", "mycompanycore", file), false);
    }
}
```

## Building and Deploying

### Build Commands

```bash
# Navigate to platform directory
cd [HYBRIS_HOME]/bin/platform

# Set environment
. ./setantenv.sh

# Clean and build all
ant clean all

# Build specific extension
ant build -Dextension.name=mycompanycore

# Generate models only
ant build -Dextension.name=mycompanycore -Dtarget=models
```

### System Update

After building, perform system update to apply schema changes:

1. **Via HAC** (Hybris Administration Console):
   - Navigate to: http://localhost:9001/hac
   - Platform -> Update -> Update running system

2. **Via Ant**:
   ```bash
   ant updatesystem
   ```

3. **Via API**:
   ```java
   @Autowired
   private SystemSetupAuditService systemSetupAuditService;
   
   public void performUpdate() {
       systemSetupAuditService.performUpdate();
   }
   ```

## Extension Templates

### Core Extension Template
```
mycompanycore/
|-- src/
|   `-- com/mycompany/core/
|       |-- constants/
|       |   `-- MycompanyCoreConstants.java
|       |-- service/
|       |   |-- MycompanyService.java
|       |   `-- impl/
|       |       `-- DefaultMycompanyService.java
|       |-- dao/
|       |   |-- MycompanyDao.java
|       |   `-- impl/
|       |       `-- DefaultMycompanyDao.java
|       |-- event/
|       |-- interceptor/
|       |-- job/
|       `-- setup/
|           `-- MycompanyCoreSystemSetup.java
|-- resources/
|   |-- mycompanycore-items.xml
|   |-- mycompanycore-spring.xml
|   |-- localization/
|   |   `-- mycompanycore-locales_en.properties
|   `-- impex/
|       |-- essential-data.impex
|       `-- sample-data.impex
|-- testsrc/
|   `-- com/mycompany/core/
|       `-- service/
|           `-- MycompanyServiceTest.java
|-- extensioninfo.xml
|-- project.properties
`-- build.xml
```

### Facades Extension Template
```
mycompanyfacades/
|-- src/
|   `-- com/mycompany/facades/
|       |-- facade/
|       |   |-- MycompanyFacade.java
|       |   `-- impl/
|       |       `-- DefaultMycompanyFacade.java
|       |-- data/
|       |   `-- MycompanyData.java
|       |-- converter/
|       |   `-- MycompanyConverter.java
|       `-- populators/
|           `-- MycompanyPopulator.java
|-- resources/
|   `-- mycompanyfacades-spring.xml
|-- testsrc/
`-- extensioninfo.xml
```

## Best Practices

### DO
- Use `extgen` to generate extension scaffolding
- Follow standard directory structure
- Declare all dependencies in `extensioninfo.xml`
- Use meaningful extension names (company prefix)
- Separate concerns: core, facades, storefront
- Version control extension configuration
- Document extension purpose and dependencies
- Use system setup for data initialization
- Run `ant clean all` after items.xml changes
- Perform system update after schema changes

### DON'T
- Modify platform or accelerator extensions directly
- Create circular dependencies
- Mix concerns (e.g., web logic in core)
- Hardcode extension names in code
- Skip system update after items.xml changes
- Commit generated files (gensrc, classes)
- Use deprecated HMC module
- Create monolithic extensions
- Ignore extension dependency order

## Common Pitfalls

### 1. Missing Dependencies
```xml
<!-- Wrong: Missing required dependency -->
<requires-extension name="commerceservices"/>
<!-- But using classes from basecommerce -->

<!-- Correct: Declare all dependencies -->
<requires-extension name="core"/>
<requires-extension name="commerceservices"/>
<requires-extension name="basecommerce"/>
```

### 2. Circular Dependencies
```
Wrong:
mycompanycore -> mycompanyfacades -> mycompanycore

Correct:
mycompanycore -> mycompanyfacades -> mycompanystorefront
```

### 3. Skipping System Update
```bash
# Wrong: Build without system update
ant clean all
# Start server - schema changes not applied!

# Correct: Build and update
ant clean all
ant initialize  # or updatesystem
```

## Resources

- **SAP Commerce Documentation**: Extension Development
- **Accelerator Template**: Reference implementation
- **extgen Tool**: Extension generation utility
