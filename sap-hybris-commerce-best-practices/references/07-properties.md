# Properties Configuration

## Purpose
Externalize configuration using properties files for environment-specific settings, feature flags, and application parameters. Properties enable configuration without code changes.

## Key Concepts

### Property Files Hierarchy
```
1. project.properties (extension-level, build-time)
2. local.properties (instance-level, runtime)
3. advanced.properties (platform defaults)
4. Environment variables (highest priority)
```

### Property Resolution Order
```
Environment Variables -> local.properties -> project.properties -> advanced.properties
```

## Property Files

### 1. project.properties (Extension Level)

Located in extension root: `[extension]/project.properties`

```properties
# Extension metadata
extension.name=mycompanycore
extension.dependencies=core,commerceservices

# Build configuration
javac.source=21
javac.target=21
javac.debug=true

# Web application
webapp.main.contextroot=/mycompanycore

# Custom properties
mycompany.feature.enabled=true
mycompany.api.timeout=5000
mycompany.batch.size=100
```

### 2. local.properties (Instance Level)

Located in: `[HYBRIS_HOME]/config/local.properties`

```properties
# Database configuration
db.url=jdbc:mysql://localhost:3306/hybris
db.driver=com.mysql.jdbc.Driver
db.username=hybris
db.password=*****
db.tableprefix=

# Solr configuration
solr.config.urls=http://localhost:8983/solr

# Media storage
media.default.storage.strategy=localFileMediaStorageStrategy
media.folder.root=/opt/hybris/media

# Email configuration
mail.smtp.server=smtp.gmail.com
mail.smtp.port=587
mail.smtp.user=noreply@mycompany.com
mail.smtp.password=********
mail.use.tls=true

# Custom application properties
mycompany.api.url=https://api.mycompany.com
mycompany.api.key=secret-api-key
mycompany.feature.newCheckout=true
mycompany.cache.ttl=3600
```

### 3. Environment-Specific Properties

```properties
# Development
mycompany.environment=dev
mycompany.debug.enabled=true
mycompany.cache.enabled=false

# Staging
mycompany.environment=staging
mycompany.debug.enabled=false
mycompany.cache.enabled=true

# Production
mycompany.environment=prod
mycompany.debug.enabled=false
mycompany.cache.enabled=true
mycompany.monitoring.enabled=true
```

## Accessing Properties in Code

### 1. Using ConfigurationService

```java
package com.mycompany.core.service.impl;

import de.hybris.platform.servicelayer.config.ConfigurationService;
import org.springframework.stereotype.Service;

@Service
public class DefaultMyCompanyService implements MyCompanyService {
    
    private final ConfigurationService configurationService;
    
    public DefaultMyCompanyService(ConfigurationService configurationService) {
        this.configurationService = configurationService;
    }
    
    @Override
    public void performOperation() {
        // Get string property
        String apiUrl = configurationService.getConfiguration()
            .getString("mycompany.api.url");
        
        // Get string with default
        String apiKey = configurationService.getConfiguration()
            .getString("mycompany.api.key", "default-key");
        
        // Get boolean property
        boolean featureEnabled = configurationService.getConfiguration()
            .getBoolean("mycompany.feature.enabled", false);
        
        // Get integer property
        int timeout = configurationService.getConfiguration()
            .getInt("mycompany.api.timeout", 5000);
        
        // Get long property
        long cacheTtl = configurationService.getConfiguration()
            .getLong("mycompany.cache.ttl", 3600L);
        
        // Get double property
        double threshold = configurationService.getConfiguration()
            .getDouble("mycompany.threshold", 0.5);
        
        // Use properties
        LOG.info("API URL: {}", apiUrl);
        LOG.info("Feature enabled: {}", featureEnabled);
    }
}
```

### 2. Property Injection via Spring

```java
package com.mycompany.core.service.impl;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class DefaultApiService implements ApiService {
    
    private final String apiUrl;
    private final String apiKey;
    private final int timeout;
    private final boolean featureEnabled;
    
    // Constructor injection with @Value
    public DefaultApiService(
            @Value("${mycompany.api.url}") String apiUrl,
            @Value("${mycompany.api.key}") String apiKey,
            @Value("${mycompany.api.timeout:5000}") int timeout,
            @Value("${mycompany.feature.enabled:false}") boolean featureEnabled) {
        this.apiUrl = apiUrl;
        this.apiKey = apiKey;
        this.timeout = timeout;
        this.featureEnabled = featureEnabled;
    }
    
    @Override
    public void callApi() {
        if (!featureEnabled) {
            LOG.info("Feature is disabled");
            return;
        }
        
        // Use injected properties
        RestTemplate restTemplate = new RestTemplate();
        restTemplate.setRequestFactory(
            new HttpComponentsClientHttpRequestFactory() {{
                setConnectTimeout(timeout);
                setReadTimeout(timeout);
            }}
        );
        
        // Make API call
        String response = restTemplate.getForObject(apiUrl, String.class);
        LOG.info("API response: {}", response);
    }
}
```

