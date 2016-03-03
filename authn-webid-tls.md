# Solid WebID-TLS Protocol Spec

**Note:** This spec is a component of the parent
[Solid specification](README.md); the parent spec and all its components are
versioned as a whole.

The [WebID-TLS protocol](http://www.w3.org/2005/Incubator/webid/spec/tls/)
enables secure, efficient authentication on the Web. It enables users to
authenticate on any site by simply choosing one of the certificates proposed to
them by their browser. These certificates can be created by any Web Site for any
purpose. A user may have multiple client certificates, bound to one or multiple
WebIDs.

Basically, WebID-TLS relies on matching a public key received from a client
certificate, to the public key published in the WebID profile obtained by
dereferencing the WebID included in the `SubjectAlternativeName` field of the
client certificate. In other words, users must prove they own a public key they
publish in their WebID profiles.


The tech at work here is the familiar [TLS
protocol](https://en.wikipedia.org/wiki/Transport_Layer_Security) used by
browsers to establish secure HTTPS connections to known websites, but with two
important differences. 
