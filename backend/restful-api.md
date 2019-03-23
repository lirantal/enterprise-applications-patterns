# RESTful APIs

In 2000, Roy Fielding proposed Representational State Transfer (REST) as an architectural approach to designing web services. REST is an architectural style for building distributed systems based on hypermedia. REST is independent of any underlying protocol and is not necessarily tied to HTTP. However, most common REST implementations use HTTP as the application protocol, and this guide focuses on designing REST APIs for HTTP.

**Disclaimer:**
REST is an architectural pattern and not a strict protocol, therefore many derivatives of it exist and as a general consensus there isn’t a definitive _bad_ or _good_. This document aims to provide a guideline for a recommended approach that attempts to follow the original RESTful architectural style.

There will not always be a silver bullet for how to design the right API (RESTful vs RPC vs GraphQL), since different systems and integrations may use different designs. For example, there is another notion of aggregators and backends-for-frontends (BFF) where you might indeed create a new API for them that would be more tailored to their needs. I'm not too fond of this because it opens a can of worms to always create a new endpoint for each use-case and then you end up maintaining similar purpose endpoints for different consumers.

## Resources

REST APIs are designed around resources, which are any kind of object, data, or service that can be accessed by the client.

A resource has an identifier, which is a URI that uniquely identifies that resource. For example, the URI for a particular customer order might be:
http://adventure-works.com/orders/1

Clients interact with a service by exchanging representations of resources. Many web APIs use JSON as the exchange format. For example, a GET request to the URI listed above might return this response body:

```json
{
	"orderId": 1,
	"orderValue": 99.9,
	"productId": 1,
	"quantity": 1
}
```

REST APIs use a uniform interface, which helps to decouple the client and service implementations. For REST APIs built on HTTP, the uniform interface includes using standard HTTP verbs perform operations on resources. The most common operations are GET, POST, PUT, PATCH, and DELETE.

### Methods (aka verbs)

By using the variety of HTTP methods specific in the HTTP spec we can drive different actions on resources.

The following is a list of methods and their intended actions:

- GET - retrieving of information for a resource.
  For example: `GET /schools` will retrieve a list of schools

- POST - creating a resource entry.
  For example: `POST /schools` will create a new school

- PUT - updating a resource entry in it's entirety. a PUT operation is idempotent since it always sets the same state on the server and doesn't change it.
  For example: `PUT /schools/1` will update school 1 with the data, overriding everything that school id had before.

- PATCH - partial update for a resource entry.
  For example: `PATCH /schools/1` will update specific data in `/schools/1` for example will just update the school name, without having to send and update the entire school object.

When employing a PATCH strategy, one needs to take into account one of two available strategies (backed with RFCs) on how to handle such API calls:

- **JSON merge-patch** (RFC 7396 - "application/merge-patch+json") - When fields are specified with _null_ they convey the intent to remove them from the document.
- **JSON patch** (RFC 6902 - "application/json-patch+json") - Specifies the changes as a sequence of operations to perform, such as “add”, “remove”, etc.

* DELETE - remove a resource entry.
  For example: `DELETE /schools/2/teachers/3` will remove teacher entry 3 from schools

### Methods for Bulks

Often, you may face a situation where it is required to retrieve data in bulk. For example: get all classes for the following list of teacher IDs, which might end up looking like this endpoint:
`GET /classes?teacherId[]=1& teacherId[]=2& teacherId[]=3`

The problem that often arise with that endpoint is that web application servers, proxies or firewalls might end up truncating or completely limiting such request because they impose a content length limit for the amount of characters provided in the query params.

> Tip: the original HTTP spec doesn't mention any such restrictions that should apply on GET requests (or other methods), however the new HTTP 1.1 spec that replaced it recommends a maximum length of 8k for query parameters length.

How to deal with the query length problem then? Two options:

1. Use POST for the endpoint, instead of GET, and then accept all the teacher ids in the body of the request, overcoming any issues related to query param content length.
2. Use GET, to maintain compatibility with RESTful architecture patterns, but limit the query param to a reasonable amount of input length. API consumers can then iterate over the full list of data teacher ids they want to handle and may also send the requests in bulk to parallelize the operation if needed.

Option 1 is a big not recommended.
Why option 2 is better?

- RESTful approach
- Enforce a maximum limit for bulk operations to protect your server from
  heavy load, to make sure it can cope with a given request.

**Counter-argument:**

> how will you do this ? what if tomorrow someone else will add another array to the query parameters and after a month another array, you will again have the same problem with input length

_Response:_

