# Recommendations for Server Implementations

**Note:** This spec is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.

## Default Containers

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

```ttl
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

```ttl
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

```ttl
<#me>
    <http://www.w3.org/ns/pim/space#preferencesFile> <../settings/preferences> ;
```

##### `/inbox/` (Inbox)

A container to serve as a default primary channel for
notifications.

**Default ACL:** append-only by public, read by owner.

Discoverable from profile using the [ldp:inbox](http://www.w3.org/ns/ldp#inbox) property as specified in [W3C Linked Data Notifications](https://www.w3.org/TR/ldn/).

```ttl
<#me>
    <http://www.w3.org/ns/ldp#inbox> <../inbox/> ;
```

## CORS - Cross Origin Resource Sharing

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

```http
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

```http
Access-Control-Allow-Origin: *
```

The star character (`*`) signifies "allow all". If you want to learn more about
CORS, please visit this page: http://enable-cors.org/
