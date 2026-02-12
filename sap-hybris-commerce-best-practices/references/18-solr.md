# Solr Integration

## Purpose
Configure and use Apache Solr for search functionality in SAP Commerce, including indexing, faceted search, and search optimization.

## Configuration

### 1. Solr Server Configuration

```properties
# local.properties
solr.config.urls=http://localhost:8983/solr
solrserver.instances.default.autostart=true
solrserver.instances.default.port=8983
solrserver.instances.default.memory=512m
```

### 2. Indexed Type Configuration

```xml
<!-- solrconfig.xml -->
<solrIndexedType name="Product">
    <indexedProperties>
        <indexedProperty name="code" type="string"/>
        <indexedProperty name="name" type="text"/>
        <indexedProperty name="description" type="text"/>
        <indexedProperty name="price" type="double"/>
        <indexedProperty name="categories" type="string" multiValue="true"/>
    </indexedProperties>
    
    <facets>
        <facet name="categoryFacet" priority="100">
            <indexedProperty>categories</indexedProperty>
        </facet>
        <facet name="priceFacet" priority="50">
            <indexedProperty>price</indexedProperty>
        </facet>
    </facets>
</solrIndexedType>
```

## Indexing

### 1. Full Index

```java
@Service
public class SolrIndexService {
    
    private final SolrIndexerService solrIndexerService;
    
    public void performFullIndex() {
        solrIndexerService.performFullIndex();
    }
}
```

### 2. Incremental Index

```java
public void performIncrementalIndex() {
    solrIndexerService.performIncrementalIndex();
}
```

## Search

### 1. Basic Search

```java
@Service
public class ProductSearchService {
    
    private final SolrSearchService solrSearchService;
    
    public SearchResult<ProductModel> searchProducts(String query) {
        SearchQuery searchQuery = new SearchQuery();
        searchQuery.setQuery(query);
        
        return solrSearchService.search(searchQuery);
    }
}
```

### 2. Faceted Search

```java
public SearchResult<ProductModel> searchWithFacets(String query) {
    SearchQuery searchQuery = new SearchQuery();
    searchQuery.setQuery(query);
    searchQuery.addFacet("categoryFacet");
    searchQuery.addFacet("priceFacet");
    
    return solrSearchService.search(searchQuery);
}
```

## Best Practices

### DO
- Index regularly
- Use facets for filtering
- Optimize field types
- Monitor Solr performance
- Use incremental indexing
- Configure proper memory
- Use Solr Cloud for production
- Test search relevance
- Implement search analytics
- Cache search results

### DON'T
- Use embedded Solr in production
- Skip index optimization
- Index unnecessary fields
- Ignore search performance
- Use full index frequently
- Skip Solr monitoring
- Hardcode Solr URLs
- Ignore search errors
- Skip facet configuration
- Use default Solr settings

## Commands

```bash
# Start Solr
ant startSolrServer

# Stop Solr
ant stopSolrServer

# Full index
ant solrindex

# Incremental index
ant solrindex -Dincremental=true
```

## Resources

- **Solr Documentation**: solr.apache.org
- **SAP Commerce Solr**: Help Portal -> Solr Integration
