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