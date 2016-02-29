# Solid Specification
[![](https://img.shields.io/badge/project-Solid-7C4DFF.svg?style=flat-square)](https://github.com/solid/solid)
[![Join the chat at https://gitter.im/solid/solid-spec](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/solid/solid-spec)

**Disclaimer: this is a living spec. Expect it to change often!**

**Current Spec version:** `v.0.6.0` (see [CHANGELOG.md](CHANGELOG.md))

## Table of contents

* [Overview](#overview)
* [Identity](#identity)
* [Profiles](#profiles)
  * [WebID Profile Documents](#webid-profile-documents)
* [Authorization and Access Control](#web-access-control)
* [Reading](#reading)
* [Subscribing](#subscribing)
* [Creating content](#creating-content)
* [Authentication](#authentication)
* [Implementations](#implementations)
* [Example](#example)
* [Other Considerations](#other-considerations)

## Overview

[Solid](https://github.com/solid/solid) (derived from "**so**cial **li**nked
**d**ata") is a proposed set of conventions and tools for building
*decentralized social applications* based on [Linked
Data](http://www.w3.org/DesignIssues/LinkedData.html) principles. Solid is
modular and extensible. It relies as much as possible on existing
[W3C](http://www.w3.org/) standards and protocols.

See Also:

* [About Solid](https://github.com/solid/solid#about-solid)
* [Contributing to Solid](https://github.com/solid/solid#contributing-to-solid)
  * [Pre-Requisites](https://github.com/solid/solid#pre-requisites)
  * [Solid Project
      Workflow](https://github.com/solid/solid#solid-project-workflow)
* [Standards Used](https://github.com/solid/solid#standards-used)
* [Platform Notes](https://github.com/solid/solid#solid-platform-notes)
* [Solid Project directory](https://github.com/solid/solid#project-directory)

## Identity

Solid uses [WebID](http://www.w3.org/2005/Incubator/webid/spec/identity/) URIs
as universal usernames or actor identifiers. Frequently referred to simply as  
*WebIDs*, these URIs form the basis of most other Solid-related technologies,
such as authentication, authorization, access control, user profiles, discovery
of user preferences and server capabilities, and more.

WebIDs provide globally unique decentralized identifiers, enable cross-service
federated signin, prevent service provider lock-in, and give users control over
their own identity. *The WebID URI's primary function is to point to the
location of a public [WebID Profile document](#profiles) (see below).*

**Example WebIDs:** `https://alice.databox.com/profile/card#me` or
`http://somepersonalsite.com/#webid`

## Profiles

Solid uses WebID Profile Documents for management of user identity and security
credentials (such as public keys), and user preferences discovery.

Although here we mostly refer to them in the context of user profiles,
other types of actors use these profiles as well, such as groups, organizations,
devices, and software applications.

### WebID Profile Documents

A WebID URI, when dereferenced, yields a WebID Profile Document in a
Linked Data format ([Turtle](http://www.w3.org/TR/turtle/) by default, but
often available as JSON-LD or HTML+RDFa). Parsing this document provides a
client application with useful information, such as the user's name and
profile image, links to user preferences and related documents, and lists of
public key certificates or other relevant identity credentials.

**See component spec:
  [Solid WebID Profiles Specification](solid-webid-profiles.md)**

### Default Containers

Upon account creation, a series of default LDP Containers are initialized in the
user's data space, along with their corresponding ACL resources. The
particulars are left up to server implementers, but the recommended list is as
follows.

##### `/profile/` (Profile)
The container which will house the WebID profile document and its
various components/linked docs. **Default ACL:** read-public.
The Profile container is discoverable from the WebID -- a user pastes in a WebID
URL (for example, `https://accountname.databox.me/profile/card#me`), which
de-references to a profile document containing, among other things:

```
<>
    a <http://xmlns.com/foaf/0.1/PersonalProfileDocument> ;
    <http://xmlns.com/foaf/0.1/maker> <#me> ;
    <http://xmlns.com/foaf/0.1/primaryTopic> <#me> .
```

##### `/` (root)
The root/default container for the account. **Default ACL:** private
(owner only). Discoverable from profile via [pim :
space#storage](http://www.w3.org/ns/pim/space#storage) property.
Discoverable via:

```
<#me>
    <http://www.w3.org/ns/pim/space#storage> <../> ;
```

##### `/settings/` (Settings)

This is a private, protected container that houses User preferences and settings
(such as preferred language, date format and time zone, etc), the content Type
Registry, and app preferences and configs. **Default ACL:** private (owner
only). Note that individual resources *within* the Settings container may be
public (that is, override the default ACL).

Discoverable from profile via [pim :
space#preferencesFile](http://www.w3.org/ns/pim/space#preferencesFile) property.

```
<#me>
    <http://www.w3.org/ns/pim/space#preferencesFile> <../settings/preferences.ttl> ;
```

##### `/inbox/` (Inbox)

A container to serve as a default primary channel for
notifications. Note that this is complementary (and not necessarily replacing)
app-specific notification inboxes such as the one used by
[Dokieli](https://github.com/linkeddata/dokieli).
**Default ACL:** append-only by public, read by owner.

Discoverable from profile via
[solid-terms](https://github.com/solid/vocab/blob/master/solid-terms.ttl)#Inbox
property.

```
<#me>
    <http://www.w3.org/ns/solid/terms#inbox> <../inbox/> ;
```

### Creating new accounts

Before creating new accounts, client applications must be able to check whether
or not an account exists. To do that, clients only need to send a `HEAD` request
to the account root URI. For example, let's assume the user *Alice* wants to
create an account on `example.org`, using the username `alice`. The client will
perform a `HEAD` request to the `alice.example.org` subdomain.

REQUEST:

```
HEAD / HTTP/1.1
Host: alice.example.org
```

RESPONSE:

```
HTTP/1.1 200 OK
```

If the HTTP status code returned is `200`, then it means an account with that
name exists already.

If the status code returned is `404`, it means that the account is available.

Once the client application has verified that the account is available, it can
now proceed to create it. To do so, it must submit a form (or emulate it) to the
*account URI* it previously checked (e.g. alice.example.org), containing at
least the following form parameter names:

 * `username` (required) - the account name that will be used as the subdomain
  (i.e. `alice`)
 * `email` (optional) - the email of the user, which may be used for account
  recovery and/or account validation

**IMPORTANT** At this point, the server should also automatically consider the
user to be authenticated, and issue a cookie. This will allow the user to
properly manage the following steps that may require authentication.

Once submitted, the server will take charge of creating the necessary
workspaces, setting the access control policies and creating the user's WebID
profile document.

#### Issuing the client certificate

**Attention!** Because creating client certificates requires the [keygen HTML5
element](http://www.w3schools.com/tags/tag_keygen.asp),
which does not work with AJAX requests, the client must submit a form to the
**account host URI** -- i.e. `https://user.example.org/`. This restriction means
that a predefined set of form element names must be respected on the server.
Here is the minimum list of form element names (case sensitive!) that **MUST**
be sent by signup applications, in order to achieve interoperability:

 * `spkac` - contains the *certificate signing request* (CSR) generated by the
  `<keygen>` element. (see [SPKAC](https://en.wikipedia.org/wiki/SPKAC))
 * `webid` - the WebID of the user
 * `name` - the name (CN) that will be used in the certificate

The server will update the user's profile by adding a representation of the
public key (as modulus and exponent) it obtained from the certificate, according
to the [WebID-TLS specification](http://www.w3.org/2005/Incubator/webid/spec/tls/#vocabulary).

**IMPORTANT** Servers should only return the certificate in the response, while
also setting the Content-Type header to the proper mime type value (as seen
below), otherwise the certificate will fail to install in the browser.

```
Content-Type: application/x-x509-user-cert
```

Unfortunately, there is currently no browser API to discover whether or not a
certificate was properly installed in the browser.

### Finding out the identity currently used

Regardless of the authentication mechanism that was used during an HTTP request,
the server must always return a `User` header, which contains a URI representing
the user's identity. For example, the `User` header may contain either an HTTP
URI (i.e. the WebID of the user that was just authenticated), or a different URI
(`mailto:`, `dns:`, `tel:`, etc.).

The `User` header can also be used to verify that a user has successfully
authenticated (specifically, that the HTTP URI that the header contains points
to a valid WebID profile), as well as to bootstrap the way apps personalize the
user experience, since apps have an easy way of discovering the user's identity.

Here is an example:

REQUEST:

```
GET /data/ HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK
...
User: https://alice.example.org/card#me
```

## Web Access Control

[Web Access Control (WAC)](https://www.w3.org/wiki/WebAccessControl) is a
decentralized system that allows different users and groups various forms of
access to resources where users and groups are identified by HTTP URIs. The
system is similar to the access control system used within many file systems
except that the documents controlled, the users, and the groups, are all
identified by URIs. Users are identified by WebIDs. Groups of users are
identified by the URI of a class of users which, if you look it up, returns a
list of users in the class. This means a WebID hosted by any server can be a
member of a group hosted some other server.

**IMPORTANT:** Users do not need to have an account (i.e. WebID) on a given
server to have access to documents on it.

ACL resources are not publicly listed by the server when browsing files
(typically when doing a GET on an LDP container). However, they can still be
read/written by client apps using the above mentioned ways of writing data.
An ACL resource is advertised through a **Link** header having **rel="acl"** and
can be discovered when doing HTTP GET/HEAD on regular resources. The naming of
an ACL resource is arbitrary and may change from one server implementation to
another.

For example, the container `https://example.org/data/` may have a corresponding
ACL resource with the URI: `https://example.org/data/.acl`. A resource
`https://example.org/data/test` may have a corresponding ACL resource at
`https://example.org/data/test.acl`. The following is an example of a typical
request.


REQUEST:
```
GET /data/ HTTP/1.1
Host: example.org
```

RESPONSE:
```
Link: <https://example.org/data/.acl>; rel="acl"
```

WAC policies are applied to resources, instead of RDF triples. This means that
policies can be set for [LDPRs](http://www.w3.org/TR/ldp/#ldpr) as well as for
[LDPCs](http://www.w3.org/TR/ldp/#ldpc). A special case is applied to LDPCs,
where policies can be defined as "default" for everything in a container,
meaning that all the members of that specific container will inherited them.

## Reading Resources

Resources can be commonly accessed (i.e. read) using HTTP GET requests. Solid
servers are encouraged to perform content negotiation for RDF resources,
depending on the value of the `Accept` header.

**IMPORTANT:** a default `Content-Type: text/turtle` will be used for requests
for RDF resources or views (containers) that do not have an `Accept`
header.

### Content representation

The [Resource Description Framework (RDF)](http://www.w3.org/TR/rdf11-concepts/)
is a framework for representing information on the Web, It was originally
designed as a graph-based data model, where the core structure of the abstract
syntax is a set of triples, each consisting of a subject, a predicate and an
object.

Solid uses several serialization syntaxes for storing and exchanging RDF such as
[Turtle](http://www.w3.org/TR/turtle/) and
[JSON-LD](http://www.w3.org/TR/json-ld/). When creating new RDF resources, the
preferred *default* serialization is Turtle. Solid-compliant servers should
implement content negotiation in order to handle different serialization
formats.

### Objects

A very important aspect of Solid revolves around naming resources, and keep the
namespaces consistent across both the Web and local file systems.

Our motivation is threefold:

 1. Aspect: app developers care about URLs -- i.e. `https://example.org/posts/1`
  instead of `https://example.org/posts/1.ttl`

 2. Portability: resources should still resolve after being exported/imported into
  different servers. For instance, if Server A decides to store RDF resources as
  Turtle files, then Server B needs to be able to understand and use those
  resources (including doing content negotiation on them -- i.e. serve JSON-LD
  even though the resource is stored as a Turtle file).

 3. Direct mapping: the URLs map directly to the file system resources -- i.e.
  `https://example.org/test.ttl` maps to `/home/user/www/test.ttl`

Servers must support the HEAD method for reading data. This returns a list of
headers related to the resource in question. Among these headers, two very
important `Link` headers contain pointers to corresponding ACL and metadata
resources. More information on naming conventions for these resources can be
found in the [Access Control section](#access-control).

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

### Metadata

The metadata (extra RDF triples such as types, titles, comments, and so on)
about non-RDF resources (e.g. containers, images, binaries, etc.) will be stored
in a corresponding *meta* resource.

The metadata resource is a "special" type of resource, which is not publicly
listed by the server when browsing files (typically when doing a GET on an LDP
container). However, it can still be modified by client apps using the methods
described in this section. The corresponding metadata resource is advertised
through a **Link** header having **rel="describedby"**, which can be discovered
when doing HTTP GET/HEAD on regular resource, as mentioned before in case of ACLs.
The naming of a *meta* resource is also arbitrary and may change from one server
implementation to another.

For example, the corresponding metadata resource for a container called */data/*
may be accessible through this URI: `https://example.org/data/.meta`.
Alternatively, the photo at `https://example.org/data/image.jpg` may have it's
metadata stored in `https://example.org/data/image.jpg.meta`. The following is an
example of a typical request.


REQUEST:
```
GET /data/ HTTP/1.1
Host: example.org
```

RESPONSE:
```
Link: <https://example.org/data/.meta>; rel="describedby"
```

### Streams

Being LDP (BasicContainer) compliant, Solid servers MUST return a full listing
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

REQUEST:

```
GET /
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK

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

#### Globbing (inlining on GET)

We have found that in some cases, using the existing LDP features was not
enough. For instance, to optimize certain applications we needed to aggregate
all RDF resources from a container and retrieve them with a single GET
operation. We implemented this feature on the servers and decided to call it
"globbing". Similar to [UNIX shell
glob](https://en.wikipedia.org/wiki/Glob_(programming)), doing a GET on any URI
which ends with a `*` will return an aggregate view of all the resources that
match the indicated pattern.

For example, let's assume that `/data/res1` and `/data/res2` are two resources
containing one triple each, which defines their type as follows:

For *res1*:

```
<> a <https://example.org/ns/type#One> .
```

For *res2*:

```
<> a <https://example.org/ns/type#Two> .
```

If one would like to fetch all resources of a container beginning with `res`
(e.g. `/data/res1`, `/data/res2`) in one request, they could do a GET on
`/data/res*` as follows.

REQUEST:

```
GET /data/res* HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK

<res1>
    a <https://example.org/ns/type#One> .

<res2>
    a <https://example.org/ns/type#Two> .
```

Alternatively, one could ask the server to inline *all* resources of a
container, which includes the triples corresponding to the container itself:

REQUEST:

```
GET /data/* HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 200 OK

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
1.0](http://www.w3.org/TR/rdf-sparql-query/), where each resource is its own
SPARQL endpoint, accepting basic SELECT, INSERT and DELETE statements.

To read (query) a resource, the client can send a SPARQL `SELECT` through a
form-encoded HTTP GET request. The server will use the given resource as the
default graph that is being queried. The resource can be an RDF document or even
a container. The response will be serialized using `application/json` mime type.

For instance, the client can send the following form-encoded query `SELECT *
WHERE { ?s ?p ?o . }`:

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

## Subscribing

Live updates are currently only supported through websockets. This describes a
subscription mechanism through which clients can be notified in real time of
changes affecting a give resource.

The PubSub system is very basic. Clients only need to open a websocket
connection and *sub*(scribe) to a given resource URI. If any change occurs in
that resource, a *pub*(lish) event will be sent to all the subscribed clients.

The websocket server URI is the same for any resource located on a given data
space (same hostname). To discover the URI of the websocket server, clients can
send an HTTP OPTIONS. The server will then include an `Updates-Via` header in
the response:

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

To subscribe to a resource, clients will need to send the keyword `sub` followed
by an empty space and then the URI of the resource:

```
sub https://example.org/data/test
```

If a change occurs and the client is subscribed to that resource, it will
receive a websocket message composed of the keyword `pub`, followed by an empty
space and the URI of the resource that has changed:

```
pub https://example.org/data/test
```

Subscribing to a container can also be really useful, since all CRUD operations
(POST, PUT, PATCH, DELETE) performed on resources of that container will trigger
a notification for the container URI. This makes synchronization between
multiple apps really easy.

For example, a client subscribes to the `data/` container:

```
sub https://example.org/data/
```

If another client deletes the resource `foo` inside `data/`:

REQUEST:

```
DELETE /data/foo HTTP/1.1
Host: example.org
```

Then the following notification message will be sent:

```
pub https://example.org/data/
```

Here is a Javascript example on how to subscribe to live updates for a `test`
resource at `https://example.org/data/test`:

```javascript
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
`https://example.org/`, the client will need to send the following POST request,
with the Content-Type header set to `text/turtle`:

REQUEST:

```
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Link: <http://www.w3.org/ns/ldp#BasicContainer>; rel="type"
Slug: data

<> <http://purl.org/dc/terms/title> "Basic container" .
```

RESPONSE:

```
HTTP/1.1 201 Created
```

### Creating documents (files)

To create a new resource, the `Link` header value must be set to the following
value:
`Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"`

For example, to create a resource called `test` under
`https://example.org/data/`, the client will need to send the following POST
request, with the Content-Type header set to `text/turtle`:

REQUEST:

```
POST / HTTP/1.1
Host: example.org
Content-Type: text/turtle
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
Slug: test

<> <http://purl.org/dc/terms/title> "This is a test file" .
```

RESPONSE:

```
HTTP/1.1 201 Created
```

More examples can be found in the LDP [Primer document](http://www.w3.org/TR/ldp-primer/).

#### HTTP PUT to create

An alternative (though not standard) way of creating new resources is to use
HTTP PUT. Although HTTP PUT is commonly used to overwrite resources, this way is
usually preferred when creating new non-RDF resources (i.e. using a mime type
different than `text/turtle`).

REQUEST:

```
PUT /picture.jpg HTTP/1.1
Host: example.org
Content-Type: image/jpeg
...
```

RESPONSE :

```
HTTP/1.1 201 Created
```

Another useful Solid feature that is not yet part of the LDP spec deals with
using HTTP PUT to *recursively* create new resources. This feature is really
useful when the client wants to make sure it has absolute control over the URI
namespace -- e.g. migrating from one personal data store to another. Although
this feature is defined in HTTP1.1
[RFC2616](https://tools.ietf.org/html/rfc2616), we decided to improve it
slightly by having servers create the full path to the resource (including
intermediate containers), if it didn't exist before. (If you're familiar with
the Unix command line, think of it as `mkdir -p`.) For instance, a calendar app
uses a URI pattern (structure) based on dates when storing new events (i.e.
yyyy/mm/dd). Instead of performing several POST requests to create a month and a
day container when switching to a new month, it could send the following request
to create a new event resource called `event1`:

REQUEST:

```
PUT /2015/05/01/event1 HTTP/1.1
Host: example.org
```

RESPONSE:

```
HTTP/1.1 201 Created
```

This request would then create a new resource called `event1`, as well as the
missing intermediate resources -- containers for the month `05` and the day `01`
under the parent container `/2015/`.

To avoid accidental overwrites, Solid servers must support `ETag` checking
through the use of [If-Match or
If-None-Match](https://tools.ietf.org/html/rfc2616#section-14.24) HTTP headers.

**IMPORTANT:** Using PUT to create standalone containers is not supported,
because the behavior of PUT (overwrite) is not well defined for containers. You
MUST use POST (as defined by LDP) to create containers alone.

#### Alternative: Using SPARQL

To write data, clients can send an HTTP PATCH request with a SPARQL payload to
the resource in question. If the resource doesn't exist, it should be created
through an LDP POST or through a PUT.

For instance, to update the `title` of the container from the previous example,
the client would have to send a DELETE statement, followed by an INSERT
statement. Multiple statements (delimited by a `;`) can be sent in the same
PATCH request.

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

**IMPORTANT:** There is currently no support for blank nodes and RDF lists in
our SPARQL patches.

## Authentication

### WebID-TLS

The  [WebID-TLS protocol](http://www.w3.org/2005/Incubator/webid/spec/tls/)
enables secure, efficient authentication on the Web. It enables users to
authenticate on any site by simply choosing one of the certificates proposed to
them by their browser. These certificates can be created by any Web Site for any
purpose. A user may have multiple client certificates, bound to one or multiple
WebIDs.

Basically, WebID-TLS relies on matching a public key received from a client
certificate, to the public key published in the WebID profile obtained by
dereferencing the WebID included in the `SubjectAlternativeName` field of the
client certificate. In other words, users must prove they own a public key they
publish in their WebID profiles.

WebID-TLS is currently the preferred authentication mechanism in Solid.

**Important:** Javascript clients MUST set the XHR flag `withCredentials` to
`true` when making authenticated requests, in order to force browsers to send
the client certificate. For example, Firefox will not send the client
certificate unless this flag is set to true.

## Implementations
See:

- [solid/solid-platform](https://github.com/solid/solid-platform) for a
  list of Solid servers and developer tools
- [solid/solid-apps](https://github.com/solid/solid-apps) for an example list of
  Apps built using Solid

## Example

This example is taken from [W3C Social Web
WG](http://www.w3.org/wiki/Socialwg/)'s [user
stories](https://github.com/solid/solid-spec/tree/master/UserStories), where it
is called ["user posts a
note"](http://www.w3.org/wiki/Socialwg/Social_API/User_stories#User_posts_a_note):

 1. Eric writes a short note to be shared with his followers.
 2. After posting the note, he notices a spelling error. He edits the note and
  re-posts it.
 3. Later, Eric decides that the information in the note is incorrect. He
  deletes the note.

Here is how Solid would handle the three steps, using
[curl](http://curl.haxx.se/) as the client application:

1) Eric writes a short note to be shared with his followers. The `Slug` header
is optional but useful for controlling the resulting URL.

```
curl -H"Content-Type: text/turtle" \
     -H"Slug: social-web-2015" \
     -X POST \
     --data ' @prefix as: <http://www.w3.org/ns/activitystreams#>. <> a as:Note; as:content "Going to Social Web WG".' \
     https://eric.example.org/notes/
```

The URL of the new note can be found in the `Location` header returned by the
server. In this example it is likely to be
`https://eric.example.org/notes/social-web-2015` (since the `social-web-2015`
part was politely requested by the `Slug` header).

2) After posting the note, he notices a spelling error. He edits the note and
re-posts it. Solid servers can handle updates in two different ways: a PUT
(overwrite) or a PATCH with `application/sparql-update` content type.

Use HTTP PUT, when you just want to replace the data:

```
curl -H"Content-Type: text/turtle" \
     -X PUT \
     --data ' @prefix as: <http://www.w3.org/ns/activitystreams#>. <> a as:Note; as:content "Going to Social Web WG in Paris".' \
     https://eric.example.org/notes/social-web-2015
```

Or you can use HTTP PATCH with SPARQL if you only want to change certain parts
of the resource, leaving the others unchanged (perhaps because other
applications are modifying them):

```
curl -H"Content-Type: application/sparql-update" \
     -X PATCH \
     --data 'DELETE DATA {<> <http://www.w3.org/ns/activitystreams#content> "Going to Social Web WG" .}; INSERT DATA {<> <http://www.w3.org/ns/activitystreams#content> "Going to Social Web WG in Paris" .} ' \
     https://eric.example.org/notes/social-web-2015
```

If no match is found for the triple to DELETE, the request safely aborts without
changing any data.

3) Later, Eric decides that the information in the note is incorrect. He deletes
  the note.

```
curl -X DELETE https://eric.example.org/notes/social-web-2015
```

Note that all three actions have been performed through RESTful HTTP requests.

In these example, data was sent to the server using `text/turtle` (which is
mandated by LDP), but other content types (such as JSON-LD) could be used if
implemented by servers.

## Other Considerations

### CORS - Cross Origin Resource Sharing

There are two different ways CORS support must be implemented on Solid servers.
First, when the request is sent through a browser that sets the `Origin` header.
And second, when clients do not set an `Origin` header (e.g. curl or non-browser
clients).

**1)** When the `Origin` header is set:

1. Client (browser) loads an app from `https://app.org` and wants to send an XHR
  (ajax) request to the server at `https://example.org`. Before sending the
  request over the wire, the browser adds the `Origin` header: `Origin:
  https://app.org`, which corresponds to the domain from where the app was loaded.

2. The server running on https://example.org receives the request and looks at the
  `Origin` header. It sees `https://app.org`, stores the value and handles the
  request.

3. The server responds to the request and sets the value of the request `Origin`
  header to the CORS header in the HTTP response:

    ```
    Access-Control-Allow-Origin: https://app.org
    ```

**2)** Without an `Origin` header:

1. A curl request is sent from the terminal to `https://example.org`. Unless
  explicitly specified though a curl parameter, the `Origin` header will not be
  set.

2. The server running on `https://example.org` receives the request and does not
  find an `Origin` header.

3. The server responds to the request and sets a default "all" value for the
  `Access-Control-Allow-Origin` header in the HTTP response:

    ```
Access-Control-Allow-Origin: *
    ```

The star character (`*`) signifies "allow all". If you want to learn more about
CORS, please visit this page: http://enable-cors.org/

### Discovering server capabilities - the OPTIONS method

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
