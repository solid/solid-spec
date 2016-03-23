# Solid Web Access Control Spec

**Note:** This spec is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.

[Web Access Control (WAC)](https://www.w3.org/wiki/WebAccessControl) is a
decentralized system that allows different users and groups various forms of
access to resources where users and groups are identified by HTTP URIs. The
system is similar to the access control system used within many file systems
except that the documents controlled, the users, and the groups, are all
identified by URIs. Users are identified by WebIDs. Groups of users are
identified by the URI of a class of users which, if you look it up, returns a
list of users in the class. This means a WebID hosted by any server can be a
member of a group hosted some other server.

Users do not need to have an account (i.e. WebID) on a given server to have
access to documents on it.

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
```http
GET /data/ HTTP/1.1
Host: example.org
```

RESPONSE:
```http
Link: <https://example.org/data/.acl>; rel="acl"
```

WAC policies are applied to resources, instead of RDF triples. This means that
policies can be set for [LDPRs](http://www.w3.org/TR/ldp/#ldpr) as well as for
[LDPCs](http://www.w3.org/TR/ldp/#ldpc). A special case is applied to LDPCs,
where policies can be defined as "default" for everything in a container,
meaning that all the members of that specific container will inherited them.
