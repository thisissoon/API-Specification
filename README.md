# REST API Specification

We write a lot of services that communicate with one another, it's important that we
have a standard source of truth for how we should write and consume our services via API's.
This document aims to provide that one source of truth.

## Versions

Current Version: [2015.8.25](https://github.com/thisissoon/API-Specification/tree/2015.8.25)

### Past Versions

* [2015.8.4](https://github.com/thisissoon/API-Specification/tree/2015.8.4)

### Change Log

All changes to this document are tracked in [CHANGELOG.md](https://github.com/thisissoon/API-Specification/blob/master/CHANGELOG.md)

## Contents

* [Credits](#credits)
* [Requirements](#requirements)
* [Versioning](#versioning)
* [Restful Resources / URL Structure](#restful-resources--url-structure)
  * [HTTP Verbs](#http-verbs)
  * [Related Resources](#related-resources)
  * [External Relations](#external-relations)
  * [Limiting Fields](#limiting-fields)
  * [Sorting](#sorting)
  * [Filtering](#filtering)
  * [Limiting](#limiting-list-results)
  * [Common Queries](#common-queries)
  * [Returning Resource Representations](#returning-resource-representations)
* [Request Bodies](#request-bodies)
* [Status Codes](#status-codes)
* [Responses](#responses)
  * [Camel Cased](#camel-cased)
  * [Pretty Print](#pretty-print-by-default)
  * [Do Not Use an Envelope](#do-not-use-an-envelope)
  * [Date Formats](#date-formats)
  * [Response Body Format](#response-body-format)
  * [Links](#links)
  * [Embedded Links](#embedded-links)
  * [Pagination](#pagination)
* [Errors](#errors)
  * [Service Errors](#service-errors-5xx)
  * [Validation Errors](#validation-errors-422)
* [Correlation IDs](#correlation-ids)
* [CORS](#cors)
* [Authentication](#authentication)
  * [Example Flow](#example-flow)
* [Service Identification](#service-identification)
* [HMAC](#hmac-request-verification)
  * [Example](#example)
* [Caching](#caching)

## Credits

This document is inspired by: http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api

Everyone at [SOON\_](http://thisissoon.com)

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
* `OPTIONS /products/123` - Tells what methods are supported by the resource

#### `OPTIONS`

All resources should support `OPTIONS` requests. Sending `OPTIONS` to the resource should
tell us what HTTP Methods are supported by the resource.

``` http
HTTP/1.1 200 OK
Allow: HEAD, OPTIONS, GET, POST
Connection: keep-alive
Content-Length: 0
Content-Type: text/html; charset=utf-8
Date: Tue, 04 Aug 2015 09:50:25 GMT
Server: nginx/1.6.2
```

The above tells us the resource allows `HEAD`, `OPTIONS`, `GET`, and `POST`.  Other verbs such as
`DELETE`, `PUT` and `PATCH` would result in a 405.

##### Bonus Data

Optionally to help out the consumer the `OPTIONS` request could return a data strucutre of
exactly what is expected by a resource. This is not a hard and fast rule but it's always
handy to have.

``` json
{
    "GET": {
        "description": "Returns the representation of the object."
    },
    "POST": {
        "description": "This does stuff!",
        "parameters": {
            "title": {
                "type": "string",
                "description": "bla bla bla.",
                "required": true
            }
        }
    }
}
```

Here the `POST` object describes the fields that are expected by the resoure and `GET` describes what
a `GET` request would do.

### Related Resources

If a resource exists that's tied to another they can be mapped together in the URL structure. For example
our `products` may have a set of customer generated reviews against them:

* `GET /products/123/reviews` - Gets a list of all the reviews for the product
* `GET /products/123/reviews/456` - Gets a specific review of the product
* `POST /products/123/reviews` - Creates a new review
* `PUT /products/123/reviews/456` - Updates the review in it's entirety
* `PATCH /products/123/reviews/456` - Partially updates a review
* `DELETE /products/123/reviews/456` - Deletes the review
* `OPTIONS /products/123/reviews/456` - Tells what methods are supported by the resource

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
comma list of fields to sort on, each prefixed with a `asc` or `desc` to indicate ascending or
descending ordering.

`GET /products?sort=desc:price,asc:name`

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

Query parameters should be consistent with the data model and should be camelCased. For example;
where `Product` has a property `productName` the filter should be:

`GET /products?productName=abc`

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

### Filtering Metadata

To make it clear what filters are available on a resource, the `OPTIONS` request could provide a
`_filters` object with details of each filter. These could either include an embedded list of
query options or a link to another resource, for example:

``` json
"_filters": {
    "region": {
        "options": [
            {
                "label": "Europe",
                "value": "EU"
            }
        ]
    },
    "section": {
        "link": "/api/sections",
        "value": "slug"
    }
}
```

### Returning Resource Representations

When issuing `POST`, `PUT` or `PATCH` requests to create or update objects should always
return a representation of the resource that has just been created or updated.

This avoids the consumer from having to hit the service again to get the latest
representation of the resource.

## Request Bodies

Request bodies should always be `JSON` encoded, services should return a `400` if request
bodies cannot be unmarshaled from `JSON` into valid data structures.

A `415` can also be returned if the content type is also not `application/json`.

### Business Requirements

Clients may also require our public facing API's to use other formats, such as `XML`. In
that situation the API gateway would need to provide some sort of content negotiation to
communicate with other services using `JSON`.

## Status Codes

Using HTTP status codes correctly and intuitively will allow the consumer to better handle
different types of responses from the service and return meaningful information.

### `200 OK`

`200` responses should be issued when a request was successful and has been completed, for
example a `GET`, `PUT` or `PATCH`. `POST` Requests should **NOT** return a `200`. See `201`
and `202` status codes below for valid `POST` response status codes.

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

### Camel Cased

Since `JSON` is part of the Javascript family we should use Javascript naming conventions for `JSON`.
This does mean using camel case for field names.

`productOrderDate` and not `product_order_date`.

Some languages such as Python prefer snake case over camel however we could write some middleware
which converts snake cased attributes to Camel Case for API responses to make life easy.

### Pretty Print by Default

API Responses should be easily readable by a Human so that means they should include white space.
The bytes used for this is minimal provided gzip is on (2% vs 8% when gzip is off).

However each service should provide a `pretty` query parameter that accepts a boolean value to
override so responses can trim the white space if they so desire.

`GET /products?pretty=false`

### Do not use an envelope

Response should not be wrapped in an envelope, for example:

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

### Date Formats

Dates should follow the `RFC3339` specification. That means:

* `2016-01-06T14:45:00Z` for UTC (not `2016-01-06T14:45:00+00:00`)
* `2016-01-06T14:45:00+01:00` for Time Zones

By default all dates should be UTC dates and not include a timze zone offset.

### Response Body Format

We will be using the `HAL` specification.

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

The reserved `_links` property is **REQUIRED** and should always contain a link to itself
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
    "pages": 16,
    "currentPage": 1,
    "limit": 2,
    "shipped": 12,
    "processing": 20,
    "total": 32,
}
```

Here we can see the `_links` property contains our `next` link data. The `_embedded` property
contains an `orders` property which has our list of embedded objects.

The object contains meta data about the currently paginated resource:

* `pages`: The total number of pages we have
* `currentPage`: The current page we are on
* `limit`: The current number of objects returned by each page

The other properties returned here (`shipping`, `processing`) are specific to this resources
current state. Note that `total` as part of the resource state.

## Errors

Errors are guaranteed to occur, either in the form of validation errors or actual service errors.
Errors should always follow a consistent format.

We will use an `_errors` object to return error information allowing consumers to check for this
objects existence when error status codes are retuned.

### Service Errors (`5XX`)

In the case of `5XX` errors it is unlikely we will know much about the error, it could be that
a server is down (`502`) or something has gone wrong processing the request (`500`). These errors
should be formatted as follows:

``` json
{
    "_links": {
        "self": {
            "href": "/orders",
            "title": "Orders"
        },
    },
    "_errors": {
        "message": "Internal Server Error"
    }
}
```

If we are operating in a debug QA / Development environment this could also contain a `stack` attribute:

``` json
{
    "_links": {
        "self": {
            "href": "/orders",
            "title": "Orders"
        },
    },
    "_errors": {
        "message": "Internal Server Error",
        "stack": "...ImportError: cannot import name authenticated...",
    }
}
```

### Validation Errors (`422`)

If the consumer has sent in-valid data the service will respond with a `422`. The `_errors` object
will also contain those validation errors.

``` json
{
    "_links": {
        "self": {
            "href": "/orders",
            "title": "Orders"
        },
    },
    "_errors": {
        "message": "Validation Errors",
        "errors": {
            "field_name": "First name can only contain ASCII characters",
            "password": "Password is requried",
        }
    }
}
```

Here `_errors` now containers an `errors` object which contains field level validation errors.

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

## CORS

Try and avoid it where possible, Frontends should have Backends specifically designed for
the FE to talk to other services avoiding `CORS` all together.

## Authentication

Authentication should use the `HTTP` `Authorization` header. Session tokens should be returned
in the form of `{id}:{session_id}` where `id` is a hash of the user `id` and `session_id` is
a unique Session ID used to validate the user.

### Example Flow

* First `POST` credentials to the appropriate authentication service, here we will use
  `auth.service` which would be an internal micro service handling user authentication.

``` http
POST / HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Connection: keep-alive
Host: auth.service

{
    "email": "foo@bar.com",
    "password": "1234"
}
```

``` http
HTTP/1.0 201 Created
Content-Type: application/json; charset=utf-8
Date: Mon, 03 Aug 2015 11:37:25 GMT

{
    "token": "$dsjkfhdskjfh:jhfjklsadhfkdhsafjhasdkhfjksdhfkh"
}
```

**Note:** A `201` has been returned since a token was created.

* Once the token has been generated it can then be used in the `Authorization` header for resources
  that require authentication. This should be a `Base 64` encoded version of the token.

``` http
GET /me HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate
Authorization: Basic OmpoZmprbHNhZGhma2Roc2Fmamhhc2RraGZqa3NkaGZraA==
Connection: keep-alive
Host: users.service
```

``` http
HTTP/1.0 200 OK
Content-Type: application/json; charset=utf-8
Date: Mon, 03 Aug 2015 11:37:25 GMT

{
    "_links": {
        "self": {
            "href": "/me",
            "title": "Me"
        },
    },
    "username": "krak3n",
    "email": "foo@bar.com",
    "activated": true,
}
```

## Service Identification

A service should identify itself via a HTTP Header. This is useful for debugging.

``` http
HTTP/1.0 200 OK
Content-Type: application/json; charset=utf-8
Date: Mon, 03 Aug 2015 11:37:25 GMT
Service: FooServiceName
```

## HMAC Request Verification

Each service should sign it's requests to other services using a `Base 64` `SHA256` `HMAC` using
a private key. Each down stream service will then validate each request to ensure the request
has come from a known service and has not been tampered with.

Each service should send their request signature using the `Signature` HTTP Header which follows
this format:

```
Signature: base64(id:signature)
```

If a service does not contain a `Signature` header or the signature is not valid the service
should return a `400 Bad Request` and Flag the request as suspicious.

Binary blob data (such as images) should be excluded from the signature and verification process.

### Example

The follow example shows a Python Server and Go Client.

#### Server

**Note:** This is pseudo code.

``` python
import base64
import hashlib
import hmac

from flask import Flask, request
app = Flask(__name__)

clients = {
    'FooService': '123abc'
}

@app.route("/")
def hello():
    # Get the Signature, splitting at the : which gives us the client id and the request signature
    cleint_id, client_signature = request.headers.get('Signature').split(':')
    # Get the Private Key for the Client
    key = clients.get(client_id)
    # Generate the Same Base64 encoded SHA256 HMAC from the Clients Key
    signature = base64.b64encode(
        hmac.new(
            key,
            request.data,
            hashlib.sha256).digest())

    # Ensure they match, if not throw a 400
    if client_signature != signature:
        return 'Invalid Signature', 400

    # Everything is cool...
    return "Hello World!", 200

if __name__ == "__main__":
    app.run()
```

#### Client

**Note:** This is pseudo code.

``` go
package main

import (
	"bytes"
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"fmt"
	"log"
	"net/http"
)

var key = "123abc"
var id = "FooService"

func main() {
	// JSON Data to send in the body
	data := []byte(`{"foo": "bar"}`)

	// Generate Signature - SHA256 HMAC from Private Key Base64 Encoded
	mac := hmac.New(sha256.New, []byte(key))
	mac.Write(data)
	signature := base64.StdEncoding.EncodeToString(mac.Sum(nil))

	// Make the request to the service - adding the custmom header
	req, _ := http.NewRequest("POST", "http://hello.service", bytes.NewBuffer(data))
	req.Header.Set("Signasture", base64.StdEncoding.EncodeToString([]byte(fmt.Sprintf("%s:%s", id, signature))))
	client := &http.Client{}
	res, err := client.Do(req)

	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(res)
}
```

## Caching

Where appropriate consumers and services should cache responses. This can be achieved via the `ETag` HTTP
specification.

When a consumer requests data from a resource the response should include an `Etag` header. This is a hash
of the response data, for example: `94232c5b8fc9272f6f73a1e36eb68fcf`. A `Cache-Control` header should
also be provided detailing the length of time to cache for, for example: `max-age=86400`. The consumer
should then store a cache of the response locally. Servers should also cache locally if possible.

When a consumer requests the resource again it should send the `Etag` in the `If-None-Match` header, this
triggers the server to check the `Etag` against that of the consumer. If they match the server will respond
with an empty `304 Not Modified` which tells the consumer to use it's local cache. If the `Etag` and `If-None-Match`
do not match this means the data has changed and the server should respond as normal, forcing the
consumer to delete the old cache and cache the new response.

### Example

The example shows 4 HTTP reqpuest and responses outline the flow.

#### First Request

The first request to the resource.

``` http
GET / HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, deflate
Connection: keep-alive
Host: foo.service/123
```

This causes the server to respond with a `200` including an `ETag` and `Cache-Control` headers. The consumer
should now cache locally.

``` http
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 193
Content-Type: application/json
Location: https://foo.service/123
ETag: 94232c5b8fc9272f6f73a1e36eb68fcf
Cache-Control: max-age=86400

{"foo": "bar"}
```

#### Second Request

We make the same request again but this time with the `If-None-Match` header which contains the `ETag` we
got from the last request response.

``` http
GET / HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, deflate
Connection: keep-alive
Host: foo.service/123
If-None-Match: 94232c5b8fc9272f6f73a1e36eb68fcf
```

The server now responds with a `304` and no data forcing the consumer to load from it's local cache.

``` http
HTTP/1.1 304 Not Mofified
Connection: keep-alive
Content-Length: 0
Content-Type: application/json
Location: https://foo.service/123
```

#### Thrird Request

We make the same request agaain.

``` http
GET / HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, deflate
Connection: keep-alive
Host: foo.service/123
If-None-Match: 94232c5b8fc9272f6f73a1e36eb68fcf
```

This time the content has changed and therefore a new `200` response is rerurned with a new `ETag` and
`Cache-Control` headers.

``` http
HTTP/1.1 200 OK
Connection: keep-alive
Content-Length: 193
Content-Type: application/json
Location: https://foo.service/123
ETag: 8594ed77e6c35d97bb7e07c9fa765b15
Cache-Control: max-age=86400

{"foo": "baz"}
```

The consumer should now clear it's previous cache and store the new object.

#### Fourth Request

We now make the same request with a new `If-None-Match` header container the new `ETag`.

``` http
GET / HTTP/1.1
Accept: application/json
Accept-Encoding: gzip, deflate
Connection: keep-alive
Host: foo.service/123
If-None-Match: 8594ed77e6c35d97bb7e07c9fa765b15
```

Once again we get a `304`.

``` http
HTTP/1.1 304 Not Mofified
Connection: keep-alive
Content-Length: 0
Content-Type: application/json
Location: https://foo.service/123
```
