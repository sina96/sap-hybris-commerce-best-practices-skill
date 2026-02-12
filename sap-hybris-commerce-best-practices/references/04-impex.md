# Using ImpEx

## Purpose
Master ImpEx (Import/Export) for data management in SAP Commerce. ImpEx is a CSV-based scripting language for importing, exporting, and manipulating data.

## Key Concepts

### ImpEx Basics
- **CSV-based syntax**: Semicolon-separated values
- **Header + Data rows**: Define structure then provide data
- **Macros**: Variables for reusable values
- **Modes**: INSERT, UPDATE, INSERT_UPDATE, REMOVE
- **Translators**: Convert string values to objects

### Common Use Cases
- Initial data setup (catalogs, users, products)
- Sample data import
- Data migration
- Bulk updates
- Testing data creation

## Basic Syntax

### INSERT Mode
```impex
# Insert new items (fails if item exists)
INSERT_UPDATE Language;isocode[unique=true];active
;en;true
;de;true
;fr;true

# Insert with mode explicitly set
INSERT Product;code[unique=true];name;catalogVersion(catalog(id),version)
;PROD001;Test Product;Default:Staged
```

### UPDATE Mode
```impex
# Update existing items only (fails if not exists)
UPDATE Product;code[unique=true];name;description
;PROD001;Updated Name;Updated description
```

### INSERT_UPDATE Mode (Most Common)
```impex
# Insert if not exists, update if exists
INSERT_UPDATE Product;code[unique=true];name;catalogVersion(catalog(id),version)
;PROD001;Product 1;Default:Staged
;PROD002;Product 2;Default:Staged
;PROD003;Product 3;Default:Staged
```

### REMOVE Mode
```impex
# Remove items
REMOVE Product;code[unique=true];catalogVersion(catalog(id),version)
;PROD001;Default:Staged
;PROD002;Default:Staged
```

## Header Syntax

### Basic Header
```impex
INSERT_UPDATE Product;code[unique=true];name;description
```

### Modifiers
```impex
# unique: Part of unique key
INSERT_UPDATE Product;code[unique=true];name

# default: Default value if not provided
INSERT_UPDATE Product;code[unique=true];name;active[default=true]

# allownull: Allow null values
INSERT_UPDATE Product;code[unique=true];name;description[allownull=true]

# translator: Custom value translator
INSERT_UPDATE Product;code[unique=true];price[translator=de.hybris.platform.impex.jalo.translators.PriceTranslator]

# dateformat: Date parsing format
INSERT_UPDATE Product;code[unique=true];launchDate[dateformat=yyyy-MM-dd]

# numberformat: Number parsing format
INSERT_UPDATE Product;code[unique=true];weight[numberformat=0.00]

# lang: Localized attribute language
INSERT_UPDATE Product;code[unique=true];name[lang=en];name[lang=de]
```

## Macros and Variables

### Define Macros
```impex
# Define variables at top of file
$catalog=Default
$catalogVersion=catalogVersion(catalog(id[default=$catalog]),version[default=Staged])[unique=true,default=$catalog:Staged]
$approved=approvalstatus(code)[default='approved']
$contentCV=catalogVersion(catalog(id[default='ContentCatalog']),version[default='Staged'])[unique=true]

# Use macros in headers
INSERT_UPDATE Product;code[unique=true];name;$catalogVersion;$approved
;PROD001;Product 1
;PROD002;Product 2
```

### System Properties
```impex
# Use system properties
$siteUid=electronics
INSERT_UPDATE CMSSite;uid[unique=true];name
;$siteUid;Electronics Site
```

## Reference Syntax

### Simple Reference
```impex
# Reference by unique attribute
INSERT_UPDATE Product;code[unique=true];primaryCategory(code)
;PROD001;CAT001
```

### Composite Reference
```impex
# Reference with multiple attributes
INSERT_UPDATE Product;code[unique=true];catalogVersion(catalog(id),version)
;PROD001;Default:Staged
```

### Nested Reference
```impex
# Deep reference navigation
INSERT_UPDATE Product;code[unique=true];primaryCategory(code,catalogVersion(catalog(id),version))
;PROD001;CAT001:Default:Staged
```

