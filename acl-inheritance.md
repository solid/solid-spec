# ACL

This document explores two different types of permission inheritance for Web Access Control, [Web Access Control, WAC](http://www.w3.org/wiki/WebAccessControl): `default` and `defaultForNew`.

## Explicit and inherited permissions
In WAC, permissions can be defined explicitly or inherited from parent containers. Explicit access control entries are described via `acl:accessTo` in the ACL file of the resource. Inherited permissions are defined in parent containers via `acl:defaultForNew`, whose possible interpretations are described in this document. The predicate `defaultForNew` can only be used in containers ACLs.

The need for inherited ACL comes from two key needs:

  - Avoid having an ACL file for each resource. For example, changing permissions for a folder would mean change individual children resource's  ACL.
  - Define Access Control for resources that do not exist yet.

## Different inherited permissions

There are two key implementations to consider: `default` and `defaultForNew`. The key differences between those are: (1) whether permissions are defined by the __most significant__ ACL entry or are cumulative, hence (2) the permission check algorithm's direction of the walk through the resource path.

### Strategy 1) default

In `default`, ACL permissions are cumulative (inherited from the ancestors) and the permission check algorithm sums permissions from left-to-right. The path is explored from root, `/` to the end.

#### Pro
- Simple hierarchical permission (e.g. everything in `/shared` is shared)

#### Cons
- It is slower than `defaultForNew`, since all the path must be taken into consideration.
- It can't have private subfolders within shared folders. Given that permissions cannot be reverted (with the current WAC specification), a subfolder cannot be private in a shared folder. A possible solution is include Windows' `DENY` or `DENY all` in the WAC specification. These entries would take precedence to the other (_allow_) permissions).
- User has to be aware of the permissions given to the parent folders

### Strategy 2) defaultForNew

In `defaultForNew`, ACL permissions are inherited from the most significant ACL. The permission check algorithm iterates from the end of path to the beginning stopping at the first valid ACL. Note: Different permission check algorithm may be implemented to find the most significant ACL.

#### Pro
- It can have private subfolders within shared folders

#### Cons
- Users may lose access to his resource by creating an ACL file that does not contain themselves.
- Changing permissions recursively to a folder will require changing permission on each subfolder's ACL
