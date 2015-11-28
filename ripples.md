# Solid ripples

## Sending (aka starting a ripple)

1. A relationship is created between resource A and resource B.
2. Sender (could be a client or server) `GET`s resource B and looks for the `rel="pingback:to"` Link Header.
2.1 If not found, the sender checks the containing Container, and continues up the hierarchy until halted or rel found.
3. The value of `rel="pingback:to"` is an LDP Container, publicly writeable by default.
4A. *(Plain triple version)* Sender writes the triple for the relationship in 1. to the Container in 3.
4B. *(Pingback object version)* Sender writes the following triples to the Container in 3.

```
@prefix pingback: http://purl.org/net/pingback/
@prefix as2: http://www.w3.org/ns/activitystreams

[] a pingback:Request ;
   pingback:source <http://example.com/resource-a> ;
   pingback:property as2:inReplyTo ;
   pingback:target <http://elpmaxe.net/resource-b> .
```

If the sender is not authorized to write, processing stops.

## Verifying a relationship

1. There exists a client with the ability to verify the existance of relationships between resources, whether all that were ever sent, or a subset that this particular client is interested in.
2. The client locates the list of asserted relationships via the `rel="pingback:to"` Link Header of resources/containers/users it is interested in. Presumably the user has told it what to be interested in, or it has a predefined list of types of thing it's interested in because it's a client with a specific purpose, and can follow links to more if need be.
3. For each triple (relationship assertion) of interest, the client:
3.1 `GET`s the object (`pingback:target`) and verifies that it exists.
3.2 `GET`s the subject (`pingback:source`) and verifies that it exists, and containes a link to the object, via the specified predicate (`pingback:property`).
3.3 If both of these conditions are met, the client may optionally proceed to check against other criteria as desired before accepting.
4. Once verified, the client does something useful like displays it to the user / fetches the subject resource and displays that to the user / etc.
4.1 Does the client also need to do something with it to mark it as verified? For both itself, and other clients that might be interested.

## Setting up the pingback container

Any `ldp:Container` can be also be a `pingback:Container`.

Perhaps all resources can receive pingbacks by default, in which case all top level containers (the workspaces?) have a `rel="pingback:to"` pre-set.

If the user wants to refine pingback reciept for specific containers/resources, they can set preferences to point to different `rel="pingback:to"` which may have different ACLs set, allowing blacklisting or whitelisting or total prevention of responses, if desired.

A user should have a link to a `pingback:to` from their WebID, for if someone creates a relationship with the user directly, rather than one of their resources.