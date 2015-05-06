# SoLiD - Social Linked Data platform

[![Join the chat at https://gitter.im/linkeddata/SoLiD](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/linkeddata/SoLiD)

## Table of contents

 1. [Quick intro](#quick-intro)
 2. [Brief example of SoLiD in action](#brief-example-of-solid-in-action)
 3. [RDF](#rdf)
 4. [Reading and writing data using LDP](#reading-and-writing-data-using-ldp)
 5. [Reading and writing data using SPARQL](#reading-and-writing-data-using-sparql)
 6. [CORS](#cors---cross-origin-resource-sharing)
 7. [Live updates](#live-updates)
 8. [Identity management](#identity-management-based-on-webid)
 9. [Authentication](#authentication)
 10. [Access control](#access-control)
 11. [Software implementing SoLiD](#software-implementing-solid)


## Quick intro

SoLiD is a proposed set of conventions for building decentralized social applications on the Linked Data stack.  This document contains design notes on the individual components used, intended to be a guide for developers who plan to build servers or applications.

SoLiD is modular and extensible. It relies as much as possible on existing [W3C](http://www.w3.org/) standards.

SoLiD applications are somewhat like multiuser applications where instances talk to each other through a shared filesystem, and the Web is that filesystem.

Features:

1. Servers are application-agnostic, so that new applications can be developed without needing to modify servers.  For example, even though LDP 1.0 contains nothing specific to "social", many of the SocialWG User Stories can be implemented using only **application logic**, with no need to change code on the server.  The design ideal is to keep a small standard data management core and extend it as necessary to support increasingly powerful classes of applications. 
2. The basic protocol is REST, as refined by LDP with minor extensions.   New items are created in a *container* (which could be called a collection or directory) by sending them to the container URL with an HTTP POST or issuing an HTTP PUT within its URL space.  Items are updated with HTTP PUT or HTTP PATCH.  Items are removed with HTTP DELETE.  Items are found using HTTP GET and following links.   A GET on the container returns an enumeration of the items in the container.
3. The data model is RDF.  This means the data can be transmitted in various syntaxes like Turtle, JSON-LD (JSON with a "context"), or RDFa (HTML attributes).  RDF is REST-friendly, using URLs everywhere, and it provides **decentralized extensibility**, so that a set of applications can cooperate in sharing a new kind of data without needing approval from any central authority.


## Brief example of SoLiD in action

This example is taken from W3C's [Social Web WG](http://www.w3.org/wiki/Socialwg/) user stories, where it is called ["user posts a note"](http://www.w3.org/wiki/Socialwg/Social_API/User_stories#User_posts_a_note):

 1. Eric writes a short note to be shared with his followers.
 2. After posting the note, he notices a spelling error. He edits the note and re-posts it.
 3. Later, Eric decides that the information in the note is incorrect. He deletes the note.

Here is how SoLiD would handle the three steps, using [curl](http://curl.haxx.se/) as the client application:

1) Eric writes a short note to be shared with his followers. The *Slug* header is optional but useful for controlling the resulting URL.
```
curl -H"Content-Type: text/turtle" \
     -H"Slug: social-web-2015" \
     -X POST \
     --data ' @prefix as: <http://www.w3.org/ns/activitystreams#>. <> a as:Note; as:content "Going to Social Web WG".' \
     https://eric.example.org/notes/
```

The URL of the new note can be found in the *Location* header returned by the server.  In this example it is likely to be: https://eric.example.org/notes/social-web-2015

2) After posting the note, he notices a spelling error. He edits the note and re-posts it. SoLiD servers can handle updates in two different ways: PUT (overwrite) or PATCH with *sparql-update* content type.

Use HTTP PUT, when you just want to replace the data:
```
curl -H"Content-Type: text/turtle" -X PUT --data ' @prefix as: <http://www.w3.org/ns/activitystreams#>. <> a as:Note; as:content "Going to Social Web WG in Paris".' https://eric.example.org/notes/social-web-2015
```

Or you can use HTTP PATCH with SPARQL if you only want to change certain parts of the resource, leaving the others unchanged (perhaps because other applications are modifying them):

```
curl -H"Content-Type: application/sparql-update" \
     -X PATCH \
     --data 'DELETE DATA {<> <http://www.w3.org/ns/activitystreams#content> "Going to Social Web WG" .}; INSERT DATA {<> <http://www.w3.org/ns/activitystreams#content> "Going to Social Web WG in Paris" .} ' \
     https://eric.example.org/notes/social-web-2015
```

If no match is found for the triple to DELETE, the request safely aborts without changing any data.

3) Later, Eric decides that the information in the note is incorrect. He deletes the note.

```
curl -X DELETE https://eric.example.org/notes/social-web-2015
```

Note that all three actions have been performed through RESTful HTTP requests.

In these example, data was sent to the server using text/turtle (which is mandated in LDP), but other content types (such as JSON-LD) could be used if implemented by servers.

More examples of user stories can be found [here](https://github.com/linkeddata/SoLiD/tree/master/UserStories).

## RDF
The Resource Description Framework (RDF) is a framework for representing information in the Web [[RDF1.1](http://www.w3.org/TR/rdf11-concepts/)], originally designed as a graph-based data model, where the core structure of the abstract syntax is a set of triples, each consisting of a subject, a predicate and an object.

SoLiD uses several serialization syntaxes for storing and exchanging RDF such as [Turtle](http://www.w3.org/TR/turtle/) and [JSON-LD](http://www.w3.org/TR/json-ld/). When creating new RDF resources, the preferred **default** serialization is Turtle. SoLiD-compliant servers should implement content negotiation in order to handle different serialization formats.

## Reading and writing data using LDP
To simplify data portability, we opted for a design that follows the classic POSIX standards. For instance, resources are stored directly on the file system instead of using a database. This allows people to read/write data both using desktop applications as well as Web-based ones, and also to share the same disk drive between different machines/servers.

For this precise reason, we decided to use a fairly recent spec proposed by the W3C, called Linked Data Platform. 

The [LDP specification](http://www.w3.org/TR/ldp/) defines a set of rules for HTTP operations on Web resources, some based on RDF, to provide an architecture for reading and writing Linked Data on the Web. The most important feature of LDP is that it provides us with a standard way of RESTfully writing resources (documents) on the Web, without having to rely on less flexible conventions (APIs) based around sending form-encoded data using POST.

To find out how LDP works, you can take a look at the examples in the LDP [Primer document](http://www.w3.org/TR/ldp-primer/). 

However, we have found that while LDP is usually sufficient, there are some cases where it doesn't cover some of our use cases, and therefore we had to extend it.

### Getting information on resources and server capabilities

Our existing servers support the following HTTP methods for reading data:

####The HEAD method
Returns a list of headers related to the resource in question. Among these headers, two very important Link headers contain pointers to corresponding ACL and metadata resources. More information on naming conventions for these resources can be found [here](#wac).

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

#### The OPTIONS method
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
Access-Control-Allow-Methods: OPTIONS, HEAD, GET, PATCH, POST, PUT, DELETE
Access-Control-Allow-Origin: *
Access-Control-Expose-Headers: User, Triples, Location, Link, Vary, Last-Modified, Content-Length
Allow: OPTIONS, HEAD, GET, PATCH, POST, PUT, DELETE
```

### Resources names and extensions
A very important aspect of SoLiD revolves around naming resources, and keep the namespaces consistent across both the Web and local file systems.

Our motivation is threefold:

 1. Aspect: app developers care about URLs -- i.e. `https://example.org/posts/1` instead of `https://example.org/posts/1.ttl`
 2. Portability: resources should still resolve after being exported/imported into different servers. For instance, if Server A decides to store RDF resources as Turtle files, then Server B needs to be able to understand and use those resources (including doing content negotiation on them -- i.e. serve JSON-LD even though the resource is stored as a Turtle file).
 3. Direct mapping: the URLs map directly to the file system resources -- i.e. `https://example.org/test.ttl` maps to `/home/user/www/test.ttl`


### Creating new resources
When creating new resources (directories or documents) using LDP, the client must indicate the type of the new resource that is going to be created. LDP uses Link headers with specific URI values, which in turn can be dereferenced to obtain additional information about each type of resource. Currently, our LDP implementation supports only [Basic Containers](http://www.w3.org/TR/ldp/#ldpbc).

LDP also offers a mechanism through which clients can provide a preferred name for the new resource through a header called **Slug**.

#### Creating containers (directories)
To create a new **basic container** resource, the Link header value must be set to the following value:
`Link: <https://www.w3.org/ns/ldp#BasicContainer>; rel="type"`

For example, to create a basic container called **data** under https://example.org/, the client will need to send the following POST request, with the Content-Type header set to `text/turtle`:

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

For example, to create a resource called **test** under https://example.org/data/, the client will need to send the following POST request, with the Content-Type header set to `text/turtle`:

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

More examples can be found in the LDP [Primer document](http://www.w3.org/TR/ldp-primer/).

An alternative, though not standard way of creating new resources is to use HTTP PUT. Although HTTP PUT is commonly used to overwrite resources, this way is usually preferred when creating new non-RDF resources (i.e. using a mime type different than *text/turtle*).

REQUEST:
```
PUT / HTTP/1.1
Host: example.org
Content-Type: image/jpeg
...
```
RESPONSE :
```
HTTP/1.1 201 Created
```

#### Handling metadata for non-RDF resources
The metadata (i.e. extra RDF triples such as types, titles, comments, etc.) about non-RDF resources (e.g. containers, images, binaries, etc.) will be stored in a corresponding *meta* resources. SoLiD servers use a specific naming convention when referring to these meta resources. Basically, every non-RDF resource may have a corresponding metadata resource, with a name composed of the resource's name together with a **.meta** suffix.

For example, the corresponding metadata resource for the newly created container will be accessible through this URI: `https://example.org/data/.meta`. Alternatively, the photo at `https://example.org/data/image.jpg` will have it's metadata stored in `https://example.org/data/image.jpg.meta`.

Metadata resources are a "special" type of resources, which are not publicly listed by the server when browsing files (typically when doing a GET on an LDP container). However, they can still be modified by client apps using the methods described in this section. The corresponding metadata resources are advertised and can be discovered when doing HTTP GET/HEAD on regular resources, as mentioned before.

### Reading resources
Resources can be commonly accessed (i.e. read) using HTTP GET requests. SoLiD servers are encouraged to perform content negotiation for RDF resources, depending on the value of the *Accept* header.

**IMPORTANT:** a default **text/turtle** Content-Type will be used for requests for RDF resources or views (e.g. containers) that do not have an *Accept* header.

## Reading and writing data using SPARQL

Another possible way of reading and writing data is to use SPARQL. Currently, our SoLiD servers support a subset of [SPARQL 1.0](http://www.w3.org/TR/rdf-sparql-query/), where each resource is its own SPARQL endpoint, accepting basic SELECT, INSERT and DELETE statements.

### Reading data using SPARQL

To read (query) a resource, the client can send a SPARQL SELECT through a form-encoded HTTP GET request. The server will use the given resource as the default graph that is being queried. The resource can be an RDF document or even a container. The response will be serialized using `application/json` mime type.

For instance, the client can send the following form-encoded query `SELECT * WHERE { ?s ?p ?o . }`:

REQUEST:
```
GET /data/?query=SELECT%20*%20WHERE%20%7B%20%3Fs%20%3Fp%20%3Fo%20.%20%7D HTTP/1.1
Host: example.org
```
RESPONSE:
```
HTTP/1.1 200 OK

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

### Extensions to LDP

#### Globbing (inlining on GET)

We have found that in some cases, using the existing LDP features was not enough. For instace, to optimize certain applications we needed to aggregate all resources from a container and retrieve them with a single GET operation. We implemented this feature on the servers and decided to call it "globbing". Simiar to UNIX shell globbing, doing a GET on any URI which ends with a * will return an aggregate of all the resources that match the indicated pattern. For instance, if one would like to fetch all resources of a container in one request, they could do a GET on https://example.org/data/*. The aggregation process is not recursive, therefore it will not apply to children containers.

#### HTTP PUT to create

Another useful feature that is not yet part of LDP deals with using HTTP PUT to create new resources. This feature is really useful when the clients wants to make sure it has absolute control over the URI namespace -- e.g. migrating from one pod to another. Although this feature is defined in HTTP1.1 [RFC2616](https://tools.ietf.org/html/rfc2616), we decided to improve it slightly by having servers create the full path to the resource, if it didn't exist before. For instance, a calendar app uses a URI pattern (structure) based on dates when storing new events (i.e. yyyy/mm/dd). Instead of performing several POST requests to create a month and a day container when switching to a new month, it could send the following request to create a new event resource called \textit{event1}:

REQUEST:
```
PUT /2015/05/01/event1 HTTP/1.1\par
Host: example.org
```
RESPONSE:
```
HTTP/1.1 201 Created
```

This request would then create a new resource called *event1*, as well as the missing month (i.e. 05) and day (i.e. 01) containers under /2015/.

To avoid accidental overwrites, SoLiD servers must support ETag checking through the use of [If-Match or If-None-Match](https://tools.ietf.org/html/rfc2616#section-14.24) HTTP headers.

### Writing/deleting data using SPARQL
To write data, clients can send an HTTP PATCH request with a SPARQL payload to the resource in question. If the resource doesn't exist, it should be created through an LDP POST or through a PUT.

For instance, to update the *title* of the container from the previous example, the client would have to send a DELETE statement, followed by an INSERT statement. Multiple statements  (delimited by a **;**) can be sent in the same PATCH request.

REQUEST:
```
PATCH /data/ HTTP/1.1
Host: example.org
Content-Type: application/sparql-update

DELETE DATA { <> <http://purl.org/dc/terms/title> "Basic container" };
INSERT DATA { <> <http://purl.org/dc/terms/title> "My data container" }
```
RESPONSE:
```
HTTP/1.1 200 OK
```

**IMPORTANT:** There is currently no support for blank nodes and RDF lists in our SPARQL patches.

## CORS - Cross Origin Resource Sharing
There are two different ways CORS support must be implemented on SoLiD servers. First, when the request is sent through a browser that sets the Origin header. And second, when clients do not set an Origin header (e.g. curl or non-browser clients).

When the Origin header is set:

1. Client (browser) loads an app from https://app.org and wants to send an XHR (ajax) request to the server at https://example.org. Before sending the request over the wire, the browser adds the Origin header: `Origin: https://app.org`, which corresponds to the domain from where the app was loaded.

2. The server running on https://example.org receives the request and looks at the Origin header. It sees https://app.org, stores the value and handles the request.

3. The server responds to the request and sets the value of the request Origin header to the CORS header in the HTTP response:
     Access-Control-Allow-Origin: https://app.org
 
Without an Origin header:

1. A curl request is sent from the terminal to https://example.org. Unless explicitly specified though a curl parameter, the Origin header will not be set.

2. The server running on https://example.org receives the request and does not find an Origin header.

3. The server responds to the request and sets a default "all" value for the Access-Control-Allow-Origin header in the HTTP response:
     Access-Control-Allow-Origin: *

The star character (*) signifies "allow all". If you want to learn more about CORS, please visit this page: http://enable-cors.org/

## Live updates
### Websockets
Live updates are currently only supported through websockets. There are two ways in which clients can be notified in real time of changes affecting a give resource. One possible way is through a subscription mechanism, while the other way is through SPARQL patches.

#### PubSub notifications
The PubSub system is very basic. Clients only need to open a websocket connection and *sub*(scribe) to a given resource URI. If any change occurs in that resource, a *pub*(lish) event will be sent to all the subscribed clients. 

The websocket server URI is the same for any resource located on a given data space (same hostname). To discover the URI of the websocket server, clients can send an HTTP OPTIONS. The server will then include an **Updates-Via** header in the response:

REQUEST:
```
OPTIONS /data/test HTTP/1.1
Host: example.org
```
RESPONSE:
```
HTTP/1.1 200 OK
...
Updates-Via: wss://example.org/
```

To subscribe to a resource, clients will need to send the keyword **sub** followed by an empty space and then the URI of the resource:

```sub https://example.org/data/test```

If a change occurs and the client is subscribed to that resource, it will receive a websocket message composed of the keyword **pub**, followed by an empty space and the URI of the resource that has changed:

```pub https://example.org/data/test```

Here is a Javascript example on how to subscribe to live updates for our *test* resource at `https://example.org/data/test`:

```
var socket = new WebSocket('wss://example.org/');
socket.onopen = function() {
	this.send('sub https://example.org/data/test');
};
socket.onmessage = function(msg) {
	if (msg.data && msg.data.slice(0, 3) === 'pub') {
        // resource updated, refetch resource
	}
};
```

#### SPARQL patches

@@@TODO 

## Identity management based on WebID
Identity management as well as unique identifiers are the core of any social system. SoLiD uses [WebID](http://www.w3.org/2005/Incubator/webid/spec/identity/), an HTTP(S) URI, to uniquely refer to users (people or agents). The advantage of WebID is that the URI can be dereferenced to a WebID profile document, in order to reveal useful information about the user. Also, since WebID profiles can be hosted anywhere (including your basement server), users are no longer trapped inside Identity Provider Silos (e.g. Twitter, Facebook, Google+, etc.).

### Creating new accounts

#### Client - server API
SoLiD-compliant servers must implement a very simple API, indicating whether an account name is available or not on the server. Clients (e.g. the signup Web component) send an HTTP POST request containing the following JSON structure, where *accountName* contains the target account name (e.g. a preferred username):

```
{
	method:		 "accountStatus",
	accountName: "alice"
}
```

The server response has to contain the following JSON structure:

```
{
	method:   "accountStatus",
	status:	  "success",
	formURI:  "https://example.org/api/spkac",
	loginURI: "https://example.org/",
	response: {
				accountURL: "https://user.example.org/",
				available:	 true
			}
}
```

**Attention!** Because creating client certificates requires the &lt;KEYGEN&gt; HTML element, which does not work with AJAX requests, the client must submit a form to the *formURI* it receives from the server. This restriction means that a predefined set of form element names must be respected on the server. Here is a list of form element names  (case sensitive!) that are sent by the signup component:

 * `spkac` - SPKAC containing the public key generated by the KEYGEN element
 * `username` - account user name
 * `name` - user's full name
 * `email` - user's email address
 * `img` - user's picture URL

Finally, if the *status* indicates success, and the *available* flag is set to *true*, then the form containing user details can be submitted, using the URI provided by the server (i.e. the value of *formuri*).

The value of *loginURI* is used to indicate where the app can fake a WebID-TLS login, in order to find the user's WebID.

Once the WebID certificate is installed in the browser, the user is presented with a button that finishes setting up the account when clicked. At the end, the user ends up with a series of default workspaces, access control policies (ACLs) for them, and also a *preferences* document (file).

### Personal data workspaces
Upon account creation, a series of dedicated workspaces (i.e. LDP containers) are created in the user's data space, together with their corresponding ACL resources. At the moment, the list contains the following workspaces:

 * Public
 * Private
 * Work
 * Family
 * Friends
 * Preferences

You can consider workspaces to be dedicated containers, which store application-specific data. For example, one of the reasons we decided to use this concept of workspaces is that complicated ACL logic can be set per workspace, and then all data inside the workspace will inherit the same policies.

### Preferences document
The *preferences* document is used to describe useful information about the user and the data server, which can later on be used by applications. This resource currently lists the workspaces that were just created. In the future it may contain user preferences such as a preferred language, date format, etc.

By default, the preferences resource will be created in the *Preferences* workspaces -- i.e. `https://user.example.org/Preferences/prefs`.

A triple pointing to the preferences file will also be added to the user's WebID profile.

## Authentication
### WebID-TLS
The WebID-TLS protocol ([W3C draft](http://www.w3.org/2005/Incubator/webid/spec/tls/)) enables secure, efficient authentication on the Web. It enables users to authenticate onto any site by simply choosing one of the certificates proposed to them by their browser. These certificates can be created by any Web Site for any purpose. A user may have multiple client certificates, bound to one or multiple WebIDs.

Basically, WebID-TLS relies on matching a public key received from a client certificate, to the public key published in the WebID profile obtained by dereferencing the WebID included in the SubjectAlternativeName field of the client certificate. In other words, users must prove they own a public key they publish in their WebID profiles.

WebID-TLS is currently the preferred authentication mechanism in SoLiD. 

More information on WebID-TLS can be found here: http://www.w3.org/2005/Incubator/webid/spec/tls/.

### WebID-RSA
WebID-RSA is somehow similar to WebID-TLS, in that a public RSA key is published in the WebID profile, and the user will sign a token with the corresponding private key that matches the public key in the profile.

The client receives a secure token from the server, which it signs and then sends back to the server. The implementation of WebID-RSA is somehow similar to [Digest access authentication](https://tools.ietf.org/html/rfc2617) in HTTP, in that it reuses the same headers.

Here is a step by step example that covers the authentication handshake. 

First, the client attempts to access a protected resource at `https://example.org/data/`.

REQUEST:
```
GET /data/ HTTP/1.1
Host: example.org
```
RESPONSE:
```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: WebID-RSA source="example.org", nonce="securestring"
```

Next, the client sets the username value to the user's WebID and signs the concatenated value of **source + username + nonce** before resending the request. It is important that clients return the proper source value they received from the server, in order to avoid main-in-the-middle attacks.

REQUEST:
```
GET /data/ HTTP/1.1
Host: example.org
Authorization: Digest source="example.org",
                      username="https://alice.example.org/card#me", 
                      nonce="securestring",
                      sig="signatureOverUsernamePlusNonce"
```
RESPONSE:
```
HTTP/1.1 200 OK
```

One important advantage of WebID-RSA over WebID-TLS is that keys can be generated on the fly to sign and encrypt data. The way client certificate management is currently implemented in browsers, it does not offer the means to access keys inside certificates, for purposes other than authentication.

@@TODO: the server must send it's URI together with the token, otherwise a MitM can forward the claim to the client. Also, clients will also have to return the same server URI.

@@TODO: Instead of sending the WebID during the response, the client could directly send the URI of the public key that is need in order to verify the claim. For instance, Alice could list public keys in her own profile, using fragment identifiers (e.g. <#key1>):

```
....
<#me> cert:key <#key1>, <#key2> .

<#key1> a cert:RSAPublicKey;
        cert:modulus "00cb24ed85d64d794b..."^^xsd:hexBinary;
        cert:exponent 65537  .
```

The client would then send the following response:

```
GET /data/ HTTP/1.1
Host: example.org
Authorization: Digest keyuri="https://alice.example.org/card#key1", 
                      nonce="securestring",
                      sig="signatureOverUsernamePlusNonce"
```

The server would then be able to link the key that was used to sign the response to the user that owns it.

## Access Control
### Web Access Control
Web Access Control (WAC) is a decentralized system that allows different users and groups various forms of access to resources where users and groups are identified by HTTP URIs. The system is similar to the access control system used within many file systems except that the documents controlled, the users and the groups are all identified by URIs. Users are identified by WebIDs. Groups of users are identified by the URI of a class of users which, if you look it up, returns a list of users in the class. This means a WebID hosted by any server can be a member of a group hosted some other server.

**IMPORTANT:** Users do not need to have an account (i.e. WebID) on a given server to have access to documents on it.

Same as for metadata resources, ACL resources are not publicly listed by the server when browsing files (typically when doing a GET on an LDP container). However, they can still be read/written by client apps using the above mentioned ways of writing data. The corresponding ACL resources are advertised and can be discovered when doing HTTP GET/HEAD on regular resources.

Similar to the metadata resource naming convention, SoLiD servers use a specific naming convention for ACL resources. This convention relies on appending a **.acl** suffix to its corresponding resource.

For example, the container `https://example.org/data/` will have a corresponding ACL resource with the URI: `https://example.org/data/.acl`. A resource `https://example.org/data/test` will have a corresponding ACL resource at `https://example.org/data/test.acl`

More information on Web Access Control can be found here: https://www.w3.org/wiki/WebAccessControl.

@@TODO: add section on delegation

# Software implementing SoLiD
## Servers

Name | Maintained | LDP | Cors | WebID provider | WebID-TLS | WebID-RSA | WebID-Delegation | WAC   
-----|------------|-----|------|----------------|-----------|-----------|------------------|----
[rww-play](https://github.com/read-write-web/rww-play)|Yes|Basic Containers, file storage, SEARCH|Proxy|Yes|Yes|No|N/A|Yes
[gold](https://github.com/linkeddata/gold)|Yes|Basic Containers, file storage|Proxy|Yes|Yes|Yes|Yes|Yes
[ldphp](https://github.com/linkeddata/ldphp)|No|Basic Containers, file storage|Proxy|Yes|Yes|No|No|Yes
[ld-node](https://github.com/linkeddata/node-ldp-httpd)|Yes|In progress, file storage|N/A|Yes|Yes|No|No|Yes
[meccano](https://github.com/Qatar-Computing-Research-Institute/qcri-crosscloudP/tree/meccano)|Yes|Basic Containers (adaptor), SPARQL store|No|No|No|No|No|Partial

## Applications
 - Warp -- https://github.com/linkeddata/warp
 - Profile editor -- https://github.com/linkeddata/profile-editor
 - Cimba -- https://github.com/linkeddata/cimba
 - Meeting scheduler -- https://github.com/linkeddata/app-schedule
 - Contacts manager -- https://github.com/mzereba/contacts
 - Todo list -- https://github.com/mzereba/todo