## Collections

### Simple Collection
```impex
# Comma-separated values
INSERT_UPDATE Product;code[unique=true];supercategories(code)
;PROD001;CAT001,CAT002,CAT003
```

### Collection with References
```impex
# Collection with composite references
INSERT_UPDATE Category;code[unique=true];products(code,catalogVersion(catalog(id),version))
;CAT001;PROD001:Default:Staged,PROD002:Default:Staged,PROD003:Default:Staged
```

## Localized Attributes

### Multiple Languages
```impex
INSERT_UPDATE Product;code[unique=true];name[lang=en];name[lang=de];name[lang=fr]
;PROD001;Product Name;Produktname;Nom du produit
;PROD002;Another Product;Ein anderes Produkt;Un autre produit
```

### Separate Language Blocks
```impex
# English
INSERT_UPDATE Product;code[unique=true];name[lang=en];description[lang=en]
;PROD001;Product Name;English description

# German
UPDATE Product;code[unique=true];name[lang=de];description[lang=de]
;PROD001;Produktname;Deutsche Beschreibung

# French
UPDATE Product;code[unique=true];name[lang=fr];description[lang=fr]
;PROD001;Nom du produit;Description francaise
```

## Media Import

### Import Media Files
```impex
# Define media
INSERT_UPDATE Media;code[unique=true];realfilename;@media[translator=de.hybris.platform.impex.jalo.media.MediaDataTranslator];mime[default='image/jpeg'];folder(qualifier)[default='images']
;product-001.jpg;product-001.jpg;$siteResource/images/product-001.jpg

# Assign media to product
INSERT_UPDATE Product;code[unique=true];picture(code)
;PROD001;product-001.jpg
```

### Media from JAR
```impex
$jarResource=jar:com.mycompany.core.setup.MycompanyCoreSystemSetup&/mycompanycore/import

INSERT_UPDATE Media;code[unique=true];@media[translator=de.hybris.platform.impex.jalo.media.MediaDataTranslator];mime[default='image/jpeg']
;logo.jpg;$jarResource/images/logo.jpg
```

## User and Security

### Create Users
```impex
INSERT_UPDATE Employee;uid[unique=true];name;groups(uid);password[default=$defaultPassword]
;admin@mycompany.com;Admin User;admingroup
;user@mycompany.com;Regular User;customergroup
```

### User Groups
```impex
INSERT_UPDATE UserGroup;uid[unique=true];groups(uid)
;customergroup;
;employeegroup;
;admingroup;employeegroup
```

### Passwords
```impex
# Plain text (will be encoded)
INSERT_UPDATE Customer;uid[unique=true];name;password
;customer@test.com;Test Customer;password123

# Pre-encoded password
INSERT_UPDATE Customer;uid[unique=true];name;encodedPassword
;customer@test.com;Test Customer;*:5f4dcc3b5aa765d61d8327deb882cf99
```

## Catalog Management

### Create Catalog Structure
```impex
# Catalog
INSERT_UPDATE Catalog;id[unique=true];name[lang=en];defaultCatalog
;Default;Default Catalog;true

# Catalog Versions
INSERT_UPDATE CatalogVersion;catalog(id)[unique=true];version[unique=true];active;defaultCurrency(isocode)
;Default;Staged;false;USD
;Default;Online;true;USD

# Sync Job
INSERT_UPDATE CatalogVersionSyncJob;code[unique=true];sourceVersion(catalog(id),version);targetVersion(catalog(id),version)
;sync Default Staged->Online;Default:Staged;Default:Online
```

### Products with Catalog Version
```impex
$catalogVersion=catalogVersion(catalog(id[default='Default']),version[default='Staged'])[unique=true,default='Default:Staged']

INSERT_UPDATE Product;code[unique=true];name;$catalogVersion
;PROD001;Product 1
;PROD002;Product 2
;PROD003;Product 3
```

## Categories

