# SoLiD - Social Linked Data platform
This document contains design notes on individual components used by SoLiD. They are intended to be a guide for developers who plan to build social linked data servers and applications.

# Software implementing SoLiD


# Components
SoLiD was designed from the ground up to be modular and openly extensible. It relies as much as possible on existing [W3C](http://www.w3.org/) standards, except in one case in which it extends the [LDP standard](http://www.w3.org/TR/ldp/). Among the technologies it uses, you can find:

 - [RDF](#rdf)
 - [WebID](#webid)
 - [WebID-TLS](#webid-tls)
 - [WebID-RSA](#webid-tls)
 - [WebAccessControl](#wac)
 - [LDP](#ldp)

## <a name="rdf"></a>RDF
The Resource Description Framework (RDF) is a framework for representing information in the Web [[RDF1.1](http://www.w3.org/TR/rdf11-concepts/)], originally designed as a graph-based data model, where the core structure of the abstract syntax is a set of triples, each consisting of a subject, a predicate and an object.

SoLiD uses several serialization syntaxes for storing and exchanging RDF such as [Turtle](http://www.w3.org/TR/turtle/) and [JSON-LD](http://www.w3.org/TR/json-ld/). When creating new RDF resources, the preferred **default** serialization is Turtle. SoLiD-compliant servers should implement content negotiation in order to handle different serialization formats.

## Reading and writing data using LDP
To simplify data portability, we opted for a design that follows the classic POSIX standards. For instance, resources are stored directly on the file system instead of using a database. This allows people to read/write data both using desktop applications as well as Web-based ones, and also to share the same disk drive between different machines/servers.

For this precise reason, we decided to use a fairly recent spec proposed by the W3C, called Linked Data Platform. 

<a name="ldp"></a>The [LDP specification](http://www.w3.org/TR/ldp/) defines a set of rules for HTTP operations on Web resources, some based on RDF, to provide an architecture for reading and writing Linked Data on the Web. The most important feature of LDP is that it provides us with a standard way of RESTfully writing resources (documents) on the Web, without having to rely on less flexible conventions (APIs) based around sending form-encoded data using POST.

To find out how LDP works, you can take a look at the examples in the LDP [Primer document](http://www.w3.org/TR/ldp-primer/). 

However, we have found that while LDP is usually sufficient, there are some cases where it doesn't cover some of our use cases, and therefore we had to extend it.

### Getting information on resources and server capabilities

Our existing servers support the following HTTP methods for reading data:

####HEAD
Returns a list of headers related to the resource in question. Among these headers, two very important Link headers contain pointers to corresponding ACL and metadata resources:
REQUEST:
```
HEAD /data/ HTTP/1.1
Host: example.org
```
RESPONSE:
```
HTTP/1.1 200 OK
....
Link: <https://example.org/data/.acl>; rel="acl"
Link: <https://example.org/data/.meta>; rel="describedby"
```

####OPTIONS
Returns a list of headers describing the server's capabilities.

REQUEST:
```
OPTIONS /data/ HTTP/1.1
Host: example.org
```
RESPONSE:
```
HTTP/1.1 200 OK
Accept-Patch: application/json, application/sparql-update
Accept-Post: text/turtle, application/ld+json
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: OPTIONS, HEAD, GET, PATCH, POST, PUT, MKCOL, DELETE, COPY, MOVE, LOCK, UNLOCK
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: User, Triples, Location, Link, Vary, Last-Modified, Content-Length
Allow: OPTIONS, HEAD, GET, PATCH, POST, PUT, MKCOL, DELETE, COPY, MOVE, LOCK, UNLOCK
```

### Creating new resources
When creating new resources (directories or documents) using LDP, the client must indicate the type of the new resource that is going to be created. LDP uses Link headers with specific URI values, which in turn can be dereferenced to obtain additional information about each type of resource. Currently, our LDP implementation supports only [Basic Containers](http://www.w3.org/TR/ldp/#ldpbc).

LDP also offers a mechanism through which clients can provide a preferred name for the new resource through a header called **Slug**.

#### Creating containers (directories)
To create a new **basic container** resource, the Link header value must be set to the following value:
`Link: <http://www.w3.org/ns/ldp#BasicContainer>; rel="type"`

For example, to create a basic container called **data** under http://example.org/, the client will need to send the following POST request, with the Content-Type header set to `text/turtle`:

REQUEST:
```
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Slug: data

<> a <http://www.w3.org/ns/ldp#BasicContainer> ;
   <http://purl.org/dc/terms/title> "Basic container" .
```
RESPONSE:
```
HTTP/1.1 201 Created
```

#### Creating documents (files)
To create a new resource, the Link header value must be set to the following value:
`Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"`

For example, to create a resource called **test** under http://example.org/data/, the client will need to send the following POST request, with the Content-Type header set to `text/turtle`:

REQUEST:
```
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Slug: test

<> a <http://www.w3.org/ns/ldp#Resource> ;
   <http://purl.org/dc/terms/title> "This is a test file" .
```
RESPONSE:
```
HTTP/1.1 201 Created
```

## Reading and writing data using SPARQL

Another possible way of reading and writing data is to use SPARQL. Currently, our SoLiD servers support a subset of [SPARQL 1.0](http://www.w3.org/TR/rdf-sparql-query/), where each resource is its own SPARQL endpoint, accepting basic SELECT, INSERT and DELETE statements.

### Reading data using SPARQL

To read (query) a resource, the client can send a SPARQL SELECT query through an HTTP GET request. The server will use the given resource as the default graph that is being queried. The response will be serialized using `application/json` mime type.

For instance, the client can send the following form-encoded query:

REQUEST:
```
GET /data/ HTTP/1.1
Host: example.org
Query: SELECT * WHERE { ?s ?p ?o . }
```
RESPONSE:
```
{
  "head": {
    "vars": [ "s", "p", "o" ]
  },
  "results": {
    "ordered" : false,
    "distinct" : false,
    "bindings" : [
      {
        "s" : { "type": "uri", "value": "http://example.org/data/" },
        "p" : { "type": "uri", "value": "http://www.w3.org/1999/02/22-rdf-syntax-ns#type" },
        "o" : { "type": "uri", "value": "http://www.w3.org/ns/ldp#BasicContainer" }
      },
      {
        "s" : { "type": "uri", "value": "http://example.org/data/" },
        "p" : { "type": "uri", "value": "http://purl.org/dc/terms/title" },
        "o" : { "type": "literal", "value": "Basic container" }
      }
    ]
  }
}
```

### Writing/deleting data using SPARQL
To write data, clients can send an HTTP PATCH request with a SPARQL payload to the resource in question. For instance, to update the *title* of the container from the previous example, the client would have to send a DELETE statement, followed by an INSERT statement.

REQUEST:
```
PATCH /data/resource HTTP/1.1
Host: example.org

DELETE DATA { <> <http://purl.org/dc/terms/title> "Basic container" };
INSERT DATA { <> <http://purl.org/dc/terms/title> "My data container" }
```

**IMPORTANT:** There is currently no support for blank nodes and RDF lists for the SPARQL statements.

## CORS

## Live updates
### Websockets

## <a name="webid"></a>Identity management based on WebID
Identity management as well as unique identifiers are the core of any social system. SoLiD uses [WebID](http://www.w3.org/2005/Incubator/webid/spec/identity/), an HTTP(S) URI, to uniquely refer to users (people or agents). The advantage of WebID is that the URI can be dereferenced to reveal useful information about the user. Also, since WebID profiles can be hosted anywhere (including your basement server), users are no longer trapped inside Identity Provider Silos (e.g. Twitter, Facebook, Google+, etc.).

### Personal Data Workspaces

## Authentication
### <a name="webid-tls"></a>WebID-TLS

### <a name="webid-rsa"></a>WebID-RSA

## Access Control
### <a name="wac"></a>Web Access Control


Here is an example of an LDP request to create a new container in an existing one (i.e. /data/):

REQUEST:
```
POST /data/ HTTP/1.1
Host: example.org
Content-Type: text/turtle
Slug: contacts
Link: <http://www.w3.org/ns/ldp#BasicContainer>; rel="type"
   
<>  a ldp:BasicContainer;
    dc:title "Contacts container" .
```

RESPONSE:
```
HTTP/1.1 201 Created
Location: https://example.org/data/contacts/
Link: <https://example.org/data/contacts/.acl>; rel=acl
Link: <https://example.org/data/contacts/.meta>; rel=meta
```


