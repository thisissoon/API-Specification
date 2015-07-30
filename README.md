# SOON\_ REST API Specification

We write a lot of services that communicate with one another, it's important that we
have a standard source of truth for how we should write and consume our services via API's.
This document aims to provide that one source of truth.

## Credits

This document is inspired by: http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api

## Requirements

An API is a UI for both Humans and Machines. These requirements outline the general
goals of our API's.

* It should comply with web standards where possible and sane
* It should be technology agnostic, anything should be able to use / consume the API
* It should be service / consumer agnostic
* It should be intuitive to navigate
* It should be **simple**

## Versioning

Each service should support the ability to run different versions if required. For example:

* `/v1` Would route to all the first version endpoints
* `/v2` Would route to the new version endpoints

`/v1` and `/v2` would usually not be forwards or backwards compatible.

## RESTful Resources / URL Structure

In general each service should expose resources on which actions can be taken. Not everything
will map neatly to specific data models but they should map to concepts within the overall application
in general (Customer, Session, Product, Search etc)

### HTTP Verbs

Services should user standard HTTP verbs together with resources which provide the intended
action against a resource, for example:

* `GET /products` - Get a list of products
* `GET /products/123` - Get a specific product
* `POST /products` - Create a new product
* `PUT /products/123` - Update a product in it's entirety
* `PATCH /products/123` - Partially updates a product
* `DELETE /products/123` - Delete a specific product

### Related Resources

If a resource exists that's tied to another they can be mapped together in the URL structure. For example
our `products` may have a set of customer generated reviews against them:

* `GET /products/123/reviews` - Gets a list of all the reviews for the product
* `GET /products/123/reviews/456` - Gets a specific review of the product
* `POST /products/123/reviews` - Creates a new review
* `PUT /products/123/reviews/456` - Updates the review in it's entirety
* `PATCH /products/123/reviews/456` - Partially updates a review
* `DELETE /products/123/reviews/456` - Deletes the review

### External Relations

It sometimes will not make sense to include all of a resources relations, you can provide a link
off to these other relations using the `HATEOAS` standard which we will come too later when
we define our data structure.

### Actions that don't fit the mould

There will always be actions don't fit the normal `CRUD` style operations, for example
activating a user account. This could be achieved with a `PATCH` request to `/users/123`
resource and updating an `activated` flag from `0` to `1`.

Try and map to stubs to RESTful principles, for example, the SOON\_ FM API allows you to
pause the player by sending `POST` request to `/player/pause` and to resume the player
a `DELETE` request to `/player/pause`.

### Limiting Fields

A consumer may not want all the data which the resource returns, a consumer should be able
to specify the fields it wants by using a `fields` query parameter in the URL. This would
contain a comma separated list of fields to ask for.

`GET /products?fields=id,name,price`

Would only return the `id`, `name`, and `price` fields.

**Note:** Services should be flexible and not break if a consumer asks for a field that is
not supported to avoid the consumer breaking. Likewise consumers should also be flexible
when dealing with fields that do not exist to avoid consumers breaking.

### Sorting

Some service resources will need provide sorting options, for example sorting a list of
products by price. This should be achieved via a `sort` query parameter followed by a
comma list of fields to sort on, each prefixed with a `+` or `-` to indicate ascending or
descending ordering.

`GET /products?sort=-price,+name`

This would ask the products list resource to return results sorted descending by price and
then ascending by name.

**Note:** As with field limiting, services should not break if a consumer asks to sort on a
field that is not supported.

### Filtering

Consumers may also need to filter results, for example only show products that are of a
specific category. This should be done by using the field we want to switch on as the query
parameter name and value being the value we want to filter on.

`GET /products?category=1,2,6`

The above example uses a `category` query parameter with the value of `1,2,6` which tells the
service to only return products where they are in the `1`, `2` or `6` categories.

**Note:** Services should not break if the consumer asks to filter on invalid query parameters
or values.

### Limiting List Results

This ties slightly into pagination which will be covered in more detail later, however it is
worth covering here that for resources which return multiple objects a consumer may
want to only get a specific amount of objects, this should be done via the `limit` query
parameter.

`GET /products?limit=4`

**Note:** A service should only every return a maximum number of records, for example `50`
to avoid consumers potentially being able to break services by asking for too much data.

