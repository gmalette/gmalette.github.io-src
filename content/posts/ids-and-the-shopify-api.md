---
title: IDs, and the Shopify API
description: Shopify uses different schemes for ID encoding, and it's annoying.
categories: ["GraphQL", "Ruby"]
date: 2021-06-04T09:42:45-04:00
draft: true
---

I recently started working on a new project, in which we want to sell tickets to events. The normal stuff. Each event has a finite number of seats. To purchase a ticket, the customers need to first have an account on our event-hosting website. Each customer may purchase at most one seat per event. 

For many (hopefully obvious), I was looking for a provider to handle the payments. After considering Shopify and Stripe, I settled for Shopify because they handle inventory limits, such that we couldn't oversell the event. I figured that my app would need to:
1. Create a `Product` for the event
2. Manage carts on our website, to ensure customers are logged in, and enforce purchase limits
3. Create `Checkout` once the customer manifests an intent to pay
4. Receive `Webhooks` notifications after the purchase, to subscribe the customer to the events

To perform these tasks, Shopify uses different APIs. I would need to use the Admin GraphQL API to create the `Product`, the Storefront GraphQL API to create the `Checkout`, and the Webhook API to receive the notifications.

All this sounds (and should be?) pretty trivial, but has caused me some headaches, so let's dive in! I'll show the problems I faced and the solutions I found.

## Step 1: Create a Product

The first step is to create a `Product` within Shopify. Since I have read the documentation for `Checkout`, I also know that we need the `ProductVariant` ID, because Checkouts are created using `ProductVariant`, not `Product`, so we can select it right away.

```graphql
mutation($input: ProductInput!) {
  productCreate(input: $input) {
    product {
      id
      variants(first: 1) {
        edges {
          node {
            id
          }
        }
      }
    }
    userErrors {
      field
      message
    }
  }
}
```

We can then execute the mutation and create the products and product variants.

```ruby
ShopifyAPI::GraphQL
  .client
  .query(
    CREATE_PRODUCT_QUERY,
    variables: {
      input: {
        title: "Ticket to Event",
        status: "ACTIVE",
        variants: [
          {
            requiresShipping: false,
            price: "10.00",
            inventoryQuantities: [
              {
                availableQuantity: 30,
                locationId: LOCATION_ID, 
              }
            ],
            inventoryManagement: "SHOPIFY",
            inventoryPolicy: "DENY",
          }
        ]
      }
    },
  )
  .to_h
```

From the returned value we can keep only the IDs.

```ruby
{ 
  "data" => {
    "productCreate" => {
      "product" => {
        "id" => "gid://shopify/Product/6822055674006",
        "variants" => {
          "edges" => [{
            "node" => {
              "id" => "gid://shopify/ProductVariant/39988171440278"
            }
          }]
        }
      },
      "userErrors" => []
    }
  }
}

# Keep the ids around
PRODUCT_ID = "gid://shopify/Product/6822055674006"
PRODUCT_VARIANT_ID = "gid://shopify/ProductVariant/39988171440278"
```

Before users can buy products, they also need to be published, so let's do just that. If the mutation is successful we don't need to select anything, really, so we can select only the errors.

```graphql
mutation($id: ID!) {
  publishablePublishToCurrentChannel(id: $id) {
    publishable {
      __typename
    }
    userErrors {
      field
      message
    }
  }
}
```

```ruby
ShopifyAPI::GraphQL
  .client
  .query(
    PUBLISH_PRODUCT_QUERY,
    variables: { id: "gid://shopify/Product/6822055674006" },
  )
  .to_h
```

The response is successful, great!

```ruby
{
  "data" => {
    "publishablePublishToCurrentChannel" => {
      "publishable" => {"__typename" => "Product"},
      "userErrors" => []
    }
  }
}
```

Now that we've successfully created and published a product and product variant, it's time to start selling it!

## Step 2: Create a Checkout

Now that we've created a product to represent our events, it's time to sell it using Shopify's Checkouts. In Shopify's APIs, Checkouts are exposed through the Storefront API. We can create the GraphQL mutation to create a Checkout:

```graphql
mutation CheckoutCreate($input: CheckoutCreateInput!) {
  checkoutCreate(input: $input) {
    checkout {
      id
    }
    checkoutUserErrors {
      code
      field
      message
    }
  }
}
```

We can then use it to create our first Checkout.

```ruby
ShopifyStorefrontAPI::GraphQL
  .client
  .query(
    CREATE_CHECKOUT_QUERY,
    variables: {
      input: {
        lineItems: [
          {
            quantity: 1,
            variantId: PRODUCT_VARIANT_ID,
          }
        ],
      },
    },
  )
  .to_h
```

