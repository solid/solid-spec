## User Profile Management

Originally put forward by Evan Prodromou ([permalink](https://www.w3.org/wiki/Socialwg/Social_API/User_stories#User_profile_management)):

> 1. Kim creates a personal profile with her name, avatar picture, and home town.
> 2. Kim updates her profile to include her job title, phone number and company name.
> 3. Kim reconsiders her personal privacy boundaries; she updates her profile to remove her phone number.

### Background

This story requires some initial background context: Kim needs previous to this to have created an account. This would give her software access to a root LDP Container. Let us assume that Kim also bought her domain name and that the root container is just `<https://kim.name/>`

We may assume that during a short period ( a few hours or a day) her software is the only one to be able to authenticate using a cookie. This is the time Kim has to create her profile.

Her client may have followed a link to the initial collection and made the simple GET Request:

```http
GET / HTTP/1.1
Host: kim.name
Cookie: cryptohash:123cafebabe
Accept: text/turtle;q=0.8,application/json+ld;q=0.7
```

which on success returns the empty LDP Basic Container:

```http
HTTP/1.1 200 Ok
Accept-Patch: application/sparql-update
Access-Control-Allow-Origin: *
Allow: OPTIONS, GET, HEAD, POST, SEARCH
Content-Type: text/turtle
ETag: "1417390950000|Success(922)"
Last-Modified: Sun, 1 April 2015 23:42:30 GMT
Content-Type: text/turtle
Content-Length: 100
Link: <.acl>; rel=acl
Link: <http://www.w3.org/ns/ldp#BasicContainer>; rel="type", <http://www.w3.org/ns/ldp#Resource>; rel="type"
```
```ttl
@prefix ldp: <http://www.w3.org/ns/ldp#> .

<> a ldp:BasicContainer .
```

So the LDP container is empty, accessible, and can be POSTed to.

### 1. Create the profile

> Kim creates a personal profile with her name, avatar picture, and home town.


Kim's software can get Kim's profile information in a number of ways, by asking her to fill in a form, or by asking her to give her access to existing social network sites, or by getting the information directly from another application on the operating system....

Once this information is gathered the User Agent can then translate it to a graph and then POST it in either Turtle or JSON-LD. Whatever you post, you will be able to retrieve it in the format you like.

#### POST Turtle

You can post

```http
POST / HTTP/1.1
Host: kim.name
Cookie: cryptohash:123cafebabe
Content-Length: 449
Slug: card
Content-Type: text/turtle
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
```
```ttl
@prefix foaf: <http://xmlns.com/foaf/0.1/> .
@prefix contact: <http://www.w3.org/2000/10/swap/pim/contact#> .

<> a foaf:PersonalProfileDocument;
   foaf:primaryTopic <#k> .

<#k> a foaf:Person;
   foaf:logo <http://tinyurl.com/ltxw3px> ;
   foaf:mbox <mailto:kim@example.com>;
   foaf:givenName "Kim";
   contact:home <#hme> .

<#hme> contact:address [
          contact:country "Australia";
          contact:city "WonderCity";
        ] .
```

which on success would return:

```http
HTTP/1.1 201 Created
Location: https://kim.name/card
Link: <card.acl>; rel=acl
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
```

The `<card.acl>` link allows Kim's software agent to find out who has access to this file. As this story does not go into access control issues. More examples on that [elsewhere](https://github.com/read-write-web/rww-play/wiki/Curl-Interactions).

#### POST Json-LD

For those favoring JSON-LD, the client sends the following content
which described a graph isomorphic to the one above:

```http
POST / HTTP/1.1
Host: kim.name
Cookie: cryptohash:123cafebabe
Content-Length: 574
Slug: card
Content-Type: application/ld+json
```
```json
{
  "@context": {
     "foaf": "http://xmlns.com/foaf/0.1/",
     "contact": "http://www.w3.org/2000/10/swap/pim/contact#"
  },
  "@id": "",
  "@type": "foaf:PersonalProfileDocument",
  "foaf:primaryTopic":  {
    "@id": "#k",
    "@type": "foaf:Person",
    "foaf:logo": { "@id": "http://tinyurl.com/ltxw3px" },
    "foaf:mbox": { "@id": "mailto:kim@example.com" },
    "foaf:givenName": "Kim",
    "contact:home": {
       "@id": "#hme",
       "contact:address": {
          "contact:country": "Australia",
          "contact:city": "WonderCity"
       }
    }
  }
}
```

and receives the following result:

```http
HTTP/1.1 201 Created
Location: https://kim.name/card
Link: <card.acl>; rel=acl
Link: <http://www.w3.org/ns/ldp#Resource>; rel="type"
ETag: "version0"
```

### 2. Update profile

> Kim updates her profile to include her job title, phone number and company name.

She can do this either with a PUT which requires re-sending the full document as above, or for efficiency with [Sparql-Update](http://www.w3.org/TR/sparql11-update/) or with [LD-PATCH](http://www.w3.org/TR/ldpatch/).

#### update using PUT

PUT is the easiest for the client, as it does not have to calculate a diff between the original version it has and the version it wishes to have. We use If-Match, to make sure we don't override someone elses content.

```http
PUT /card HTTP/1.1
Host: kim.name
Cookie: cryptohash:123cafebabe
Content-Length: 574
If-Match: "version0"
Slug: card
Content-Type: application/ld+json
```
```json
{
  "@context": {
     "foaf": "http://xmlns.com/foaf/0.1/",
     "contact": "http://www.w3.org/2000/10/swap/pim/contact#",
     "schema": "https://schema.org/"
  },
  "@id": "",
  "@type": "foaf:PersonalProfileDocument",
  "foaf:primaryTopic":  {
    "@id": "#k",
    "@type": "foaf:Person",
    "foaf:logo": { "@id": "http://tinyurl.com/ltxw3px" },
    "foaf:mbox": { "@id": "mailto:kim@example.com" },
    "foaf:givenName": "Kim",
    "schema:jobTitle": "Kung Fu Koach",
    "schema:worksFor": { "schema:name": "Kung Fu Kat Inc." },
    "contact:home": {
       "@id": "#hme",
       "contact:phone": { "@id": "tel:+61755555555" },
       "contact:address": {
          "contact:country": "Australia",
          "contact:city": "WonderCity"
       }
    }
  }
}
```
```http
HTTP/1.1 201 Created
Location: https://kim.name/card
Link: <card.acl>; rel=acl
ETag: "version0"
```

#### update using PATCH with Sparql-Update

```http
PATCH /card HTTP/1.1
Content-Type: application/sparql-update; utf-8
If-Match: "version0"
Content-Length: 45
```
```sparql
PREFIX contact: <http://www.w3.org/2000/10/swap/pim/contact#> .
PREFIX foaf: <http://xmlns.com/foaf/0.1/> .
PREFIX schema: <https://schema.org/>
INSERT DATA {
  <#k> schema:jobTitle "Kung Fu Koach";
       schema:worksFor [ schema:name "Kung Fu Kat Inc." ] .
  <#hme> contact:phone <tel:+61755555555> .
}
```

#### update using PATCH with LD-PATCH

One can also use the W3C Candidate Recommendation [LD Patch](http://www.w3.org/TR/ldpatch/).

{>> please fill in <<}

### Deleting data

> Kim reconsiders her personal privacy boundaries; she updates her profile to remove her phone number.

Deleting a few triples can be done exactly the same way as above using PUT, so we won't repeat it here. Here is a shorter version using PATCH.

```http
PATCH /card HTTP/1.1
Content-Type: application/sparql-update; utf-8
If-Match: "version0"
Content-Length: 45
```
```spqrql
PREFIX contact: <http://www.w3.org/2000/10/swap/pim/contact#> .
DELETE DATA {
  <#hme> contact:phone <tel:+61755555555> .
}
```

### Notes

There is nothing definitive in the selection of vocabularies. Some notes ( see [discussion](https://lists.w3.org/Archives/Public/public-socialweb/2015May/0005.html) )

  * [Foaf](http://xmlns.com/foaf/0.1/) is widely deployed, but quite limited for company info, and needs to be complemented with a lot of other vocabularies.
  * [Schema.org](https://schema.org/) is very complete, but confuses documents and things, has URLs as string in the examples, and is so vague that it would probably not be useful for an efficient social web. It is clearly an improvement over the search engines current position, but not designed for building light weight apps in a linked data space. It would be counterproductive.
  * [vcard](http://www.w3.org/TR/vcard-rdf/) has the advantages and disadvantages of sticking closely to the old and widely deployed vcard format. The disadvantage is the odd decision to have relations to objects and Relation objects - there are as many types of relations, that it would have been easier to create the relations directly.

Todo:

 * discuss complexities of patching with blank nodes for each format

 
