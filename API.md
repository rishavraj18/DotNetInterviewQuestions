# API

## API History:

![API History](/img/APIHistory.JPG "API History")

## HTTP Request:

![Http Request](/img/HttpRequest1.JPG "Http Request")

### E.g

![Http Request](/img/HttpRequest2.JPG "Http Request")

### Request Deconstructed:

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

* Data to send to server
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