```ruby
{
  "errors" => [
    {
      "message" => "Variable $input of type CheckoutCreateInput! 
          was provided invalid value for lineItems.0.variantId 
          (Invalid global id `gid://shopify/ProductVariant/39988171440278`)",
      "locations" => [{"line" => 1, "column" => 52}],
    }
  ]
}
```

Oh no! This query didn't work. What's more, apparently the `variantId` is not a valid. Well, that's definitely weird... what's that about?

## Step 2.1: Find what the problem is with the ID

We'll have to investigate and see why our IDs were not accepted. We can start by checking whether the Storefront API sees the product we created in Step 1.

```graphql
query { 
  products(first: 1) {
    edges {
      node {
        id
        title
      }
    }
  }
}
```

```ruby
ShopifyStorefrontAPI::GraphQL
  .client
  .query(PRODUCTS_QUERY)
  .to_h
```

The response is successful, and our product is returned, but... something looks amiss.

```ruby
{
  "data" => {
    "products" => {
      "edges" => [
        {
          "node" => {
            "id" => "Z2lkOi8vc2hvcGlmeS9Qcm9kdWN0LzY4MjIwNTU2NzQwMDY=",
            "title" => "Ticket to Event"
          }
        }
      ]
    }
  }
}
```

The ID looks nothing like the one from the Admin API.

## Step 2.2: Are those Base64?

At first glance, that ID looks like it's Base64-encoded; the last character (`=`) is a pretty good indicator of padding. If it is, maybe we can use it to our advantage?

```ruby
Base64.decode64("Z2lkOi8vc2hvcGlmeS9Qcm9kdWN0LzY4MjIwNTU2NzQwMDY=")
# => "gid://shopify/Product/6822055674006"
```

Sure enough, it's the exact same GlobalID, but this time it's Base64-encoded. 

We can retry creating the Checkout.

```ruby
ShopifyStorefrontAPI::GraphQL.client.query(
    CREATE_CHECKOUT_QUERY,
    variables: {
      input: {
        lineItems: [
          {
            quantity: 1,
            # Do we only need to base64-encode the ID?
            variantId: Base64.encode64(PRODUCT_VARIANT_ID).strip, 
          }
        ],
      },
    },
  )
  .to_h
```

Sweet! It works!

```ruby
{
  "data" => {
    "checkoutCreate" => {
      "checkout" => {
        "id" => "Z2lkOi8vc2hvcGlmeS9DaGVja291dC81ZGIzMDRiNzNjZTQ3YjIyZDE1Y2QwOTM1ZDBlNDgxZD9rZXk9ZWZmODg4Zjc5YmZhYjVlOGVmMWZmMDliNTllMjBmYjI=",
      }, 
      "checkoutUserErrors" => []
    }
  }
}
```

We'll need to keep the Checkout ID around, to create a registration once it's successfully paid.

```ruby
CHECKOUT_ID = "Z2lkOi8vc2hvcGlmeS9DaGVja291dC81ZGIzMDRiNzNjZTQ3YjIyZDE1Y2QwOTM1ZDBlNDgxZD9rZXk9ZWZmODg4Zjc5YmZhYjVlOGVmMWZmMDliNTllMjBmYjI="
```

It looks like the Checkout ID is longer than the other Base64 IDs we had previously, I wonder what it holds.

```ruby
Base64.decode64(CHECKOUT_ID)
=> "gid://shopify/Checkout/5db304b73ce47b22d15cd0935d0e481d?key=eff888f79bfab5e8ef1ff09b59e20fb2"
```

Interestingly, the identifier is not numeric this time around. It also has a `key` parameter. 

## Step 3: Webhooks

I'll spare you the details of setting up and receiving webhooks, this topic has been covered in many other places. We'll skip to the part where the webhooks are fully setup, and we've just finished paying for the checkout created in Step 2. Our Webhook endpoint gets called with a payload with this information (fields irrelevant to this post have been omitted):

```json
{
    "id": 3711321832,
    "checkout_id": 1231231902380,
    "checkout_token": "5db304b73ce47b22d15cd0935d0e481d",
    "financial_status": "paid",
    "line_items": [{
      "id": 102938901203,
      "product_id": 6822055674006,
      "quantity": 1,
      "variant_id": 39988171440278
    }]
}
```

The `product_id` and `variant_id` are no longer GID nor Base64-encoded, but they are the same integer values as previously. However, the `checkout_id` doesn't have the same identifier we had previously seen. Instead, it's the `checkout_token` which corresponds to the `ID` returned by the Storefront GraphQL API, minus the `key` parameter.

# Recap

In the previous section, we found out interesting things. 
- For both `Product` and `ProductVariant`, the `ID` returned by all APIs (Admin, Storefront, Webhooks), all have a stable numeric identifier, and only differ in how they're encoded/represented. 
- For the `Checkout`, the Storefront API encodes more data (the `key` parameter) than the Webhook API, but the identifier is stable although under a different name.

Now would be a good time to look at the [documentation](https://shopify.dev/api/admin-graphql/2021-10/objects/ProductVariant#fields-ProductVariant-OBJECT) to see if I missed something. In doing so, I noticed the `ProductVariant.storefrontID` field, which could've been used to create the Checkout. However, I haven't been able to find a value to match the `Checkout` in the Webhook event payload.

# Persistence


