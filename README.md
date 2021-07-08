# Solid Specification Draft
[![](https://img.shields.io/badge/project-Solid-7C4DFF.svg?style=flat-square)](https://github.com/solid/solid)
[![Join the chat at https://gitter.im/solid/solid-spec](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/solid/solid-spec)

**Latest version:** [`v.0.7.0`](https://github.com/solid/solid-spec/tree/v0.7.0) (see [CHANGELOG.md](CHANGELOG.md))

**Publication status**: Unofficial Draft

**Current development version:** `v.0.7.0-next` (evolving)

**This document contains an informal description of implementation guidelines for Solid servers and clients.
A normative specification is in the making at https://github.com/solid/specification/.
For the time being, the present document contains the best approximation of expected server and client behavior.**

## Table of Contents

1. [Overview](#overview)
2. [Identity](#identity)
3. [Profiles](#profiles)
    * [WebID Profile Documents](#webid-profile-documents)
4. [Authentication](#authentication)
    * [Primary Authentication](#primary-authentication)
      * [WebID-OIDC](#webid-oidc)
      * [WebID-TLS](#webid-tls)
    * [Secondary Authentication: Account
        Recovery](#secondary-authentication-account-recovery)
5. [Authorization and Access Control](#authorization-and-access-control)
    * [Web Access Control](#web-access-control)
6. [Content Representation](#content-representation)
7. [Reading and Writing Resources](#reading-and-writing-resources)
    * [HTTPS REST API](#https-rest-api)
    * [WebSockets API](#websockets-api)
8. [Social Web App Protocols](#social-web-app-protocols)
    * [Notifications](#notifications)
    * [Friends Lists, Followers and
        Following](#friends-lists-followers-and-following)
9. [Recommendations for Server
      Implementation](#recommendations-for-server-implementations)
10. [Recommendations for Client App
      Implementation](#recommendations-for-client-app-implementations)
11. [Examples](#examples)
12. [Current Implementations](#current-implementations)

## Overview

[Solid](https://github.com/solid/solid)
is a proposed set of conventions and tools for building
*decentralized applications* based on [Linked
Data](https://www.w3.org/DesignIssues/LinkedData) principles. Solid is
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

Solid uses WebID Profile Documents for management of user identity and optionally security credentials (such as public keys), and user preferences discovery.

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

## Authentication

Authentication is the process of determining a user’s identity, of asking the
question “How do I know you are who you say?”.

How do web applications typically authenticate users (that is, how do they
verify identity)? The most common method is usernames and passwords. A
*username* uniquely identifies a user (and ties them to a user profile), and a
*password* verifies that the user is who they say they are. Many applications or
services also have a *secondary authentication mechanism* (usually an external
email address) that they use for account recovery (in case the user forgets or
loses their primary authentication tokens, username and password).

Solid currently uses WebID-OIDC as its primary authentication mechanism.
Alternative complementary mechanisms are also being actively investigated.
In addition, Solid recommends that server implementations also offer secondary
authentication available for users for Account Recovery (via email or some
other out-of-band mechanism).

### Primary Authentication

Solid, being a decentralized web application platform, has a set of requirements
for its authentication mechanisms that are not commonly encountered by most
platforms and ecosystems. Specifically, it requires *cross-domain*,
de-centralized authentication mechanisms not tied to any particular identity
provider or certificate authority.

#### WebID-OIDC

WebID-OIDC is based on the OAuth2/OpenID Connect
protocols, adapted for WebID based decentralized use cases.

Implementations of WebID-OIDC IDPs for Solid SHOULD implement TLS as a login method
alongside other login methods such as passwords.

**See component spec:
  [WebID-OIDC Specification](https://github.com/solid/webid-oidc-spec)**

#### WebID-TLS (Optional)

**Note:** Several browser vendors (Chrome, Firefox) have removed support
for the `KEYGEN` element, on which WebID-TLS relied for in-browser certificate
generation.

Solid servers MAY implement the [WebID-TLS
protocol](http://www.w3.org/2005/Incubator/webid/spec/tls/) as one of their
primary authentication mechanisms.

**See component spec:
  [Solid WebID-TLS Specification](authn-webid-tls.md)**

### Secondary Authentication: Account Recovery

Regardless of the primary authentication mechanism, bearer tokens and other
proofs of identity tend to get lost by users. Passwords can be forgotten,
browser certificates can be lost to hardware failure, and so on. Solid
recommends that secondary Account Recovery mechanisms are provided by server
implementers, to aid in these scenarios.

## Authorization and Access Control

Authorization is the process of deciding whether a user has *access* to a
particular resource. If authentication asks "who is the user?", authorization
is concerned with "what is the user allowed to do?".

Solid currently uses the Web Access Control (WAC) mechanism for cross-domain
authorization for all its resources.

### Web Access Control

[Web Access Control (WAC)](https://github.com/solid/web-access-control-spec) is
a decentralized system that allows different users and groups various forms of
access to resources where users and groups are identified by HTTP URIs. The
system is similar to the access control system used within many file systems
except that the documents controlled, the users, and the groups, are all
identified by URIs. Users are identified by WebIDs. Groups of users are
identified by the URI of a class of users which, if you look it up, returns a
list of users in the class. This means a WebID hosted by any server can be a
member of a group hosted some other server.

Users do not need to have an account (i.e. WebID) on a given server to have
access to documents on it.

**See component spec:
[Solid WAC Specification](https://github.com/solid/web-access-control-spec)**

## Content Representation

Solid deals with reading and writing two kinds of resources:

1. Linked Data resources (RDF in the form of JSON-LD, Turtle, HTML+RDFa, etc)
2. Everything else (binary data and non-linked-data structured text)

While you can build Solid applications with non-linked data resources, using
actual RDF-based Linked Data provides you with considerable benefits in terms
of interoperability with the rest of the Solid app ecosystem.

Resources are grouped in directory-like **Containers** (currently conforming
to the [LDP Basic Container spec](https://www.w3.org/TR/ldp/#ldpbc)).

**See component spec: [Solid Content
  Representation](content-representation.md)**

## Reading and Writing Resources

### HTTPS REST API

Solid extends the [Linked Data Platform spec](https://www.w3.org/TR/ldp/) to
provide a simple REST API for CRUD operations on resources and containers.

**See component spec: [HTTPS REST API](api-rest.md)**

### WebSockets API

Solid also provides a WebSockets based API for a PubSub (Publish/Subscribe)
mechanism, through which clients can be notified in real time of
changes affecting a give resource.

**See component spec: [WebSockets API](api-websockets.md)**

## Social Web App Protocols

In addition to read/write operations on resources, Solid provides a number of
specs and recommendations to help developers achieve interoperability between
various social web applications that are part of the ecosystem.

### Notifications

**See component spec: [Linked Data Notifications](https://www.w3.org/TR/ldn/)**

### Friends Lists, Followers and Following

API recommendations for managing subscriptions and friends lists are still
being discussed. TBD.

## Recommendations for Server Implementations

**See component spec: [Recommendations for Server
  Implementations](recommendations-server.md)**

## Recommendations for Client App Implementations

**See component spec: [Recommendations for Client
  Implementations](recommendations-client.md)**

## Examples

* [User Posts a Note](examples/user-posts-note.md)

## Current Implementations

**Server Implementations:** See
[solid/solid-platform](https://github.com/solid/solid-platform#servers) for a
list of Solid servers and developer tools.
Note: The Solid team uses
[`node-solid-server`](https://github.com/solid/node-solid-server) as
its main server implementation.

**Client App Implementations:** See
[`solid-client`](https://github.com/solid/solid-client) for the main client
library, and [solid/solid-apps](https://github.com/solid/solid-apps) for an
example list of Apps built using Solid.