### Common Queries

It is likely consumers will often be performing the same common requests, it is entirely acceptable
where it makes sense that a specific resource can be created for these common queries to
make it easier for the consumer, for example:

`GET /products/new`

The above might be equivalent too:

`GET /products?sort=-created&limit=4`

Which would return only the latest 4 new products.

### Returning Resource Representations

When issuing `POST`, `PUT` or `PATCH` requests to create or update objects should always
return a representation of the resource that has just been created or updated.

This avoids the consumer from having to hit the service again to get the latest
representation of the resource.

## Request Bodies

Request bodies should always be `JSON` encoded, services should return a `400` if request
bodies cannot be unmarshaled from `JSON` into valid data structures.

A `415` can also be returned if the content type is also not `application/json`.

## Status Codes

Using HTTP status codes correctly and intuitively will allow the consumer to better handle
different types of responses from the service and return meaningful information.

### `200 OK`

`200` responses should be issued when a request was successful and has been completed, for
example a `GET`, `PUT` or `PATCH`. `200` can also be used as valid `POST` response **IF**
the request has not resulted in a new resource being created.

### `201 Created`

Only used for `POST` request responses that have resulted in the creation of a new
resource. This should also be used in combination with the `Location` header which would
contain a URI to the newly created resource.

### `202 Accepted`

This status could should be returned if the request has not been acted upon, for example
it maybe in an asynchronous queue and will be acted upon later. The `Location` header
could also be used with this status code to give the URI to eventual location of
the result of this action. This could for a `POST` request to an email service for example.

### `204 No Content`

Only used for `DELETE` request responses when a resource has been successfully deleted.

### `400 Bad Request`

Used when a request is malformed, for example the request body cannot be unmarshaled
by the service.

### `401 Unauthorized`

When a request does not provide valid credentials for a protected resource that requires
authentication.

### `403 Forbidden`

Used when a request us authorized but does not have sufficient access rights to perform
the action.

### `404 Not Found`

When a resource does not exist.

### `405 Method Not Allowed`

Returned when a request is made to a service resource with a HTTP method that is not
supported by the resource.

### `415 Unsupported Media Type`

If a request to the service with an incorrect content type.

### `422 Unprocessable Entity`

Used for validation errors with the request body.

### `429 Too Many Requests`

If the consumer has been rate limited and has made too many requests.

## Responses

It is very important that all our API's stick to a standard response format. General principles are:

* `JSON` All the way
* They stick to `HATEOAS` principles - which can be provided by the `HAL` specification

### :snake: or :camel: case?

Since `JSON` is part of the Javascript family we should use Javascript naming conventions for `JSON`.
This does mean using camel case for field names.

