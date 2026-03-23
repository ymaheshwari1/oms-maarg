We have added support to create/update the productCategory record in the ProductAttribute entity. This is curently only supported for the Product Store Setting "Product Category Attribute". When this setting is enabled, the product category will be created/updated as an attribute of the product. This allows for easier integration with external systems that may require the product category to be an attribute of the product.

```xml
<Enumeration enumId="PROD_CAT_ATTR" enumName="Product Category Attribute" enumTypeId="PROD_STR_STNG"/>

<ProductStoreSetting fromDate="2026-03-03 05:32:24.482" productStoreId="STORE" settingTypeEnumId="PROD_CAT_ATTR" settingValue="Y"/>
```