> I assume you mean that another developer will extend the API contract to add another query param? You should enforce a reasonable maximum limit, say allow bulks of 500 teacher ids, and allow room for the extra query params for the query itself. Possibly also thinking the length of your query param names. Practically you may end up with a dozen or so query params besides the bulk ids. If you end up adding tens of query params to your API then at this point you should question yourself if you're not abusing the API contract.

### Resource Naming

Simply put - prefer plurals.

- Good: `/schools`, `/projects`, `/teachers`
- Bad: `/school`, `/class`

How to handle multi-words?

- **Resources**: prefer hyphens
  For example: `/schools/1/teaching-assistants`

- **Query Strings**: prefer snake case
  For example: `/schools/1?students_count=1234`

> Tip: In more complex systems, it can be tempting to provide URIs that enable a client to navigate through several levels of relationships, such as `/customers/1/orders/99/products`. However, this level of complexity can be difficult to maintain and is inflexible if the relationships between resources change in the future.

### Resource Identifiers

Identifying and accessing a resource could be achieved in any of the following ways:

- Bad: `/schools?id=:id`
- Good: `/schools/:id`

RESTful resources follow the second pattern, although a use case argument for (1) would be to support bulk ids such as specifying an array of ids in the query string.
For example: `/schools?id[]=1&id[]=2&id[]=3`

However, REST is also about representing resource hierarchies or relationships, for example accessing a specific school in a specific city, consider the following two options:

- `/city/school?city_id=:id&school_id=:id`
- `/city/:id/school/:id`

Option 2 is more verbose, readable and friendly.

### Resource query params

A query parameter that represents an array should be stated using the square brackets notation.
For example: `/schools?id[]=1&id[]=2&id[]=3`

There's a semantic benefit that `id[]` could be argued to clearly represent an array, while a list of plain id parameters such as `id=1&id=2` don't necessarily convey that intent.