### Category Hierarchy
```impex
$catalogVersion=catalogVersion(catalog(id[default='Default']),version[default='Staged'])[unique=true,default='Default:Staged']

# Root categories
INSERT_UPDATE Category;code[unique=true];name[lang=en];$catalogVersion
;electronics;Electronics
;clothing;Clothing
;home;Home & Garden

# Subcategories
INSERT_UPDATE Category;code[unique=true];name[lang=en];supercategories(code,$catalogVersion);$catalogVersion
;laptops;Laptops;electronics
;phones;Phones;electronics
;mens-clothing;Men's Clothing;clothing
;womens-clothing;Women's Clothing;clothing
```

## Prices

### Price Rows
```impex
INSERT_UPDATE PriceRow;product(code,catalogVersion(catalog(id),version))[unique=true];currency(isocode)[unique=true];price;unit(code)[default='pieces']
;PROD001:Default:Staged;USD;99.99
;PROD001:Default:Staged;EUR;89.99
;PROD002:Default:Staged;USD;149.99
;PROD002:Default:Staged;EUR;139.99
```

### Volume Prices
```impex
INSERT_UPDATE PriceRow;product(code,catalogVersion(catalog(id),version))[unique=true];currency(isocode)[unique=true];minqtd;price;unit(code)[default='pieces']
;PROD001:Default:Staged;USD;1;99.99
;PROD001:Default:Staged;USD;10;89.99
;PROD001:Default:Staged;USD;50;79.99
```

## Stock Levels

### Warehouse Stock
```impex
INSERT_UPDATE Warehouse;code[unique=true];name
;warehouse_1;Main Warehouse
;warehouse_2;Secondary Warehouse

INSERT_UPDATE StockLevel;productCode[unique=true];warehouse(code)[unique=true];available;inStockStatus(code)
;PROD001;warehouse_1;100;inStock
;PROD002;warehouse_1;50;inStock
;PROD003;warehouse_1;0;outOfStock
```

## Advanced Techniques

### Conditional Import
```impex
# Import only if condition is met
INSERT_UPDATE Product;code[unique=true];name;active[default=true]
#% if: impex.getLastImportedItem().getCode().startsWith("PROD")
;PROD001;Product 1
;PROD002;Product 2
#% endif
```

### BeanShell Scripts
```impex
# Execute BeanShell code
#% impex.info("Starting product import");
INSERT_UPDATE Product;code[unique=true];name
;PROD001;Product 1
#% impex.info("Product import completed");
```

### Groovy Scripts
```groovy
#% groovy%
import de.hybris.platform.core.model.product.ProductModel

def products = []
for (int i = 1; i <= 100; i++) {
    products << "PROD${String.format('%03d', i)}"
}
return products
#% groovy%

INSERT_UPDATE Product;code[unique=true];name
#% for: code : products
;${code};Product ${code}
#% endfor
```

## Import from File

### Import ImpEx File
```java
@Service
public class DataImportService {
    
    @Autowired
    private ImportService importService;
    
    public void importData(String impexFile) {
        final ImportConfig config = new ImportConfig();
        config.setScript(new StreamBasedImpExResource(
            getClass().getResourceAsStream(impexFile), "UTF-8"));
        config.setSynchronous(true);
        config.setLegacyMode(false);
        
        final ImportResult result = importService.importData(config);
        
        if (result.isError()) {
            throw new ImportException("Import failed: " + result.getUnresolvedLines());
        }
    }
}
```

### System Setup Import
```java
@SystemSetup(extension = "mycompanycore")
public class MycompanyCoreSystemSetup {
    
    @SystemSetup(type = SystemSetup.Type.PROJECT, process = SystemSetup.Process.ALL)
    public void importData(final SystemSetupContext context) {
        importImpexFile(context, "/impex/products.impex");
        importImpexFile(context, "/impex/categories.impex");
    }
    
    private void importImpexFile(SystemSetupContext context, String file) {
        logInfo(context, "Importing: " + file);
        getSetupImpexService().importImpexFile(
            String.format("/%s%s", context.getExtensionName(), file), false);
    }
}
```

## Export Data

