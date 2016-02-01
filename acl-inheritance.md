# ACL

This document explores two different types of permission inheritance for Web Access Control, [WAC](http://www.w3.org/wiki/WebAccessControl): `default` and `defaultForNew`. `Gold` and `Ldnode` currently implements `defaultForNew`.

## Explicit vs inherited permissions
In WAC, permissions can be defined explicitly or inherited from parent containers. Explicit access control entries are described via `acl:accessTo` in the ACL file of the resource. Inherited permissions are defined in parent containers via `acl:defaultForNew`, whose possible interpretations are described in this document. The predicate `defaultForNew` can only be used in containers ACLs.

The need for inherited ACL comes from two main issues:

  - Avoid having an ACL file for each resource. For example, changing permissions for a folder would mean change individual children resource's ACL.
  - Define Access Control for resources that do not exist yet.


## Different inheritance strategies

There are two relevant implementations to consider: `default` and `defaultForNew`. The key differences between those are: (1) whether permissions are defined by the __most significant__ ACL entry or are cumulative, hence (2) the permission check algorithm's direction of the walk through the resource path.

### Strategy 1) `monotonic`

In `monotonic`, ACL permissions are cumulative (inherited from the ancestors) and the permission check algorithm sums permissions over the path. The path is explored in any direction, as the permission is the union of all the permissions from each ACL file. The search can stop when any ACL file is which gives permission.

#### Pro
- Not as fast as defaultForNew but can be 
- Simple hierarchical permission (e.g. everything in `/shared` is shared) 
- Can be fast as it only has to find one ACL file to give the permission it needs
- Monotonic: Once a user or any agent knows the ACL it can apply it as a rule. An ACL is a first class fact. It can be digitally signed, transported, and used to demand access at a later date, etc.  Monotonicness is useful.

#### Cons
- It is slower than `defaultFor new`, but the search stops the moment it finds success.
- It can't have private subfolders within shared folders. Given that permissions cannot be reverted (with the current WAC specification), a subfolder cannot be private in a shared folder.    This system is monotonic.
- User has to be aware of the permissions given to the parent folders 

((A possible solution is NOT include Windows' `DENY` or `DENY all` in the WAC specification, where these entries would take precedence to the other (_allow_) permissions)  That would not  be monotonic.

### Strategy 2) `default`

In `defaultLocal`, ACL permissions are inherited from the most local ACL file which exists, and no others are searched.. The permission check algorithm iterates from the end of path to the beginning stopping at the first existing ACL. Note: Different permission check algorithm may be implemented to find the most significant ACL.

#### Pro
- It can have private subfolders within shared folders

#### Cons
- Users may lose access to their resource by creating an ACL file that does not contain themselves. (Software stops that happening) 
- Changing permissions recursively to a folder will require changing permission on each subfolder's ACL

### Strategy 3) `defaultForNew`

In `defaultForNew`, ACL permissions are inherited from the whole path as in 'monotonic', but done from the end of the path top the root. With this method, however, whenever a file does not have a local ACL, one is generated for it, so that in future the search will hit it immediately.

#### Pro
- Fast

#### Cons
- Generates a storage requirement for all the ACL files, which is a pain, especialy in a fiel space shared with other systems.
- Users may lose access to their resource by creating an ACL file that does not contain themselves.
- Changing permissions recursively to a folder will require changing permission on each subfolder's ACL
 

### Strategy 4) resource centric

Also see wiki page [regexes in ACLs](https://github.com/solid/solid/wiki/Regexes-in-ACLs).

[rww-play](https://github.com/read-write-web/rww-play) takes a resource centric view of acls. Any resource `<doc>`'s acl link relation (given by the http header `Link: <doc.acl>; rel="acl"`) specifies the acl starting point for that resource. This must be the case as that is the only way a client can find out about an acl for a resource. The client and the server must therefore start from that acl and follow any [wac:include](http://www.w3.org/ns/auth/acl#include) relations which should be logically mergeable monotonically to create the complete acl for that resource. Given that these mergers happen monotonically the server can stop at any moment it finds an acl that gives the user permission, in the knowledge that any other acl included cannot undo the statement added.
Note: if it can be shown that ACLs can be inconsistent, then a SoLiD server may want to check the consistency of these acls before allowing a write to succeed.

Given the above, here is one way for a container to deal with defaults for new resources that works by reference using a 
relation `wac:defaultInclude` that points from an acl file to another one that will be included by default in any new resource created.

This would work like this. Consider an `ldp:Container` named `</container/>`'s whose acl file `</container/.acl>`  includes the statement

```Turtle
<.> wac:defaultInclude <default.acl> .
```

On creation of a new resource eg, by `POST`ing some content to `</container/>` with Slug "cat", thereby creating a new resource `</container/cat>` and the associated `</container/cat.acl>` the server on finding the above wac:defaultInclude statement in the container's acl will add the statement:

```
<> wac:include <default.acl> .
```

in the created resource's acl.

#### Pro
 * If somone wants to change the all the resources that have a certain default, they can do so just by changing `<default.acl>`. 
 * If someone wants a particular resource to not have the defaults, they can just remove the `wac:include` triple.
 * Reduces storage requirements
 * Is very maintainable
 * Is monotonic 
 * client and server verification mechanisms follow the same discovery process

#### Cons
  * verifying an ACL will often require fetching one new default acl, but that is local so it should be very fast.
  * this does require some form of regular expression (e.g. a simple version based on globbing such as `"/*.acl"`) as the `wac:include`ed `default.acl` needs to make statements about sets of resources such as 
```Turtle
 [] acl:accessToClass [ acl:regex "https://jack.example/.*[.]acl" ];
   acl:mode acl:Read;
   acl:agentClass foaf:Agent .
```

#### Issues

##### Truth

It does mean that on a naive reading of `wac:regex` some acls will not actually be true of all files specified in the regular expression, as they are only valid if the resource's acl includes them using `wac:include`. Perhaps there is a way of thinking of the `acl:regex` relation in way that does not create such false statements. Perhaps it should be read as defining the subclass of resources that fit the given pattern _and_ that whose acls are linked to via a set of `wac:include`s to the resource that contains the regular expression. On this reading one cannot deduce that `https://jack.example/cat.acl` is readable by everyone only from the acl shown in the cons section above. One also needs to know:
 * that `<https://jack.example/cat.acl>` exists
 * that `<https://jack.example/cat.acl>` has an `acl` link header to a resource that through a chain of `wac:include`s refers back to `<default.acl>`

#### Better Pattern Languages

The problems with full regexes are:

* one needs to know the full url of the resource
* different languages have different implementations of regexes
* they can be turing complete 

This should not stop one. Regexes are already standardised by the W3C in RDF via the [POWDER spec](https://www.w3.org/TR/powder-dr/), which also provided simpler less powerful vocabularies to enable use cases that did not require the full regex power. So one could invent a simple regular expression based on globbing such as `"/*"` for all resources in a folder, or `"/**"` for all resources in a folder and sub-folders.  This could look like the following:

```Turtle
[] acl:accessToClass [ acl:urlPattern [  acl:base <.>; acl:match "*.acl" ]];
   acl:mode acl:Read;
   acl:agentClass foaf:Agent .
```

This should be read as saying that everybody can read all resources that match the pattern "*.acl" in the current directory, and for which this is an acl through wac:include chain from the resource's acl. (in this case this is a rule on acls)

To allow all files to be readable and writeable by the owner in this folder and sub-folders one could use

```Turtle
[] acl:accessToClass [ acl:urlPattern [  acl:base <.>; acl:match "**" ]];
   acl:mode acl:Read, acl:Write;
   acl:agent </card#i> .
```

assuming of course the user's WebID is `</card#i>` . 
Note again that the `acl:urlPattern` gives a class that is larger than the class of resources for which this is true, as the only resources for which that rule is valid are those whose acls link to the acl in which this is written.



The advantage of such a pattern language is that it allows the pattern to be relative to a resource, and so to be written out even for a client that does not know the full url of the resource.


--

## References

[1] [Grunbacher, A. POSIX Access Control Lists on Linux. 2003.](https://www.usenix.org/legacy/events/usenix03/tech/freenix03/full_papers/gruenbacher/gruenbacher.pdf)