### 3. Configuration Bean

```java
package com.mycompany.core.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MyCompanyConfig {
    
    @Value("${mycompany.api.url}")
    private String apiUrl;
    
    @Value("${mycompany.api.key}")
    private String apiKey;
    
    @Value("${mycompany.api.timeout:5000}")
    private int timeout;
    
    @Value("${mycompany.batch.size:100}")
    private int batchSize;
    
    @Value("${mycompany.feature.enabled:false}")
    private boolean featureEnabled;
    
    // Getters
    public String getApiUrl() {
        return apiUrl;
    }
    
    public String getApiKey() {
        return apiKey;
    }
    
    public int getTimeout() {
        return timeout;
    }
    
    public int getBatchSize() {
        return batchSize;
    }
    
    public boolean isFeatureEnabled() {
        return featureEnabled;
    }
}
```

## Property Patterns

### 1. Feature Flags

```properties
# Feature flags
mycompany.feature.newCheckout=true
mycompany.feature.recommendations=false
mycompany.feature.socialLogin=true
mycompany.feature.guestCheckout=true
```

```java
@Service
public class CheckoutService {
    
    private final ConfigurationService configurationService;
    
    public void processCheckout(CartModel cart) {
        boolean newCheckoutEnabled = configurationService.getConfiguration()
            .getBoolean("mycompany.feature.newCheckout", false);
        
        if (newCheckoutEnabled) {
            processNewCheckout(cart);
        } else {
            processLegacyCheckout(cart);
        }
    }
}
```

### 2. API Configuration

```properties
# External API configuration
mycompany.api.url=https://api.example.com
mycompany.api.key=secret-key
mycompany.api.timeout=5000
mycompany.api.retries=3
mycompany.api.retryDelay=1000
```

```java
@Service
public class ExternalApiService {
    
    private final String apiUrl;
    private final String apiKey;
    private final int timeout;
    private final int retries;
    private final int retryDelay;
    
    public ExternalApiService(ConfigurationService configurationService) {
        Configuration config = configurationService.getConfiguration();
        this.apiUrl = config.getString("mycompany.api.url");
        this.apiKey = config.getString("mycompany.api.key");
        this.timeout = config.getInt("mycompany.api.timeout", 5000);
        this.retries = config.getInt("mycompany.api.retries", 3);
        this.retryDelay = config.getInt("mycompany.api.retryDelay", 1000);
    }
}
```

### 3. Batch Processing Configuration

```properties
# Batch processing
mycompany.batch.size=100
mycompany.batch.threads=4
mycompany.batch.enabled=true
mycompany.batch.schedule=0 0 2 * * ?
```

```java
@Service
public class BatchProcessingService {
    
    private final int batchSize;
    private final int threads;
    
    public BatchProcessingService(ConfigurationService configurationService) {
        Configuration config = configurationService.getConfiguration();
        this.batchSize = config.getInt("mycompany.batch.size", 100);
        this.threads = config.getInt("mycompany.batch.threads", 4);
    }
    
    public void processBatch(List<ProductModel> products) {
        Lists.partition(products, batchSize).forEach(batch -> {
            // Process batch
        });
    }
}
```

### 4. Cache Configuration

```properties
# Cache settings
mycompany.cache.enabled=true
mycompany.cache.ttl=3600
mycompany.cache.maxSize=10000
mycompany.cache.evictionPolicy=LRU
```

### 5. Email Configuration

```properties
# Email settings
mycompany.email.from=noreply@mycompany.com
mycompany.email.replyTo=support@mycompany.com
mycompany.email.enabled=true
mycompany.email.async=true
```

## Environment Variables

### Override Properties with Environment Variables

```bash
# Set environment variable
export MYCOMPANY_API_URL=https://prod-api.mycompany.com
export MYCOMPANY_API_KEY=prod-secret-key

# Start Hybris
./hybrisserver.sh
```

