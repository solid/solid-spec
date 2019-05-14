## Private Sharing

Originally put forward by Ben Roberts ([permalink](https://www.w3.org/wiki/Socialwg/Social_API/User_stories#Private_Sharing)):

> 1. Ian wants to share a file with Jane
> 2. Ian posts the file file to his site with it set to only show to Jane.
> 3. Jane receives a notice that Ian has shared a file with her.
> 4. Jane views the file and decides to leave a thank you comment on the file for Ian

Point 1 is just a desire. We cut point 2. into two parts: the first is uploading a file, and the second is limiting access.

### Background

This story has a privacy aspect so we will use [WebID-OIDC authentication](https://github.com/solid/webid-oidc-spec) to illustrate it. Other authentication methods should also work with [Web Access Control](http://www.w3.org/2005/Incubator/webid/spec/), such as WebID which is easy, and others that need to be looked at.


Ian has WebID `<https://ian.name/card#me>`.

```http
GET /card HTTP/1.1
Host: ian.name:443
Accept: text/turtle, application/ld+json
```
```http
HTTP/1.1 200 Ok
Accept-Patch: application/sparql-update
Access-Control-Allow-Origin: *
Allow: OPTIONS, GET, HEAD, SEARCH, PATCH
Content-Type: text/turtle
ETag: "1417390950000|Success(922)"
Last-Modified: Sun, 1 April 2015 23:42:30 GMT
Content-Type: text/turtle
Content-Length: 545
Link: <card.acl>; rel=acl
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
```
```ttl
@prefix foaf: <http://xmlns.com/foaf/0.1/> .
@prefix cert: <http://www.w3.org/ns/auth/cert#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

<card> a foaf:PersonalProfileDocument;
   foaf:primaryTopic <card#me> .

<card#me> a foaf:Person ;
     foaf:name "Ian;
     foaf:knows <https://jane.org/profile#me> ;
```


In order to be able to do command line curl demos, we will assume that
Ian has already obtained a Bearer token via the [WebId-OIDC process](https://github.com/solid/webid-oidc-spec). 


### Ian posts the file

Here curl makes the connection, and authenticates Ian with his Certificate. As a result the content is created.

```sh
$ curl -X POST -k -i -H "Content-Type: text/turtle" \
   -H "Slug: financials" \
   -H 'authorization: Bearer ey...'
   --data-binary @financials.ttl  https://ian.name/2014/   
```
```http
HTTP/1.1 201 Created
Accept-Patch: application/sparql-update
Access-Control-Allow-Origin: *
Allow: OPTIONS, GET, HEAD, SEARCH, PATCH
Content-Type: text/turtle
ETag: "v0"
Last-Modified: Sun, 1 April 2015 23:42:30 GMT
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
Location: /2014/financials
Link: <financials.acl>; rel=acl
```

So the `<financials>` resource is created in the LDP container `</2014/>` . Let us imagine that the `<financials.acl>` resource indeed limits it currently to only be viewed by the owner Ian.

```sh
$ curl -X GET -k -H "Content-Type: text/turtle" \
   -H 'authorization: Bearer ey...'
   https://ian.name/2014/financials.acl
```
```ttl
@prefix acl: <http://www.w3.org/ns/auth/acl#> .
@prefix foaf: <http://xmlns.com/foaf/0.1/> .

[] acl:accessTo </2014/financials>, <>;
   acl:mode acl:Read, acl:Write;
   acl:agent </card#me> .
```   

### Allow Access to Jane

To allow access to the resource to Jan, Ian must send the following
PATCH, using his certificate as he is the only one authorised to patch the resource.

```http
PATCH /2014/financials.acl HTTP/1.1
Host: ian.name:443
Content-Type: application/sparql-update; utf-8
Content-Length: 120
Authorization: Bearer ey...
```
```sparql
Prefix acl: <http://www.w3.org/ns/auth/acl#> .
INSERT DATA {
[] acl:accessTo </2014/financials>;
   acl:mode acl:Read;   
   acl:agent <https://jane.org/profile#me> .
 }
```

After doing this Jane will be able to read the resource.

### Send notice to Jane

Ian's software ( server or client - it does not matter ) somehow needs to find out how to ping Jane. They know Jane's WebID is `<https://jane.org/profile#me>`, so they can dereference the document at `<https://jane.org/profile>` for which they probably already have a cached version. If they don't have the full version but just want to find out what the relation they need is they can send the request. A simple GET on the profile will do.

Given that we have shown the obvious way to query in other examples, we show here out of interest a potential optimisation that would send the query in the body of the GET (see [discussion on http-wg list](https://lists.w3.org/Archives/Public/ietf-http-wg/2015AprJun/0317.html) ). (The query could also be in a `Query` header.)

```http
GET /profile HTTP/1.1
Host: jane.org:443
Accept: text/turtle
Content-Type: application/sparql-query; charset=UTF-8
Content-Length: 123
```
```sparql
PREFIX solid: <http://solid.info/notification/ping#>
CONSTRUCT { <#me> as:ping ?where }
WHERE { <#me> as:ping ?where }
```

If the server does not understand the query, it just returns the full document by default - that is what most servers will do actually do. This still needs to be finalised by ldp and http-wg. A more interesting type of query is probably the SPARQL "DESCRIBE" query which gives the minimal bounded graph around the URI.

The response may then in the best of case just be one short line:

```ttl
@prefix solid: <http://solid.info/notification/ping#> .
<#me> as:ping </pingInbox/> .
```

### Send a notice

To send a notice the agent could send an activity stream event to the
`<https://jane.org/pingInbox/>` ldp:BasicContainer .

```http
POST /pingInbox/ HTTP/1.1
Host: jane.org:443
Content-Type: text/turtle
Content-Length: 145
```
```ttl
@prefix as: <http://www.w3.org/ns/activitystreams#> .
[] as:Post ;
  as:published "2015-02-10T15:04:55Z"^^xsd:dateTime ;
  as:actor <https://ian.name/card#me> ;
  as:object <https://ian.name/2014/financials> ;
  as:target <https://ian.name/2014/> .
```

{>> We are probably here abusing [AS2.0 vocabulary](http://www.w3.org/TR/activitystreams-core/) here. We need to work out with the AS2.0 team what they think the right manner of doing this would be. <<}

And the server responds with a 201 created:

```http
HTTP/1.1 201 Created
Accept-Patch: application/sparql-update
Access-Control-Allow-Origin: *
Allow: OPTIONS, GET, HEAD, SEARCH, PATCH
ETag: "v0"
Last-Modified: Sun, 1 April 2015 23:52:12 GMT
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
Location: /pingInbox/ping20
Link: <ping20.wac>; rel=acl
```

The acl is in `</pingInbox/ping20.wac>` and it may say that the resource is only readable by the owner of the `</pingInbox/>` container and the sender of the resource in R/W.

{>> we need to find a way to have an ACL that automatically adds the
 author of the ACL to the authorisation <<} .

At this point we have the following set of links:

![relations between docs, acls, and WebIDs](img/PrivateSharing.png)

### Jane views the file

Jane reads her inbox at some point, and just does a normal GET on the `<https://ian.name/2014/financials> resource, using her certificate containing a WebID.
