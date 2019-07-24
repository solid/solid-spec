# Solid Content Representation Spec

**Note:** This spec is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.

## Table of Contents

* [Overview](#overview)
* [Objects](#objects)
* [Metadata](#metadata)

## Overview

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

## Objects

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
  `https://example.org/avatar.png` maps to `/home/user/www/avatar.png`

Servers must support the HEAD method for reading data. This returns a list of
headers related to the resource in question. Among these headers, two very
important `Link` headers contain pointers to corresponding ACL and metadata
resources. More information on naming conventions for these resources can be
found in the [Authorization and Access Control
section](README.md#authorization-and-access-control).

REQUEST:

```http
HEAD /data/ HTTP/1.1
Host: example.org
```

RESPONSE:

```http
HTTP/1.1 200 OK
....
Link: <https://example.org/data/.acl>; rel="acl"
Link: <https://example.org/data/.meta>; rel="describedby"
```

## Metadata

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
```http
GET /data/ HTTP/1.1
Host: example.org
```

RESPONSE:
```http
Link: <https://example.org/data/.meta>; rel="describedby"
```
