# Mastering Sling Models in AEM
### A Senior Architect's Definitive Technical Reference

---

## Table of Contents

1. [Core Annotations — The Complete Hierarchy](#1-core-annotations)
2. [Injection Strategy — Field vs. Constructor](#2-injection-strategy)
3. [The Adaptation Process — ModelFactory & Adaptables](#3-the-adaptation-process)
4. [Logic & Lifecycle — @PostConstruct, Optional & Defaults](#4-logic--lifecycle)
5. [Sling Model Exporters — Jackson & Headless JSON](#5-sling-model-exporters)
6. [Performance Patterns — Caching](#6-performance-patterns)
7. [Advanced Mapping — @Via, ResourceSuperType & Collections](#7-advanced-mapping)
8. [Gold Standard Code Example](#8-gold-standard-code-example)
9. [Troubleshooting Reference Table](#9-troubleshooting-reference-table)

---

## 1. Core Annotations

Sling Model annotations exist in a clear precedence hierarchy. Understanding which injector wins is critical to avoiding silent injection failures.

### 1.1 `@Model` — The Declaration Gate

`@Model` is the entry point. It registers a POJO with the Sling `ModelAdapterFactory` and defines the contract for how the model can be constructed.

```java
@Model(
    adaptables  = SlingHttpServletRequest.class,
    adapters    = { ProductModel.class, ComponentExporter.class },
    resourceType = "mysite/components/product",
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class ProductModelImpl implements ProductModel { ... }
```

**Key parameters:**

| Parameter | Purpose | AEMaaCS Best Practice |
|---|---|---|
| `adaptables` | The source object (Request or Resource) | Prefer `SlingHttpServletRequest` when page context is needed |
| `adapters` | The interface(s) the model is exposed as | Always declare the interface, never the implementation class |
| `resourceType` | Scopes the model to a Sling resource type | Mandatory for component models |
| `defaultInjectionStrategy` | `REQUIRED` (throws on null) or `OPTIONAL` | Use `OPTIONAL` at class level |

---

### 1.2 The Injector Hierarchy

```
Priority (High to Low)
1. @Self          — The adaptable object itself
2. @ScriptVariable— HTL/JSP script bindings
3. @ValueMapValue — Resource property from ValueMap
4. @ChildResource — A named child Resource node
5. @RequestAttribute — From the SlingHttpServletRequest
6. @ResourcePath  — A Resource resolved from a path prop
7. @OSGiService   — An OSGi service reference
8. @SlingObject   — Sling objects (ResourceResolver, etc.)
9. @Inject        — Generic fallback (tries all injectors)
```

> Always use the specific injector annotation. `@Inject` forces Sling to iterate all injectors at runtime.

---

### 1.3 Annotation Reference Card

#### `@ValueMapValue`

```java
@ValueMapValue(name = "productTitle", injectionStrategy = InjectionStrategy.OPTIONAL)
private String productTitle;

@ValueMapValue
@Default(values = "Default Product Name")
private String displayName;
```

#### `@ScriptVariable`

```java
@ScriptVariable
private Page currentPage;

@ScriptVariable(injectionStrategy = InjectionStrategy.OPTIONAL)
private Style currentStyle;
```

#### `@SlingObject`

```java
@SlingObject
private ResourceResolver resourceResolver;

@SlingObject
private Resource resource;
```

#### `@Self`

```java
@Self
private SlingHttpServletRequest request;

@Self
private LinkManager linkManager;
```

#### `@OSGiService`

```java
@OSGiService
private UrlExternalizerService urlExternalizer;

@OSGiService(injectionStrategy = InjectionStrategy.OPTIONAL)
private XFAService xfaService;
```

---

## 2. Injection Strategy

### 2.1 Field Injection (Legacy — Avoid)

```java
@Model(adaptables = SlingHttpServletRequest.class, ...)
public class FieldInjectedModel {
    @ValueMapValue
    private String title;

    @OSGiService
    private UrlExternalizerService urlExternalizer;
}
```

Problems: fields cannot be `final`, hard to unit test, null injections are invisible until NPE.

### 2.2 Constructor Injection (Gold Standard)

```java
@Model(adaptables = SlingHttpServletRequest.class, adapters = ProductModel.class,
       resourceType = "mysite/components/product")
public class ProductModelImpl implements ProductModel {

    private final String title;
    private final String description;
    private final UrlExternalizerService urlExternalizer;
    private final Resource resource;

    @Inject
    public ProductModelImpl(
            @ValueMapValue(name = "jcr:title") String title,
            @ValueMapValue(name = "jcr:description", injectionStrategy = InjectionStrategy.OPTIONAL) String description,
            @OSGiService UrlExternalizerService urlExternalizer,
            @SlingObject Resource resource) {
        this.title = title;
        this.description = description != null ? description : "No description available";
        this.urlExternalizer = urlExternalizer;
        this.resource = resource;
    }
}
```

**Unit test — zero container required:**

```java
@Test
void testGetTitle() {
    UrlExternalizerService mockService = mock(UrlExternalizerService.class);
    Resource mockResource = mock(Resource.class);
    ProductModelImpl model = new ProductModelImpl("My Product", "A great product", mockService, mockResource);
    assertEquals("My Product", model.getTitle());
}
```

| Aspect | Field Injection | Constructor Injection |
|---|---|---|
| Field immutability | Cannot use `final` | All fields are `final` |
| Unit testability | Requires Mockito reflection or full context | Plain `new MyModel(mock1, mock2)` |
| Null visibility | Silent until NPE | Fails fast at model construction |
| Readability | Dependencies scattered | Full dependency graph in constructor |

---

## 3. The Adaptation Process

### 3.1 How `ModelAdapterFactory` Works

```
SlingHttpServletRequest.adaptTo(MyModel.class)
        |
        v
ModelAdapterFactory (OSGi Service)
        |
        |-- 1. Finds @Model class registered for MyModel.class interface
        |-- 2. Validates adaptable matches declared adaptables[]
        |-- 3. Runs all injectors to populate fields/constructor params
        |-- 4. Calls @PostConstruct method(s)
        |-- 5. Returns the fully initialized model instance
```

**Programmatic adaptation via `ModelFactory`:**

```java
@OSGiService
private ModelFactory modelFactory;

public ProductTeaserModel getRelatedProduct() {
    Resource productResource = resourceResolver.getResource("/content/products/my-product");
    if (productResource != null
            && modelFactory.isModelAvailableForAdaptable(productResource, ProductTeaserModel.class)) {
        return modelFactory.getModelFromWrappedRequest(request, productResource, ProductTeaserModel.class);
    }
    return null;
}
```

### 3.2 Request vs. Resource — Choosing the Right Adaptable

**`adaptables = Resource.class`** — use when:
- The model is purely data-driven (reads JCR properties)
- Must be used in background threads or scheduled jobs
- Building a reusable data layer / API model

Cannot inject: `@ScriptVariable`, `SlingHttpServletRequest`, WCM Mode, Style System.

**`adaptables = SlingHttpServletRequest.class`** — use when:
- Model needs HTL script variables (currentPage, etc.)
- Model needs WCM Mode awareness
- Model uses Style System
- Model implements ComponentExporter

**Adapting both:**

```java
@Model(adaptables = { SlingHttpServletRequest.class, Resource.class }, adapters = MyDataModel.class)
public class MyDataModelImpl implements MyDataModel {

    @ValueMapValue(name = "jcr:title")
    private String title;

    @ScriptVariable(injectionStrategy = InjectionStrategy.OPTIONAL)
    private Page currentPage;

    @SlingObject
    private Resource resource;
}
```

---

## 4. Logic & Lifecycle

### 4.1 `@PostConstruct` — The Initialization Hook

```java
@Model(adaptables = SlingHttpServletRequest.class, adapters = NavigationModel.class)
public class NavigationModelImpl implements NavigationModel {

    @ValueMapValue(injectionStrategy = InjectionStrategy.OPTIONAL)
    private String rootPath;

    @SlingObject
    private ResourceResolver resourceResolver;

    @OSGiService
    private UrlExternalizerService urlExternalizer;

    private List<NavigationItem> navigationItems;
    private String externalizedRootUrl;

    @PostConstruct
    private void init() {
        if (StringUtils.isBlank(rootPath)) {
            navigationItems = Collections.emptyList();
            return;
        }
        Resource rootResource = resourceResolver.getResource(rootPath);
        if (rootResource == null) {
            navigationItems = Collections.emptyList();
            return;
        }
        navigationItems = buildNavigationTree(rootResource);
        externalizedRootUrl = urlExternalizer.externalLink(null, UrlExternalizerService.PUBLISH, rootPath);
    }
}
```

**Key rules:**
1. Must be `private` or `protected`.
2. Must return `void`.
3. Use only one — execution order of multiple is not guaranteed.
4. Unchecked exceptions cause `ModelInstantiationException`.

### 4.2 Optional vs. Default Values

```java
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class ArticleModelImpl {

    @ValueMapValue
    private Optional<String> subtitle;

    @ValueMapValue
    @Default(values = "Read More")
    private String ctaLabel;

    @ValueMapValue
    @Default(booleanValues = false)
    private boolean openInNewTab;

    @ValueMapValue
    @Default(intValues = 4)
    private int maxItems;

    @PostConstruct
    private void init() {
        String subtitleValue = subtitle.orElse("Default Subtitle");
    }
}
```

---

## 5. Sling Model Exporters

### 5.1 The Jackson Exporter Architecture

```
GET /content/mysite/page/jcr:content/root/product.model.json
         |
         v
Sling resolves selector "model" + extension "json"
         |
         v
ModelExporterServlet
         |
         |-- Looks up @Exporter annotation on the model class
         |-- Delegates to JacksonExporter (name = "jackson")
         |-- Jackson serializes all getter methods (JavaBean convention)
         |-- Returns: Content-Type: application/json
```

### 5.2 The `@Exporter` Annotation

```java
@Model(
    adaptables  = SlingHttpServletRequest.class,
    adapters    = { ProductModel.class, ComponentExporter.class },
    resourceType = "mysite/components/product"
)
@Exporter(
    name       = ExporterConstants.SLING_MODEL_EXPORTER_NAME,
    extensions = ExporterConstants.SLING_MODEL_EXTENSION,
    selector   = ExporterConstants.SLING_MODEL_SELECTOR
)
public class ProductModelImpl implements ProductModel, ComponentExporter { ... }
```

### 5.3 Jackson Serialization Control

```java
public String getProductId() { return productId; }

@JsonIgnore
public String getInternalSku() { return internalSku; }

@JsonProperty("product_display_name")
public String getTitle() { return title; }

@JsonInclude(JsonInclude.Include.NON_NULL)
public String getSubtitle() { return subtitle; }

@Override
@NotNull
public String getExportedType() {
    return "mysite/components/product";
}
```

**Resulting JSON:**

```json
{
  ":type": "mysite/components/product",
  "productId": "PRD-001",
  "product_display_name": "Premium Widget",
  "price": { "amount": 49.99, "currency": "USD" }
}
```

### 5.4 Container Exporter for SPA Page Models

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters = { MyContainer.class, ContainerExporter.class })
@Exporter(name = "jackson", extensions = "json")
public class MyContainerImpl implements MyContainer, ContainerExporter {

    @OSGiService
    private ModelFactory modelFactory;

    @SlingObject
    private Resource resource;

    @Self
    private SlingHttpServletRequest request;

    private Map<String, ComponentExporter> children;

    @PostConstruct
    private void init() {
        children = new LinkedHashMap<>();
        for (Resource child : resource.getChildren()) {
            ComponentExporter exporter = modelFactory.getModelFromWrappedRequest(
                request, child, ComponentExporter.class);
            if (exporter != null) {
                children.put(child.getName(), exporter);
            }
        }
    }

    @Override @NotNull
    public Map<String, ComponentExporter> getExportedChildren() { return children; }

    @Override @NotNull
    public String[] getExportedItemsOrder() {
        return children.keySet().toArray(new String[0]);
    }

    @Override @NotNull
    public String getExportedType() { return "mysite/components/mycontainer"; }
}
```

---

## 6. Performance Patterns

### 6.1 Model Caching — `cache = true`

```java
@Model(
    adaptables = SlingHttpServletRequest.class,
    adapters = SiteConfigurationModel.class,
    cache = true
)
public class SiteConfigurationModelImpl implements SiteConfigurationModel {

    @OSGiService
    private SiteConfigurationService siteConfigService;

    @ScriptVariable
    private Page currentPage;

    private String cdnBaseUrl;
    private String analyticsId;
    private Locale siteLocale;

    @PostConstruct
    private void init() {
        SiteConfiguration config = siteConfigService.getConfigForPage(currentPage);
        cdnBaseUrl  = config.getCdnBaseUrl();
        analyticsId = config.getAnalyticsId();
        siteLocale  = config.getLocale();
    }

    @Override public String getCdnBaseUrl() { return cdnBaseUrl; }
    @Override public String getAnalyticsId() { return analyticsId; }
    @Override public Locale getSiteLocale() { return siteLocale; }
}
```

**Cache scope:**
- Resource-adapted: cached per Resource object
- Request-adapted: cached per SlingHttpServletRequest
- Cache cleared at the end of the request lifecycle

**Always use `cache = true` for:**
- Configuration models read from Context-Aware Configurations
- Models that make JCR queries in `@PostConstruct`
- Models that call external services (DAM, Commerce APIs)
- Models injected into HTL templates used in loops

---

## 7. Advanced Mapping

### 7.1 `@Via` — Redirecting the Injection Source

```java
@Model(adaptables = SlingHttpServletRequest.class, adapters = ProductModel.class)
public class ProductModelImpl implements ProductModel {

    @ValueMapValue
    @Via(type = ChildResource.class, value = "pricing")
    private String price;

    @Self
    @Via(type = ResourceSuperType.class)
    private BaseProductModel baseModel;

    @ValueMapValue
    @Via("jcr:content")
    private String pageTitle;
}
```

### 7.2 Handling Collections of Child Resources

```java
@Model(adaptables = SlingHttpServletRequest.class, ...)
public class CarouselModelImpl implements CarouselModel {

    @ChildResource
    private List<CarouselItemModel> items;

    @SlingObject
    private Resource resource;

    private List<CarouselItemModel> carouselItems;

    @PostConstruct
    private void init() {
        carouselItems = StreamSupport
            .stream(resource.getChildren().spliterator(), false)
            .filter(child -> !child.getName().startsWith("jcr:"))
            .map(child -> child.adaptTo(CarouselItemModel.class))
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
}
```

### 7.3 Component Versioning with `@Via(type = ResourceSuperType.class)`

```java
// v1 model
@Model(adaptables = SlingHttpServletRequest.class, adapters = TitleModel.class,
       resourceType = "mysite/components/title/v1/title")
public class TitleModelV1 implements TitleModel {
    @ValueMapValue(name = "jcr:title")
    protected String title;

    @Override public String getTitle() { return title; }
    @Override public String getType()  { return "h2"; }
}

// v2 model — delegates to v1 via ResourceSuperType
@Model(adaptables = SlingHttpServletRequest.class, adapters = TitleModel.class,
       resourceType = "mysite/components/title/v2/title")
@Exporter(name = "jackson", extensions = "json")
public class TitleModelV2 implements TitleModel, ComponentExporter {

    @Self
    @Via(type = ResourceSuperType.class)
    private TitleModel v1Delegate;

    @ValueMapValue(injectionStrategy = InjectionStrategy.OPTIONAL)
    private String titleLink;

    @Override public String getTitle() { return v1Delegate.getTitle(); }
    @Override public String getType()  { return v1Delegate.getType(); }
    public String getTitleLink()       { return titleLink; }

    @Override public String getExportedType() { return "mysite/components/title/v2/title"; }
}
```

---

## 8. Gold Standard Code Example

### ArticleCardModel.java (Interface)

```java
package com.mysite.core.models;

public interface ArticleCardModel {
    String getTitle();
    String getDescription();
    String getExternalArticleUrl();
    String getResolvedImageUrl();
    String getCtaLabel();
    boolean isOpenInNewTab();
    String getAuthorName();
    String getFormattedDate();
    boolean isValid();
}
```

### ArticleCardModelImpl.java (Implementation)

```java
package com.mysite.core.models;

import java.util.*;
import javax.annotation.PostConstruct;
import org.apache.commons.lang3.StringUtils;
import org.apache.sling.api.SlingHttpServletRequest;
import org.apache.sling.api.resource.Resource;
import org.apache.sling.api.resource.ResourceResolver;
import org.apache.sling.models.annotations.*;
import org.apache.sling.models.annotations.injectorspecific.*;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.adobe.cq.export.json.ComponentExporter;
import com.adobe.cq.export.json.ExporterConstants;
import com.day.cq.wcm.api.Page;
import com.fasterxml.jackson.annotation.JsonIgnore;
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonProperty;
import com.mysite.core.services.UrlExternalizerService;
import com.mysite.core.services.ImageResolverService;

@Model(
    adaptables  = SlingHttpServletRequest.class,
    adapters    = { ArticleCardModel.class, ComponentExporter.class },
    resourceType = ArticleCardModelImpl.RESOURCE_TYPE,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL,
    cache = true
)
@Exporter(
    name       = ExporterConstants.SLING_MODEL_EXPORTER_NAME,
    extensions = ExporterConstants.SLING_MODEL_EXTENSION
)
public class ArticleCardModelImpl implements ArticleCardModel, ComponentExporter {

    private static final Logger LOG = LoggerFactory.getLogger(ArticleCardModelImpl.class);
    static final String RESOURCE_TYPE = "mysite/components/content/article-card";

    // --- INJECTED FIELDS ---

    @ValueMapValue(name = "jcr:title")
    private String title;

    @ValueMapValue(name = "jcr:description", injectionStrategy = InjectionStrategy.OPTIONAL)
    private String description;

    @ValueMapValue(injectionStrategy = InjectionStrategy.OPTIONAL)
    private String articlePath;

    @ValueMapValue(injectionStrategy = InjectionStrategy.OPTIONAL)
    private String imageReference;

    @ValueMapValue
    @Default(values = "Read Article")
    private String ctaLabel;

    @ValueMapValue
    @Default(booleanValues = false)
    private boolean openInNewTab;

    @ScriptVariable
    private Page currentPage;

    @SlingObject
    private ResourceResolver resourceResolver;

    @SlingObject
    private Resource resource;

    @OSGiService
    private UrlExternalizerService urlExternalizerService;

    @OSGiService(injectionStrategy = InjectionStrategy.OPTIONAL)
    private ImageResolverService imageResolverService;

    // --- COMPUTED FIELDS ---

    private String externalArticleUrl;
    private String resolvedImageUrl;
    private String authorName;
    private String formattedDate;
    private boolean valid;

    // --- LIFECYCLE ---

    @PostConstruct
    private void init() {
        LOG.debug("ArticleCardModel initializing for resource: {}", resource.getPath());

        if (StringUtils.isBlank(title)) {
            LOG.warn("ArticleCardModel at '{}' has no title.", resource.getPath());
            valid = false;
            return;
        }
        valid = true;

        if (StringUtils.isNotBlank(articlePath)) {
            try {
                externalArticleUrl = urlExternalizerService.externalizeUrl(resourceResolver, articlePath);
            } catch (Exception e) {
                LOG.error("Failed to externalize URL for path '{}': {}", articlePath, e.getMessage());
                externalArticleUrl = articlePath;
            }
        }

        if (imageResolverService != null && StringUtils.isNotBlank(imageReference)) {
            resolvedImageUrl = imageResolverService.resolveImageUrl(imageReference, "card-hero");
        } else if (StringUtils.isNotBlank(imageReference)) {
            resolvedImageUrl = imageReference;
        }

        if (StringUtils.isNotBlank(articlePath)) {
            Resource articleResource = resourceResolver.getResource(articlePath + "/jcr:content");
            if (articleResource != null) {
                authorName    = articleResource.getValueMap().get("articleAuthor", String.class);
                String rawDate = articleResource.getValueMap().get("publishDate", String.class);
                formattedDate = formatPublishDate(rawDate);
            }
        }
    }

    private String formatPublishDate(String rawDate) {
        if (StringUtils.isBlank(rawDate)) return null;
        try { return rawDate.substring(0, 10); } catch (Exception e) { return null; }
    }

    // --- PUBLIC METHODS ---

    @Override @Nullable public String getTitle() { return title; }

    @Override @Nullable @JsonInclude(JsonInclude.Include.NON_NULL)
    public String getDescription() { return description; }

    @Override @Nullable @JsonProperty("articleUrl")
    public String getExternalArticleUrl() { return externalArticleUrl; }

    @Override @Nullable @JsonInclude(JsonInclude.Include.NON_NULL)
    public String getResolvedImageUrl() { return resolvedImageUrl; }

    @Override public String getCtaLabel() { return ctaLabel; }
    @Override public boolean isOpenInNewTab() { return openInNewTab; }

    @Override @Nullable @JsonInclude(JsonInclude.Include.NON_NULL)
    public String getAuthorName() { return authorName; }

    @Override @Nullable @JsonInclude(JsonInclude.Include.NON_NULL)
    public String getFormattedDate() { return formattedDate; }

    @Override @JsonIgnore public boolean isValid() { return valid; }

    @Override @NotNull public String getExportedType() { return RESOURCE_TYPE; }
}
```

### ArticleCardModelImplTest.java (Unit Test)

```java
@ExtendWith(AemContextExtension.class)
class ArticleCardModelImplTest {

    private final AemContext ctx = new AemContext();

    @BeforeEach
    void setUp() {
        ctx.load().json("/content/article-card-test.json", "/content");
        ctx.addModelsForClasses(ArticleCardModelImpl.class);

        UrlExternalizerService mockExternalizer = mock(UrlExternalizerService.class);
        when(mockExternalizer.externalizeUrl(any(), anyString()))
            .thenAnswer(inv -> "https://www.mysite.com" + inv.getArgument(1) + ".html");
        ctx.registerService(UrlExternalizerService.class, mockExternalizer);
    }

    @Test
    void testGetTitle() {
        ctx.currentResource("/content/article-card-test");
        ArticleCardModel model = ctx.request().adaptTo(ArticleCardModel.class);
        assertNotNull(model);
        assertEquals("How AEM Cloud Changes Everything", model.getTitle());
    }

    @Test
    void testInvalidModelWithNoTitle() {
        ctx.currentResource("/content/article-card-notitle");
        ArticleCardModel model = ctx.request().adaptTo(ArticleCardModel.class);
        assertNotNull(model);
        assertFalse(model.isValid());
        assertNull(model.getExternalArticleUrl());
    }

    @Test
    void testExternalUrlIsComputed() {
        ctx.currentResource("/content/article-card-test");
        ArticleCardModel model = ctx.request().adaptTo(ArticleCardModel.class);
        assertNotNull(model);
        assertEquals("https://www.mysite.com/en/articles/my-article.html",
            model.getExternalArticleUrl());
    }
}
```

### Resulting JSON Output

```json
{
  ":type": "mysite/components/content/article-card",
  "title": "How AEM Cloud Changes Everything",
  "description": "A deep dive into AEMaaCS architecture",
  "articleUrl": "https://www.mysite.com/en/articles/my-article.html",
  "resolvedImageUrl": "/content/dam/mysite/hero-image.png",
  "ctaLabel": "Read Article",
  "openInNewTab": false,
  "authorName": "Jane Architect",
  "formattedDate": "2026-04-08"
}
```

---

## 9. Troubleshooting Reference Table

| Error / Symptom | Root Cause | Diagnostic Steps | Resolution |
|---|---|---|---|
| `Could not adapt to model` / `ModelInstantiationException` | Model not registered, wrong `adaptables`, or `@PostConstruct` threw | Check `error.log`. Enable DEBUG for `org.apache.sling.models` | Verify `@Model` on class, bundle Active in OSGi console, no unchecked exceptions in `@PostConstruct` |
| `NullPointerException` inside `@PostConstruct` | A `REQUIRED` injection silently failed | Add null checks. Check WARN logs for injection failures | Add `injectionStrategy = InjectionStrategy.OPTIONAL` and null-guard the field |
| `@ValueMapValue` returns `null` for existing property | Property name mismatch (Java field name != JCR property name) | Print `resource.getValueMap().keySet()` to logs | Use `@ValueMapValue(name = "exact-jcr-property-name")` |
| `@ScriptVariable` fails with `ModelInstantiationException` | Model adapted from `Resource` instead of `SlingHttpServletRequest` | Check call site: `resource.adaptTo()` vs `request.adaptTo()` | Add `injectionStrategy = InjectionStrategy.OPTIONAL` or change adaptable to Request |
| `@OSGiService` injects as `null` | Service bundle not active, or wrong service interface | Check OSGi console → Services | Fix bundle state, correct interface class, or use `InjectionStrategy.OPTIONAL` |
| JSON export returns `404` | `@Exporter` missing, or `ComponentExporter` not in `adapters[]` | Try `.model.json` suffix in browser | Add `@Exporter(name="jackson", extensions="json")` and `ComponentExporter.class` to adapters |
| `getExportedType()` wrong in SPA Page Model | Method not implemented or returns empty string | Inspect JSON — `:type` will be empty | Implement returning exact `resourceType` string |
| Model instantiated multiple times per request | `cache = true` missing | Add timing logs in `@PostConstruct` | Add `cache = true` to `@Model` annotation |
| `@ChildResource List<Model>` is empty | Children cannot be adapted (missing `@Model`, wrong adaptable) | Verify `sling:resourceType` on children matches item `@Model` resourceType | Register `@Model` for item class with `adaptables = Resource.class` |
| `@Self MyOtherModel` injects as `null` | Adaptable cannot adapt to `MyOtherModel` | Verify `MyOtherModel` is a registered Sling Model | Ensure `@Model` on the class and bundle is active |
| `@Via(type = ResourceSuperType.class)` returns `null` | No `sling:resourceSuperType` on component, or parent model not registered | Check CRXDE for `sling:resourceSuperType` property | Define `sling:resourceSuperType` pointing to correct parent type |
| `No injector for field 'X'` | Using `@Inject` and no injector satisfies the field type | Enable DEBUG on `org.apache.sling.models.impl` | Replace `@Inject` with correct specific injector |
| `IllegalArgumentException: adaptables must not be null` | Calling `adaptTo()` on a null object | Add null check before `adaptTo()` | `if (resource != null) { model = resource.adaptTo(...); }` |
| Stale data from model cache | `cache = true` on a model that mutates state post-`@PostConstruct` | Audit for mutable state modified outside `@PostConstruct` | Ensure model is immutable after `@PostConstruct`, or remove `cache = true` |

---

## Quick Reference — Decision Tree

```
Do you need request context (WCM Mode, Style System, HTL bindings)?
  YES -> adaptables = SlingHttpServletRequest.class
  NO  -> adaptables = Resource.class

Is this model referenced more than once per page?
  YES -> cache = true (mandatory)
  NO  -> cache = false (default)

Does this component need to expose JSON (headless / SPA)?
  YES -> Add @Exporter + implement ComponentExporter + add to adapters[]

Do you need initialization logic or computed properties?
  YES -> @PostConstruct private void init() { ... }

Which injector annotation to use?
  JCR property value   -> @ValueMapValue
  HTL binding variable -> @ScriptVariable
  Sling platform object-> @SlingObject
  OSGi service         -> @OSGiService
  Self / other model   -> @Self
  Child resource       -> @ChildResource
  NEVER                -> @Inject (too generic, performance cost)
```

---

*Generated on 2026-04-14 | AEM Cloud Service (AEMaaCS) | Sling Models API 1.5+*