### Access in Code

```java
@Service
public class EnvironmentAwareService {
    
    public void checkEnvironment() {
        // Environment variable takes precedence
        String apiUrl = System.getenv("MYCOMPANY_API_URL");
        
        if (apiUrl == null) {
            // Fallback to properties
            apiUrl = configurationService.getConfiguration()
                .getString("mycompany.api.url");
        }
        
        LOG.info("Using API URL: {}", apiUrl);
    }
}
```

## Property Validation

### Validate Required Properties

```java
@Component
public class PropertyValidator implements InitializingBean {
    
    private final ConfigurationService configurationService;
    
    public PropertyValidator(ConfigurationService configurationService) {
        this.configurationService = configurationService;
    }
    
    @Override
    public void afterPropertiesSet() throws Exception {
        Configuration config = configurationService.getConfiguration();
        
        // Validate required properties
        validateRequired(config, "mycompany.api.url");
        validateRequired(config, "mycompany.api.key");
        
        // Validate numeric ranges
        int timeout = config.getInt("mycompany.api.timeout", 5000);
        if (timeout < 1000 || timeout > 60000) {
            throw new IllegalStateException(
                "Invalid timeout value: " + timeout + " (must be 1000-60000)");
        }
        
        LOG.info("Property validation successful");
    }
    
    private void validateRequired(Configuration config, String key) {
        if (!config.containsKey(key)) {
            throw new IllegalStateException("Required property missing: " + key);
        }
    }
}
```

## Encrypted Properties

### Encrypt Sensitive Properties

```properties
# Encrypted password
db.password=ENC(encrypted-value-here)
mycompany.api.key=ENC(encrypted-api-key)
```

### Decrypt in Code

```java
@Service
public class SecureConfigService {
    
    private final ConfigurationService configurationService;
    private final EncryptionService encryptionService;
    
    public String getDecryptedProperty(String key) {
        String value = configurationService.getConfiguration().getString(key);
        
        if (value != null && value.startsWith("ENC(") && value.endsWith(")")) {
            String encrypted = value.substring(4, value.length() - 1);
            return encryptionService.decrypt(encrypted);
        }
        
        return value;
    }
}
```

## Best Practices

### DO
- Externalize all configuration to properties
- Use meaningful property names with prefixes
- Provide default values for optional properties
- Document properties in README or wiki
- Use environment variables for secrets
- Validate required properties at startup
- Use feature flags for gradual rollouts
- Version control property templates
- Use different properties per environment
- Encrypt sensitive values

### DON'T
- Hardcode configuration in code
- Commit sensitive data (passwords, API keys)
- Use generic property names
- Skip default values
- Mix concerns in property names
- Use properties for business logic
- Ignore missing required properties
- Use plain text for secrets
- Duplicate property definitions
- Change property names without migration

## Common Pitfalls

### 1. Missing Default Values

```java
// Wrong: No default value
int timeout = configurationService.getConfiguration()
    .getInt("mycompany.api.timeout"); // Throws exception if missing

// Correct: Provide default
int timeout = configurationService.getConfiguration()
    .getInt("mycompany.api.timeout", 5000);
```

### 2. Hardcoded Values

```java
// Wrong: Hardcoded URL
String apiUrl = "https://api.example.com";

// Correct: Use property
String apiUrl = configurationService.getConfiguration()
    .getString("mycompany.api.url");
```

### 3. Sensitive Data in Version Control

```properties
# Wrong: Plain text password in git
db.password=secretpassword123

# Correct: Use environment variable or encrypted value
db.password=${DB_PASSWORD}
# Or
db.password=ENC(encrypted-value)
```

## Testing with Properties

### Override Properties in Tests

```java
@IntegrationTest
class ServiceWithPropertiesTest extends ServicelayerTest {
    
    @Autowired
    private ConfigurationService configurationService;
    
    @Autowired
    private MyCompanyService myCompanyService;
    
    @Before
    public void setUp() {
        // Override property for test
        Config.setParameter("mycompany.feature.enabled", "true");
        Config.setParameter("mycompany.api.timeout", "1000");
    }
    
    @Test
    public void testWithCustomProperties() {
        // Test with overridden properties
        myCompanyService.performOperation();
        
        // Verify behavior
        assertTrue(myCompanyService.isFeatureEnabled());
    }
}
```

## Resources

- **Configuration Service**: Help Portal -> Configuration
- **Property Files**: Platform documentation
- **Environment Variables**: Deployment guide
