# SoLiD - Social Linked Data platform
This document contains design notes on individual components used by SoLiD. They are intended to be a guide for developers who plan to build social linked data servers and applications.

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

SoLiD uses several serialization syntaxes for storing and exchanging RDF such as [Turtle](http://www.w3.org/TR/turtle/) and [JSON-LD](http://www.w3.org/TR/json-ld/).


## Read-Write data handling
To simplify data portability, we opted for a design that follows the classic POSIX standards. For instance, resources are stored directly on the file system instead of using a database. This allows people to read/write data both using desktop applications as well as Web-based ones, and also to share the same disk drive between different machines/servers.

For this precise reason, we opted to use a fairly recent spec proposed by the W3C, called Linked Data Platform.

### <a name="ldp"></a>LDP - Linked Data Platform
The [LDP specification](http://www.w3.org/TR/ldp/) defines a set of rules for HTTP operations on Web resources, some based on RDF, to provide an architecture for reading and writing Linked Data on the Web. The most important feature of LDP is that it provides us with a standard way of RESTfully writing resources (documents) on the Web [[examples](http://www.w3.org/TR/ldp-primer/)], without having to rely on less flexible conventions (APIs) based around sending form-encoded data using POST.

### Websockets

## <a name="webid"></a>Identity through WebID
Identity management as well as unique identifiers are the core of any social system. SoLiD uses [WebID](http://www.w3.org/2005/Incubator/webid/spec/identity/), an HTTP(S) URI, to uniquely refer to users (people or agents). The advantage of WebID is that the URI can be dereferenced to reveal useful information about the user. Also, since WebID profiles can be hosted anywhere (including your basement server), users are no longer trapped inside Identity Provider Silos (e.g. Twitter, Facebook, Google+).

## Authentication
### <a name="webid-tls"></a>WebID-TLS

### <a name="webid-rsa"></a>WebID-RSA

## Access Control
### <a name="wac"></a>Web Access Control


