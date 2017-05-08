## Browsing the Friendship Graph

([permalink](https://www.w3.org/wiki/Socialwg/Social_API/User_stories#Browsing_the_Friendship_Graph)):


> 1. Samwise is looking for new contacts (for a surprise party, new business, your guess)
> 2. Samwise is browsing his list of contacts.
> 3. Alissa has previously allowed Samwise to see her list of contact.
> 4. Samwise also browses the list of Alissa (his contact).

### Background

The Linked Data People graph extends to several million organic user profiles over hundreds of multiple domains.  If facebook is included (they are producers of linked data via turtle) it extends to over 1.4 billion profiles.  

Facebook profiles generally require an OAuth token to use the API, this enables access control and viewing priveledges.  Solid in general uses WebAccessControl and ACL's to achieve the same thing, but implementors may choose which approach they wish to take.

In Solid people are denoted by an HTTP URI.  Normally, as a best practice the URI contains a fragment identifier (#).  This is to help software disambiguate between an HTTP document and the person it talks about, in much the same way that a passport contains information about a person but a passport ID is the ID of that document, not the person.

Solid's implementation is based on the W3C REC RDF, and in general uses the WebID spec, and FOAF vocabulary, but is not limited to either.

Friendship connections are open ended, but normally based on the foaf : knows predicate, to indicate one person knows another.  In practice, over the past 10+ years of foaf, knows has loosely been associated with friends, as assumed by software, but this could change in future, with consensus.

There are several implementations of viewers.

* http://linkeddata.github.io/profile-editor/#/friends/view?webid=http:%2F%2Fbblfish.net%2Fpeople%2Fhenry%2Fcard%23me
* http://foaf-visualizer.gnu.org.ua/?uri=http://danbri.org/foaf.rdf
* https://deiu.me/profile#me
* https://graphite.ecs.soton.ac.uk/browser/?uri=http%3A%2F%2Fmelvincarvalho.com%2F

### Notes

There is nothing definitive in the selection of vocabularies. 
 
  * [Foaf](http://xmlns.com/foaf/0.1/) is widely deployed, but quite limited for company info, and needs to be complemented with a lot of other vocabularies.
  * [Schema.org](https://schema.org/) is very complete, but confuses documents and things, has URLs as string in the examples, and is so vague that it would probably not be useful for an efficient social web. It is clearly an improvement over the search engines current position, but not designed for building light weight apps in a linked data space. It would be counterproductive.
  * [vcard](http://www.w3.org/TR/vcard-rdf/) has the advantages and disadvantages of sticking closely to the old and widely deployed vcard format. The disadvantage is the odd decision to have relations to objects and Relation objects - there are as many types of relations, that it would have been easier to create the relations directly.
