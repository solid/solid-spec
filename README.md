# SoLiD - Social Linked Data platform
This document contains design notes on individual components used by SoLiD. They are intended to be a guide for developers who plan to build social linked data servers and applications.

# Components
SoLiD was designed from the ground up to be modular and openly extensible. Among the technologies it uses, you can find:

 - [RDF](#rdf)
 - [WebID](#webid)
 - [WebID-TLS](#webid-tls)
 - [WebID-RSA](#webid-tls)
 - [WebAccessControl](#wac)
 - [LDP](#ldp)

## <a name="rdf"></a>RDF

## <a name="webid"></a>Identity through WebID
Identity management as well as unique identifiers are the core of any social system. SoLiD uses [WebID](http://www.w3.org/2005/Incubator/webid/spec/identity/), an HTTP(S) URI, to uniquely refer to users (people or agents). The advantage of WebID is that the URI can be dereferenced to reveal useful information about the user. Also, since WebID profiles can be hosted anywhere (including your basement server), users are no longer trapped inside Identity Provider Silos (e.g. Twitter, Facebook, Google+).

## Authentication
### <a name="webid-tls"></a>WebID-TLS

### <a name="webid-rsa"></a>WebID-RSA

## Access Control
### <a name="wac"></a>Web Access Control

## Resource manipulation
### <a name="ldp"></a>LDP - Linked Data Platform

