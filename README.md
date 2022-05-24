# Protected prices example

Example of setup where 2 sources are defined CMS and PIM. PIM products have prices defined per customer (for B2B). Desired outcome is to display product prices per customer, without exposing prices for other customers.

# Sources
## CMS source
Created source named "cms-example"

### Source entities

```json
{
	"sourceId": "gid://Source/79c32351-34c8-4935-be52-390fb5a3e43e",
	"id": "gid://Source/79c32351-34c8-4935-be52-390fb5a3e43e/Entity/cms-id-1",
	"type": "productPage",
	"originId": "cms-id-1",
	"originParentId": "",
	"url": "/products/coca-cola",
	"redirects": [],
	"properties": {
		"name": "Coca-Cola",
		"description": "We are here to refresh the world and make a difference. Learn more about the Coca-Cola Company, our brands, and how we strive to do business the right way.",
		"sku": "12345678",
        // TODO: you can include some of non-protected PIM product information too!
	}
}
```

## PIM source
Created source named "pim-example"

### Source entities
```json
{
	"sourceId": "gid://Source/ee3a84d5-9163-4009-b5ab-c48cc024a1b0",
	"id": "gid://Source/ee3a84d5-9163-4009-b5ab-c48cc024a1b0/Entity/pim-id-1",
	"type": "product",
	"originId": "pim-id-1",
	"originParentId": null,
	"url": null,
	"redirects": [],
	"properties": {
		"sku": "12345678",
		"inStock": true,
		"prices": {
			"default": 20,
			"customer1": 10,
			"customer2": 15
		}
	}
}
```

# Environments

Concept is to push sensitive information only to 'protected' environment. So we create 2 environments - `Production` and `Protected production`.

Each of them has their own environment client. `Production` environment client is something that can be publicly available if needed, but `Protected production` environmenet client key must be kept as a secret.

# Mapping schemas

## (CMS) Product page
```json
{
	"sourceEntityTypes": [
		"productPage"
	],
	"route": {
		"url": "{url}"
	},
	"properties": {
		"name": "{p.name}",
		"description": "{p.description}",
		"sku": "{p.sku}",
        // OPTIONAL. I like to have it here to act as a contract and being visible to all parties.
		"price": "<SET-PER-CUSTOMER>"
	}
}
```

Deploy this schema only to `Production` environment and for `CMS` source (or also to `PIM` source if you have `PIM` enriched mapping).

## (PIM) Product prices
```json
{
	"sourceEntityTypes": [
		"product"
	],
	"route": {
		"handles": [
			"products/{p.sku}/prices"
		]
	},
	"properties": {
        "*": "p.prices"
	}
}
```

Deploy this schema only to `Protected production` environment and only for `PIM` source.

# Serving right price to the end user

Enterspeed is not responsible for figuring out who is current end user and what company he belongs to, it is part that you have the knowledge for your specific use case and business.

For this example, we are going to setup separate layer in front of Enterspeeds delivery API, that will be used to retrieve products information with proper enrichment based on the callers company.

Current simplified snippet is **extreme happy flow**, that doesn't include actual JWT validation rules or company retrieval approach for caller, since it is very specific per implementation and requirements.

```csharp
[FunctionName("products")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = null)] HttpRequest req)
{
    // Validate JWT and try to retrieve callers company identifier
    var customerId = string.Empty;
    if (req.TryValidateJwt(out var User))
    {
        customerId = User.CustomerId;
    }

    var url = req.Query["url"].ToString();

    // Arrange dependencies
    var configuration = new EnterspeedDeliveryConfiguration();
    var configurationProvider = new InMemoryConfigurationProvider(configuration);
    var deliveryConnection = new EnterspeedDeliveryConnection(configurationProvider);
    var serializer = new SystemTextJsonSerializer();
    var deliveryService = new EnterspeedDeliveryService(deliveryConnection, configurationProvider, serializer);

    // Retrieve product info (CMS), use public key
    var productResponse = await deliveryService.Fetch("PUBLIC-ENV-CLIENT-KEY", (b) => b.WithUrl(url));

    // Retrieve product prices (PIM), use protected key
    var pimProductResponse = await deliveryService.Fetch("PROTECTED-ENV-CLIENT-KEY", (b) => b.WithHandle($"products/{productResponse.Response.Route["sku"]}/prices"));
    var productPrices = (Dictionary<string, object>)pimProductResponse.Response.Views["productPrices"];
    // Map current customers price, or fallback to 'default' price
    productResponse.Response.Route["price"] = productPrices.ContainsKey(customerId) ? productPrices[customerId] : productPrices["default"];
    return new OkObjectResult(productResponse);
}
```

## Example of CURL request
Caller corresponds to `Customer1`
```bash
curl -H 'Accept: application/json' -H "Authorization: Bearer ${JWT_TOKEN}" https://{custom-layer-hostname}/api/products?url=/products/coca-cola
```

## Example of CURL response
```json
{
    "statusCode": 200,
    "message": null,
    "response": {
        "meta": {
            "status": 200,
            "redirect": null
        },
        "route": {
            "name": "Coca-Cola",
            "description": "We are here to refresh the world and make a difference. Learn more about the Coca-Cola Company, our brands, and how we strive to do business the right way.",
            "sku": "12345678",
            "price": "10" // Price for current callers customer
        },
        "views": {}
    }
}
```