> This notation of query param name with square brackets has been made popular with languages and frameworks like jQuery, PHP, Ruby and others. Furthermore, this is also related to the [HTTP Parameter Pollution (HPP)](<https://www.owasp.org/index.php/Testing_for_HTTP_Parameter_pollution_(OTG-INPVAL-004)>) attack vector where-as different platforms and web servers handle duplicate query param names in different ways - some will take the first occurrence, while others will take the last, or treat it as an array altogether. Neither is a problem on its own as long as the developer handles it correctly.

## Errors

When errors happen the server should provide relevant context to the problem:

- A meaningful explanation to clients using a friendly generic message.
- The server should never have to implement internationalization as this is a job for consuming clients. To support that, every error message should be accompanied with an error status (either a numeric ID or text) that identifies the problem.

An example error payload:

```json
{
	"message": "Error when validation fields",
	"status": "FIELDS_VALIDATION_ERROR"
}
```

Examples of user related errors from the HTTP spec:

- 400 for when the requested information is incomplete or malformed.
- 422 for when the requested information is okay, but invalid.
- 404 for when everything is okay, but the resource doesn’t exist.
- 409 for when a conflict of data exists, even with valid information

Example use-cases:

- If the email field is missing, return a 400 .
- If the password field is too short, return a 422 .
- If the email field isn’t a valid email, return a 422 .
- If the email is already taken, return a 409 .

**Counter arguments:**

> Why should we duplicate http status in body of the response?

_Response:_

> It is usually more easily accessible to consumers as part of the payload, and another reason is that if you log the response payload then you have this context of the response for later inquiry. I don't see it as breaking REST because it is not replacing the HTTP status, but instead just mirroring that part of the context.

### References:

1. [Problem Details for HTTP APIs](https://tools.ietf.org/html/rfc7807)

## Return codes

RESTful API services should adhere to architectural pattern and make use of the HTTP transport to communicate the intent of their response based on the existing HTTP error codes as specific in the [spec](https://tools.ietf.org/html/rfc7231).

Example use-cases:

- GET request for bulk APIs with more data than you are allowing to handle? return a 414 URI Too Long error.
- GET request for an API but you have no results to return? As in the resource is empty as opposed to not existing which is different state altogether. Return an empty set: an empty object {} or empty array [].
- POST for creating resources - created resource successfully? return 201. Optionally provide in the body the resource data (id, etc).
- PUT or DELETE operations on a resource - updated or deleted successfully? return 200.
- PUT or DELETE operations on a resource, but there is no sense in returning any data? return 204 (the body payload should not exist). For "204 No Content" it would be advised to carefully consider using it and choosing the cases that truly makes sense. For example, doing an "Unlike" operation on a Facebook or Twitter type of social network, or say deleting a document like in Google Docs. These use-cases might send a DELETE request, where sending back a response may not make sense, there-fore the "Unlike" was successful, but there's no content to send, hence 204 could be a good fit.
- PUT / PATCH requests aren’t allowed for a resource? return 409 to convey conflict.
-

A question that often comes up with regards with regards to creating or modifying data is: when does it make sense for a POST or PUT requests to return a response that also includes the object that was created?
A rule of thumb for answering this is to ask whether the returned object will include more meaningful data than was sent by the client? For example, perhaps there is a computed field that there is no control for the user to send this data, but would make sense to return it for interested clients.

## Payload Formatting

### Data Envelopes

Responding with more than just the actual data payload (DTO) to your clients may prove useful for debugging, monitoring and instrumentation operations.

For that end, it is suggested to use envelopes, wrapping all responses in objects, even if a response is an array list items, and providing an additional metadata on the request. Metadata that can prove useful is the request time, or a correlated transaction id that can be later referred to in logs.

> Security note: returning an object as a JSON response is a way to mitigate JSON Hijacking attacks where malicious parties may try to target victims by exploiting means of requesting third-party website JSONs on the user's behalf. Google, Facebook and others are using similar mitigations such as prefixing a JSON response with custom text for example `while(1);{"hello": "world"}` to throw off offending parties.
> [OWASP JSON object reference](https://www.owasp.org/index.php/AJAX_Security_Cheat_Sheet#Always_return_JSON_with_an_Object_on_the_outside)

- Data is always in a dedicated data key: `{ data: {} }`
- Metadata about the request is always in a dedicated meta key: `{ meta: {} }`
  _ request time
  _ request id \* returned status code

An example successful response might look like this:

```json
{
  "data": {
    "// data payload goes here": ""
  },
  "meta": {
    "requestTime": 123453453,
    "requestId": 123,
    "message": "SUCCESS",
    "status": null,
    "statusCode": 200
  }
}
```

**Error payloads**

In cases of errors, such as when 400 or 500 responses are sent back by servers, the response may look as follows:

```json
{
  "data": {
    "// in cases of API errors, the errors information is inside the data field": "",
    "description": "One or more fields raised validation errors",
    "invalidPath": {
      "// the joi schema validation path": ""
    }
  },
  "meta": {
    "requestTime": 123453453,
    "requestId": 123,
    "message": "Error when validation fields",
    "status": "FIELDS_VALIDATION_ERROR",
    "statusCode": 422
  }
}
```

## Asynchronous Operations

Sometimes operations can take a while to complete.
When that is the case, return status code 202 (accepted) to convey a message was received but not yet completed, return a `Location` header for the status endpoint where clients can query and poll for status information about the operation.

For example:

```json
HTTP/1.1 202 Accepted
Location: /api/status/12345
```

Then, an implemented status API end point should reply with useful information:

```json
HTTP/1.1 200 OK
Content-Type: application/json


{
    "status":"In progress",
    "link": {
		"rel":"cancel",
		"method":"delete",
		"href":"/api/status/12345"
	  }
}
```

## RESTful heaven

I have not yet found any advantage in implementing the following but they are part of the pure RESTful architectural style.

In 2008, Leonard Richardson proposed the following maturity model for web APIs:

- Level 0: Define one URI, and all operations are POST requests to this URI.
- Level 1: Create separate URIs for individual resources.
- Level 2: Use HTTP methods to define operations on resources.
- Level 3: Use hypermedia (HATEOAS, described below).

Level 3 corresponds to a truly RESTful API according to Fielding's definition. In practice, many published web APIs fall somewhere around level 2.

## HATEOAS

Use HATEOAS to enable navigation to related resources
One of the primary motivations behind REST is that it should be possible to navigate the entire set of resources without requiring prior knowledge of the URI scheme. Each HTTP GET request should return the information necessary to find the resources related directly to the requested object through hyperlinks included in the response, and it should also be provided with information that describes the operations available on each of these resources. This principle is known as HATEOAS, or Hypertext as the Engine of Application State. The system is effectively a finite state machine, and the response to each request contains the information necessary to move from one state to another; no other information should be necessary.

REST APIs are driven by hypermedia links that are contained in the representation. For example, the following shows a JSON representation of an order. It contains links to get or update the customer associated with the order.

```json
{
  "orderID": 3,
  "productID": 2,
  "quantity": 4,
  "orderValue": 16.6,
  "links": [
    {
      "rel": "product",
      "href": "http://adventure-works.com/customers/3",
      "action": "GET",
      "types": [
        "application/x-www-form-urlencoded"
      ]
    },
    {
      "rel": "product",
      "href": "http://adventure-works.com/customers/3",
      "action": "PUT"
    }
  ]
}
```

# References

- Adobe's Audience Manager's API Guide: https://bank.demdex.com/portal/swagger/index.html
- Stripe API Guide: https://stripe.com/docs/api
- GitHub API Guide: https://developer.github.com/v3/
