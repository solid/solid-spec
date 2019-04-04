# Solid HTTPS REST API Spec

***Note:** This spec is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.*

## Reading Resources

Resources can be commonly accessed (i.e., read) using HTTP `GET` requests. Solid
servers are encouraged to perform content negotiation for RDF resources,
depending on the value of the `Accept` header.

***IMPORTANT:** a default `Content-Type: text/turtle` will be used for requests
for RDF resources or views (containers) that do not have an `Accept`
header.*

### Streams

Being LDP (`BasicContainer`) compliant, Solid servers MUST return a full listing
of container contents when receiving requests for containers. For every resource
in a container, a Solid server may include additional metadata, such as the time
the resource was modified, the size of the resource, and more importantly any
other RDF type specified for the resource in its metadata. You will notice in
the example below that the `<profile>` resource has the extra RDF type
`<http://xmlns.com/foaf/0.1/PersonalProfileDocument>`, and also that the
resource `<workspace/>` has the RDF type
`<http://www.w3.org/ns/pim/space#Workspace>`.

Extra metadata can also be added, describing whether each resource in the
container maps to a file or a directory on the server, using the [POSIX
vocabulary](http://www.w3.org/ns/posix/stat#). Here is an example that reflects
how our current server implementations handle such a request:

**REQUEST**

```http
GET /
Host: example.org
```

**RESPONSE**

```http
HTTP/1.1 200 OK
```
```ttl
@prefix rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .

<>
    a <http://www.w3.org/ns/ldp#BasicContainer>, <http://www.w3.org/ns/ldp#Container>, <http://www.w3.org/ns/posix/stat#Directory> ;
    <http://www.w3.org/ns/ldp#contains> <profile>, <data/>, <workspace/> ;
    <http://www.w3.org/ns/posix/stat#mtime> "1436281776" ;
    <http://www.w3.org/ns/posix/stat#size> "4096" .

<profile>
    a <http://xmlns.com/foaf/0.1/PersonalProfileDocument>, <http://www.w3.org/ns/posix/stat#File> ;
    <http://www.w3.org/ns/posix/stat#mtime> "1434583075" ;
    <http://www.w3.org/ns/posix/stat#size> "780" .
```

#### Globbing (inlining on `GET`)

**Note: this feature is _at risk_ of being
[changed](https://github.com/solid/solid-spec/pull/148)
or [removed](https://github.com/solid/solid-spec/pull/151).
Please join the discussion.
Code depending on this will still work for now.**

In some cases, we have found that using the existing LDP features was not
enough. For instance, to optimize certain applications, we needed to aggregate
all RDF resources from a container and retrieve them with a single `GET`

operation. We implemented this feature on the servers and decided to call it
"globbing". Similar to [UNIX shell
glob](https://en.wikipedia.org/wiki/Glob_(programming)), doing a `GET` on any URI
which ends with a `*` will return an aggregate view of all the resources that
match the indicated pattern.

For example, let's assume that `/data/res1` and `/data/res2` are two resources
containing one triple each, which defines their type as follows:

**For *res1***

```ttl
<> a <https://example.org/ns/type#One> .
```

**For *res2***

```ttl
<> a <https://example.org/ns/type#Two> .
```

If one would like to fetch all resources of a container beginning with `res`
(e.g., `/data/res1`, `/data/res2`) in one request, they could do a `GET` on
`/data/res*` as follows.

**REQUEST**

```http
GET /data/res* HTTP/1.1
Host: example.org
```

**RESPONSE**

```http
HTTP/1.1 200 OK
```
```ttl
<res1>
    a <https://example.org/ns/type#One> .

<res2>
    a <https://example.org/ns/type#Two> .
```

Alternatively, one could ask the server to inline *all* resources of a
container, which includes the triples corresponding to the container itself:

**REQUEST**

```http
GET /data/* HTTP/1.1
Host: example.org
```

**RESPONSE**

```http
HTTP/1.1 200 OK
```
```ttl
<>
    a <http://www.w3.org/ns/ldp#BasicContainer> ;
    <http://www.w3.org/ns/ldp#contains> <res1>, <res2> .

<res1>
    a <https://example.org/ns/type#One> .

<res2>
    a <https://example.org/ns/type#Two> .
```

Note: the aggregation process is not currently recursive, therefore it will not
apply to children containers.

### Alternative: using SPARQL

Another possible way of reading and writing data is to use SPARQL. Currently,
our Solid servers support a subset of [SPARQL
1.1](https://www.w3.org/TR/sparql11-overview/), where each resource is its own
SPARQL endpoint, accepting basic `SELECT`, `INSERT`, and `DELETE` statements.

To read (query) a resource, the client can send a SPARQL `SELECT` through a
form-encoded HTTP `GET` request. The server will use the given resource as the
default graph that is being queried. The resource can be an RDF document or even
a container. The response will be serialized using `application/json` MIME type.

For instance, the client can form-encode and send the query `SELECT *
WHERE { ?s ?p ?o . }`:

**REQUEST**

```http
GET /data/?query=SELECT%20*%20WHERE%20%7B%20%3Fs%20%3Fp%20%3Fo%20.%20%7D HTTP/1.1
Host: example.org
```

**RESPONSE**

```http
HTTP/1.1 200 OK
```
```json
{
  "head": {
    "vars": [ "s", "p", "o" ]
  },
  "results": {
    "ordered" : false,
    "distinct" : false,
    "bindings" : [
      {
        "s" : { "type": "uri", "value": "https://example.org/data/" },
        "p" : { "type": "uri", "value": "http://www.w3.org/1999/02/22-rdf-syntax-ns#type" },
        "o" : { "type": "uri", "value": "http://www.w3.org/ns/ldp#BasicContainer" }
      },
      {
        "s" : { "type": "uri", "value": "https://example.org/data/" },
        "p" : { "type": "uri", "value": "http://purl.org/dc/terms/title" },
        "o" : { "type": "literal", "value": "Basic container" }
      }
    ]
  }
}
```

## Creating content

When creating new resources (directories or documents) using LDP, the client
must indicate the type for the new resource that is going to be created. LDP
uses `Link` headers with specific URI values, which in turn can be dereferenced
to obtain additional information about each type of resource. Currently, our LDP
implementation supports only [Basic Containers](http://www.w3.org/TR/ldp/#ldpbc).

LDP also offers a mechanism through which clients can provide a preferred name
for the new resource through a header called `Slug`.

### Creating containers (directories)

To create a new *basic container* resource, the Link header value must be set
to the following value:
`Link: <http://www.w3.org/ns/ldp#BasicContainer>; rel="type"`

For example, to create a basic container called `data` under
`https://example.org/`, the client will need to send the following `POST` request,
with the `Content-Type` header set to `text/turtle`:

**REQUEST**

```http
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Link: <http://www.w3.org/ns/ldp#BasicContainer>; rel="type"
Slug: data
```
```ttl
<> <http://purl.org/dc/terms/title> "Basic container" .
```

**RESPONSE**

```http
HTTP/1.1 201 Created
```

### Creating documents (files)

To create a new resource, the `Link` header value must be set to the following
value:
`Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"`

For example, to create a resource called `test` under
`https://example.org/data/`, the client will need to send the following `POST`
request, with the `Content-Type` header set to `text/turtle`:

**REQUEST**

```http
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
Slug: test
```
```ttl
<> <http://purl.org/dc/terms/title> "This is a test file" .
```

**RESPONSE**

```http
HTTP/1.1 201 Created
```

More examples can be found in the LDP [Primer document](http://www.w3.org/TR/ldp-primer/).

#### HTTP `PUT` to create

An alternative (though not standard) way of creating new resources is to use
HTTP `PUT`. Although HTTP `PUT` is commonly used to overwrite resources, this way is
usually preferred when creating new non-RDF resources (i.e., using a MIME type
other than `text/turtle`).

**REQUEST**

```http
PUT /picture.jpg HTTP/1.1
Host: example.org
Content-Type: image/jpeg
...
```

**RESPONSE**

```http
HTTP/1.1 201 Created
```

Another useful Solid feature that is not yet part of the LDP spec deals with
using HTTP `PUT` to *recursively* create new resources. This feature is really
useful when the client wants to make sure it has absolute control over the URI
namespace -- e.g., migrating from one personal data store to another. Although
this feature is defined in HTTP1.1
[RFC2616](https://tools.ietf.org/html/rfc2616), we decided to improve it
slightly by having servers create the full path to the resource (including
intermediate containers), if it didn't exist before. (If you're familiar with
the Unix command line, think of it as `mkdir -p`.) For instance, a calendar app
uses a URI pattern (structure) based on dates when storing new events (i.e.,
`yyyy/mm/dd`). Instead of performing several POST requests to create a month and a
day container when switching to a new month, it could send the following request
to create a new event resource called `event1`:

REQUEST:

```http
PUT /2015/05/01/event1 HTTP/1.1
Host: example.org
```

RESPONSE:

```http
HTTP/1.1 201 Created
```

This request would then create a new resource called `event1`, as well as the
missing intermediate resources -- containers for the month `05` and the day `01`
under the parent container `/2015/`.

To avoid accidental overwrites, Solid servers must support `ETag` checking
through the use of [`If-Match` or
`If-None-Match`](https://tools.ietf.org/html/rfc2616#section-14.24) HTTP headers.

***IMPORTANT:** Using `PUT` to create standalone containers is not supported,
because the behavior of `PUT` (overwrite) is not well defined for containers. You
MUST use `POST` (as defined by LDP) to create containers alone.*

#### Alternative: Using SPARQL

To write data, clients can send an HTTP `PATCH` request with a SPARQL payload to
the resource in question. If the resource doesn't exist, it should be created
through an LDP `POST` or through a `PUT`.

For instance, to update the `title` of the container from the previous example,
the client would have to send a `DELETE` statement, followed by an `INSERT`
statement. Multiple statements (delimited by a `;`) can be sent in the same
`PATCH` request.

**REQUEST**

```http
PATCH /data/ HTTP/1.1
Host: example.org
Content-Type: application/sparql-update
```
```sparql
DELETE DATA { <> <http://purl.org/dc/terms/title> "Basic container" };
INSERT DATA { <> <http://purl.org/dc/terms/title> "My data container" }
```

**RESPONSE**

```http
HTTP/1.1 200 OK
```

***IMPORTANT:** There is currently no support for blank nodes nor RDF lists in
our SPARQL patches.*

### Discovering server capabilities - the `OPTIONS` method

Returns a list of headers describing the server's capabilities (with an 
asterisk \[`*`\] as the `request-target`), or the subset of those capabilities 
which are made available on a specific resource (if specified as the
`request-target`).

**REQUEST**

```http
OPTIONS * HTTP/1.1
Host: example.org
```

**RESPONSE**

```http
HTTP/1.1 200 OK
Accept-Patch: application/json, application/sparql-update
Accept-Post: text/turtle, application/ld+json
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: OPTIONS, HEAD, GET, PATCH, POST, PUT, DELETE
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: User, Triples, Location, Link, Vary, Last-Modified, Content-Length
Allow: OPTIONS, HEAD, GET, PATCH, POST, PUT, DELETE
MS-Author-Via: DAV, SPARQL
```

**REQUEST**

```http
OPTIONS /data/ HTTP/1.1
Host: example.org
```

**RESPONSE**

```http
HTTP/1.1 200 OK
Accept-Patch: application/sparql-update
Accept-Post: text/turtle
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: OPTIONS, HEAD, GET, PATCH, POST
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: User, Triples, Location, Link, Vary, Last-Modified, Content-Length
Allow: OPTIONS, HEAD, GET, PATCH, POST
MS-Author-Via: SPARQL

```
