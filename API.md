# API

## API History:

![API History](/img/APIHistory.JPG "API History")

## HTTP Request:

![Http Request](/img/HttpRequest1.JPG "Http Request")

### E.g

![Http Request](/img/HttpRequest2.JPG "Http Request")

### Request Deconstructed:

#### URI
* URI & Querystring

#### Verb

* Action to perform on the server.
* GET : Request resource
* POST : Create resource
* PUT : Update resource
* PATCH : Update partial resource
* DELETE : Delete resource

#### Headers:

* Metadata about the request
* Content-Type: The format of content
* Content-Length: Size of content
* Authorization: Who's making the call
* Accept: What type(s) can accept
* Cookies: Client data in the request

#### Content:

* Request body : Data to send to server
* HTML, CSS, JavaScript, XML, JSON
* Content is not valid with some verbs
* Information to help fulfill request
* Binary and blobs common (e.g. jpg)

## HTTP Response:

![Http Response](/img/HttpResponse1.JPG "Http Response")

### E.g

![Http Response](/img/HttpResponse2.JPG "Http Response")

### Response Deconstructed:

#### Status code:

* Operation status
* 100-199: Informational
* 200-299: Success
* 300-399: Redirection
* 400-499: Client Errors
* 500-599: Server Errors

#### Headers:

* Metadata about the response
* Content-Type: The format of content
* Expires: When to consider stale
* Cookies: Passenger data in the request

#### Content:

* Response body
* HTML, CSS, JavaScript, XML, JSON
* Binary and blobs common (e.g. jpg)
* APIs often have their own types

## Sample code

### Testing API

```sample code
@site=https://restdesign.dev
GET {{site}}
```

### 1) GET Method

#### Request

```sample code
GET {{site}}/api/customers
Accept: application/json
```

#### Response

```sample code
HTTP/1.1 200 OK
Connection: close
Content-Type: application/json; charset=utf-8
Date: Sat, 07 Mar 2026 08:13:27 GMT
Server: Kestrel
Cache-Control: public,max-age=60
ETag: "BE2FACBEE107B16CE352A6E8010D3A01"
Expires: Sat, 07 Mar 2026 08:14:28 GMT
Last-Modified: Sat, 07 Mar 2026 08:13:28 GMT
Transfer-Encoding: chunked
Vary: Accept, Accept-Language, Accept-Encoding
api-supported-versions: 2.0
api-deprecated-versions: 1.0

[
  {
    "id": 9,
    "companyName": "Blanda and Sons",
    "contact": "Claudie Lynch",
    "phoneNumber": "623-369-5083 x379",
    "email": null,
    "addressLine1": "715 Benny Center",
    "addressLine2": null,
    "addressLine3": null,
    "city": "Fernandoburgh",
    "stateProvince": "MN",
    "postalCode": "77240-3310",
    "country": null,
    "projects": []
  },
  {
    "id": 22,
    "companyName": "Cole LLC",
    "contact": "Ardella Pollich",
    "phoneNumber": "1-570-784-7651 x491",
    "email": null,
    "addressLine1": "480 Shanon Bridge",
    "addressLine2": null,
    "addressLine3": null,
    "city": "Rooseveltside",
    "stateProvince": "TX",
    "postalCode": "59829-8000",
    "country": null,
    "projects": []
  }
  ]
```

### 2) POST Method

#### A) 415 Unsupported Media Type

#### Request

```sample code
POST {{site}}/api/customers
Accept: application/json

Foo bar
```

#### Response

```sample code
HTTP/1.1 415 Unsupported Media Type
```

#### B) 400 Bad Request

#### Request

```sample code
POST {{site}}/api/customers
Accept: application/json
Content-Type: application/json


Foo bar
```

#### Response

```sample code
HTTP/1.1 400 Bad Request
```

## REST:

* Represenatational state transfer
* Separation of client and server
* Servers are stateless
* Cacheable requests
* Uniform interface

### Problems:

* To difficult to be qualified as "REST"
* Dogma of REST vs Pragmatism

a) Structure architecture style<br/>
b) They need to be productive

## Designing RESTful web APIs:

* Design first
* REST requests
* Verbs
* Resources(Nouns)
* Idempotency
* Designing results
* Formatting results

### Design your API first

* Can't fix an API after publishing
* Too easy to add ad-hoc endpoints
* Helps understand the requirements
* Well designed API can mature

#### URIs:

* URIs are just paths to resources
<br/>e.g. api.yourserver.com/people

* Query strings for non-data elements
<br/>e.g. format, sorting, searching, paging etc.

#### Resources:

Real world objects:
* People
* Invoices
* Payments
* Products

![Resources](/img/Resources.JPG "Resources")


#### Identifiers in URI:

Use unique identifiers
Doesnot have to be primary keys

```sample code
api/customers
api/customers/2
api/customers/microsoft
api/customers/msft
```

#### Query Strings:

Use for non-resource properties

```sample code
/customers?includeProjects=true/tickets?page=1
```

![REST-Verb](/img/REST-Verb.JPG "REST-Verb")

![Verbs-URIs](/img/Verbs-URIs.JPG "Verbs-URIs")

#### Idempotency:

* Operations result in same side effect
<br/>GET, PUT, PATCH, and DELETE

* POST is not idempotent

#### Decide formats during design:

Abide by Accept header
```sample code
Accept: application/json, text/html
```

Return Sane default (usually JSON)
```sample code
Content-Type: application/json
```

Prefer NOT to use query strings for formats:
```sample code
/api/customer?format=json  // <- Antipattern
```

#### Common Formats:

* JSON : application/json
* XML : text/xml
* JSONP : application/javascript
* RSS : application/xml+rss
* ATOM : application/xml+atom

#### Designing Associations / Relational APIs:

* For sub-objects: Use URI Navigation
<br/>api/customers/123/Invoices

* Should return list - same shapes
<br/>api/customers/123/invoices
<br/>api/invoices

* Can have multiple associations
<br/>api/customers/123/invoices
<br/>api/customers/123/payments
<br/>api/customers/123/shipments

* Search should use queries
<br/>api/customers?st=GA
<br/>api/customers?st=GA&salesid=144
<br/>api/customers?hasOpenOrders=true

Paging:

![Paging](/img/Paging.JPG "Paging")

#### Error Handling:

* Not just status codes
* How do you communicate errors
* How do you help the user recover

* Return object with error info

```sample code
400 Bad Request
{
  error: "Failed to supply id"
}
```

* Not necessary for obvious errors

```sample code
404 Not Found 
```

#### Caching:

* Basic tenet of REST APIs
* Server-side caching is good
* But isn't what they mean
* Use Http for caching mechanism

![HTTPCaching](/img/HTTPCaching.JPG "HTTPCaching")

![HTTPCaching1](/img/HTTPCaching1.JPG "HTTPCaching1")

#### Entity Tags(ETags)

* Strong and Weak Caching support
* Returned in the response

![ETag1](/img/ETag1.JPG "ETag1")

![ETag2](/img/ETag2.JPG "ETag2")

* Use 304 to indicate that it's cached

```sample code
HTTP/1.1 304 Not Modified
```
![ETag3](/img/ETag3.JPG "ETag3")

```sample code
HTTP/1.1 412 Preconditioned Failed
```

#### Functional APIs

* Be Pragmatic
* Make sure these are documented
* Should be completely functional
* Not an excuse to build an RPC API
* Should be exception rather than rule

#### Async APIs

* Some APIs aren't RESTful in nature
* Need long-life polling
* Non-REST solutions are useful

#### Async API solutions to consider:

* Comet
* gRPC
* SignalR
* Firebase
* Socket.IO

#### Versioning your API:

<br/>Should you version your API ?

* Once we publish, it's set in stone
* User rely on the API not changing
* But requirements will change

<br/> Evolve the API without breaking clients:

- API version isn't product version
- Don't tie them together

<br/> API versioning is harder

- Needs to support both new and old
- Side-by-side deployment isn't feasible

#### Versioning strategies:

<br/>1) Versioning in the URI path

```sample code
https://foo.org/api/v2/Customers
```

<br/>Pros:

- Very clear to clients where the version is handled

<br/>Cons: