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

In `defaultForNew`, ACL permissions are inherited from the whole path as in 'momotonic', but done from the end of the path top the root. With this method, however, whenever a file dopes not have a local ACL, one is generated for it, so that in future the search will hit it immediately.

#### Pro
- Fast

#### Cons
- Generates a storage reuirement for all the ACL files, which is a pain, especialy in a fiel space shared with other systems.
- Users may lose access to their resource by creating an ACL file that does not contain themselves.
- Changing permissions recursively to a folder will require changing permission on each subfolder's ACL

--

## References

[1] [Grunbacher, A. POSIX Access Control Lists on Linux. 2003.](https://www.usenix.org/legacy/events/usenix03/tech/freenix03/full_papers/gruenbacher/gruenbacher.pdf)