### Export ImpEx
```java
@Service
public class DataExportService {
    
    @Autowired
    private ExportService exportService;
    
    public void exportProducts() throws IOException {
        final ExportConfig config = new ExportConfig();
        config.setScript(
            "INSERT_UPDATE Product;code[unique=true];name;catalogVersion(catalog(id),version)\n" +
            "#% impex.exportItems(\"SELECT {pk} FROM {Product}\", 1000, true);"
        );
        
        final File outputFile = new File("products-export.impex");
        config.setTargetFile(outputFile);
        
        final ExportResult result = exportService.exportData(config);
        
        if (result.isError()) {
            throw new ExportException("Export failed");
        }
    }
}
```

## Best Practices

### DO
- Use INSERT_UPDATE for idempotent imports
- Define macros for reusable values
- Use meaningful variable names
- Add comments to explain complex logic
- Test ImpEx scripts in HAC first
- Use catalog version macros
- Validate data before import
- Use transactions for large imports
- Handle errors gracefully
- Version control ImpEx files

### DON'T
- Use INSERT for data that may already exist
- Hardcode catalog versions
- Import without validation
- Skip error handling
- Use plain text passwords
- Import large files synchronously
- Mix concerns in single file
- Use deprecated syntax
- Ignore import errors
- Commit sensitive data

## Common Pitfalls

### 1. Missing Unique Keys
```impex
# Wrong: No unique key
INSERT_UPDATE Product;code;name
;PROD001;Product 1

# Correct: Unique key specified
INSERT_UPDATE Product;code[unique=true];name
;PROD001;Product 1
```

### 2. Incorrect Reference Syntax
```impex
# Wrong: Missing catalog version
INSERT_UPDATE Product;code[unique=true];primaryCategory(code)
;PROD001;CAT001

# Correct: Full reference
INSERT_UPDATE Product;code[unique=true];primaryCategory(code,catalogVersion(catalog(id),version))
;PROD001;CAT001:Default:Staged
```

### 3. Circular Dependencies
```impex
# Wrong: Circular reference
INSERT_UPDATE Category;code[unique=true];supercategories(code)
;CAT001;CAT002
;CAT002;CAT001

# Correct: Proper hierarchy
INSERT_UPDATE Category;code[unique=true];supercategories(code)
;CAT001;
;CAT002;CAT001
```

### 4. Missing Catalog Version
```impex
# Wrong: No catalog version
INSERT_UPDATE Product;code[unique=true];name
;PROD001;Product 1

# Correct: Include catalog version
$catalogVersion=catalogVersion(catalog(id[default='Default']),version[default='Staged'])[unique=true]
INSERT_UPDATE Product;code[unique=true];name;$catalogVersion
;PROD001;Product 1
```

## Testing ImpEx

### HAC Console
1. Navigate to: http://localhost:9001/hac
2. Console -> ImpEx Import
3. Paste ImpEx script
4. Click "Import content"
5. Review results and errors

### Unit Test
```java
@IntegrationTest
class ImpExImportTest extends ServicelayerTest {
    
    @Autowired
    private ImportService importService;
    
    @Autowired
    private ProductService productService;
    
    @Test
    void testProductImport() throws Exception {
        // Given
        final String impex = 
            "INSERT_UPDATE Product;code[unique=true];name\n" +
            ";TEST001;Test Product\n";
        
        // When
        importImpEx(impex);
        
        // Then
        final ProductModel product = productService.getProductForCode("TEST001");
        assertNotNull(product);
        assertEquals("Test Product", product.getName());
    }
    
    private void importImpEx(String impex) throws ImpExException {
        final ImportConfig config = new ImportConfig();
        config.setScript(new StreamBasedImpExResource(
            new ByteArrayInputStream(impex.getBytes()), "UTF-8"));
        
        final ImportResult result = importService.importData(config);
        assertFalse(result.isError(), "Import failed: " + result.getUnresolvedLines());
    }
}
```

## Resources

- **HAC ImpEx Console**: http://localhost:9001/hac -> Console -> ImpEx Import
- **ImpEx Syntax**: Help Portal -> ImpEx
- **Sample ImpEx**: Platform extensions -> resources/impex
