### User Profile Management

Originally put forward by Evan Prodromou ([permalink](https://www.w3.org/wiki/Socialwg/Social_API/User_stories#User_profile_management)):

> 1. Kim creates a personal profile with her name, avatar picture, and home town.
> 2. Kim updates her profile to include her job title, phone number and company name.


##### Background

This story requires some initial background context: Kim needs previous to this to have created an account. This would give her software access to a root LDP Container. Let us assume that Kim also bought her domain name and that the root container is just `<https://kim.name/>`

We may assume that during a short period ( a few hours or a day) her software is the only one to be able to authenticate using a cookie. This is the time Kim has to create her profile.

```http
GET / HTTP/1.1
Host: kim.name
Cookie: cryptohash:123cafebabe
Accept: text/turtle;q=0.8,application/json+ld;q=0.7
```
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

@prefix ldp: <http://www.w3.org/ns/ldp#> .

<> a ldp:BasicContainer .
```

So the LDP container is empty, accessible, and can be POSTed to.

#### 1. Create the profile

> Kim creates a personal profile with her name, avatar picture, and home town.


Kim's software can get Kim's profile information in a number of ways, by asking her to fill in a form, or by asking her to give her access to existing social network sites, or by getting the information directly from another application on the operating system.... 

Once this information is gathered the User Agent can then translate it to a graph and then POST it in either Turtle or JSSON-LD. Here is the Turtle version.

```http
POST / HTTP/1.1
Host: kim.name
Cookie: cryptohash:123cafebabe
Content-Length: 350
Slug: card
Content-Type: text/turtle

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
```http
HTTP/1.1 201 Created
Location: https://kim.name/card
Link: <card.acl>; rel=acl
```

The `<card.acl>` link allows Kim's software agent to find out who has access to this file. As this story does not go into access control issues. More
examples on that [elsewhere](https://github.com/read-write-web/rww-play/wiki/Curl-Interactions).

### 2. Update profile

> Kim updates her profile to include her job title, phone number and company name.

She can do this with Sparql-Update like this

```HTTP
PATCH /card HTTP/1.1
Content-Type: application/sparql-update; utf-8
Content-Length: 45

PREFIX contact: <http://www.w3.org/2000/10/swap/pim/contact#> .
PREFIX foaf: <http://xmlns.com/foaf/0.1/> .
PREFIX schema: <https://schema.org/>
INSERT DATA {
  <#k> schema:jobTitle "Kung Fu Koach";
       schema:worksFor [ schema:name "Kunk Fu Kat" ] .
  <#hme> contact:phone <tel:+61755555555> .
}
```

Or with the W3C Candidate Recommendation [LD Patch](http://www.w3.org/TR/ldpatch/).

Note: there is nothing definitive in the selection of vocabularies. Some notes:
 
  * [Foaf](http://xmlns.com/foaf/0.1/) is widely deployed, but quite limited for company info, and needs to be complemented with a lot of other vocabs
  * [Schema.org](https://schema.org/) is very complete, but confuses documents and things, has URLs as string in the examples, and is so vague that it would probably not be useful for an efficient social web. It is clearly an improvement over the search engines current position, but not designed for building light weight apps in a linked data space. It would be counterproductive.
  * [vcard](http://www.w3.org/TR/vcard-rdf/) has the advantages and disadvantages of sticking closely to the old and widely deployed vcard format. The disadvantage is the odd decision to have relations to objects and Relation objects - there are as many types of relations, that it would have been easier to create the relations directly.

### 3.  Deletion

> Kim reconsiders her personal privacy boundaries; she updates her profile to remove her phone number.

```HTTP
PATCH /card HTTP/1.1
Content-Type: application/sparql-update; utf-8
Content-Length: 45

PREFIX contact: <http://www.w3.org/2000/10/swap/pim/contact#> .
DELETE DATA {
  <#hme> contact:phone <tel:+61755555555> .
}
```

Note in both cases an HTTP `PUT` could have done too.

Todo:
 
 * add conditional PATCH
 * show LD-Patch version
 * discuss complexities of patching with blank nodes for each format
 
 