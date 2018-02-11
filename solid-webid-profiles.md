# Solid WebID Profiles Spec

**Note:** This spec is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.

## Table of Contents

* [Overview](#overview)
* [Profile Representation Formats](#profile-representation-formats)
* [Required Profile Information](#required-profile-information)
* [Minimum Recommended Profile
      Information](#minimum-recommended-profile-information)
    * [Recommendation for User Names in
      Profiles](#recommendation-for-user-names-in-profiles)
* [Public and Private Profiles](#public-and-private-profiles)
    * [Extended Profile](#extended-profile)
* [Public Key Certificates](#public-key-certificates)
* [Account Resource Discovery](#account-resource-discovery)
    * [Storage Discovery](#storage-discovery)
    * [Inbox Discovery](#inbox-discovery)
    * [Type Registry Index Discovery](#type-registry-index-discovery)

## Overview

Solid uses WebID Profile Documents for management of user identity and security
credentials (such as public keys), and user preferences discovery.

## Profile Representation Formats

From the
[WebID Profile](https://www.w3.org/2005/Incubator/webid/spec/identity/#publishing-the-webid-profile-document)
spec:

> A WebID Profile Document is an RDF Web resource that MUST be available as
> [text/turtle](http://www.w3.org/TR/turtle/), but MAY be available in other RDF
> serialization formats (such as JSON-LD or
> [HTML+RDFa](http://www.w3.org/TR/rdfa-core/)) if requested through content
> negotiation.

## Required Profile Information

There are only 3 statements required for a valid (though not very useful)
WebID Profile Document:

1. Identifying the document as a `foaf:PersonalProfileDocument` instance
2. Having a `foaf:primaryTopic` predicate
3. Having that primary topic be a valid `foaf:Agent` type, such as `foaf:Person`

Here's an example of a minimum valid profile, in Turtle (`text/turtle`) format:

```ttl
@prefix foaf: <http://xmlns.com/foaf/0.1/>.
<https://alice.databox.com/profile/card>
    a foaf:PersonalProfileDocument ;
    foaf:primaryTopic <#me> .

<#me>
    a foaf:Person .
```

Same profile, in JSON-LD (`application/ld+json`) format:

```json
{
  "@context": {
    "foaf": "http://xmlns.com/foaf/0.1/"
  },
  "@graph": [
    {
      "@id": "https://alice.databox.com/profile/card",
      "@type": "foaf:PersonalProfileDocument",
      "foaf:primaryTopic": {
        "@id": "#me"
      }
    },
    {
      "@id": "#me",
      "@type": "foaf:Person"
    }
  ]
}
```

## Minimum Recommended Profile Information

The above minimal valid profile doesn't provide enough useful information
for the purposes of building distributed read-write-web applications.
In addition, Solid recommends that WebID profiles include the following
statements:

1. A profile MUST include a `foaf:name` (see the discussion
  on [user names](#recommendation-for-user-names-in-profiles) below).
  This does not have to be a real name, it can by any pseudnym, but
  a string proided for apps to use for representing the user, in chats, sharing etc etc.
2. A profile SHOULD include a public `foaf:image` of either a mugshot of the person or a chosen avatar
  to make the display of the user's contributions identifyable.
3. A profile MAY provide a `foaf:nick` nickname as a short string for use by user interfaces where
  space is limited.
3. A profile SHOULD include `cert:key` public key certificate information, for
  use with WebID+TLS (which is currently the primary Solid authentication
  mechanism).
4. A profile SHOULD point to the root storage location using `pim:storage`
  (so that applications will know where to read and write their data).

```ttl
@prefix foaf: <http://xmlns.com/foaf/0.1/>.
<https://alice.databox.com/profile/card>
    a foaf:PersonalProfileDocument ;
    foaf:primaryTopic <#me> .
<#me>
    a foaf:Person ;
    foaf:name "Alice" ;
    <http://www.w3.org/ns/auth/cert#key> <#key6b4c> ;
    <http://www.w3.org/ns/pim/space#storage> <../> ;
<#key6b4c>
    # ... certificate key statements go here, see Certificates section
```

### Recommendation for User Names in Profiles

Client-side applications frequently need to know what to name the user, both
when interacting with the user directly (such as displaying the currently
logged in user in the navigation bar), or talking about users indirectly
(an event manager app, when listing which users are invited to a meeting, needs
to know how to display their names).

The Solid recommendation for client-side application code, when discovering
what to name the user, is to perform the following steps:

1. An app SHOULD look in the user's WebID Profile for the `foaf:name` predicate,
  and use that as the name, if it's available.
2. If an app does not find a name in the user profile, it MAY fall back to using
  the WebID URL, r a part ofo it, as the username.

## Public and Private Profiles

Solid servers must be able to support the separation of public and private data
in a user's profile. As a result, Solid WebID profiles MAY be split into
multiple RDF resources with different read/write permissions, linked together
either via `owl:sameAs` and `rdfs:seeAlso` predicates, or via the Solid-specific
predicate `space:preferencesFile`.

For example, a typical Solid user would have profile-related statements split
across several RDF documents:

* `/profile/card` - their primary (public-readable) WebID Profile.
  Which would contain a `space:preferencesFile` link to:
* `/settings/prefs.ttl` - a private (only the user has read/write access)
  Preferences file which contains further profile-related statements.

### Extended Profile

The combination of the main WebID Profile document, and all of the *related*
profile documents is referred to as the **Extended Profile**.

Solid apps that interact anonymously with the WebID profile MUST also load and parse *all*
of the related public RDF resources that are linked to from the main profile using any
the following triples in the main profile document:

1. $webid   `http://www.w3.org/2002/07/owl#sameAs`  ?public
2. $webid   `http://www.w3.org/2000/01/rdf-schema#seeAlso`  ?public

Solid apps that interact as the user in question, logged in with their cedentials,
with their own WebID profile MUST also load and parse all
of the related public public resources above and also will normally
load the user's preferences file.

### Private preferences file

The private preferences file is part of the extended profile. It is found
by following a triple in the main profile (the result of looking up the webid)

3. $webid   `http://www.w3.org/ns/pim/space#preferencesFile` ?preferences

Where the subject is the user's original webid.

It is the first private file that the app discovers in thie process, and
it is the place which either stores, or leads to, all of the
data which is private to the user, including settings 
and preferences, language and display preferences, and so on 
and all the user's pesonal data, be it conntacts, pictures or health data.

The `solid:preferencesFile` link is unusual then in that it is a link
from public data to private data.  Otherwise, discovery happes in two 
parallelel but otherwise congruent ways, in a tree of public information starting from
the extended profile, and a tree of private information starting from the 
private preferences file. Developers use urged to use common software for
these cases, and also to make it extenssible in future for when 
the congruent trees may be rooted in files corresponding to groups and organizations
of which the user is a member.


## Public Key Certificates

Solid currently uses WebID+TLS as its main Authentication mechanism.
To enable this, WebID Profile documents on Solid-compliant servers MAY contain
one or more Public Key Certificate sections, linked to from the main WebID
subject via `cert:key` predicates.

Example profile with a public key certificate (created by LDNode):

```ttl
@prefix foaf: <http://xmlns.com/foaf/0.1/>.
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#>.
@prefix cert: <http://www.w3.org/ns/auth/cert#>.
@prefix dc: <http://purl.org/dc/terms/>.
@prefix XML: <http://www.w3.org/2001/XMLSchema#>.

<https://alice.databox.com/profile/card>
    a foaf:PersonalProfileDocument ;
    foaf:primaryTopic <#me> .
<#me>
    a foaf:Person ;
    foaf:name "Alice" ;
    <http://www.w3.org/ns/auth/cert#key> <#key6b4c> ;
    <http://www.w3.org/ns/pim/space#storage> <../> ;
<#key6b4c>
    dc:created
       "2016-02-12T15:07:46.916Z"^^XML:dateTime;
    dc:title
       "Created by ldnode";
    a    cert:RSAPublicKey;
    rdfs:label
       "LDNode Localhost Test Cert";
    cert:exponent
       "65537"^^XML:int;
    cert:modulus
        "970E88..(many digits here)..167801"^^XML:hexBinary.
```

## Account Resource Discovery

Solid WebID Profile documents MAY contain the following links, to support
the discovery of resources that are of interest to client side applications.

### Storage Discovery

A Solid WebID Profile SHOULD contain a link to one or more Solid Containers
that act as Storage (a space for apps to read and write data).

Example link to Root Storage (gets created
[by default](recommendations-server.md#default-containers) on account creation):

```ttl
# ...
<#me>
    a foaf:Person ;
    <http://www.w3.org/ns/pim/space#storage> <../> .
```

### Inbox Discovery

A Solid WebID Profile MAY contain a link to the Solid Inbox container (gets
created [by default](recommendations-server.md#default-containers) on account
creation).

If an inbox link exists, there MUST be only one Inbox for the profile.

Example:

```ttl
# ...
<#me>
    a foaf:Person ;
    <http://www.w3.org/ns/solid/terms#inbox> </inbox/> .
```

### Type Registry Index Discovery

A Solid WebID Profile SHOULD contain one or more links to [Type Registry
Index](https://github.com/solid/solid/blob/master/proposals/data-discovery.md)
resources.

If links to type indexes exist, there MUST be only *one link each* to a private
and a public type registry index file, respectively.

For example, a link to the Listed Type Index in the main profile document:

```ttl
# ...
<#me>
    a foaf:Person ;
    <http://www.w3.org/ns/solid/terms#publicTypeIndex>
        </settings/publicTypeIndex.ttl> .
```

And an example corresponding link to the Unlisted Type Index, in a private
resources of the Extended Profile, such as the Preferences file
(in `/settings/prefs.ttl`):

```ttl
# ...
<#me>
    <http://www.w3.org/ns/solid/terms#privateTypeIndex>
        </settings/privateTypeIndex.ttl> .
```
