# Solid Discovery (Draft)

The following describes discovery in the Solid framework using HTTP link following aka "follow your nose".

### Starting point -- WebID

##### Public Profile

The public profile is what you get when you look up someone's WebID directly.
Strip off any hash and localid part.  For example.

```
  https://example.databox.me/card#me    ->     https://example.databox.me/card
```


The starting point of Solid discovery is a [WebID](http://www.w3.org/2005/Incubator/webid/spec/identity/) user profile, which is a hash based URI, typically denoting a (FOAF) Agent.  From this profile all of your storage can be found (discovery).  The Profile typically contains public name and contact information about someone, and pointers to public data used by various apps.

#### Preferences File

* Starting Point: WebID
* Property: [pim : preferencesFile](http://www.w3.org/ns/pim/space#preferencesFile)
* Access: Private

```
  <#me>   space:preferencesFile <settings/preferences.ttl>.
```


Preferences File is a private file that is linked from your WebID, and contains miscellaneous data not in your public profile.  
In general, the same triples will be put in your publuc



### Storage Discovery

#### Storage

* Starting Point: WebID
* Type: [pim : storage](http://www.w3.org/ns/pim/space#storage)

Storage is a root place to store files and data, which is used to create workspaces (see below).  Storage can also be of type:
* Public (everyone can see)
* Personal (only the user can see)
* Controlled (allows sharing)
* 


##### Public

This storage is a non-access controlled public space.  
Dev, do not assume that the ACL APIs will work here.
User, know that you cannot control sharing here.
This may be used to point to a peice of legacy storage such as WebDAV disk.

##### Private

Nothing in this space will be shown to anyone else.  Access control may not be available.
This may be a storage which is not connected to the net, and only available from one machine.

##### Controlled

A normal space, which allows Access Control using the solid standards.

#### Spaces

A space provides both a place (URI prefix) where to store stuff and a default ACL to give to things in it.   You can have just open workspace in a storage, or you can have two, for example for your interactions with different communities, like home and work. 
A space is a bit like an account in some SNSs or a bit like a mounted (virtual) disk. 

 
* Where stored: WebID ie user's prubli profile, if public, or preferencesFile, if private
*  Predicate: (http://www.w3.org/ns/pim/space#workspace)

```
  <#me>   space:workspace <#ws1>.
```

Workspaces are where data is stored.  There are various types of workspaces, which normally will have their own type.  For example workspaces can be

Spaces have various special subclasses:

##### Master

App, do not offer this space to the user.
It is used for example for a list of  other workspaces.


## App configuration

Three ytpes of registration

### RDF Type

A list of 
```
  <#me>  space:TypeIndex  <byType>.
```
And then in there thngs like
```
 <#r1>  a solid:TypeRegistration;
  solid:forClass   vcard:addressBook;
  solid:instance </contacts/book#this>.
  
```
### By application

A list of 
```
  <#me>  space:AppIndex  <byType>.
```
And then in there thngs like
```
 <#r1>  a solid:AppRegistration;
  solid:forApp  ghld:app-shedule
  solid:instanceIndex </polls/list.ttl>.
  
```
and then in </polls/list.ttl> things like

```
TBD
```

### Timeline

A timeline registration put things on a user's timeline for a given time. 
Do this if it useful for the user or others to find stuff that way.
Can be public or private.

@@@ Melvin, what are the triples?

(Suggest the timeline triples are stored in files by month or month/day.)



#### App Configuration Workspace - OLD

* Starting Point: WebID or preferencesFile
* Type: [pim : workspace](http://www.w3.org/ns/pim/space#workspace)

The app configuration workspace is a container of many different app configurations.  It is also possile to use the "glob * " function, for convenience, to get all configurations of various apps that are in use.


#### App configuration Files

* Starting Point: App Configuration Workspace
* Type: [pim : ConfigurationFile](http://www.w3.org/ns/pim/space#configurationFile)

App configuration files contain all information related to an app.  


### Ontologies

* pim : http://www.w3.org/ns/pim/space#


### Illustration

![discovery illustration](assets/discovery.png "discovery illustration")
