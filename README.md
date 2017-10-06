# Resource Discovery with Activity Streams

Using standard, semantic technologies for easy optimization of resource discovery, crawling and indexing.

## The Approach

Following the [design principles](http://iiif.io/api/annex/notes/design_patterns/) of IIIF:
 * As Simple as Possible, and No Simpler
 * Intelligently Manage Ramping Up
 * Avoid Specific Technology implementations
 * Follow Resource-Oriented Design
 * Follow Linked Data
 * Design for JSON-LD First
 * Use Standards
 * Follow Best Practices
 * Loosely couple APIs
 * Define Success, not Failure

And the useful patterns established by the [ResourceSync](http://openarchives.org/rs/1.1/resourcesync) effort:
  * List of Resources
  * List of Changes
  * Notification of Changes

As scoped by the [IIIF Discovery charter](http://iiif.io/community/groups/discovery/charter/) and [use cases](https://github.com/iiif/iiif-stories/issues?utf8=%E2%9C%93&q=is%3Aissue%20is%3Aopen%20label%3Adiscovery), we can then design for solving the challenges faced by the community.

## Activity Streams

[Activity Streams](https://www.w3.org/TR/activitystreams-core/) (AS2) is a "model for representing potential and completed activities". IIIF already uses the AS2 Collections and Paging model, as does the [Web Annotation](https://www.w3.org/TR/annotation-model/#collections) work that we depend on.  AS2 is designed to be JSON-LD first while still respecting the Linked Open Data paradigm, and follows the best practices defined in the W3C.

### List of Resources

As IIIF uses the AS2 Collection model, a IIIF Collection is very close to an AS2 Collection of Resources already!  AS2 Collections for lists of this size will always be paged, however, and the contents in the pages.  Thus for any of the options that follow, the top resource will be a single Collection document.

* [Collection](/json/collection.json)

#### Too Simple

The simplest possible page layout would be a list of Manifests, identical in structure to a IIIF Collection.  If we were going to recommend this, we should just use IIIF Collections directly.  It does not ramp up to a full activitystream well, but this level demonstrates the similarity of the AS2 approach with existing IIIF patterns.

Usage:  Crawl the entire set of pages and dereference each manifest to see if it has changed.

* [Page: Too Simple](/json/0_page-toosimple.json)

#### Level 0: Basic Resource List

However with the addition of just a little boilerplate in the JSON, we can be on the path towards a robust set of changes.  We add a "Create" activity wrapper around the Manifest, and do not set a date for when it was created.  The order of the manifests in the pages is unimportant, but each should only appear once (as it was only created once). In terms of optimization, it provides no additional benefit over the IIIF Collection pattern, but is compatible with a system where optimization is possible, such as the following.

Usage:  Crawl the entire set of pages and dereference each manifest to see if it has changed.

* [Page: Basic Resource List](/json/1_page-resourcelist.json)


#### Level 1: Basic Change List

If we know dates, such as when the resource was created or modified, instead we can order the list by those dates, with the most recent activities occuring first.  Each Manifest 
should still appear only once in the full list.  This could be accomplished with a IIIF Collection, if we were to add created and modified timestamps to the Manifests themselves.

Usage: Record each time the list is crawled, and stop crawling when a datestamp before the previous crawl time is encountered.

* [Page: Basic Change List](/json/2_page-basicchangelist.json)

#### Level 1b: Change List with Metadata

Additional metadata can be added to the basic change list without affecting the overall crawl behavior.  This metadata can include who did the change, more information about when the change occured as opposed to when the change activity was published, and so forth.  This level (and subsequent) cannot be achieved using just IIIF Collections without significant extensions.

Usage:  Record each time the list is crawled, and stop crawling when a datestamp before the previous crawl time is encountered.

* [Page: Basic Change List](/json/3_page-changelistmetadata.json)

#### Level 2: Complete Change List

At the most complex level, a log of all of the activities that have taken place can be recorded, with multiple events per resource.  This would include Deletes, allowing a synchronization process to remove resources as well as add them. The list might thus end up very long if there are many changes to resources, however this is infrequent.  This would also allow for the complete history of a resource to be reconstructed, if each version has an archived representation.  This level cannot be achieved using IIIF Collections at all, as the deleted Manifest would have been deleted.

Usage:  Record each time the list is crawled, stop crawling when a datestamp before the previous crawl time is encountered, and only process the most recent change per resource, including deleting resources.

* [Page: Complete Change List](/json/4_page-fullchangelist.json)


## Types of Change

### Single-Resource Publisher Operations

#### Create

Minimum Level: 0

A resource came into existence. Before this time it did not exist, and the state of the resource is the first state.

#### Update / Modify

Minimum Level: 1

A resource that was already created was changed.  There is a new state of the resource that replaces the previous state.  The previous state may be available at another URI.

#### Delete

Minimum Level: 2

A resource that was already created was taken out of existence. After this time, the resource does not exist, and previous states may be available at other URIs.


### Multi-Resource Publisher Operations

These operations are compositions of the above in terms of the effects on different resources, but should be treated as a single operation for the purpose of notification and replay.  These are more appropriate for true synchronization systems, rather than simple crawling and discovery.

#### Copy

Minimum Level: 4

A source resource is duplicated to create a new target resource.

#### Rename / Move

Minimum Level: 4

A source resource is duplicated to create a new target resource, and the original is deleted.

#### Merge

Minumum Level: 4

Two or more source resources are combined to create a new target resource.  

#### Split

Minumum Level: 4

A single source resource is divided to create multiple new target resources.

### Third Party Operations

These operations are carried out by third parties, rather than the publisher.  As such they would be written to the Inbox of the resource as either a request to modify the resource, or just notification that the resource was used.


#### Linked

Minimum Level: 3

A remote source resource has a new reference to the target local resource.

#### Used

Minimum Level: 3

The target resource was used by the remote agent in some way.

#### Replace

Minimum Level: 4

The third party is requesting that the publisher replace their copy of the target resource with the source resource.


## Notifications of Change

[Linked Data Notifications](https://www.w3.org/TR/ldn/) allow for these notifications to be sent to and made available from an "inbox", and thus we would be consistent with section 3 of the charter around Notification.

Sending Manifests to an Inbox is not a "notification", it's just moving data around. Thus this level is also not possible without the notion of activity. 

LDN provides a storage and publication mechanism, but clients are still expected to come and retrieve the set of changes in the same way as above.  Assuming that either (a) systems register their interest in a resource, and the publisher notifies them directly or (b) the publisher writes to the inbox, and the inbox then auto-forwards the notification to the subscribers ... then there needs to be a method of subscribing to have the changes pushed to a remote inbox.

### Linked Data Notifications: Subscription

While there is [WebSub](https://www.w3.org/TR/websub/), this does not interoperate with LDN and is from a service oriented mindset. WebSub makes various requirements that are not appropriate for Notifications (you MUST send the entire contents of the resource that you have subscribed to) which prevents subscribing either to binary resources (you don't want the entire gigabyte TIFF) or or aggregation-level containers (you don't want every change, every time, just the new one).

A more LOD / LDP / LDN appropriate pattern would be to use a REST pattern with Containers, that allow subscription to the inbox, manage filtering and the callback endpoint.  One way to do this could be as [described here](https://docs.google.com/document/d/1JsQS1LVFt8wuJSYo_XsOzyP28pS8hfSLEhAC3BFuN6o/edit).



