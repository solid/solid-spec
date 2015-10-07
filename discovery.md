# SoLiD Discovery (Draft)

The following describes discovery in the SoLiD framework using HTTP link following aka "follow your nose".

### Starting point -- WebID

The starting point of SoLiD discovery is a [WebID](http://www.w3.org/2005/Incubator/webid/spec/identity/) user profile, which is a hash based URI, typically denoting a (FOAF) Agent.  From this profile all of your storage can be found (discovery).

### Storage Discovery

#### Storage

* Starting Point: WebID
* Type: [pim : storage](http://www.w3.org/ns/pim/space#storage)

Storage is a root place to store files and data, which is used to create workspaces (see below).  Storage can also be of type:
* Public (everyone can see)
* Personal (only the user can see)
* Controlled (allows sharing)

#### Workspaces

* Starting Point: WebID or preferencesFile
* Type: [pim : workspace](http://www.w3.org/ns/pim/space#workspace)

Workspaces are where data is stored.  There are various types of workspaces, which normally will have their own type.  For example workspaces can be
* Personal (only the user can see)
* Private (only the user can see)
* Shared (allows sharing)
* Master (a workspace about other workspaces)

#### App Configuration Workspace

* Starting Point: WebID or preferencesFile
* Type: [pim : workspace](http://www.w3.org/ns/pim/space#workspace)

The app configuration workspace is a container of many different app configurations.  It is also possile to use the "glob * " function, for convenience, to get all configurations of various apps that are in use.

#### Preferences File

* Starting Point: WebID
* Type: [pim : preferencesFile](http://www.w3.org/ns/pim/space#preferencesFile)
* Access: Private

Preferences File is a private file that is linked from your WebID, and contains miscellaneous data not in your public profile.


#### App configuration Files

* Starting Point: App Configuration Workspace
* Type: [pim : ConfigurationFile](http://www.w3.org/ns/pim/space#configurationFile)

App configuration files contain all information related to an app.  


### Ontologies

* pim : http://www.w3.org/ns/pim/space#