That said Snake case is easier read than camel case (by 20% according to
this [Study](http://www.cs.kent.edu/~jmaletic/papers/ICPC2010-CamelCaseUnderScoreClouds.pdf)) and thus
should make the API easier for humans to read since machines couldn't care less.

**we should decide this together on monday**

### Pretty Print by Default

API Responses should be easily readable by a Human so that means they should include white space.
The bytes used for this minimal provided gzip is on (2% vs 8% when gzip is off).

However each service should provide a `pretty` query parameter that accepts a boolean value to
override so responses can trim the white space.

`GET /products?pretty=false`

### Do not use an envelope

Response should not be wrapped in an enveloper, for example:

``` json
{
    "data": {
        "foo": "bar"
    }
}
```

Just return the data :wink:

``` json
{
    "foo": "bar"
}
```

### Response Body Format

All response bodies should follow the [`HAL`](https://tools.ietf.org/html/draft-kelly-json-hal-07)
specification which follows the [`HATEOAS`](https://en.wikipedia.org/wiki/HATEOAS) principle.

This ensures that links to external / embedded / related resources and actions are contained in
the `JSON` response body to make it easy for the consumer to perform other related actions, or
retrieve related data.

A generic `HAL` response would look something like this:

``` json
{
    "_links": {
        "self": {
            "href": "/products/523"
        },
        "warehouse": {
            "href": "/warehouse/56"
        },
        "company": {
            "href": "/company/873"
        }
    },
    "name": "Foo",
    "currency": "£",
    "price": 10.20
}
```

The above representation is for the `/products/523`. The document contains data about the state of resource
the resource (`name`, `currency`, and `price`). The is also a `_links` property which is an object
containing links off to other related resources as well as a link to itself. If this were to be
represent as `XML` it would look like this:

``` xml
<product>
    <link rel="self" href="/products/523" />
    <link rel="warehouse" href="/warehouse/56" />
    <link rel="company" href="/company/873" />
    <name>Foo</name>
    <currency>£</currency>
    <price>10.20</price>
</product>
```

In the above examples both the `JSON` and `XML` formats come out a 162 and 161 bytes respectively gzipped.

### Links

The reserved `_links` property is **NOT** optional and should always contain a link to itself
if no other links are required:

``` json
{
    "_links": {
        "self": {
            "href": "/you/are/here"
        }
    }
}
```

Links can also contain other properties other than `href`, see the
[Official Specification](https://tools.ietf.org/html/draft-kelly-json-hal-07#section-5.1) for
a full list of properties.

A more complete example would be:

``` json
{
    "_links": {
        "self": {
            "href": "/products",
            "title": "Products"
        },
        "next": {
            "href": "/products?page=2",
            "title": "Page 2 of Products"
        }
    },
    "foo": "bar"
}
```

This example also touches on pagination which we will get to later.

### Embedded Links

The reserved `_embedded` property is **OPTIONAL** and would contain one or more objects specifically
related to the resource object that you are embedding into the response.

So for example an order object would be related to a product for which the order is for, this would
be an embedded object since we are including the name and price of the product:

``` json
{
    "_links": {
        "self": {
            "href": "/order/12345",
            "title": "Order 12345"
        },
    },
    "_embedded": {
        "product": {
            "_links": {
                "self": {
                    "href": "/products/123",
                    "title": "Foo"
                }
            },
            "name": "Foo",
            "price": 11.20
        },
    },
    "number": "12345",
    "status": "processing",
    "placed": "2015-07-29T14:59:08+00:00"
}
```

If we didn't want to embed it then we could just put it in the `_links` attribute instead:

``` json
{
    "_links": {
        "self": {
            "href": "/order/12345",
            "title": "Order 12345"
        },
        "product": {
            "href": "/products/123",
            "title": "Foo"
        }
    },
    "number": "12345",
    "status": "processing",
    "placed": "2015-07-29T14:59:08+00:00"
}
```

This would force the consumer to make another API call if it wanted basic product data.

### Pagination

Resource which support multiple pages of data should use the `HAL` specification to
provide links to other pages within the resource using the `_links` attribute.
The resource itself should return data about the state the resource.

An example would be an orders resource (`/orders`):

``` json
{
    "_links": {
        "self": {
            "href": "/orders",
            "title": "Orders"
        },
        "next": {
            "href": "/orders?page=2",
            "title": "Orders Page 2"
        }
    },
    "_embedded": {
        "orders": [
            {
                "_links": {
                    "self": {
                        "href": "/orders/123",
                        "title": "Order 123"
                    }
                },
                "number": "123",
                "status": "processing"
            }, {
                "_links": {
                    "self": {
                        "href": "/orders/456",
                        "title": "Order 456"
                    }
                },
                "number": "456",
                "status": "shipped"
            }
        ],
    },
    "shipped": 12,
    "processing": 20,
    "total": 32,
    "limit": 2
}
```

Here we can see the `_links` property contains our `next` link data. The `_emebdded` property
contains an `orders` property which has our list of embedded objects. The other properties
returned by the resource contain data about the state of the resource we have accessed, so total
items, the current limit, how many orders we have shipped and are still processing.

## Correlation ID's

Correlation ID's allow us to track requests through various services allowing us to diagnose
errors up the request stack, like a stack trace in code.

Each service is responsible for passing the Correlation ID onto each subsequent request it
makes to other services. This should be done via HTTP Header `Correlation-ID` and should be
a UUID matching the following pattern:

`{:service_name}:{:uuid4}`

This will tell us which service initially generated the ID.

So an example maybe:

`products:5f1457e5-bdbe-4f18-b358-fb5351b10f7c`

If the caller of the service has not provided a Correlation ID it is the responsibility of the
service to generate one.
